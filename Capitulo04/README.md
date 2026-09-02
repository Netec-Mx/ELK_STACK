# Construir y depurar un pipeline de Logstash para logs de aplicaciones

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En esta práctica construirás un pipeline completo de Logstash que procesa logs de una aplicación Java con excepciones multiline. Partirás de un archivo de configuración incompleto con secciones marcadas como `TODO`, aplicarás plugins de filtrado (grok, dissect, date, json, mutate, geoip, translate) para normalizar los eventos al estándar ECS, y depurarás errores intencionales utilizando los logs de Logstash y la Dead Letter Queue. El resultado será un índice `labs-logstash-app-{fecha}` correctamente mapeado que servirá como fuente de datos para el dashboard de la Práctica 5.

## Objetivos de Aprendizaje

- [ ] Construir un pipeline de Logstash funcional con input file (codec multiline), múltiples filters y output a Elasticsearch
- [ ] Aplicar los plugins grok, dissect, mutate, date, json, geoip y translate para extraer, normalizar y enriquecer campos de logs Java
- [ ] Mapear campos extraídos al estándar ECS (log.level, log.logger, error.message, error.stack_trace)
- [ ] Configurar y utilizar la Dead Letter Queue (DLQ) para capturar y diagnosticar eventos fallidos
- [ ] Depurar errores de parsing en un pipeline de Logstash mediante análisis de logs y corrección iterativa

## Prerrequisitos

### Conocimiento Previo

- Comprensión de la arquitectura de Logstash (inputs, filters, outputs, codecs) — Lección 4.1
- Familiaridad con expresiones regulares básicas para patrones grok
- Práctica 2 completada: index template `labs-logstash-template` y política ILM `labs-policy` activos en Elasticsearch
- Práctica 3 completada: experiencia con flujos de datos end-to-end

### Acceso Requerido

- Acceso al host Ubuntu 22.04 con Docker Engine y Docker Compose operativos
- Contenedores `es01` y `kibana01` en ejecución y saludables
- Puerto 9200 (Elasticsearch) y 5601 (Kibana) accesibles desde el host
- Directorio de trabajo `~/elastic-labs/` con permisos de escritura

## Entorno del Laboratorio

### Software Utilizado

| Componente | Versión | Rol |
|---|---|---|
| Logstash | 8.14.1 | Motor de procesamiento de pipeline |
| Elasticsearch | 8.14.1 | Almacenamiento e indexación de eventos |
| Kibana | 8.14.1 | Validación visual de documentos |
| Docker / Docker Compose | 26.1.4 / 2.27.1 | Orquestación de contenedores |

### Configuración Inicial del Entorno

Verifica que los contenedores base estén en ejecución:

```bash
cd ~/elastic-labs
docker compose ps --format "table {{.Name}}\t{{.Status}}"
```

Resultado esperado: `es01` y `kibana01` con estado `healthy`.

Verifica que el index template de la Práctica 2 exista:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:9200/_index_template/labs-logstash-template | jq .
```

Si no existe, créalo antes de continuar (consulta la guía de la Práctica 2).

---

## Paso a Paso

### Paso 1: Crear el archivo de logs de muestra (java_exception.log)

**Objetivo:** Generar el archivo de logs de aplicación Java con excepciones multiline que Logstash procesará.

**Instrucciones:**

1. Crea el directorio para los logs de entrada:

```bash
mkdir -p ~/elastic-labs/logs/logstash-input
```

2. Crea el archivo de logs de muestra con eventos multiline:

```bash
cat > ~/elastic-labs/logs/logstash-input/java_exception.log << 'EOF'
2024-06-15 10:23:45.123 ERROR com.labs.service.PaymentService - Payment processing failed | client_ip=203.0.113.45 | error_code=PAY_001 | context={"transaction_id":"txn-7890","amount":150.00,"currency":"USD"}
java.lang.NullPointerException: Cannot invoke method on null object
	at com.labs.service.PaymentService.processPayment(PaymentService.java:142)
	at com.labs.controller.PaymentController.handleRequest(PaymentController.java:87)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:897)
2024-06-15 10:23:46.456 WARN com.labs.service.AuthService - Authentication attempt suspicious | client_ip=198.51.100.22 | error_code=AUTH_003 | context={"user_id":"usr-1234","attempts":5,"locked":true}
2024-06-15 10:23:47.789 ERROR com.labs.service.OrderService - Order creation failed | client_ip=192.0.2.100 | error_code=ORD_002 | context={"order_id":"ord-5678","items":3,"total":299.99}
java.io.IOException: Connection reset by peer
	at com.labs.service.OrderService.createOrder(OrderService.java:203)
	at com.labs.controller.OrderController.submitOrder(OrderController.java:54)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:897)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:750)
2024-06-15 10:23:48.012 INFO com.labs.service.UserService - User profile updated successfully | client_ip=45.33.32.156 | error_code=NONE | context={"user_id":"usr-9999","fields_updated":["email","phone"]}
2024-06-15 10:23:49.345 ERROR com.labs.service.InventoryService - Stock check failed | client_ip=104.236.198.48 | error_code=INV_004 | context={"product_id":"prod-4321","warehouse":"US-EAST","requested":10}
java.sql.SQLException: Connection pool exhausted
	at com.labs.service.InventoryService.checkStock(InventoryService.java:88)
	at com.labs.controller.InventoryController.getAvailability(InventoryController.java:33)
2024-06-15 10:23:50.678 WARN com.labs.service.NotificationService - Email delivery delayed | client_ip=203.0.113.12 | error_code=NOTIF_005 | context={"recipient":"user@example.com","retry_count":3,"queue_depth":150}
2024-06-15 10:23:51.901 ERROR com.labs.service.CacheService - Cache invalidation error | client_ip=198.51.100.55 | error_code=CACHE_006 | context={"cache_region":"products","keys_affected":42,"ttl_remaining":0}
java.util.concurrent.TimeoutException: Redis operation timed out after 5000ms
	at com.labs.service.CacheService.invalidateRegion(CacheService.java:156)
	at com.labs.scheduler.CacheMaintenanceJob.execute(CacheMaintenanceJob.java:29)
EOF
```

3. Verifica el archivo:

```bash
wc -l ~/elastic-labs/logs/logstash-input/java_exception.log
```

**Resultado esperado:** `25 /home/.../java_exception.log` (25 líneas totales que formarán 7 eventos lógicos tras agrupar multiline).

**Verificación:** El archivo contiene 4 eventos ERROR con stack traces y 3 eventos sin stack trace (WARN/INFO).

---

### Paso 2: Crear el diccionario de traducción para el plugin translate

**Objetivo:** Preparar el archivo CSV que mapea códigos de error a descripciones legibles.

**Instrucciones:**

1. Crea el directorio de configuración de Logstash:

```bash
mkdir -p ~/elastic-labs/config/logstash/dictionaries
```

2. Crea el archivo de diccionario:

```bash
cat > ~/elastic-labs/config/logstash/dictionaries/error_codes.csv << 'EOF'
PAY_001,Payment gateway timeout or rejection
AUTH_003,Multiple failed authentication attempts detected
ORD_002,Order persistence failure due to backend connectivity
INV_004,Inventory database connection pool exhausted
NOTIF_005,Notification service delivery delay
CACHE_006,Cache backend operation timeout
NONE,No error - successful operation
EOF
```

3. Verifica el contenido:

```bash
cat ~/elastic-labs/config/logstash/dictionaries/error_codes.csv
```

**Resultado esperado:** 7 líneas con formato `CÓDIGO,Descripción`.

---

### Paso 3: Crear el pipeline de Logstash completo

**Objetivo:** Construir el archivo de configuración del pipeline con input multiline, filters de procesamiento y output a Elasticsearch.

**Instrucciones:**

1. Crea el directorio para el pipeline:

```bash
mkdir -p ~/elastic-labs/config/logstash/pipeline
```

2. Crea el archivo de configuración del pipeline:

```bash
cat > ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf << 'PIPELINE'
# Pipeline: labs-app-pipeline
# Procesa logs de aplicación Java con excepciones multiline
# Normaliza al estándar ECS y enriquece con geoip y translate

input {
  file {
    path => "/var/log/app/java_exception.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => multiline {
      pattern => "^%{TIMESTAMP_ISO8601}"
      negate => true
      what => "previous"
    }
    type => "java_app"
  }
}

filter {
  # --- FASE 1: Extracción inicial con grok ---
  grok {
    match => {
      "message" => "^%{TIMESTAMP_ISO8601:raw_timestamp}\s+%{LOGLEVEL:raw_level}\s+%{JAVACLASS:raw_logger}\s+-\s+%{DATA:raw_message}\s+\|\s+client_ip=%{IP:raw_client_ip}\s+\|\s+error_code=%{WORD:raw_error_code}\s+\|\s+context=%{GREEDYDATA:raw_context}"
    }
    tag_on_failure => ["_grokparsefailure"]
  }

  # --- FASE 2: Extraer stack trace si existe ---
  if [message] =~ /\n\t/ {
    dissect {
      mapping => {
        "message" => "%{raw_first_line}
%{raw_stack_trace}"
      }
    }
    # Limpiar: solo conservar desde la primera línea de excepción
    mutate {
      gsub => [
        "raw_stack_trace", "^[^\n]*\n", ""
      ]
    }
    # Extraer el nombre de la excepción del stack trace
    grok {
      match => {
        "message" => "\n(?<raw_exception_class>[a-zA-Z0-9_.]+Exception|[a-zA-Z0-9_.]+Error):\s*(?<raw_exception_message>[^\n]+)"
      }
      tag_on_failure => ["_grok_exception_parse_failure"]
    }
  }

  # --- FASE 3: Parsear el campo de contexto JSON ---
  if [raw_context] {
    json {
      source => "raw_context"
      target => "event_context"
      tag_on_failure => ["_jsonparsefailure"]
    }
  }

  # --- FASE 4: Normalizar timestamp ---
  date {
    match => ["raw_timestamp", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
    timezone => "UTC"
    tag_on_failure => ["_dateparsefailure"]
  }

  # --- FASE 5: Enriquecimiento GeoIP ---
  if [raw_client_ip] {
    geoip {
      source => "raw_client_ip"
      target => "source.geo"
      tag_on_failure => ["_geoip_lookup_failure"]
    }
  }

  # --- FASE 6: Traducción de códigos de error ---
  if [raw_error_code] {
    translate {
      field => "raw_error_code"
      destination => "error.description"
      dictionary_path => "/usr/share/logstash/config/dictionaries/error_codes.csv"
      fallback => "Unknown error code"
    }
  }

  # --- FASE 7: Mapeo a ECS y limpieza ---
  mutate {
    rename => {
      "raw_level" => "[log][level]"
      "raw_logger" => "[log][logger]"
      "raw_client_ip" => "[source][ip]"
      "raw_error_code" => "[error][code]"
      "raw_message" => "[message_parsed]"
    }
  }

  # Mapear excepción a campos ECS de error
  if [raw_exception_class] {
    mutate {
      rename => {
        "raw_exception_class" => "[error][type]"
        "raw_exception_message" => "[error][message]"
      }
    }
    # Extraer stack trace completo para ECS
    if [raw_stack_trace] {
      mutate {
        rename => {
          "raw_stack_trace" => "[error][stack_trace]"
        }
      }
    }
  }

  # Normalizar log.level a minúsculas para consistencia ECS
  if [log][level] {
    mutate {
      lowercase => ["[log][level]"]
    }
  }

  # Añadir metadatos del pipeline
  mutate {
    add_field => {
      "[event][dataset]" => "java_app.logs"
      "[event][module]" => "custom"
      "[@metadata][target_index]" => "labs-logstash-app-%{+YYYY.MM.dd}"
    }
  }

  # Limpiar campos temporales
  mutate {
    remove_field => ["raw_timestamp", "raw_first_line", "raw_context", "host", "path"]
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    index => "%{[@metadata][target_index]}"
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_enabled => true
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca/ca.crt"]
    template_name => "labs-logstash-template"
    manage_template => false
  }

  # Output de depuración - descomentar para troubleshooting
  # stdout { codec => rubydebug }
}
PIPELINE
```

3. Verifica la sintaxis del archivo:

```bash
grep -c "^}" ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf
```

**Resultado esperado:** `3` (cierre de los bloques input, filter y output).

**Verificación:** El archivo contiene las tres secciones principales (input, filter, output) con todos los plugins configurados.

---

### Paso 4: Configurar Logstash con Dead Letter Queue habilitada

**Objetivo:** Crear la configuración de Logstash que habilita la DLQ para capturar eventos que fallan en el output.

**Instrucciones:**

1. Crea el archivo `logstash.yml`:

```bash
cat > ~/elastic-labs/config/logstash/logstash.yml << 'EOF'
# Configuración principal de Logstash
http.host: "0.0.0.0"
http.port: 9600

# Dead Letter Queue - captura eventos que fallan en el output
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1024mb
dead_letter_queue.storage_policy: drop_newer
dead_letter_queue.retain.age: 7d

# Configuración de pipeline
pipeline.workers: 2
pipeline.batch.size: 125
pipeline.batch.delay: 50

# Logging
log.level: info

# Monitoreo
xpack.monitoring.enabled: false
EOF
```

2. Crea el archivo `pipelines.yml`:

```bash
cat > ~/elastic-labs/config/logstash/pipelines.yml << 'EOF'
- pipeline.id: labs-app
  path.config: "/usr/share/logstash/pipeline/labs-app-pipeline.conf"
  pipeline.workers: 2
  dead_letter_queue.enable: true
EOF
```

**Resultado esperado:** Dos archivos de configuración creados sin errores.

---

### Paso 5: Configurar y lanzar el contenedor de Logstash

**Objetivo:** Crear el servicio de Logstash en Docker Compose y ejecutarlo.

**Instrucciones:**

1. Crea un archivo Docker Compose específico para Logstash (para no alterar el compose principal):

```bash
cat > ~/elastic-labs/docker-compose-logstash.yml << 'EOF'
version: "3.8"

services:
  logstash01:
    image: docker.elastic.co/logstash/logstash:8.14.1
    container_name: logstash01
    hostname: logstash01
    environment:
      - XPACK_MONITORING_ENABLED=false
      - LS_JAVA_OPTS=-Xms1g -Xmx1g
    volumes:
      - ./config/logstash/pipeline/labs-app-pipeline.conf:/usr/share/logstash/pipeline/labs-app-pipeline.conf:ro
      - ./config/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./config/logstash/pipelines.yml:/usr/share/logstash/config/pipelines.yml:ro
      - ./config/logstash/dictionaries:/usr/share/logstash/config/dictionaries:ro
      - ./logs/logstash-input/java_exception.log:/var/log/app/java_exception.log:ro
      - ./config/certs:/usr/share/logstash/config/certs:ro
      - logstash_data:/usr/share/logstash/data
    networks:
      - elastic-net
    depends_on:
      - es01
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9600/_node/stats | grep -q '\"status\":\"green\"' || curl -s http://localhost:9600 | grep -q 'logstash'"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  logstash_data:
    driver: local

networks:
  elastic-net:
    external: true
EOF
```

2. Verifica que la red Docker `elastic-net` existe:

```bash
docker network ls | grep elastic-net
```

Si no existe, créala:

```bash
docker network create --subnet=172.20.0.0/24 elastic-net
```

3. Verifica que los certificados están disponibles:

```bash
ls ~/elastic-labs/config/certs/ca/ca.crt
```

Si el archivo no existe, copia los certificados del contenedor de Elasticsearch:

```bash
mkdir -p ~/elastic-labs/config/certs/ca
docker cp es01:/usr/share/elasticsearch/config/certs/ca/ca.crt ~/elastic-labs/config/certs/ca/ca.crt
```

4. Lanza el contenedor de Logstash:

```bash
cd ~/elastic-labs
docker compose -f docker-compose-logstash.yml up -d logstash01
```

5. Observa los logs en tiempo real (espera ~30 segundos para que Logstash inicie):

```bash
docker logs -f logstash01 2>&1 | head -80
```

**Resultado esperado:** Después de 20-40 segundos verás mensajes como:
```
[INFO ] Pipelines running {:count=>1, :running_pipelines=>[:labs-app], :non_running_pipelines=>[]}
[INFO ] Successfully started Logstash API endpoint {:port=>9600, :ssl_enabled=>false}
```

**Verificación:**

```bash
docker exec logstash01 curl -s http://localhost:9600/_node/pipelines?pretty | jq '.pipelines["labs-app"].status'
```

Debe devolver `"running"`.

---

### Paso 6: Verificar la indexación en Elasticsearch

**Objetivo:** Confirmar que los eventos se procesaron correctamente y se indexaron en Elasticsearch.

**Instrucciones:**

1. Espera 15 segundos tras ver el pipeline activo, luego verifica los índices:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-logstash-app-*?v&h=index,docs.count,store.size"
```

**Resultado esperado:**
```
index                           docs.count store.size
labs-logstash-app-2024.06.15          7       45kb
```

2. Consulta un documento para verificar la estructura ECS:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search?size=1&q=log.level:error" | jq '.hits.hits[0]._source | {log, error, source, event_context, "event.dataset": .event.dataset}'
```

**Resultado esperado:** Un documento con campos ECS correctamente mapeados:
```json
{
  "log": {
    "level": "error",
    "logger": "com.labs.service.PaymentService"
  },
  "error": {
    "code": "PAY_001",
    "type": "java.lang.NullPointerException",
    "message": "Cannot invoke method on null object",
    "description": "Payment gateway timeout or rejection",
    "stack_trace": "..."
  },
  "source": {
    "ip": "203.0.113.45",
    "geo": { ... }
  },
  "event_context": {
    "transaction_id": "txn-7890",
    "amount": 150.0,
    "currency": "USD"
  },
  "event.dataset": "java_app.logs"
}
```

3. Verifica el conteo por nivel de log:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search" -H 'Content-Type: application/json' -d '{
  "size": 0,
  "aggs": {
    "by_level": {
      "terms": { "field": "log.level" }
    }
  }
}' | jq '.aggregations.by_level.buckets'
```

**Resultado esperado:**
```json
[
  { "key": "error", "doc_count": 4 },
  { "key": "warn", "doc_count": 2 },
  { "key": "info", "doc_count": 1 }
]
```

**Verificación:** 7 documentos indexados con 4 errores, 2 warnings y 1 info.

---

### Paso 7: Introducir un error intencional y depurar con logs de Logstash

**Objetivo:** Practicar la depuración de errores de parsing modificando el pipeline para provocar un fallo y corregirlo.

**Instrucciones:**

1. Detén Logstash:

```bash
docker compose -f docker-compose-logstash.yml down logstash01
```

2. Elimina el índice existente para reprocesar:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-logstash-app-*"
```

3. Introduce un error en el patrón grok (cambia `TIMESTAMP_ISO8601` por un patrón incorrecto):

```bash
sed -i 's/TIMESTAMP_ISO8601:raw_timestamp/TIMESTAMP_ISO8601_INVALID:raw_timestamp/' \
  ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf
```

4. Relanza Logstash:

```bash
docker compose -f docker-compose-logstash.yml up -d logstash01
```

5. Observa los logs para detectar el error:

```bash
sleep 30
docker logs logstash01 2>&1 | grep -i "error\|failure\|_grokparsefailure" | tail -20
```

**Resultado esperado:** Verás mensajes indicando que el patrón no se puede compilar o que los eventos reciben la etiqueta `_grokparsefailure`:
```
[ERROR] ... Unknown pattern: TIMESTAMP_ISO8601_INVALID
```
o bien el pipeline no arranca y muestra un error de configuración.

6. Corrige el error:

```bash
sed -i 's/TIMESTAMP_ISO8601_INVALID:raw_timestamp/TIMESTAMP_ISO8601:raw_timestamp/' \
  ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf
```

7. Reinicia Logstash:

```bash
docker compose -f docker-compose-logstash.yml restart logstash01
```

8. Verifica que los eventos se procesan correctamente:

```bash
sleep 30
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-logstash-app-*?v&h=index,docs.count"
```

**Resultado esperado:** El índice debe mostrar 7 documentos nuevamente.

**Verificación:** El pipeline se recupera tras corregir el patrón grok y los documentos se indexan sin tags de error.

---

### Paso 8: Probar la Dead Letter Queue con un error de mapping

**Objetivo:** Provocar un error en el output que active la DLQ y verificar que los eventos fallidos se capturan.

**Instrucciones:**

1. Detén Logstash y elimina los índices:

```bash
docker compose -f docker-compose-logstash.yml down logstash01
curl -s -k -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-logstash-app-*"
```

2. Crea un mapping conflictivo en el índice destino (forzar que `log.level` sea de tipo `integer` cuando Logstash enviará un string):

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/labs-logstash-app-2024.06.15" \
  -H 'Content-Type: application/json' -d '{
  "mappings": {
    "properties": {
      "log": {
        "properties": {
          "level": { "type": "integer" }
        }
      }
    }
  }
}'
```

3. Relanza Logstash:

```bash
docker compose -f docker-compose-logstash.yml up -d logstash01
```

4. Espera 40 segundos y verifica los logs de error:

```bash
sleep 40
docker logs logstash01 2>&1 | grep -i "mapper_parsing_exception\|DLQ\|dead_letter" | tail -10
```

**Resultado esperado:** Mensajes indicando errores de mapping y eventos enviados a la DLQ:
```
[WARN ] ... response=>{"index"=>{"error"=>{"type"=>"mapper_parsing_exception"...}}}
```

5. Verifica que la DLQ tiene eventos:

```bash
docker exec logstash01 find /usr/share/logstash/data/dead_letter_queue -name "*.log" -exec ls -la {} \;
```

**Resultado esperado:** Uno o más archivos `.log` en el directorio de la DLQ con tamaño > 0.

6. Limpia el índice con mapping incorrecto y recrea correctamente:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-logstash-app-2024.06.15"
```

7. Reinicia Logstash para reprocesar (como usamos `sincedb_path => "/dev/null"`, releerá el archivo):

```bash
docker compose -f docker-compose-logstash.yml restart logstash01
```

8. Verifica la indexación correcta:

```bash
sleep 30
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_count" | jq '.count'
```

**Resultado esperado:** `7`

**Verificación:** La DLQ capturó los eventos fallidos y tras corregir el mapping, el reprocesamiento fue exitoso.

---

### Paso 9: Validar campos ECS y enriquecimiento completo en Kibana

**Objetivo:** Verificar visualmente en Kibana que todos los campos están correctamente mapeados y enriquecidos.

**Instrucciones:**

1. Accede a Kibana en `https://localhost:5601` con las credenciales `elastic` / `ElasticLabs2024!`.

2. Navega a **Management → Stack Management → Data Views** y crea un nuevo Data View:
   - Name: `labs-logstash-app-*`
   - Index pattern: `labs-logstash-app-*`
   - Timestamp field: `@timestamp`

3. Navega a **Discover** y selecciona el data view `labs-logstash-app-*`.

4. Verifica que los siguientes campos existen en la lista de campos disponibles:
   - `log.level`
   - `log.logger`
   - `error.code`
   - `error.type`
   - `error.message`
   - `error.description`
   - `error.stack_trace`
   - `source.ip`
   - `source.geo.country_name` (puede estar vacío para IPs privadas/reservadas)
   - `event_context.transaction_id`
   - `event.dataset`

5. Filtra por `log.level: error` y expande un documento para verificar que el stack trace se capturó completo.

6. Verifica el campo `error.description` traducido desde el diccionario CSV.

**Resultado esperado:** Los 7 documentos aparecen con todos los campos ECS correctamente poblados. Los eventos ERROR tienen `error.stack_trace` con las líneas de la excepción Java.

**Verificación mediante API:**

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search" -H 'Content-Type: application/json' -d '{
  "size": 1,
  "query": { "term": { "error.code": "PAY_001" } },
  "_source": ["log.level", "log.logger", "error", "source.ip", "event.dataset", "message_parsed"]
}' | jq '.hits.hits[0]._source'
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que la práctica se completó exitosamente:

### Test 1: Conteo de documentos

```bash
DOCS=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_count" | jq '.count')
echo "Documentos indexados: $DOCS"
[ "$DOCS" -eq 7 ] && echo "✅ PASS: 7 documentos indexados" || echo "❌ FAIL: Se esperaban 7 documentos"
```

### Test 2: Campos ECS presentes

```bash
FIELDS=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_mapping" | jq '[.. | objects | keys[]] | unique | map(select(. == "level" or . == "logger" or . == "stack_trace" or . == "dataset")) | length')
echo "Campos ECS encontrados: $FIELDS"
[ "$FIELDS" -ge 4 ] && echo "✅ PASS: Campos ECS presentes" || echo "❌ FAIL: Faltan campos ECS"
```

### Test 3: Plugin translate funcionando

```bash
TRANSLATED=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search?q=error.description:*gateway*" | jq '.hits.total.value')
echo "Documentos con traducción 'gateway': $TRANSLATED"
[ "$TRANSLATED" -ge 1 ] && echo "✅ PASS: Plugin translate funcional" || echo "❌ FAIL: Translate no aplicó"
```

### Test 4: Multiline correcto (stack traces agrupados)

```bash
STACK=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search" -H 'Content-Type: application/json' -d '{
  "size": 1,
  "query": { "exists": { "field": "error.stack_trace" } },
  "_source": ["error.stack_trace"]
}' | jq -r '.hits.hits[0]._source.error.stack_trace' | grep -c "at ")
echo "Líneas 'at' en stack_trace: $STACK"
[ "$STACK" -ge 2 ] && echo "✅ PASS: Multiline agrupó stack traces" || echo "❌ FAIL: Stack traces no agrupados"
```

### Test 5: Pipeline de Logstash activo

```bash
STATUS=$(docker exec logstash01 curl -s http://localhost:9600/_node/pipelines | jq -r '.pipelines["labs-app"].status')
echo "Estado del pipeline: $STATUS"
[ "$STATUS" = "running" ] && echo "✅ PASS: Pipeline activo" || echo "❌ FAIL: Pipeline no está running"
```

---

## Solución de Problemas

### Problema 1: Logstash no arranca — error "Unknown pattern" en grok

**Síntomas:**
- El contenedor se reinicia continuamente o se detiene inmediatamente
- Los logs muestran: `[ERROR] Configuration has a syntax error... Unknown pattern: XXXX`
- `docker ps` muestra el contenedor en estado `Restarting`

**Causa:** Se ha utilizado un nombre de patrón grok que no existe en la biblioteca estándar de Logstash. Los nombres de patrones son case-sensitive y deben coincidir exactamente con los definidos en `/usr/share/logstash/vendor/bundle/jruby/*/gems/logstash-patterns-core-*/patterns/`.

**Solución:**

```bash
# 1. Verificar el error exacto en los logs
docker logs logstash01 2>&1 | grep -i "unknown pattern\|syntax error"

# 2. Listar patrones disponibles para referencia
docker exec logstash01 find /usr/share/logstash/vendor/bundle -name "grok-patterns" -exec head -50 {} \;

# 3. Corregir el patrón en el archivo de configuración
# Los patrones comunes correctos son: TIMESTAMP_ISO8601, LOGLEVEL, JAVACLASS, IP, GREEDYDATA, DATA, WORD
nano ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf

# 4. Reiniciar Logstash
docker compose -f docker-compose-logstash.yml restart logstash01
```

---

### Problema 2: Documentos indexados con tag `_grokparsefailure` — campos no extraídos

**Síntomas:**
- Los documentos aparecen en Elasticsearch pero solo contienen `message` y `tags: ["_grokparsefailure"]`
- Los campos `log.level`, `log.logger`, `source.ip` no existen en los documentos
- En Kibana Discover, el campo `message` muestra el log completo sin parsear

**Causa:** El patrón grok no coincide con el formato real del log. Esto ocurre frecuentemente cuando el codec multiline altera el contenido del campo `message` (por ejemplo, añadiendo `\n` entre líneas) o cuando hay diferencias sutiles en espaciado/separadores.

**Solución:**

```bash
# 1. Habilitar stdout para ver el contenido real del campo message
# Editar el pipeline y descomentar el output stdout:
sed -i 's/# stdout { codec => rubydebug }/stdout { codec => rubydebug }/' \
  ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf

# 2. Reiniciar y observar la salida
docker compose -f docker-compose-logstash.yml restart logstash01
sleep 30
docker logs logstash01 2>&1 | grep -A 5 '"message"'

# 3. Copiar el valor exacto del campo message y probarlo en el Grok Debugger de Kibana:
#    Kibana → Dev Tools → Grok Debugger
#    Pegar el mensaje y ajustar el patrón hasta que coincida

# 4. Problema común: el multiline añade \n literal que debe manejarse
#    Solución: anclar el patrón grok al inicio con ^ y usar (?m) si es necesario
#    O asegurarse de que el patrón grok solo procese la primera línea

# 5. Una vez corregido, volver a comentar stdout y reiniciar
sed -i 's/stdout { codec => rubydebug }/# stdout { codec => rubydebug }/' \
  ~/elastic-labs/config/logstash/pipeline/labs-app-pipeline.conf
docker compose -f docker-compose-logstash.yml restart logstash01
```

---

## Limpieza

Ejecuta los siguientes comandos para liberar recursos al finalizar la práctica:

```bash
# Detener el contenedor de Logstash
cd ~/elastic-labs
docker compose -f docker-compose-logstash.yml down

# Eliminar el volumen de datos de Logstash (incluye DLQ)
docker volume rm elastic-labs_logstash_data 2>/dev/null || true

# (OPCIONAL) Eliminar los índices de esta práctica si no necesitas los datos para Práctica 5
# NOTA: NO ejecutar si vas a continuar con Práctica 5
# curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/labs-logstash-app-*"

# Los archivos de configuración se mantienen en ~/elastic-labs/config/logstash/
# para referencia y uso en prácticas posteriores
echo "Limpieza completada. Archivos de configuración preservados en ~/elastic-labs/config/logstash/"
```

> **⚠️ Importante:** No elimines los índices `labs-logstash-app-*` si planeas continuar con la Práctica 5, ya que estos datos serán la fuente del dashboard operativo.

---

## Resumen

En esta práctica has construido un pipeline de Logstash completo que:

1. **Lee logs multiline** de una aplicación Java usando el codec multiline para agrupar stack traces con sus eventos padre
2. **Extrae campos estructurados** mediante grok (timestamp, nivel, logger, mensaje, IP, código de error, contexto JSON)
3. **Parsea JSON embebido** dentro del campo de contexto usando el plugin json
4. **Normaliza el timestamp** con el plugin date para establecer `@timestamp` correctamente
5. **Enriquece con geolocalización** las IPs de cliente mediante geoip
6. **Traduce códigos de error** a descripciones legibles usando un diccionario CSV con translate
7. **Mapea al estándar ECS** renombrando campos a `log.level`, `log.logger`, `error.type`, `error.message`, `error.stack_trace`, `source.ip`
8. **Envía a Elasticsearch** con el template e ILM configurados en Práctica 2
9. **Captura eventos fallidos** en la Dead Letter Queue para diagnóstico

### Conceptos Clave Reforzados

- La arquitectura input → filter → output de Logstash permite construir transformaciones complejas de manera modular
- El codec multiline es esencial para logs Java con excepciones (el patrón `^TIMESTAMP` con `negate => true` y `what => previous` agrupa líneas de stack trace con el evento anterior)
- La DLQ es una herramienta fundamental de depuración para errores que ocurren en la fase de output (mapping conflicts, timeouts)
- Los tags de fallo (`_grokparsefailure`, `_dateparsefailure`, `_jsonparsefailure`) son el primer indicador de problemas en los filtros

### Recursos Adicionales

- [Documentación del codec multiline](https://www.elastic.co/guide/en/logstash/current/plugins-codecs-multiline.html)
- [Referencia de patrones grok predefinidos](https://github.com/logstash-plugins/logstash-patterns-core/tree/main/patterns)
- [Grok Debugger en Kibana](https://www.elastic.co/guide/en/kibana/current/xpack-grokdebugger.html)
- [Configuración de Dead Letter Queue](https://www.elastic.co/guide/en/logstash/current/dead-letter-queues.html)
- [Elastic Common Schema (ECS) - Field Reference](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html)
