# Corregir un flujo de logs con errores de conectividad, permisos y parsing

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio recibirás un entorno Elastic Stack 8.13.0 deliberadamente degradado con cuatro categorías de fallos inyectados: conectividad TLS entre Elastic Agent y Fleet Server, permisos de archivos de log, errores de parsing en un pipeline de Logstash (Grok y timestamp), y conflictos de mapping en un índice de Elasticsearch. Tu misión es diagnosticar cada problema de forma sistemática, aplicar las correcciones necesarias y validar que el flujo de ingestión queda completamente restaurado con cero documentos rechazados.

## Objetivos de Aprendizaje

- [ ] Identificar y resolver errores de conectividad entre Elastic Agent y Fleet Server usando logs de diagnóstico y la API de estado del agente
- [ ] Corregir problemas de permisos en archivos de log y en roles de usuario que impiden la ingestión correcta de eventos
- [ ] Diagnosticar y reparar errores de parsing incluyendo timestamps mal formateados y patrones Grok incorrectos
- [ ] Detectar y solucionar conflictos de mapping en índices que causan el rechazo de documentos por parte de Elasticsearch

## Prerrequisitos

### Conocimientos Necesarios

- Comprensión de la arquitectura de ingestión de logs con Elastic Stack (Elasticsearch, Kibana, Logstash, Elastic Agent, Fleet)
- Familiaridad con comandos Linux básicos (`chmod`, `chown`, `grep`, `tail`, `curl`)
- Conocimiento de expresiones Grok y mappings de Elasticsearch
- Experiencia básica con certificados TLS y `openssl`

### Acceso Requerido

- Clúster Elasticsearch 8.13.0 operativo en `localhost:9200`
- Kibana 8.13.0 accesible en `localhost:5601`
- Fleet Server 8.13.0 en `localhost:8220`
- Elastic Agent 8.13.0 enrollado con política `lab-policy-default`
- Logstash 8.13.0 con pipeline activo
- Acceso `sudo` en el host del laboratorio

## Entorno del Laboratorio

### Infraestructura de Servicios

| Servicio | Contenedor | Puerto | Estado Esperado |
|----------|-----------|--------|-----------------|
| Elasticsearch | es01 | 9200 | Operativo |
| Kibana | kibana01 | 5601 | Operativo |
| Fleet Server | fleet-server01 | 8220 | Operativo |
| Logstash | logstash01 | 5044 / 8080 | Con errores (a corregir) |
| Nginx | nginx-app | 80 | Operativo |
| Python App | python-app | - | Generando logs |

### Credenciales

| Usuario | Contraseña | Uso |
|---------|-----------|-----|
| `elastic` | `ElasticLabs2024!` | Administración general |
| `kibana_system` | `KibanaLabs2024!` | Comunicación Kibana-ES |

### Preparación del Entorno Degradado

Ejecuta el siguiente script para inyectar los cuatro fallos en el entorno operativo. Este script simula un escenario real donde múltiples problemas se acumulan:

```bash
cd ~/elastic-labs/

# Crear el script de inyección de fallos
cat > scripts/inject-faults.sh << 'EOF'
#!/bin/bash
set -e

echo "=== Inyectando Fallo 1: Conectividad TLS ==="
# Modificar la URL de Fleet Server en la configuración del agente
# y reemplazar el certificado CA con uno inválido
sudo cp /var/log/elastic-agent/elastic-agent-*.ndjson /tmp/agent-backup-log.ndjson 2>/dev/null || true

# Crear un certificado CA falso (expirado)
openssl req -x509 -newkey rsa:2048 -keyout /tmp/fake-ca.key \
  -out ~/elastic-labs/config/certs/fake-ca.crt -days 0 -nodes \
  -subj "/CN=Fake CA/O=Lab Fault Injection" 2>/dev/null

# Sobrescribir la CA del agente con la falsa
sudo cp ~/elastic-labs/config/certs/fake-ca.crt \
  /opt/Elastic/Agent/ca.crt

# Modificar la URL de Fleet en la configuración del agente
sudo sed -i 's|https://fleet-server01:8220|https://fleet-server01:9999|g' \
  /opt/Elastic/Agent/fleet.yml 2>/dev/null || \
sudo sed -i 's|url: .*:8220|url: https://fleet-server01:9999|g' \
  /opt/Elastic/Agent/fleet.enc 2>/dev/null || true

echo "=== Inyectando Fallo 2: Permisos de archivos ==="
# Cambiar permisos de los logs de nginx a 600:root
sudo chmod 600 /var/log/nginx/*.log 2>/dev/null || \
sudo chmod 600 ~/elastic-labs/logs/nginx/*.log 2>/dev/null || true
sudo chown root:root /var/log/nginx/*.log 2>/dev/null || \
sudo chown root:root ~/elastic-labs/logs/nginx/*.log 2>/dev/null || true

echo "=== Inyectando Fallo 3: Pipeline Logstash con Grok incorrecto ==="
# Crear pipeline con patrón Grok que no coincide con el formato real
cat > ~/elastic-labs/config/logstash/app-logs.conf << 'PIPELINE'
input {
  file {
    path => "/var/log/python-app/app.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => plain
  }
}

filter {
  grok {
    # PATRÓN INCORRECTO: espera formato Apache pero los logs son JSON-like
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    # FORMATO INCORRECTO: espera dd/MMM/yyyy pero el log usa ISO8601
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca.crt"]
    index => "labs-logstash-app-%{+YYYY.MM.dd}"
  }
}
PIPELINE

# Reiniciar Logstash para aplicar el pipeline roto
docker restart logstash01 2>/dev/null || sudo systemctl restart logstash

echo "=== Inyectando Fallo 4: Conflicto de Mapping ==="
# Crear un índice con mapping incorrecto para response_code como text
curl -sk -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)" \
  -H "Content-Type: application/json" \
  -d '{
  "mappings": {
    "properties": {
      "response_code": { "type": "text" },
      "message": { "type": "text" },
      "@timestamp": { "type": "date" },
      "client_ip": { "type": "ip" },
      "duration_ms": { "type": "float" }
    }
  }
}'

echo ""
echo "=== Todos los fallos inyectados correctamente ==="
echo "Fallos activos:"
echo "  1. Conectividad: URL Fleet incorrecta + certificado CA falso"
echo "  2. Permisos: logs nginx con permisos 600:root"
echo "  3. Parsing: Grok pattern incorrecto + date format incorrecto"
echo "  4. Mapping: response_code definido como 'text' (debería ser 'integer')"
EOF

chmod +x scripts/inject-faults.sh
sudo bash scripts/inject-faults.sh
```

Espera 30 segundos para que los fallos se manifiesten en los logs antes de comenzar el diagnóstico.

---

## Procedimiento Paso a Paso

### Paso 1: Evaluación Inicial del Estado del Entorno

**Objetivo:** Obtener una vista panorámica del estado de todos los componentes para identificar qué flujos están afectados.

**Instrucciones:**

1. Verifica el estado general del clúster Elasticsearch:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cluster/health?pretty"
```

2. Verifica el estado de los índices relevantes:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-*?v&s=index"
```

3. Verifica el estado del Elastic Agent:

```bash
sudo elastic-agent status
```

4. Revisa los logs recientes del agente buscando errores:

```bash
sudo tail -50 /var/log/elastic-agent/elastic-agent-*.ndjson 2>/dev/null | \
  grep -i "error\|failed\|refused" | tail -20
```

5. Verifica el estado de Logstash:

```bash
docker logs logstash01 --tail 30 2>/dev/null || \
  sudo journalctl -u logstash --no-pager -n 30
```

**Salida Esperada:**

El estado del agente mostrará `DEGRADED` o `FAILED`. Los logs del agente contendrán mensajes como `connection refused` o `x509: certificate signed by unknown authority`. Los logs de Logstash mostrarán `_grokparsefailure` tags.

**Verificación:**

Confirma que puedes identificar al menos tres de los cuatro problemas a partir de la información recopilada. Documenta mentalmente qué componentes están afectados antes de proceder a las correcciones.

---

### Paso 2: Corregir el Error de Conectividad (Fleet Server URL + Certificado TLS)

**Objetivo:** Restaurar la comunicación entre Elastic Agent y Fleet Server corrigiendo la URL de Fleet y el certificado CA.

**Instrucciones:**

1. Diagnostica el problema de conectividad verificando la URL configurada:

```bash
# Ver la configuración actual de Fleet en el agente
sudo cat /opt/Elastic/Agent/fleet.yml 2>/dev/null | grep -i url
# o buscar en el archivo encriptado
sudo elastic-agent inspect 2>/dev/null | grep -i fleet
```

2. Verifica que Fleet Server está operativo en el puerto correcto:

```bash
curl -sk --max-time 5 "https://localhost:8220/api/status"
```

3. Verifica el certificado CA actual del agente:

```bash
openssl x509 -in /opt/Elastic/Agent/ca.crt -noout -dates -subject 2>/dev/null
```

Deberías ver que el certificado está expirado (`notAfter` es una fecha pasada o la fecha actual).

4. Restaura el certificado CA correcto:

```bash
# Copiar el certificado CA válido del stack
sudo cp ~/elastic-labs/config/certs/ca/ca.crt /opt/Elastic/Agent/ca.crt
```

5. Verifica que el nuevo certificado es válido:

```bash
openssl x509 -in /opt/Elastic/Agent/ca.crt -noout -dates -subject
```

6. Corrige la URL de Fleet Server (de puerto 9999 a 8220):

```bash
# Corregir la URL en la configuración del agente
sudo sed -i 's|https://fleet-server01:9999|https://fleet-server01:8220|g' \
  /opt/Elastic/Agent/fleet.yml 2>/dev/null

# Si el archivo es fleet.enc o la configuración está en otro formato:
sudo sed -i 's|:9999|:8220|g' /opt/Elastic/Agent/fleet.yml 2>/dev/null
sudo sed -i 's|:9999|:8220|g' /opt/Elastic/Agent/fleet.enc 2>/dev/null
```

7. Reinicia el servicio del Elastic Agent:

```bash
sudo systemctl restart elastic-agent
```

8. Espera 15 segundos y verifica la reconexión:

```bash
sleep 15
sudo elastic-agent status
```

**Salida Esperada:**

```
Status: HEALTHY
Message: Running
Fleet State: CONNECTED
Fleet Message: (connected)
```

**Verificación:**

```bash
# Confirmar conectividad con Fleet Server usando el certificado correcto
openssl s_client -connect localhost:8220 \
  -CAfile /opt/Elastic/Agent/ca.crt </dev/null 2>/dev/null | \
  grep "Verify return code"
```

La salida debe mostrar `Verify return code: 0 (ok)`.

Adicionalmente, verifica en Kibana navegando a **Management → Fleet → Agents** que el agente aparece con estado **Healthy** (verde).

---

### Paso 3: Corregir los Permisos de Archivos de Log de Nginx

**Objetivo:** Restaurar los permisos de los archivos de log de nginx para que Elastic Agent pueda leerlos.

**Instrucciones:**

1. Identifica el problema de permisos revisando los logs del agente:

```bash
sudo grep -i "permission denied\|cannot open\|access denied" \
  /var/log/elastic-agent/filebeat-*.ndjson 2>/dev/null | tail -5
```

2. Verifica los permisos actuales de los archivos de log de nginx:

```bash
ls -la /var/log/nginx/*.log 2>/dev/null || \
ls -la ~/elastic-labs/logs/nginx/*.log
```

Deberías ver permisos `600` y propietario `root:root`, lo que impide la lectura por otros usuarios.

3. Identifica el usuario bajo el que corre el Elastic Agent:

```bash
ps aux | grep elastic-agent | grep -v grep | awk '{print $1}'
```

4. Corrige los permisos para permitir lectura por el grupo:

```bash
# Opción A: Si el agente corre como root (caso más común)
# El problema podría estar en un directorio padre. Verificar:
sudo ls -la /var/log/nginx/ 2>/dev/null
sudo ls -la ~/elastic-labs/logs/nginx/ 2>/dev/null

# Corregir permisos de los archivos de log
sudo chmod 644 /var/log/nginx/*.log 2>/dev/null
sudo chmod 644 ~/elastic-labs/logs/nginx/*.log 2>/dev/null

# Asegurar que el directorio es accesible
sudo chmod 755 /var/log/nginx/ 2>/dev/null
sudo chmod 755 ~/elastic-labs/logs/nginx/ 2>/dev/null
```

5. Si el agente NO corre como root, añadir el usuario al grupo adecuado:

```bash
# Identificar el grupo del directorio de logs
NGINX_LOG_DIR="/var/log/nginx"
[ -d "$NGINX_LOG_DIR" ] || NGINX_LOG_DIR="$HOME/elastic-labs/logs/nginx"

# Cambiar grupo a uno que el agente pueda leer
sudo chown root:adm ${NGINX_LOG_DIR}/*.log
sudo chmod 640 ${NGINX_LOG_DIR}/*.log

# Añadir el usuario del agente al grupo adm (si no es root)
AGENT_USER=$(ps aux | grep elastic-agent | grep -v grep | awk '{print $1}' | head -1)
if [ "$AGENT_USER" != "root" ]; then
  sudo usermod -aG adm $AGENT_USER
fi
```

6. Verifica que los archivos son ahora legibles:

```bash
# Simular lectura como el usuario del agente
sudo -u $(ps aux | grep elastic-agent | grep -v grep | awk '{print $1}' | head -1) \
  head -5 /var/log/nginx/access.log 2>/dev/null || \
sudo head -5 ~/elastic-labs/logs/nginx/access.log
```

7. Fuerza al agente a releer los archivos (reinicio del input):

```bash
sudo systemctl restart elastic-agent
sleep 10
```

**Salida Esperada:**

Los archivos de log deben mostrar permisos `644` o `640` y ser legibles por el proceso del agente. Tras el reinicio, no deben aparecer más errores de `permission denied` en los logs.

**Verificación:**

```bash
# Verificar que ya no hay errores de permisos en los últimos logs
sudo grep -i "permission denied" \
  /var/log/elastic-agent/filebeat-*.ndjson 2>/dev/null | \
  tail -5 | grep "$(date +%Y-%m-%d)" && \
  echo "FALLO: Aún hay errores de permisos" || \
  echo "OK: No hay errores recientes de permisos"

# Verificar que llegan documentos al índice de nginx
sleep 15
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-nginx-sample/_count?pretty"
```

---

### Paso 4: Corregir el Pipeline de Logstash (Grok y Timestamp)

**Objetivo:** Reparar el patrón Grok y el formato de fecha en el pipeline de Logstash para que los logs de la aplicación Python se parseen correctamente.

**Instrucciones:**

1. Examina el formato real de los logs de la aplicación Python:

```bash
# Ver las últimas líneas del log de la aplicación
tail -5 /var/log/python-app/app.log 2>/dev/null || \
docker logs python-app --tail 5 2>/dev/null || \
tail -5 ~/elastic-labs/logs/python-app/app.log
```

El formato real de los logs debería ser similar a:

```
2025-07-01T14:23:45.123Z INFO [app.main] GET /api/users 200 45ms client=192.168.1.50
2025-07-01T14:23:46.456Z ERROR [app.auth] POST /api/login 401 12ms client=10.0.0.15
```

2. Verifica los errores actuales de Logstash:

```bash
docker logs logstash01 --tail 50 2>/dev/null | grep -i "grokparsefailure\|error\|failed" || \
sudo journalctl -u logstash --no-pager -n 50 | grep -i "grokparsefailure\|error\|failed"
```

3. Verifica los documentos con `_grokparsefailure` en Elasticsearch:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match": { "tags": "_grokparsefailure" } },
    "size": 2
  }'
```

4. Usa el Grok Debugger de Kibana o la línea de comandos para construir el patrón correcto. Basándote en el formato del log:

```
2025-07-01T14:23:45.123Z INFO [app.main] GET /api/users 200 45ms client=192.168.1.50
```

El patrón Grok correcto es:

```
%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log_level} \[%{DATA:logger}\] %{WORD:http_method} %{URIPATH:request_path} %{NUMBER:response_code:int} %{NUMBER:duration}ms client=%{IP:client_ip}
```

5. Corrige el pipeline de Logstash con el patrón Grok y formato de fecha correctos:

```bash
cat > ~/elastic-labs/config/logstash/app-logs.conf << 'PIPELINE'
input {
  file {
    path => "/var/log/python-app/app.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => plain
  }
}

filter {
  grok {
    match => { 
      "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log_level} \[%{DATA:logger}\] %{WORD:http_method} %{URIPATH:request_path} %{NUMBER:response_code:int} %{NUMBER:duration_ms:float}ms client=%{IP:client_ip}" 
    }
    tag_on_failure => ["_grokparsefailure"]
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }
  
  mutate {
    convert => {
      "response_code" => "integer"
      "duration_ms" => "float"
    }
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca.crt"]
    index => "labs-logstash-app-%{+YYYY.MM.dd}"
  }
}
PIPELINE
```

6. Copia el pipeline corregido al contenedor y reinicia Logstash:

```bash
# Si Logstash corre en Docker
docker cp ~/elastic-labs/config/logstash/app-logs.conf \
  logstash01:/usr/share/logstash/pipeline/app-logs.conf
docker restart logstash01

# Si Logstash corre como servicio nativo
sudo cp ~/elastic-labs/config/logstash/app-logs.conf \
  /etc/logstash/conf.d/app-logs.conf
sudo systemctl restart logstash
```

7. Espera a que Logstash se reinicie completamente:

```bash
sleep 30

# Verificar que Logstash está operativo
curl -s "http://localhost:9600/_node/stats/pipelines?pretty" 2>/dev/null | \
  grep -A5 "events" || \
docker logs logstash01 --tail 10 2>/dev/null | grep -i "pipeline\|started"
```

**Salida Esperada:**

Los logs de Logstash deben mostrar `Pipeline started` sin errores. Los nuevos documentos indexados no deben tener el tag `_grokparsefailure`.

**Verificación:**

```bash
# Esperar a que se procesen nuevos logs
sleep 20

# Buscar documentos recientes SIN errores de parsing
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must_not": [
          { "match": { "tags": "_grokparsefailure" } }
        ],
        "must": [
          { "exists": { "field": "log_level" } },
          { "exists": { "field": "client_ip" } }
        ]
      }
    },
    "size": 2,
    "sort": [{"@timestamp": "desc"}]
  }'
```

Deberías ver documentos con campos `log_level`, `http_method`, `request_path`, `response_code`, `duration_ms` y `client_ip` correctamente parseados.

---

### Paso 5: Resolver el Conflicto de Mapping (response_code: text → integer)

**Objetivo:** Eliminar el conflicto de mapping que causa el rechazo de documentos donde `response_code` se envía como entero pero el mapping lo define como texto.

**Instrucciones:**

1. Diagnostica el conflicto verificando los documentos rechazados:

```bash
# Buscar errores de mapping en los logs de Logstash
docker logs logstash01 --tail 100 2>/dev/null | \
  grep -i "mapper_parsing_exception\|illegal_argument\|rejected" || \
sudo journalctl -u logstash --no-pager -n 100 | \
  grep -i "mapper_parsing_exception\|rejected"
```

2. Verifica el mapping actual del índice problemático:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_mapping?pretty" | \
  grep -A3 "response_code"
```

Deberías ver:

```json
"response_code" : {
  "type" : "text"
}
```

Pero el pipeline envía `response_code` como `integer` (por la conversión en el filtro `mutate`).

3. La solución requiere eliminar el índice con mapping incorrecto y permitir que Logstash lo recree con el mapping correcto. Primero, crea un index template que garantice el mapping correcto para futuros índices:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_index_template/labs-logstash-template" \
  -H "Content-Type: application/json" \
  -d '{
  "index_patterns": ["labs-logstash-app-*"],
  "priority": 200,
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message": { "type": "text" },
        "log_level": { "type": "keyword" },
        "logger": { "type": "keyword" },
        "http_method": { "type": "keyword" },
        "request_path": { "type": "keyword" },
        "response_code": { "type": "integer" },
        "duration_ms": { "type": "float" },
        "client_ip": { "type": "ip" },
        "tags": { "type": "keyword" }
      }
    }
  }
}'
```

4. Verifica que el template se creó correctamente:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/labs-logstash-template?pretty" | \
  grep -A2 "response_code"
```

5. Si hay documentos valiosos en el índice actual, reindéxalos a un índice temporal antes de eliminar:

```bash
# Verificar si hay documentos en el índice actual
DOC_COUNT=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_count" | \
  python3 -c "import sys,json; print(json.load(sys.stdin).get('count',0))")

echo "Documentos en el índice actual: $DOC_COUNT"

if [ "$DOC_COUNT" -gt "0" ]; then
  echo "Reindexando documentos válidos..."
  curl -sk -u elastic:ElasticLabs2024! \
    -X POST "https://localhost:9200/_reindex" \
    -H "Content-Type: application/json" \
    -d "{
      \"source\": {
        \"index\": \"labs-logstash-app-$(date +%Y.%m.%d)\",
        \"query\": {
          \"bool\": {
            \"must_not\": [
              { \"match\": { \"tags\": \"_grokparsefailure\" } }
            ]
          }
        }
      },
      \"dest\": {
        \"index\": \"labs-logstash-app-backup\"
      }
    }"
fi
```

6. Elimina el índice con mapping incorrecto:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)?pretty"
```

7. Reinicia Logstash para que recree el índice con el template correcto:

```bash
docker restart logstash01 2>/dev/null || sudo systemctl restart logstash
sleep 30
```

8. Verifica que el nuevo índice se creó con el mapping correcto:

```bash
# Esperar a que Logstash procese nuevos eventos
sleep 20

curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_mapping?pretty" | \
  grep -A3 "response_code"
```

**Salida Esperada:**

```json
"response_code" : {
  "type" : "integer"
}
```

**Verificación:**

```bash
# Verificar que no hay documentos rechazados
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match_all": {} },
    "size": 3,
    "sort": [{"@timestamp": "desc"}]
  }' | grep -E "response_code|duration_ms|client_ip"
```

Los campos `response_code` deben aparecer como valores numéricos (200, 401, 500, etc.) y `client_ip` como direcciones IP válidas.

---

### Paso 6: Validación Integral del Flujo Restaurado

**Objetivo:** Confirmar que todos los cuatro problemas han sido resueltos y el flujo de ingestión funciona de extremo a extremo sin errores.

**Instrucciones:**

1. Verifica el estado completo del Elastic Agent:

```bash
sudo elastic-agent status
```

2. Verifica que no hay errores recientes en los logs del agente:

```bash
sudo grep -c "error" /var/log/elastic-agent/elastic-agent-*.ndjson 2>/dev/null | \
  awk -F: '{sum+=$2} END {print "Total errores en logs: " sum}'

# Verificar solo errores de los últimos 5 minutos
sudo grep "$(date -u +%Y-%m-%dT%H:%M 2>/dev/null || date +%Y-%m-%dT%H:%M)" \
  /var/log/elastic-agent/elastic-agent-*.ndjson 2>/dev/null | \
  grep -ic "error" || echo "0 errores recientes"
```

3. Verifica el conteo de documentos en todos los índices del laboratorio:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-*?v&s=index&h=index,docs.count,store.size"
```

4. Verifica que Logstash no tiene documentos con errores de parsing:

```bash
FAILURES=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_count" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"tags":"_grokparsefailure"}}}' | \
  python3 -c "import sys,json; print(json.load(sys.stdin).get('count',0))")

TOTAL=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_count" | \
  python3 -c "import sys,json; print(json.load(sys.stdin).get('count',0))")

echo "Documentos totales en labs-logstash-app-*: $TOTAL"
echo "Documentos con _grokparsefailure: $FAILURES"
echo "Tasa de éxito de parsing: $(echo "scale=1; ($TOTAL-$FAILURES)*100/$TOTAL" | bc 2>/dev/null || echo 'N/A')%"
```

5. Envía un documento de prueba manual para confirmar que no hay conflictos de mapping:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_doc" \
  -H "Content-Type: application/json" \
  -d '{
    "@timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%S.000Z)'",
    "message": "TEST validation document",
    "log_level": "INFO",
    "logger": "validation.test",
    "http_method": "GET",
    "request_path": "/api/health",
    "response_code": 200,
    "duration_ms": 5.2,
    "client_ip": "127.0.0.1"
  }'
```

**Salida Esperada:**

```json
{"_index":"labs-logstash-app-2025.07.01","_id":"...","_version":1,"result":"created",...}
```

No debe haber errores de tipo `mapper_parsing_exception`.

---

## Validación y Testing

Ejecuta el siguiente script de validación completa para confirmar que todos los objetivos se han cumplido:

```bash
cat > ~/elastic-labs/scripts/validate-lab06.sh << 'EOF'
#!/bin/bash
echo "╔══════════════════════════════════════════════════════╗"
echo "║  VALIDACIÓN COMPLETA - Lab 06-00-01                  ║"
echo "╚══════════════════════════════════════════════════════╝"
echo ""

PASS=0
FAIL=0

# Test 1: Elastic Agent conectado a Fleet
echo -n "[TEST 1] Elastic Agent conectado a Fleet Server... "
AGENT_STATUS=$(sudo elastic-agent status 2>/dev/null | grep -i "fleet state" | grep -ci "connected")
if [ "$AGENT_STATUS" -ge "1" ]; then
  echo "✅ PASS"
  ((PASS++))
else
  echo "❌ FAIL - Agent no conectado a Fleet"
  ((FAIL++))
fi

# Test 2: Certificado TLS válido
echo -n "[TEST 2] Certificado CA válido para Fleet Server... "
CERT_VALID=$(openssl x509 -in /opt/Elastic/Agent/ca.crt -checkend 86400 -noout 2>/dev/null && echo "valid" || echo "expired")
if [ "$CERT_VALID" = "valid" ]; then
  echo "✅ PASS"
  ((PASS++))
else
  echo "❌ FAIL - Certificado expirado o inválido"
  ((FAIL++))
fi

# Test 3: Permisos de archivos de log nginx
echo -n "[TEST 3] Permisos de logs nginx legibles... "
NGINX_LOG="/var/log/nginx/access.log"
[ -f "$NGINX_LOG" ] || NGINX_LOG="$HOME/elastic-labs/logs/nginx/access.log"
if [ -r "$NGINX_LOG" ]; then
  PERMS=$(stat -c %a "$NGINX_LOG" 2>/dev/null || stat -f %Lp "$NGINX_LOG" 2>/dev/null)
  if [ "$PERMS" != "600" ]; then
    echo "✅ PASS (permisos: $PERMS)"
    ((PASS++))
  else
    echo "❌ FAIL - Permisos aún son 600"
    ((FAIL++))
  fi
else
  echo "⚠️  SKIP - Archivo no encontrado"
  ((PASS++))
fi

# Test 4: Pipeline Logstash sin grokparsefailure recientes
echo -n "[TEST 4] Logstash sin errores de parsing recientes... "
GROK_FAILS=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_count" \
  -H "Content-Type: application/json" \
  -d '{"query":{"bool":{"must":[{"match":{"tags":"_grokparsefailure"}},{"range":{"@timestamp":{"gte":"now-5m"}}}]}}}' 2>/dev/null | \
  python3 -c "import sys,json; print(json.load(sys.stdin).get('count',0))" 2>/dev/null)
if [ "${GROK_FAILS:-0}" -eq "0" ]; then
  echo "✅ PASS (0 fallos de Grok en últimos 5 min)"
  ((PASS++))
else
  echo "❌ FAIL - $GROK_FAILS documentos con _grokparsefailure"
  ((FAIL++))
fi

# Test 5: Mapping de response_code es integer
echo -n "[TEST 5] Mapping response_code = integer... "
MAPPING_TYPE=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_mapping" 2>/dev/null | \
  python3 -c "
import sys,json
data = json.load(sys.stdin)
for idx in data:
  props = data[idx].get('mappings',{}).get('properties',{})
  if 'response_code' in props:
    print(props['response_code'].get('type','unknown'))
    break
" 2>/dev/null)
if [ "$MAPPING_TYPE" = "integer" ]; then
  echo "✅ PASS"
  ((PASS++))
else
  echo "❌ FAIL - Tipo actual: ${MAPPING_TYPE:-no encontrado}"
  ((FAIL++))
fi

# Test 6: Documento de prueba se indexa sin error
echo -n "[TEST 6] Indexación de documento sin conflicto... "
RESULT=$(curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_doc" \
  -H "Content-Type: application/json" \
  -d "{
    \"@timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)\",
    \"message\": \"validation test\",
    \"log_level\": \"INFO\",
    \"response_code\": 200,
    \"duration_ms\": 1.5,
    \"client_ip\": \"10.0.0.1\"
  }" 2>/dev/null | python3 -c "import sys,json; print(json.load(sys.stdin).get('result','error'))" 2>/dev/null)
if [ "$RESULT" = "created" ]; then
  echo "✅ PASS"
  ((PASS++))
else
  echo "❌ FAIL - Resultado: ${RESULT:-error de conexión}"
  ((FAIL++))
fi

echo ""
echo "════════════════════════════════════════════════════════"
echo "  Resultados: $PASS pasados / $FAIL fallidos de $((PASS+FAIL)) tests"
echo "════════════════════════════════════════════════════════"

if [ "$FAIL" -eq "0" ]; then
  echo "  🎉 ¡LABORATORIO COMPLETADO EXITOSAMENTE!"
else
  echo "  ⚠️  Revisa los tests fallidos antes de continuar."
fi
EOF

chmod +x ~/elastic-labs/scripts/validate-lab06.sh
bash ~/elastic-labs/scripts/validate-lab06.sh
```

**Criterio de éxito:** Los 6 tests deben pasar (✅) para considerar el laboratorio completado.

---

## Resolución de Problemas

### Problema 1: El agente sigue en estado DEGRADED después de corregir la URL y el certificado

**Síntomas:**
- `elastic-agent status` muestra `Status: DEGRADED` con mensaje `1 or more components/units are not healthy`
- Fleet State aparece como `CONNECTED` pero los componentes individuales (filebeat, metricbeat) están en `FAILED`

**Causa:**
El agente se reconectó a Fleet exitosamente, pero la política descargada todavía contiene referencias a la configuración anterior. Además, el sub-proceso filebeat puede tener un archivo de estado (`registry`) corrupto que le impide reiniciar correctamente el seguimiento de archivos.

**Solución:**

```bash
# 1. Verificar qué componente específico está fallando
sudo elastic-agent status | grep -A2 "FAILED\|DEGRADED"

# 2. Limpiar el registry de filebeat (fuerza re-lectura de archivos)
sudo find /var/lib/elastic-agent/ -name "registry" -type d -exec rm -rf {} + 2>/dev/null
sudo find /opt/Elastic/Agent/data/ -name "registry" -type d -exec rm -rf {} + 2>/dev/null

# 3. Reiniciar completamente el agente
sudo systemctl stop elastic-agent
sleep 5
sudo systemctl start elastic-agent

# 4. Esperar 30 segundos y verificar
sleep 30
sudo elastic-agent status
```

Si el problema persiste, verifica que la política en Fleet no tiene errores navegando a **Kibana → Fleet → Agent policies → lab-policy-default** y revisando las integraciones configuradas.

---

### Problema 2: Logstash indexa documentos pero response_code sigue apareciendo como texto

**Síntomas:**
- El index template se creó correctamente (se puede verificar con `_index_template`)
- Sin embargo, el índice del día actual sigue teniendo `response_code` como `text`
- Los nuevos documentos se rechazan con `mapper_parsing_exception`

**Causa:**
El índice del día actual fue creado ANTES de que se aplicara el index template. Los index templates solo afectan a índices NUEVOS; no modifican el mapping de índices existentes. Si el índice no se eliminó correctamente o Logstash lo recreó antes de que el template estuviera activo, el conflicto persiste.

**Solución:**

```bash
# 1. Verificar que el template existe y tiene prioridad correcta
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/labs-logstash-template?pretty" | \
  grep -E "priority|response_code"

# 2. Detener Logstash temporalmente para evitar que recree el índice
docker stop logstash01 2>/dev/null || sudo systemctl stop logstash

# 3. Eliminar el índice problemático
curl -sk -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)"

# 4. Verificar que no existe
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-logstash-app-*?v"

# 5. Reiniciar Logstash - el template se aplicará al crear el nuevo índice
docker start logstash01 2>/dev/null || sudo systemctl start logstash

# 6. Esperar y verificar el mapping del nuevo índice
sleep 40
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-$(date +%Y.%m.%d)/_mapping?pretty" | \
  grep -A3 "response_code"
```

La clave es que **el template debe existir ANTES de que se cree el índice**. El orden correcto es: crear template → eliminar índice viejo → dejar que Logstash cree el índice nuevo.

---

## Limpieza

Una vez completado y validado el laboratorio, ejecuta los siguientes comandos para dejar el entorno limpio como base para el siguiente laboratorio:

```bash
# Eliminar archivos temporales de la inyección de fallos
rm -f /tmp/fake-ca.key
rm -f ~/elastic-labs/config/certs/fake-ca.crt
rm -f ~/elastic-labs/scripts/inject-faults.sh

# Eliminar el índice de backup si se creó
curl -sk -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-logstash-app-backup" 2>/dev/null

# Eliminar documentos con _grokparsefailure del período de pruebas
curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:9200/labs-logstash-app-*/_delete_by_query" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"tags":"_grokparsefailure"}}}'

# Verificar estado final limpio
echo ""
echo "=== Estado Final del Entorno ==="
sudo elastic-agent status | head -5
echo ""
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-*?v&s=index&h=index,health,docs.count"
echo ""
echo "Entorno listo para el siguiente laboratorio."
```

---

## Resumen

En este laboratorio has aplicado un flujo de diagnóstico sistemático para resolver cuatro categorías de problemas comunes en la ingestión de logs con Elastic Stack:

| Problema | Herramienta de Diagnóstico | Solución Aplicada |
|----------|---------------------------|-------------------|
| Conectividad TLS | `elastic-agent status`, `openssl s_client` | Restaurar CA válida + corregir URL de Fleet |
| Permisos de archivos | `ls -la`, logs de filebeat con `permission denied` | `chmod 644` sobre archivos de log |
| Parsing Grok/Date | Grok Debugger, búsqueda de `_grokparsefailure` | Reescribir patrón Grok + formato `ISO8601` |
| Conflicto de Mapping | `_mapping` API, `mapper_parsing_exception` | Index template + eliminar/recrear índice |

**Conceptos clave reforzados:**

- El comando `elastic-agent status` es el punto de partida para cualquier diagnóstico del agente
- Los certificados TLS expirados o incorrectos son una causa frecuente y silenciosa de pérdida de datos
- Los index templates solo aplican a índices nuevos; los índices existentes requieren reindexación o recreación
- El diagnóstico efectivo requiere un enfoque por capas: servicio → red → autenticación → datos

### Recursos Adicionales

- [Elastic Agent Troubleshooting Guide](https://www.elastic.co/guide/en/fleet/8.13/fleet-troubleshooting.html)
- [Logstash Grok Filter Plugin](https://www.elastic.co/guide/en/logstash/8.13/plugins-filters-grok.html)
- [Elasticsearch Index Templates](https://www.elastic.co/guide/en/elasticsearch/reference/8.13/index-templates.html)
- [Elasticsearch Mapping Conflicts](https://www.elastic.co/guide/en/elasticsearch/reference/8.13/mapping.html#mapping-limit-settings)
