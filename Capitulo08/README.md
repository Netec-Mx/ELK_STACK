# Aplicar una política de retención y validar la recuperación de datos de logs

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio diseñarás e implementarás una política ILM completa con fases hot, warm y delete para data streams de logs, configurarás un repositorio de snapshots respaldado por MinIO (S3-compatible), ejecutarás y validarás la restauración de datos tras una eliminación simulada, y habilitarás la dead letter queue de Logstash para garantizar la recuperación de eventos durante interrupciones del flujo de ingestión.

## Objetivos de Aprendizaje

- [ ] Diseñar e implementar una política ILM (`logs-retention-policy`) con fases hot, warm y delete con criterios de rollover y retención diferenciados
- [ ] Configurar un repositorio de snapshots tipo S3 apuntando a MinIO y crear una política SLM automatizada con retención de 7 días
- [ ] Ejecutar la restauración de un snapshot en un índice alternativo y verificar la integridad de los datos recuperados
- [ ] Implementar la dead letter queue y persistent queue en Logstash para recuperar eventos perdidos durante una interrupción simulada de Elasticsearch

## Prerrequisitos

### Conocimientos previos

- Comprensión de los conceptos de ILM (fases, acciones, condiciones de transición)
- Familiaridad con data streams y backing indices en Elasticsearch
- Experiencia básica con la API REST de Elasticsearch y `curl`
- Conocimiento de Docker y Docker Compose para gestión de contenedores

### Acceso requerido

- Entorno del Lab 7 completado con seguridad TLS, usuarios y roles configurados
- Data streams `logs-nginx-*` y `logs-python-app-*` activos con datos indexados
- Usuario `elastic` con permisos de superusuario o usuario `ops-user-01` con privilegios de gestión ILM y snapshots
- Acceso a terminal con Docker Engine operativo

## Entorno del Laboratorio

### Software necesario

| Componente | Versión | Puerto |
|------------|---------|--------|
| Elasticsearch | 8.14.1 | 9200 |
| Kibana | 8.14.1 | 5601 |
| Logstash | 8.14.1 | 5044, 8080 |
| MinIO | RELEASE.2024-03-15T01-07-19Z | 9000, 9001 |
| Docker Engine | 26.1.4 | — |
| Docker Compose | 2.27.1 | — |
| curl | 7.81.0 | — |
| jq | 1.6 | — |

### Variables de entorno

```bash
# Configurar variables para el laboratorio
export ELASTIC_URL="https://localhost:9200"
export ELASTIC_USER="elastic"
export ELASTIC_PASSWORD="ElasticLabs2024!"
export MINIO_ROOT_USER="minioadmin"
export MINIO_ROOT_PASSWORD="minioadmin123"
export MINIO_ENDPOINT="http://minio01:9000"
export LAB_DIR="$HOME/elastic-labs"
```

### Preparación inicial del entorno

```bash
# Verificar que el stack está operativo
cd ~/elastic-labs

curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_cluster/health" | jq '.status'

# Verificar data streams existentes
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_data_stream/logs-*" | jq '.data_streams[].name'
```

**Salida esperada:**
```
"green"
"logs-nginx-default"
"logs-python-app-default"
```

---

## Paso a Paso

### Paso 1: Crear la política ILM `logs-retention-policy`

**Objetivo:** Definir una política ILM con fase hot (rollover a 7 días o 10 GB), fase warm (force merge a 1 segmento tras 7 días post-rollover) y fase delete (eliminación a los 30 días post-rollover).

#### Instrucciones

1. Crear la política ILM mediante la API:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_ilm/policy/logs-retention-policy" \
  -H "Content-Type: application/json" \
  -d '{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_primary_shard_size": "10gb"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}' | jq .
```

2. Verificar que la política fue creada correctamente:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_ilm/policy/logs-retention-policy" | jq '.logs-retention-policy.policy.phases | keys'
```

**Salida esperada:**
```json
["delete", "hot", "warm"]
```

3. Crear (o actualizar) el index template para asociar la política a los data streams `logs-nginx-*` y `logs-python-app-*`:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_index_template/logs-retention-template" \
  -H "Content-Type: application/json" \
  -d '{
  "index_patterns": ["logs-nginx-*", "logs-python-app-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "logs-retention-policy",
      "index.number_of_shards": 1,
      "index.number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" },
        "log.level": { "type": "keyword" },
        "service.name": { "type": "keyword" },
        "source.address": { "type": "ip" },
        "http.response.status_code": { "type": "integer" }
      }
    }
  },
  "priority": 300
}' | jq .
```

**Salida esperada:**
```json
{ "acknowledged": true }
```

4. Reducir el intervalo de comprobación de ILM para pruebas:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -d '{
  "persistent": {
    "indices.lifecycle.poll_interval": "1m"
  }
}' | jq .
```

#### Verificación

```bash
# Verificar que los data streams existentes están vinculados a la política
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/logs-nginx-*/_ilm/explain" | jq '.indices | to_entries[0].value | {index: .index, phase: .phase, policy: .policy}'
```

**Salida esperada (ejemplo):**
```json
{
  "index": ".ds-logs-nginx-default-2025.07.10-000001",
  "phase": "hot",
  "policy": "logs-retention-policy"
}
```

> **Nota:** Si los data streams existentes no recogen la nueva política automáticamente, puede ser necesario realizar un rollover manual para que el nuevo backing index herede la configuración de la template actualizada:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/logs-nginx-default/_rollover" | jq .
```

---

### Paso 2: Desplegar MinIO y configurar el repositorio de snapshots

**Objetivo:** Desplegar MinIO como almacenamiento S3-compatible, crear el bucket `elastic-snapshots`, registrar el repositorio en Elasticsearch y crear una política SLM diaria.

#### Instrucciones

1. Agregar el servicio MinIO al archivo Docker Compose. Crear el archivo de extensión:

```bash
cat > ~/elastic-labs/config/docker-compose-minio.yml << 'EOF'
services:
  minio01:
    image: minio/minio:RELEASE.2024-03-15T01-07-19Z
    container_name: minio01
    hostname: minio01
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin123
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    networks:
      - elastic-net
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  minio-data:
    driver: local

networks:
  elastic-net:
    external: true
EOF
```

2. Iniciar el contenedor MinIO:

```bash
cd ~/elastic-labs/config
docker compose -f docker-compose-minio.yml up -d
```

3. Esperar a que MinIO esté listo y crear el bucket `elastic-snapshots`:

```bash
# Esperar a que MinIO responda
sleep 10

# Instalar mc (MinIO Client) si no está disponible
docker exec minio01 mc alias set local http://localhost:9000 minioadmin minioadmin123

# Crear el bucket
docker exec minio01 mc mb local/elastic-snapshots
```

**Salida esperada:**
```
Bucket created successfully `local/elastic-snapshots`.
```

4. Instalar el plugin `repository-s3` en Elasticsearch (si no está instalado) y configurar las credenciales en el keystore:

```bash
# Verificar si el plugin ya está instalado
docker exec es01 bin/elasticsearch-plugin list | grep repository-s3

# Si no aparece, instalarlo (requiere reinicio):
# docker exec es01 bin/elasticsearch-plugin install repository-s3
# docker restart es01

# Agregar credenciales de MinIO al keystore de Elasticsearch
docker exec -it es01 bash -c '
  echo "minioadmin" | bin/elasticsearch-keystore add -f s3.client.minio.access_key --stdin
  echo "minioadmin123" | bin/elasticsearch-keystore add -f s3.client.minio.secret_key --stdin
'

# Reiniciar Elasticsearch para que cargue las nuevas credenciales del keystore
docker restart es01
sleep 30
```

5. Configurar el cliente S3 en Elasticsearch para apuntar a MinIO:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -d '{
  "persistent": {
    "s3.client.minio.endpoint": "minio01:9000",
    "s3.client.minio.protocol": "http",
    "s3.client.minio.path_style_access": "true"
  }
}' | jq .
```

6. Registrar el repositorio de snapshots:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_snapshot/minio-snapshots" \
  -H "Content-Type: application/json" \
  -d '{
  "type": "s3",
  "settings": {
    "bucket": "elastic-snapshots",
    "client": "minio",
    "base_path": "elasticsearch-snapshots"
  }
}' | jq .
```

**Salida esperada:**
```json
{ "acknowledged": true }
```

7. Verificar que el repositorio es accesible:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/_snapshot/minio-snapshots/_verify" | jq .
```

**Salida esperada:**
```json
{
  "nodes": {
    "<node_id>": {
      "name": "es01"
    }
  }
}
```

8. Crear la política SLM `daily-logs-snapshot`:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_slm/policy/daily-logs-snapshot" \
  -H "Content-Type: application/json" \
  -d '{
  "schedule": "0 0 2 * * ?",
  "name": "<daily-logs-snap-{now/d}>",
  "repository": "minio-snapshots",
  "config": {
    "indices": ["logs-*"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "7d",
    "min_count": 1,
    "max_count": 7
  }
}' | jq .
```

**Salida esperada:**
```json
{ "acknowledged": true }
```

9. Ejecutar manualmente el snapshot para validar:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/_slm/policy/daily-logs-snapshot/_execute" | jq .
```

**Salida esperada:**
```json
{
  "snapshot_name": "daily-logs-snap-2025.07.10-..."
}
```

10. Guardar el nombre del snapshot y verificar su estado:

```bash
# Obtener el nombre del snapshot ejecutado
SNAPSHOT_NAME=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_slm/policy/daily-logs-snapshot" | jq -r '.daily-logs-snapshot.last_success.snapshot_name // empty')

# Si el campo anterior está vacío, listar snapshots del repositorio
if [ -z "$SNAPSHOT_NAME" ]; then
  SNAPSHOT_NAME=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
    "${ELASTIC_URL}/_snapshot/minio-snapshots/_all" | jq -r '.snapshots[-1].snapshot')
fi

echo "Snapshot: $SNAPSHOT_NAME"

# Verificar estado
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_snapshot/minio-snapshots/${SNAPSHOT_NAME}" | jq '.snapshots[0] | {snapshot: .snapshot, state: .state, indices: .indices}'
```

**Salida esperada:**
```json
{
  "snapshot": "daily-logs-snap-2025.07.10-...",
  "state": "SUCCESS",
  "indices": [
    ".ds-logs-nginx-default-2025.07.10-000001",
    ".ds-logs-python-app-default-2025.07.10-000001"
  ]
}
```

#### Verificación

```bash
# Verificar la política SLM
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_slm/policy/daily-logs-snapshot" | jq '.daily-logs-snapshot | {policy: .policy, last_success: .last_success.snapshot_name}'
```

---

### Paso 3: Simular pérdida de datos y restaurar desde snapshot

**Objetivo:** Eliminar un backing index del data stream, restaurarlo desde el snapshot a un índice alternativo y verificar la integridad de los documentos.

#### Instrucciones

1. Identificar el backing index a eliminar y contar sus documentos:

```bash
# Listar backing indices del data stream logs-nginx
BACKING_INDEX=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_data_stream/logs-nginx-default" | jq -r '.data_streams[0].indices[0].index_name')

echo "Backing index a eliminar: $BACKING_INDEX"

# Contar documentos
DOC_COUNT=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/${BACKING_INDEX}/_count" | jq '.count')

echo "Documentos en el índice: $DOC_COUNT"
```

2. Si el data stream solo tiene un backing index (el write index), primero realizar rollover para poder eliminar el índice antiguo:

```bash
# Rollover para crear un nuevo write index
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/logs-nginx-default/_rollover" | jq .

# Actualizar la variable con el índice antiguo (ya no es write index)
BACKING_INDEX=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_data_stream/logs-nginx-default" | jq -r '.data_streams[0].indices[0].index_name')

echo "Índice a eliminar (ya no es write index): $BACKING_INDEX"
```

3. Eliminar el backing index simulando la pérdida de datos:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/${BACKING_INDEX}" | jq .
```

**Salida esperada:**
```json
{ "acknowledged": true }
```

4. Verificar que el índice ya no existe:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -o /dev/null -w "%{http_code}" \
  "${ELASTIC_URL}/${BACKING_INDEX}"
```

**Salida esperada:**
```
404
```

5. Restaurar el índice desde el snapshot con un nombre alternativo:

```bash
RESTORED_INDEX="logs-nginx-restored-000001"

curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/_snapshot/minio-snapshots/${SNAPSHOT_NAME}/_restore" \
  -H "Content-Type: application/json" \
  -d "{
  \"indices\": \"${BACKING_INDEX}\",
  \"ignore_unavailable\": true,
  \"include_global_state\": false,
  \"rename_pattern\": \"(.+)\",
  \"rename_replacement\": \"${RESTORED_INDEX}\",
  \"include_aliases\": false
}" | jq .
```

**Salida esperada:**
```json
{
  "accepted": true
}
```

6. Esperar a que la restauración complete y verificar el conteo de documentos:

```bash
# Esperar a que el índice restaurado esté en estado green
sleep 10

RESTORED_COUNT=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/${RESTORED_INDEX}/_count" | jq '.count')

echo "Documentos originales: $DOC_COUNT"
echo "Documentos restaurados: $RESTORED_COUNT"

if [ "$DOC_COUNT" -eq "$RESTORED_COUNT" ]; then
  echo "✅ INTEGRIDAD VERIFICADA: El conteo de documentos coincide."
else
  echo "❌ ERROR: Los conteos no coinciden."
fi
```

**Salida esperada:**
```
Documentos originales: <N>
Documentos restaurados: <N>
✅ INTEGRIDAD VERIFICADA: El conteo de documentos coincide.
```

7. Verificar que se pueden consultar los datos restaurados:

```bash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/${RESTORED_INDEX}/_search?size=3" | jq '.hits.hits[]._source | {timestamp: .["@timestamp"], message: .message}'
```

#### Verificación

```bash
# Estado del índice restaurado
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_cat/indices/${RESTORED_INDEX}?v&h=index,health,status,docs.count,store.size"
```

**Salida esperada (ejemplo):**
```
index                         health status docs.count store.size
logs-nginx-restored-000001    green  open          150       45kb
```

---

### Paso 4: Configurar Dead Letter Queue y Persistent Queue en Logstash

**Objetivo:** Habilitar la dead letter queue (DLQ) y la persistent queue en Logstash, simular una interrupción de Elasticsearch y verificar que los eventos se recuperan automáticamente.

#### Instrucciones

1. Editar la configuración de Logstash (`logstash.yml`) para habilitar la persistent queue y la DLQ:

```bash
cat > ~/elastic-labs/config/logstash/logstash.yml << 'EOF'
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: ["https://es01:9200"]
xpack.monitoring.elasticsearch.username: "elastic"
xpack.monitoring.elasticsearch.password: "ElasticLabs2024!"
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/usr/share/logstash/config/certs/ca/ca.crt"

# Persistent Queue
queue.type: persisted
queue.max_bytes: 1gb
queue.checkpoint.writes: 1024

# Dead Letter Queue
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 512mb
dead_letter_queue.storage_policy: drop_newer
dead_letter_queue.retain.age: 7d

path.dead_letter_queue: "/usr/share/logstash/data/dead_letter_queue"
EOF
```

2. Crear un pipeline de Logstash que envíe datos a Elasticsearch:

```bash
cat > ~/elastic-labs/config/logstash/pipeline/lab08-pipeline.conf << 'EOF'
input {
  http {
    port => 8080
    codec => json
    id => "lab08_http_input"
  }
}

filter {
  mutate {
    add_field => { "[@metadata][target_index]" => "labs-logstash-recovery-%{+YYYY.MM.dd}" }
  }
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
    remove_field => [ "timestamp" ]
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca/ca.crt"]
    index => "%{[@metadata][target_index]}"
    id => "lab08_es_output"
  }
}
EOF
```

3. Crear un pipeline para procesar la DLQ (reintento de eventos fallidos):

```bash
cat > ~/elastic-labs/config/logstash/pipeline/lab08-dlq-pipeline.conf << 'EOF'
input {
  dead_letter_queue {
    path => "/usr/share/logstash/data/dead_letter_queue"
    commit_offsets => true
    pipeline_id => "lab08-pipeline"
    id => "dlq_input"
  }
}

filter {
  mutate {
    remove_field => ["[event][original]"]
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca/ca.crt"]
    index => "labs-logstash-dlq-recovered"
    id => "dlq_es_output"
  }
}
EOF
```

4. Configurar `pipelines.yml` para incluir ambos pipelines:

```bash
cat > ~/elastic-labs/config/logstash/pipelines.yml << 'EOF'
- pipeline.id: lab08-pipeline
  path.config: "/usr/share/logstash/pipeline/lab08-pipeline.conf"
  queue.type: persisted

- pipeline.id: lab08-dlq-pipeline
  path.config: "/usr/share/logstash/pipeline/lab08-dlq-pipeline.conf"
  queue.type: memory
EOF
```

5. Reiniciar Logstash para aplicar la configuración:

```bash
docker restart logstash01
sleep 30

# Verificar que Logstash está operativo
curl -s http://localhost:9600/_node/stats/pipelines | jq '.pipelines | keys'
```

**Salida esperada:**
```json
["lab08-dlq-pipeline", "lab08-pipeline"]
```

6. Enviar eventos de prueba a Logstash antes de la interrupción:

```bash
# Enviar 50 eventos iniciales
for i in $(seq 1 50); do
  curl -s -X POST "http://localhost:8080" \
    -H "Content-Type: application/json" \
    -d "{\"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)\", \"message\": \"Evento pre-interrupcion $i\", \"log.level\": \"INFO\", \"service.name\": \"lab08-test\"}" > /dev/null
done

echo "50 eventos enviados antes de la interrupción"
sleep 5

# Verificar que llegaron
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/labs-logstash-recovery-*/_count" | jq '.count'
```

**Salida esperada:**
```
50
```

7. Simular la interrupción de Elasticsearch:

```bash
echo "=== Deteniendo Elasticsearch ==="
docker stop es01
echo "Elasticsearch detenido. Enviando eventos durante la interrupción..."

# Enviar 30 eventos mientras Elasticsearch está caído
for i in $(seq 1 30); do
  curl -s -X POST "http://localhost:8080" \
    -H "Content-Type: application/json" \
    -d "{\"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)\", \"message\": \"Evento durante-interrupcion $i\", \"log.level\": \"WARN\", \"service.name\": \"lab08-test\"}" > /dev/null
  sleep 2
done

echo "30 eventos enviados durante la interrupción (60 segundos transcurridos)"
```

8. Verificar que la persistent queue está acumulando eventos:

```bash
# Verificar estadísticas de la queue
curl -s http://localhost:9600/_node/stats/pipelines/lab08-pipeline | \
  jq '.pipelines["lab08-pipeline"].queue | {type: .type, events_count: .events_count, queue_size_in_bytes: .queue_size_in_bytes}'
```

**Salida esperada (ejemplo):**
```json
{
  "type": "persisted",
  "events_count": 30,
  "queue_size_in_bytes": 45678
}
```

9. Reiniciar Elasticsearch y esperar la reconexión:

```bash
echo "=== Reiniciando Elasticsearch ==="
docker start es01

# Esperar a que Elasticsearch esté disponible
echo "Esperando a que Elasticsearch se recupere..."
for i in $(seq 1 60); do
  STATUS=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
    -o /dev/null -w "%{http_code}" "${ELASTIC_URL}/_cluster/health" 2>/dev/null)
  if [ "$STATUS" = "200" ]; then
    echo "Elasticsearch disponible después de ${i} segundos"
    break
  fi
  sleep 1
done

# Esperar a que Logstash drene la queue
echo "Esperando 30 segundos para que Logstash procese la queue..."
sleep 30
```

10. Verificar que todos los eventos fueron procesados:

```bash
TOTAL_COUNT=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/labs-logstash-recovery-*/_count" | jq '.count')

echo "Total de eventos indexados: $TOTAL_COUNT"

if [ "$TOTAL_COUNT" -ge 80 ]; then
  echo "✅ RECUPERACIÓN EXITOSA: Se recuperaron los eventos de la persistent queue."
else
  echo "⚠️  Algunos eventos pueden estar en la DLQ. Verificando..."
fi
```

**Salida esperada:**
```
Total de eventos indexados: 80
✅ RECUPERACIÓN EXITOSA: Se recuperaron los eventos de la persistent queue.
```

11. Verificar el estado de la persistent queue después de la recuperación:

```bash
curl -s http://localhost:9600/_node/stats/pipelines/lab08-pipeline | \
  jq '.pipelines["lab08-pipeline"].queue | {events_count: .events_count, max_queue_size_in_bytes: .max_queue_size_in_bytes}'
```

**Salida esperada:**
```json
{
  "events_count": 0,
  "max_queue_size_in_bytes": 1073741824
}
```

#### Verificación

```bash
# Verificar la distribución de eventos por tipo
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/labs-logstash-recovery-*/_search" \
  -H "Content-Type: application/json" \
  -d '{
  "size": 0,
  "aggs": {
    "por_nivel": {
      "terms": { "field": "log.level.keyword" }
    }
  }
}' | jq '.aggregations.por_nivel.buckets'
```

**Salida esperada:**
```json
[
  { "key": "INFO", "doc_count": 50 },
  { "key": "WARN", "doc_count": 30 }
]
```

---

## Validación y Pruebas

### Validación integral del laboratorio

Ejecutar el siguiente script de validación completo:

```bash
#!/bin/bash
echo "=========================================="
echo "  VALIDACIÓN INTEGRAL - Lab 08-00-01"
echo "=========================================="

PASS=0
FAIL=0

# Test 1: Política ILM existe
echo -n "[1/6] Política ILM 'logs-retention-policy'... "
POLICY=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_ilm/policy/logs-retention-policy" 2>/dev/null | jq -r '.logs-retention-policy.policy.phases.hot.actions.rollover.max_age // empty')
if [ "$POLICY" = "7d" ]; then
  echo "✅ PASS"; ((PASS++))
else
  echo "❌ FAIL (max_age esperado: 7d, obtenido: $POLICY)"; ((FAIL++))
fi

# Test 2: Repositorio de snapshots verificado
echo -n "[2/6] Repositorio 'minio-snapshots' accesible... "
REPO=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/_snapshot/minio-snapshots/_verify" 2>/dev/null | jq -r '.nodes | length')
if [ "$REPO" -ge 1 ]; then
  echo "✅ PASS"; ((PASS++))
else
  echo "❌ FAIL"; ((FAIL++))
fi

# Test 3: Política SLM existe
echo -n "[3/6] Política SLM 'daily-logs-snapshot'... "
SLM=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_slm/policy/daily-logs-snapshot" 2>/dev/null | jq -r '.daily-logs-snapshot.policy.repository // empty')
if [ "$SLM" = "minio-snapshots" ]; then
  echo "✅ PASS"; ((PASS++))
else
  echo "❌ FAIL"; ((FAIL++))
fi

# Test 4: Índice restaurado existe
echo -n "[4/6] Índice restaurado 'logs-nginx-restored-000001'... "
RESTORED=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -o /dev/null -w "%{http_code}" "${ELASTIC_URL}/logs-nginx-restored-000001" 2>/dev/null)
if [ "$RESTORED" = "200" ]; then
  echo "✅ PASS"; ((PASS++))
else
  echo "❌ FAIL (HTTP $RESTORED)"; ((FAIL++))
fi

# Test 5: Persistent queue configurada en Logstash
echo -n "[5/6] Logstash persistent queue activa... "
QUEUE_TYPE=$(curl -s http://localhost:9600/_node/stats/pipelines/lab08-pipeline 2>/dev/null | \
  jq -r '.pipelines["lab08-pipeline"].queue.type // empty')
if [ "$QUEUE_TYPE" = "persisted" ]; then
  echo "✅ PASS"; ((PASS++))
else
  echo "❌ FAIL (tipo: $QUEUE_TYPE)"; ((FAIL++))
fi

# Test 6: Eventos recuperados tras interrupción
echo -n "[6/6] Eventos recuperados (≥80)... "
EVENTS=$(curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/labs-logstash-recovery-*/_count" 2>/dev/null | jq '.count // 0')
if [ "$EVENTS" -ge 80 ]; then
  echo "✅ PASS ($EVENTS eventos)"; ((PASS++))
else
  echo "❌ FAIL ($EVENTS eventos)"; ((FAIL++))
fi

echo "=========================================="
echo "  RESULTADO: $PASS/6 pruebas exitosas"
echo "=========================================="
```

---

## Resolución de Problemas

### Problema 1: El snapshot falla con error `repository_verification_exception`

**Síntomas:**
```json
{
  "error": {
    "type": "repository_verification_exception",
    "reason": "[minio-snapshots] path [elasticsearch-snapshots] is not accessible on master node"
  }
}
```

**Causa:** Elasticsearch no puede conectarse a MinIO. Esto ocurre cuando las credenciales S3 no están correctamente almacenadas en el keystore, el endpoint de MinIO no es alcanzable desde la red Docker, o el bucket no existe.

**Solución:**

```bash
# 1. Verificar conectividad desde el contenedor de Elasticsearch a MinIO
docker exec es01 curl -s http://minio01:9000/minio/health/live

# 2. Verificar que el bucket existe
docker exec minio01 mc ls local/elastic-snapshots

# 3. Recargar las credenciales del keystore sin reiniciar
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X POST "${ELASTIC_URL}/_nodes/reload_secure_settings" \
  -H "Content-Type: application/json" \
  -d '{"secure_settings_password": ""}' | jq .

# 4. Verificar la configuración del cliente S3
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  "${ELASTIC_URL}/_cluster/settings?include_defaults=true" | \
  jq '.persistent.s3'
```

---

### Problema 2: La persistent queue de Logstash no drena eventos tras reiniciar Elasticsearch

**Síntomas:** Después de reiniciar Elasticsearch, el campo `events_count` en las estadísticas de la queue de Logstash permanece en un valor > 0 y no disminuye. Los logs de Logstash muestran errores repetidos de conexión.

**Causa:** Logstash intenta reconectarse pero el certificado TLS de Elasticsearch cambió tras el reinicio, o el contenedor de Elasticsearch obtuvo una IP diferente en la red Docker y la resolución DNS interna no se actualizó inmediatamente.

**Solución:**

```bash
# 1. Verificar los logs de Logstash para identificar el error exacto
docker logs logstash01 --tail 50 2>&1 | grep -i "error\|failed\|refused"

# 2. Verificar que Elasticsearch es alcanzable desde Logstash
docker exec logstash01 curl -s -k https://es01:9200 -u elastic:ElasticLabs2024!

# 3. Si hay error de certificado, verificar que el CA está montado correctamente
docker exec logstash01 ls -la /usr/share/logstash/config/certs/ca/ca.crt

# 4. Forzar la reconexión reiniciando Logstash (la queue persiste en disco)
docker restart logstash01
sleep 30

# 5. Verificar que la queue comienza a drenarse
watch -n 5 'curl -s http://localhost:9600/_node/stats/pipelines/lab08-pipeline | jq ".pipelines[\"lab08-pipeline\"].queue.events_count"'
```

> **Nota:** La persistent queue garantiza que los datos sobreviven al reinicio de Logstash. Al reiniciar, Logstash lee los eventos pendientes del disco y los reenvía automáticamente una vez que la conexión con Elasticsearch se restablece.

---

## Limpieza

```bash
# Eliminar el índice restaurado
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/logs-nginx-restored-000001" | jq .

# Eliminar los índices de prueba de Logstash
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/labs-logstash-recovery-*" | jq .

curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/labs-logstash-dlq-recovered" | jq .

# Eliminar la política SLM (opcional - mantener si se usará en labs posteriores)
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/_slm/policy/daily-logs-snapshot" | jq .

# Eliminar snapshots del repositorio
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/_snapshot/minio-snapshots/*" | jq .

# Desregistrar el repositorio
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X DELETE "${ELASTIC_URL}/_snapshot/minio-snapshots" | jq .

# Detener y eliminar el contenedor MinIO
cd ~/elastic-labs/config
docker compose -f docker-compose-minio.yml down -v

# Restaurar el intervalo de ILM poll a su valor por defecto
curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
  -X PUT "${ELASTIC_URL}/_cluster/settings" \
  -H "Content-Type: application/json" \
  -d '{
  "persistent": {
    "indices.lifecycle.poll_interval": null
  }
}' | jq .

# Eliminar la política ILM (opcional)
# curl -s -k -u "${ELASTIC_USER}:${ELASTIC_PASSWORD}" \
#   -X DELETE "${ELASTIC_URL}/_ilm/policy/logs-retention-policy" | jq .

echo "Limpieza completada."
```

---

## Resumen

En este laboratorio has implementado un flujo completo de retención y recuperación de datos de logs:

1. **Política ILM `logs-retention-policy`**: Configuraste un ciclo de vida automatizado con rollover en la fase hot (7d/10GB), optimización en warm (force merge a 1 segmento) y eliminación en delete (30 días), vinculándola a los data streams de logs mediante una index template.

2. **Repositorio de snapshots con MinIO**: Desplegaste almacenamiento S3-compatible, registraste el repositorio en Elasticsearch y creaste una política SLM que automatiza backups diarios con retención de 7 días.

3. **Restauración de datos**: Simulaste la pérdida de un índice y lo recuperaste desde un snapshot a un índice alternativo, verificando la integridad mediante comparación de conteos de documentos.

4. **Dead Letter Queue y Persistent Queue**: Configuraste Logstash para resistir interrupciones de Elasticsearch, demostrando que los eventos acumulados durante una caída se procesan automáticamente tras la recuperación del destino.

### Conceptos clave reforzados

- Las políticas ILM automatizan la gestión del ciclo de vida sin intervención manual
- Los snapshots son la última línea de defensa contra la pérdida de datos
- La persistent queue de Logstash actúa como buffer ante fallos downstream
- La combinación de ILM + SLM garantiza que los datos se respalden antes de ser eliminados

### Recursos adicionales

- [Elasticsearch ILM API Reference](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/index-lifecycle-management.html)
- [Snapshot and Restore](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/snapshot-restore.html)
- [SLM API Reference](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/snapshot-lifecycle-management-api.html)
- [Logstash Persistent Queues](https://www.elastic.co/guide/en/logstash/8.14/persistent-queues.html)
- [Logstash Dead Letter Queues](https://www.elastic.co/guide/en/logstash/8.14/dead-letter-queues.html)
- [MinIO Docker Quickstart](https://min.io/docs/minio/container/index.html)
