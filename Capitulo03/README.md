# Incorporar una fuente de logs mediante Elastic Agent o Filebeat y validar su recepción

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |

## Descripción General

En esta práctica se implementan dos métodos de recolección de logs en paralelo — Elastic Agent 8.14.1 (administrado mediante Fleet) y Filebeat 7.17.22 (configuración legacy) — para el mismo archivo de log de aplicación. El estudiante enrollará un Elastic Agent en Fleet Server, configurará un contenedor Filebeat, y comparará la estructura de los documentos indexados por ambos agentes en Kibana Discover, evaluando diferencias en campos ECS, metadatos y experiencia de administración.

## Objetivos de Aprendizaje

- [ ] Instalar y enrollar un Elastic Agent 8.14.1 en el host Ubuntu mediante Fleet Server, asignándole una política con la integración Custom Logs
- [ ] Configurar un contenedor Filebeat 7.17.22 para recolectar logs y enviarlos directamente a Elasticsearch
- [ ] Validar la recepción correcta de logs de ambos agentes mediante consultas a la API de Elasticsearch
- [ ] Comparar la estructura de documentos indexados por cada agente identificando diferencias en campos ECS y metadatos

## Prerrequisitos

### Conocimientos Previos

- Comprensión de la arquitectura Fleet (Fleet UI → Fleet Server → Elastic Agent)
- Familiaridad con políticas de agente, integraciones y enrollment tokens
- Experiencia básica con Docker y docker-compose
- Práctica 2 completada: política ILM `labs-policy` y templates `labs-app-template` y `labs-logstash-template` creados

### Acceso Requerido

- Stack Elastic completo en ejecución (contenedores `es01`, `kibana01`, `fleet-server01`)
- Acceso sudo en el host Ubuntu 22.04
- Acceso a Kibana en `https://localhost:5601` con credenciales `elastic` / `ElasticLabs2024!`
- Python 3.12.3 disponible en el host

## Entorno del Laboratorio

### Servicios en Ejecución

| Servicio | Contenedor | Puerto Host | Propósito |
|----------|-----------|-------------|-----------|
| Elasticsearch | es01 | 9200 | Almacenamiento de índices |
| Kibana | kibana01 | 5601 | Fleet UI, Discover |
| Fleet Server | fleet-server01 | 8220 | Control de agentes |
| Filebeat | filebeat-legacy01 | — | Recolección legacy |

### Archivos Clave

| Archivo | Ubicación | Propósito |
|---------|-----------|-----------|
| Script generador de logs | `~/elastic-labs/scripts/app_log_generator.py` | Genera logs JSON estructurados |
| Archivo de log generado | `~/elastic-labs/logs/app_structured.json` | Fuente de datos para ambos agentes |
| Configuración Filebeat | `~/elastic-labs/config/filebeat-legacy.yml` | Input y output de Filebeat |
| Certificados TLS | `~/elastic-labs/config/certs/` | Comunicación segura con Elasticsearch |

### Verificación Inicial del Entorno

```bash
# Verificar que todos los contenedores están en ejecución
cd ~/elastic-labs
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Verificar conectividad con Elasticsearch
curl -s -k -u elastic:ElasticLabs2024! https://localhost:9200/_cluster/health | jq '.status'

# Verificar que Fleet Server está accesible
curl -s -k https://localhost:8220/api/status | jq '.status'
```

**Salida esperada:**

```
"green"
```

```
"HEALTHY"
```

---

## Paso 1: Generar Logs de Aplicación Sintéticos

### Objetivo

Crear el archivo de log estructurado que servirá como fuente de datos común para Elastic Agent y Filebeat.

### Instrucciones

1. Crear el script generador de logs:

```bash
mkdir -p ~/elastic-labs/scripts ~/elastic-labs/logs

cat > ~/elastic-labs/scripts/app_log_generator.py << 'EOF'
#!/usr/bin/env python3
"""Generador de logs JSON estructurados para laboratorio Elastic Stack."""

import json
import random
import time
from datetime import datetime, timezone

LEVELS = ["INFO", "WARN", "ERROR", "DEBUG"]
SERVICES = ["auth-service", "payment-service", "inventory-service", "api-gateway"]
MESSAGES = {
    "INFO": [
        "Request processed successfully",
        "User session started",
        "Cache refreshed",
        "Health check passed"
    ],
    "WARN": [
        "Response time exceeded threshold",
        "Connection pool near capacity",
        "Retry attempt initiated",
        "Deprecated API endpoint called"
    ],
    "ERROR": [
        "Database connection timeout",
        "Authentication failed for user",
        "Payment gateway unreachable",
        "Out of memory exception caught"
    ],
    "DEBUG": [
        "Entering method processOrder",
        "Query execution plan generated",
        "Thread pool stats collected",
        "Configuration reload triggered"
    ]
}

LOG_FILE = "/home/{user}/elastic-labs/logs/app_structured.json".format(
    user=__import__('os').environ.get('USER', 'ubuntu')
)

def generate_log_entry():
    level = random.choices(LEVELS, weights=[50, 25, 15, 10])[0]
    service = random.choice(SERVICES)
    message = random.choice(MESSAGES[level])
    
    entry = {
        "@timestamp": datetime.now(timezone.utc).isoformat(),
        "log.level": level,
        "service.name": service,
        "message": message,
        "event.duration": random.randint(1000, 50000),
        "http.response.status_code": random.choice([200, 201, 400, 401, 500, 503]) if level != "DEBUG" else None,
        "source.ip": f"10.0.{random.randint(1,10)}.{random.randint(1,254)}",
        "trace.id": f"{random.randint(100000,999999):06x}{random.randint(100000,999999):06x}"
    }
    # Eliminar campos nulos
    return {k: v for k, v in entry.items() if v is not None}

def main():
    print(f"Generando logs en: {LOG_FILE}")
    print("Presiona Ctrl+C para detener")
    
    count = 0
    with open(LOG_FILE, "a") as f:
        while True:
            entry = generate_log_entry()
            f.write(json.dumps(entry) + "\n")
            f.flush()
            count += 1
            if count % 10 == 0:
                print(f"  Logs generados: {count}", end="\r")
            time.sleep(random.uniform(0.5, 2.0))

if __name__ == "__main__":
    main()
EOF

chmod +x ~/elastic-labs/scripts/app_log_generator.py
```

2. Generar un lote inicial de logs (50 entradas) para tener datos disponibles inmediatamente:

```bash
cat > ~/elastic-labs/scripts/generate_batch.py << 'EOF'
#!/usr/bin/env python3
"""Genera un lote inicial de logs para el laboratorio."""

import json
import random
import os
from datetime import datetime, timezone, timedelta

LEVELS = ["INFO", "WARN", "ERROR", "DEBUG"]
SERVICES = ["auth-service", "payment-service", "inventory-service", "api-gateway"]
MESSAGES = {
    "INFO": ["Request processed successfully", "User session started", "Cache refreshed"],
    "WARN": ["Response time exceeded threshold", "Connection pool near capacity"],
    "ERROR": ["Database connection timeout", "Authentication failed for user"],
    "DEBUG": ["Entering method processOrder", "Query execution plan generated"]
}

LOG_FILE = os.path.expanduser("~/elastic-labs/logs/app_structured.json")

# Limpiar archivo existente
open(LOG_FILE, 'w').close()

base_time = datetime.now(timezone.utc) - timedelta(minutes=30)

with open(LOG_FILE, "a") as f:
    for i in range(50):
        level = random.choices(LEVELS, weights=[50, 25, 15, 10])[0]
        service = random.choice(SERVICES)
        message = random.choice(MESSAGES[level])
        ts = base_time + timedelta(seconds=i * 30)
        
        entry = {
            "@timestamp": ts.isoformat(),
            "log.level": level,
            "service.name": service,
            "message": message,
            "event.duration": random.randint(1000, 50000),
            "http.response.status_code": random.choice([200, 201, 400, 401, 500, 503]),
            "source.ip": f"10.0.{random.randint(1,10)}.{random.randint(1,254)}",
            "trace.id": f"{random.randint(100000,999999):06x}{random.randint(100000,999999):06x}"
        }
        f.write(json.dumps(entry) + "\n")

print(f"Generados 50 logs en {LOG_FILE}")
EOF

python3 ~/elastic-labs/scripts/generate_batch.py
```

3. Iniciar el generador continuo en segundo plano:

```bash
nohup python3 ~/elastic-labs/scripts/app_log_generator.py > /dev/null 2>&1 &
echo $! > ~/elastic-labs/logs/generator.pid
echo "Generador iniciado con PID: $(cat ~/elastic-labs/logs/generator.pid)"
```

### Salida Esperada

```
Generados 50 logs en /home/ubuntu/elastic-labs/logs/app_structured.json
Generador iniciado con PID: 12345
```

### Verificación

```bash
# Verificar que el archivo tiene contenido
wc -l ~/elastic-labs/logs/app_structured.json

# Verificar estructura JSON válida
head -3 ~/elastic-labs/logs/app_structured.json | jq .
```

Debe mostrar al menos 50 líneas y cada línea debe ser JSON válido con campos como `@timestamp`, `log.level`, `service.name` y `message`.

---

## Paso 2: Crear la Política de Agente en Fleet

### Objetivo

Crear una política de agente denominada `labs-agent-policy` en Fleet UI con la integración Custom Logs configurada para recolectar el archivo `app_structured.json`.

### Instrucciones

1. Acceder a Kibana Fleet UI:

```bash
# Abrir en navegador (o verificar acceso con curl)
echo "Acceder a: https://localhost:5601/app/fleet"
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:5601/api/fleet/agent_policies | jq '.items | length'
```

2. Crear la política de agente mediante la API de Fleet:

```bash
# Crear la política 'labs-agent-policy'
curl -s -k -X POST \
  -u elastic:ElasticLabs2024! \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/agent_policies" \
  -d '{
    "name": "labs-agent-policy",
    "description": "Política para laboratorio - recolección de logs de aplicación",
    "namespace": "default",
    "monitoring_enabled": ["logs", "metrics"]
  }' | jq '{id: .item.id, name: .item.name, status: .item.status}'
```

3. Guardar el ID de la política para uso posterior:

```bash
# Obtener el ID de la política recién creada
POLICY_ID=$(curl -s -k -u elastic:ElasticLabs2024! \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/agent_policies" | \
  jq -r '.items[] | select(.name=="labs-agent-policy") | .id')

echo "Policy ID: $POLICY_ID"
```

4. Añadir la integración Custom Logs a la política:

```bash
# Añadir integración Custom Logs
curl -s -k -X POST \
  -u elastic:ElasticLabs2024! \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/package_policies" \
  -d "{
    \"name\": \"custom-app-logs\",
    \"description\": \"Recolección de logs de aplicación estructurados\",
    \"namespace\": \"default\",
    \"policy_id\": \"$POLICY_ID\",
    \"enabled\": true,
    \"inputs\": [
      {
        \"type\": \"logfile\",
        \"enabled\": true,
        \"streams\": [
          {
            \"enabled\": true,
            \"data_stream\": {
              \"type\": \"logs\",
              \"dataset\": \"custom-labs.app\"
            },
            \"vars\": {
              \"paths\": {
                \"value\": [\"/var/log/app/app_structured.json\"],
                \"type\": \"text\"
              },
              \"tags\": {
                \"value\": [\"labs\", \"app-logs\", \"elastic-agent\"],
                \"type\": \"text\"
              },
              \"processors\": {
                \"value\": \"- decode_json_fields:\\n    fields: [\\\"message\\\"]\\n    target: \\\"\\\"\\n    overwrite_keys: true\",
                \"type\": \"yaml\"
              }
            }
          }
        ]
      }
    ],
    \"package\": {
      \"name\": \"log\",
      \"title\": \"Custom Logs\",
      \"version\": \"2.3.2\"
    }
  }" | jq '{id: .item.id, name: .item.name, policy_id: .item.policy_id}'
```

> **Nota:** La ruta `/var/log/app/app_structured.json` se mapea al archivo del host `~/elastic-labs/logs/app_structured.json` durante la instalación del agente. Si se instala directamente en el host, se usará la ruta real del archivo.

### Salida Esperada

```json
{
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "name": "labs-agent-policy",
  "status": "active"
}
```

```json
{
  "id": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy",
  "name": "custom-app-logs",
  "policy_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

### Verificación

```bash
# Verificar que la política tiene la integración añadida
curl -s -k -u elastic:ElasticLabs2024! \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/agent_policies/$POLICY_ID" | \
  jq '{name: .item.name, package_policies: [.item.package_policies[].name]}'
```

Debe mostrar `"custom-app-logs"` en la lista de package_policies.

---

## Paso 3: Instalar y Enrollar Elastic Agent en el Host

### Objetivo

Instalar Elastic Agent 8.14.1 en el sistema host Ubuntu y registrarlo en Fleet Server usando el enrollment token asociado a `labs-agent-policy`.

### Instrucciones

1. Obtener el enrollment token para la política:

```bash
# Obtener el enrollment token asociado a labs-agent-policy
ENROLLMENT_TOKEN=$(curl -s -k -u elastic:ElasticLabs2024! \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/enrollment_api_keys" | \
  jq -r ".items[] | select(.policy_id==\"$POLICY_ID\") | .api_key")

echo "Enrollment Token: $ENROLLMENT_TOKEN"
```

Si no existe un token para la política, crear uno:

```bash
# Crear enrollment token si no existe
if [ -z "$ENROLLMENT_TOKEN" ]; then
  ENROLLMENT_TOKEN=$(curl -s -k -X POST \
    -u elastic:ElasticLabs2024! \
    -H "Content-Type: application/json" \
    -H "kbn-xsrf: true" \
    "https://localhost:5601/api/fleet/enrollment_api_keys" \
    -d "{\"policy_id\": \"$POLICY_ID\"}" | jq -r '.item.api_key')
  echo "Token creado: $ENROLLMENT_TOKEN"
fi
```

2. Descargar e instalar Elastic Agent 8.14.1:

```bash
cd /tmp

# Descargar el paquete .deb de Elastic Agent 8.14.1
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.14.1-amd64.deb

# Instalar el paquete
sudo dpkg -i elastic-agent-8.14.1-amd64.deb
```

3. Crear un enlace simbólico para que el agente pueda acceder al archivo de log:

```bash
# Crear directorio y enlace simbólico para los logs de la app
sudo mkdir -p /var/log/app
sudo ln -sf ~/elastic-labs/logs/app_structured.json /var/log/app/app_structured.json
sudo chmod 644 ~/elastic-labs/logs/app_structured.json
```

4. Obtener el certificado CA del Fleet Server para la conexión TLS:

```bash
# Copiar el certificado CA del contenedor de Elasticsearch
docker cp es01:/usr/share/elasticsearch/config/certs/ca/ca.crt \
  ~/elastic-labs/config/certs/ca.crt

# Verificar el certificado
openssl x509 -in ~/elastic-labs/config/certs/ca.crt -text -noout | head -5
```

5. Enrollar el agente en Fleet Server:

```bash
# Enrollar Elastic Agent en Fleet Server
sudo elastic-agent enroll \
  --url=https://localhost:8220 \
  --enrollment-token="$ENROLLMENT_TOKEN" \
  --certificate-authorities=/home/$USER/elastic-labs/config/certs/ca.crt \
  --insecure
```

> **Nota:** La flag `--insecure` se usa porque los certificados son autogenerados. En producción, se usarían certificados firmados por una CA reconocida.

6. Iniciar el servicio de Elastic Agent:

```bash
# Habilitar e iniciar el servicio
sudo systemctl enable elastic-agent
sudo systemctl start elastic-agent

# Esperar 10 segundos para que el agente se estabilice
sleep 10
```

### Salida Esperada

```
Successfully enrolled the Elastic Agent.
```

### Verificación

```bash
# Verificar estado del agente
sudo elastic-agent status

# Verificar que el agente aparece en Fleet
curl -s -k -u elastic:ElasticLabs2024! \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/agents" | \
  jq '.items[] | {id: .id, status: .status, policy_name: .policy_id, local_metadata_host: .local_metadata.host.hostname}'
```

El estado debe ser `HEALTHY` y el agente debe aparecer asociado a `labs-agent-policy`.

---

## Paso 4: Configurar Filebeat 7.17.22 como Recolector Legacy

### Objetivo

Configurar el contenedor Filebeat 7.17.22 para recolectar el mismo archivo de log y enviarlo a Elasticsearch al índice `filebeat-7.17.22-labs-app`.

### Instrucciones

1. Crear el archivo de configuración de Filebeat:

```bash
cat > ~/elastic-labs/config/filebeat-legacy.yml << 'EOF'
# Filebeat 7.17.22 - Configuración Legacy para laboratorio
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/app_structured.json
    json.keys_under_root: true
    json.overwrite_keys: true
    json.add_error_key: true
    json.expand_keys: true
    tags: ["labs", "app-logs", "filebeat-legacy"]
    fields:
      lab.collector: "filebeat"
      lab.version: "7.17.22"
    fields_under_root: false

# Procesadores para enriquecer eventos
processors:
  - add_host_metadata:
      when.not.contains.tags: "forwarded"
  - add_docker_metadata: ~
  - timestamp:
      field: "@timestamp"
      layouts:
        - '2006-01-02T15:04:05.999999999Z07:00'
      test:
        - '2024-01-15T10:30:00.000000+00:00'

# Output directo a Elasticsearch
output.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "elastic"
  password: "ElasticLabs2024!"
  ssl.certificate_authorities: ["/usr/share/filebeat/config/certs/ca.crt"]
  ssl.verification_mode: "none"
  index: "filebeat-7.17.22-labs-app"

# Desactivar ILM para usar nombre de índice personalizado
setup.ilm.enabled: false
setup.template.name: "filebeat-7.17.22-labs"
setup.template.pattern: "filebeat-7.17.22-labs-*"
setup.template.enabled: false

# Logging
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 3
EOF
```

2. Crear (o actualizar) la definición del contenedor Filebeat en docker-compose:

```bash
cat > ~/elastic-labs/config/docker-compose-filebeat.yml << 'EOF'
version: "3.8"

services:
  filebeat-legacy01:
    image: docker.elastic.co/beats/filebeat:7.17.22
    container_name: filebeat-legacy01
    user: root
    command: filebeat -e -strict.perms=false
    volumes:
      - ~/elastic-labs/config/filebeat-legacy.yml:/usr/share/filebeat/filebeat.yml:ro
      - ~/elastic-labs/config/certs/ca.crt:/usr/share/filebeat/config/certs/ca.crt:ro
      - ~/elastic-labs/logs/app_structured.json:/var/log/app/app_structured.json:ro
      - filebeat-data:/usr/share/filebeat/data
    networks:
      - elastic-net
    restart: unless-stopped

volumes:
  filebeat-data:
    driver: local

networks:
  elastic-net:
    external: true
    name: elastic-net
EOF
```

3. Iniciar el contenedor Filebeat:

```bash
cd ~/elastic-labs/config

# Detener instancia previa si existe
docker stop filebeat-legacy01 2>/dev/null
docker rm filebeat-legacy01 2>/dev/null

# Iniciar Filebeat con la nueva configuración
docker compose -f docker-compose-filebeat.yml up -d filebeat-legacy01

# Esperar a que Filebeat se estabilice
sleep 15
```

4. Verificar que Filebeat está funcionando correctamente:

```bash
# Ver logs del contenedor Filebeat
docker logs filebeat-legacy01 --tail 20 2>&1 | grep -E "(Harvester|Connection|output|error)"
```

### Salida Esperada

```
INFO    [publisher_pipeline_output] pipeline/output.go:143  Connecting to backoff(elasticsearch(https://es01:9200))
INFO    [publisher]     pipeline/retry.go:219   retryer: send all events
INFO    log/harvester.go:302    Harvester started for file: /var/log/app/app_structured.json
```

### Verificación

```bash
# Verificar que Filebeat está enviando datos (buscar líneas sobre eventos publicados)
docker logs filebeat-legacy01 2>&1 | grep -c "Non-zero metrics"
```

Debe retornar al menos 1, indicando que hay actividad de envío de eventos.

---

## Paso 5: Validar Recepción de Logs en Elasticsearch

### Objetivo

Confirmar que ambos agentes están indexando documentos correctamente en sus respectivos índices/data streams.

### Instrucciones

1. Verificar la existencia de los índices creados por ambos agentes:

```bash
# Listar todos los índices del laboratorio
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/*labs*?v&h=index,docs.count,store.size&s=index"
```

2. Verificar el data stream de Elastic Agent:

```bash
# Verificar data stream de Elastic Agent (Custom Logs)
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_data_stream/logs-custom_labs.app-default" | jq '{
    name: .data_streams[0].name,
    status: .data_streams[0].status,
    backing_indices: .data_streams[0].indices | length,
    template: .data_streams[0].template
  }'
```

> **Nota:** Si el data stream no aparece con el nombre exacto, buscar variantes:

```bash
# Buscar cualquier data stream relacionado con labs
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_data_stream/*labs*" | jq '.data_streams[].name'

# O buscar por índices que contengan "custom" o "labs"
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/*custom*,*labs*?v&h=index,docs.count"
```

3. Verificar el índice de Filebeat:

```bash
# Verificar índice de Filebeat legacy
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/filebeat-7.17.22-labs-app/_count" | jq '.count'
```

4. Consultar documentos de ejemplo de cada fuente:

```bash
# Documento de ejemplo del Elastic Agent
echo "=== Elastic Agent - Documento de ejemplo ==="
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-custom_labs.app-default/_search?size=1&pretty" | \
  jq '.hits.hits[0]._source | keys'

echo ""

# Documento de ejemplo de Filebeat
echo "=== Filebeat Legacy - Documento de ejemplo ==="
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/filebeat-7.17.22-labs-app/_search?size=1&pretty" | \
  jq '.hits.hits[0]._source | keys'
```

5. Contar documentos por fuente:

```bash
# Conteo de documentos por agente
echo "Documentos Elastic Agent:"
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-custom_labs.app-default/_count" | jq '.count'

echo "Documentos Filebeat:"
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/filebeat-7.17.22-labs-app/_count" | jq '.count'
```

### Salida Esperada

```
Documentos Elastic Agent:
50
Documentos Filebeat:
50
```

> Los números pueden variar según el tiempo transcurrido desde el inicio del generador.

### Verificación

Ambos conteos deben ser mayores a 0. Si el generador continuo está activo, los conteos deben ser similares (±5 documentos de diferencia).

---

## Paso 6: Comparar Estructura de Documentos entre Agentes

### Objetivo

Analizar las diferencias estructurales entre los documentos indexados por Elastic Agent y Filebeat, identificando campos ECS, metadatos adicionales y diferencias en el procesamiento.

### Instrucciones

1. Obtener un documento completo de Elastic Agent:

```bash
echo "=== ELASTIC AGENT - Documento Completo ==="
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-custom_labs.app-default/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 1,
    "sort": [{"@timestamp": "desc"}]
  }' | jq '.hits.hits[0]._source'
```

2. Obtener un documento completo de Filebeat:

```bash
echo "=== FILEBEAT LEGACY - Documento Completo ==="
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/filebeat-7.17.22-labs-app/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 1,
    "sort": [{"@timestamp": "desc"}]
  }' | jq '.hits.hits[0]._source'
```

3. Comparar los campos presentes en cada fuente:

```bash
# Campos únicos en Elastic Agent (no presentes en Filebeat)
echo "=== Campos exclusivos de Elastic Agent ==="
EA_FIELDS=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-custom_labs.app-default/_search?size=1" | \
  jq -r '[.hits.hits[0]._source | paths(scalars) | join(".")] | sort[]')

FB_FIELDS=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/filebeat-7.17.22-labs-app/_search?size=1" | \
  jq -r '[.hits.hits[0]._source | paths(scalars) | join(".")] | sort[]')

echo "Campos Elastic Agent:"
echo "$EA_FIELDS" | head -20

echo ""
echo "Campos Filebeat:"
echo "$FB_FIELDS" | head -20
```

4. Crear un script de comparación detallada:

```bash
cat > ~/elastic-labs/scripts/compare_agents.py << 'EOF'
#!/usr/bin/env python3
"""Compara la estructura de documentos entre Elastic Agent y Filebeat."""

import json
import subprocess
import ssl

def query_es(index, size=1):
    """Consulta Elasticsearch y retorna el primer documento."""
    cmd = [
        "curl", "-s", "-k", "-u", "elastic:ElasticLabs2024!",
        f"https://localhost:9200/{index}/_search?size={size}"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    data = json.loads(result.stdout)
    if data.get("hits", {}).get("hits"):
        return data["hits"]["hits"][0]["_source"]
    return {}

def flatten_keys(d, prefix=""):
    """Aplana un diccionario anidado en claves con notación de punto."""
    keys = set()
    for k, v in d.items():
        full_key = f"{prefix}.{k}" if prefix else k
        if isinstance(v, dict):
            keys.update(flatten_keys(v, full_key))
        else:
            keys.add(full_key)
    return keys

# Obtener documentos
ea_doc = query_es("logs-custom_labs.app-default")
fb_doc = query_es("filebeat-7.17.22-labs-app")

if not ea_doc:
    # Intentar variantes del nombre del data stream
    for variant in ["logs-custom-labs.app-default", "logs-custom_labs*"]:
        ea_doc = query_es(variant)
        if ea_doc:
            break

ea_keys = flatten_keys(ea_doc)
fb_keys = flatten_keys(fb_doc)

print("=" * 60)
print("COMPARACIÓN DE ESTRUCTURA DE DOCUMENTOS")
print("=" * 60)

print(f"\n📊 Total de campos Elastic Agent: {len(ea_keys)}")
print(f"📊 Total de campos Filebeat:      {len(fb_keys)}")

common = ea_keys & fb_keys
only_ea = ea_keys - fb_keys
only_fb = fb_keys - ea_keys

print(f"\n✅ Campos comunes: {len(common)}")
for k in sorted(common):
    print(f"   • {k}")

print(f"\n🔵 Campos solo en Elastic Agent ({len(only_ea)}):")
for k in sorted(only_ea):
    print(f"   + {k}")

print(f"\n🟠 Campos solo en Filebeat ({len(only_fb)}):")
for k in sorted(only_fb):
    print(f"   + {k}")

print("\n" + "=" * 60)
print("CONCLUSIONES:")
print("=" * 60)
print("""
• Elastic Agent añade metadatos ECS más completos (agent.*, elastic_agent.*, 
  data_stream.*, ecs.version) gracias a la administración centralizada de Fleet.
• Filebeat legacy incluye campos como 'fields.lab.*' (campos personalizados) 
  y 'host.*' con estructura diferente.
• El data_stream.* (type, dataset, namespace) es exclusivo de Elastic Agent 
  y permite mejor organización en data streams.
• Ambos preservan los campos originales del log (@timestamp, log.level, 
  service.name, message).
""")
EOF

python3 ~/elastic-labs/scripts/compare_agents.py
```

### Salida Esperada

```
============================================================
COMPARACIÓN DE ESTRUCTURA DE DOCUMENTOS
============================================================

📊 Total de campos Elastic Agent: 25
📊 Total de campos Filebeat:      18

✅ Campos comunes: 8
   • @timestamp
   • event.duration
   • http.response.status_code
   • log.level
   • message
   • service.name
   • source.ip
   • trace.id

🔵 Campos solo en Elastic Agent (17):
   + agent.ephemeral_id
   + agent.id
   + agent.name
   + agent.type
   + agent.version
   + data_stream.dataset
   + data_stream.namespace
   + data_stream.type
   + ecs.version
   + elastic_agent.id
   + elastic_agent.snapshot
   + elastic_agent.version
   + host.hostname
   + host.name
   + input.type
   + tags

🟠 Campos solo en Filebeat (10):
   + agent.ephemeral_id
   + agent.hostname
   + agent.id
   + agent.name
   + agent.type
   + agent.version
   + fields.lab.collector
   + fields.lab.version
   + host.name
   + tags

============================================================
CONCLUSIONES:
============================================================
...
```

### Verificación

El script debe ejecutarse sin errores y mostrar diferencias claras entre las estructuras de ambos agentes. Los campos `data_stream.*` y `elastic_agent.*` deben aparecer solo en Elastic Agent.

---

## Paso 7: Crear Data Views en Kibana para Exploración

### Objetivo

Crear data views en Kibana para poder explorar los datos de ambos agentes en Discover.

### Instrucciones

1. Crear data view para Elastic Agent:

```bash
# Crear data view para logs del Elastic Agent
curl -s -k -X POST \
  -u elastic:ElasticLabs2024! \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/data_views/data_view" \
  -d '{
    "data_view": {
      "title": "logs-custom*labs*",
      "name": "labs-elastic-agent",
      "timeFieldName": "@timestamp"
    }
  }' | jq '{id: .data_view.id, name: .data_view.name, title: .data_view.title}'
```

2. Crear data view para Filebeat legacy:

```bash
# Crear data view para logs de Filebeat
curl -s -k -X POST \
  -u elastic:ElasticLabs2024! \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/data_views/data_view" \
  -d '{
    "data_view": {
      "title": "filebeat-7.17.22-labs-*",
      "name": "labs-filebeat-legacy",
      "timeFieldName": "@timestamp"
    }
  }' | jq '{id: .data_view.id, name: .data_view.name, title: .data_view.title}'
```

3. Verificar los data views creados:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/data_views" | \
  jq '.data_view[] | select(.name | startswith("labs-")) | {name, title}'
```

### Salida Esperada

```json
{
  "name": "labs-elastic-agent",
  "title": "logs-custom*labs*"
}
{
  "name": "labs-filebeat-legacy",
  "title": "filebeat-7.17.22-labs-*"
}
```

### Verificación

Acceder a Kibana → Discover (`https://localhost:5601/app/discover`) y seleccionar cada data view. Ambos deben mostrar documentos con timestamps recientes.

---

## Validación y Pruebas

### Test Integral de Recepción

Ejecutar el siguiente script de validación completa:

```bash
cat > ~/elastic-labs/scripts/validate_lab03.sh << 'EOF'
#!/bin/bash
# Script de validación integral - Lab 03

PASS=0
FAIL=0
ES_URL="https://localhost:9200"
KIBANA_URL="https://localhost:5601"
CREDS="elastic:ElasticLabs2024!"

check() {
    local description="$1"
    local result="$2"
    if [ "$result" == "true" ] || [ "$result" -gt 0 ] 2>/dev/null; then
        echo "✅ PASS: $description"
        ((PASS++))
    else
        echo "❌ FAIL: $description"
        ((FAIL++))
    fi
}

echo "=========================================="
echo "  VALIDACIÓN LAB 03 - Elastic Agent & Filebeat"
echo "=========================================="
echo ""

# Test 1: Elastic Agent está healthy
EA_STATUS=$(sudo elastic-agent status 2>/dev/null | grep -c "HEALTHY")
check "Elastic Agent está en estado HEALTHY" "$EA_STATUS"

# Test 2: Elastic Agent aparece en Fleet
FLEET_AGENTS=$(curl -s -k -u $CREDS "$KIBANA_URL/api/fleet/agents" | jq '.items | length')
check "Al menos un agente registrado en Fleet" "$FLEET_AGENTS"

# Test 3: Política labs-agent-policy existe
POLICY_EXISTS=$(curl -s -k -u $CREDS "$KIBANA_URL/api/fleet/agent_policies" | \
  jq '[.items[] | select(.name=="labs-agent-policy")] | length')
check "Política 'labs-agent-policy' existe en Fleet" "$POLICY_EXISTS"

# Test 4: Filebeat container está corriendo
FB_RUNNING=$(docker ps --filter name=filebeat-legacy01 --format "{{.Status}}" | grep -c "Up")
check "Contenedor filebeat-legacy01 está en ejecución" "$FB_RUNNING"

# Test 5: Data stream de Elastic Agent tiene documentos
EA_COUNT=$(curl -s -k -u $CREDS "$ES_URL/logs-custom*labs*/_count" 2>/dev/null | jq '.count // 0')
check "Data stream de Elastic Agent contiene documentos ($EA_COUNT docs)" "$EA_COUNT"

# Test 6: Índice de Filebeat tiene documentos
FB_COUNT=$(curl -s -k -u $CREDS "$ES_URL/filebeat-7.17.22-labs-app/_count" 2>/dev/null | jq '.count // 0')
check "Índice de Filebeat contiene documentos ($FB_COUNT docs)" "$FB_COUNT"

# Test 7: Archivo de log existe y tiene contenido
LOG_LINES=$(wc -l < ~/elastic-labs/logs/app_structured.json 2>/dev/null)
check "Archivo app_structured.json tiene contenido ($LOG_LINES líneas)" "$LOG_LINES"

# Test 8: Generador de logs está activo
GEN_PID=$(cat ~/elastic-labs/logs/generator.pid 2>/dev/null)
GEN_RUNNING=$(ps -p "$GEN_PID" -o pid= 2>/dev/null | wc -l)
check "Generador de logs está activo (PID: $GEN_PID)" "$GEN_RUNNING"

# Test 9: Data views creados en Kibana
DV_COUNT=$(curl -s -k -u $CREDS -H "kbn-xsrf: true" "$KIBANA_URL/api/data_views" | \
  jq '[.data_view[] | select(.name | startswith("labs-"))] | length')
check "Data views 'labs-*' creados en Kibana ($DV_COUNT encontrados)" "$DV_COUNT"

# Test 10: Documentos de Elastic Agent contienen campos ECS
EA_ECS=$(curl -s -k -u $CREDS "$ES_URL/logs-custom*labs*/_search?size=1" | \
  jq '.hits.hits[0]._source | has("ecs") or has("data_stream") or has("elastic_agent")')
check "Documentos de Elastic Agent incluyen campos ECS/data_stream" "$([[ $EA_ECS == 'true' ]] && echo 1 || echo 0)"

echo ""
echo "=========================================="
echo "  RESULTADOS: $PASS passed, $FAIL failed"
echo "=========================================="

if [ $FAIL -eq 0 ]; then
    echo "  🎉 ¡Todos los tests pasaron correctamente!"
else
    echo "  ⚠️  Revisar los tests fallidos antes de continuar."
fi
EOF

chmod +x ~/elastic-labs/scripts/validate_lab03.sh
bash ~/elastic-labs/scripts/validate_lab03.sh
```

### Criterios de Aceptación

| Criterio | Condición |
|----------|-----------|
| Elastic Agent enrollado | Aparece en Fleet UI con estado Healthy |
| Política configurada | `labs-agent-policy` con integración Custom Logs activa |
| Data stream activo | `logs-custom-labs.app-default` con >0 documentos |
| Filebeat operativo | Contenedor running, índice con >0 documentos |
| Comparación realizada | Script de comparación ejecutado mostrando diferencias |
| Data views creados | Al menos 2 data views `labs-*` en Kibana |

---

## Resolución de Problemas

### Problema 1: Elastic Agent no se conecta a Fleet Server

**Síntomas:**
- El comando `elastic-agent enroll` falla con `connection refused` o `certificate verify failed`
- El agente aparece como `Offline` o `Unhealthy` en Fleet UI
- En los logs del agente: `failed to checkin: fleet-server returned error 401`

**Causa:**
El agente no puede establecer conexión TLS con Fleet Server, ya sea porque el certificado CA no es correcto, el puerto 8220 no está accesible desde el host, o el enrollment token ha expirado/es inválido.

**Solución:**

```bash
# 1. Verificar que Fleet Server está accesible
curl -s -k https://localhost:8220/api/status | jq '.status'

# 2. Si el certificado es el problema, usar --insecure para testing
sudo elastic-agent enroll \
  --url=https://localhost:8220 \
  --enrollment-token="$ENROLLMENT_TOKEN" \
  --insecure \
  --force

# 3. Si el token es inválido, generar uno nuevo
NEW_TOKEN=$(curl -s -k -X POST \
  -u elastic:ElasticLabs2024! \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  "https://localhost:5601/api/fleet/enrollment_api_keys" \
  -d "{\"policy_id\": \"$POLICY_ID\"}" | jq -r '.item.api_key')
echo "Nuevo token: $NEW_TOKEN"

# 4. Re-enrollar con el nuevo token
sudo elastic-agent enroll \
  --url=https://localhost:8220 \
  --enrollment-token="$NEW_TOKEN" \
  --insecure \
  --force

# 5. Reiniciar el servicio
sudo systemctl restart elastic-agent
```

### Problema 2: Filebeat no indexa documentos en Elasticsearch

**Síntomas:**
- El contenedor Filebeat está running pero `curl ... /_count` retorna 0 documentos
- En los logs de Filebeat: `ERROR pipeline/output.go: Failed to connect to backoff(elasticsearch(...))` o `Harvester started` pero sin `Non-zero metrics`
- El índice `filebeat-7.17.22-labs-app` no aparece en `_cat/indices`

**Causa:**
Filebeat no puede autenticarse contra Elasticsearch (credenciales incorrectas en el YAML), el archivo de log montado no tiene permisos de lectura, o la configuración de output tiene errores de sintaxis (especialmente en la verificación SSL).

**Solución:**

```bash
# 1. Verificar logs detallados de Filebeat
docker logs filebeat-legacy01 2>&1 | tail -30

# 2. Verificar que el archivo de log está montado y es legible
docker exec filebeat-legacy01 cat /var/log/app/app_structured.json | head -2

# 3. Si hay error de permisos, ajustar en el host
chmod 644 ~/elastic-labs/logs/app_structured.json

# 4. Verificar conectividad desde el contenedor a Elasticsearch
docker exec filebeat-legacy01 curl -s -k -u elastic:ElasticLabs2024! \
  https://es01:9200/_cluster/health | head -1

# 5. Si hay error de SSL, verificar que el CA está montado correctamente
docker exec filebeat-legacy01 ls -la /usr/share/filebeat/config/certs/

# 6. Forzar re-lectura del archivo (borrar registro de posición)
docker exec filebeat-legacy01 rm -rf /usr/share/filebeat/data/registry
docker restart filebeat-legacy01

# 7. Esperar y verificar
sleep 15
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/filebeat-7.17.22-labs-app/_count" | jq '.count'
```

---

## Limpieza

Para limpiar los recursos creados en esta práctica (ejecutar solo si no se continuará con la Práctica 4):

```bash
# ⚠️  NO ejecutar si se continuará con las prácticas siguientes.
# Estos índices son necesarios para la Práctica 5.

# Detener el generador de logs
kill $(cat ~/elastic-labs/logs/generator.pid 2>/dev/null) 2>/dev/null
rm -f ~/elastic-labs/logs/generator.pid

# Detener Filebeat
docker stop filebeat-legacy01
docker rm filebeat-legacy01

# Desregistrar Elastic Agent (opcional)
# sudo elastic-agent uninstall

# Eliminar índices de prueba (SOLO si se desea reiniciar)
# curl -s -k -X DELETE -u elastic:ElasticLabs2024! \
#   "https://localhost:9200/filebeat-7.17.22-labs-app"
# curl -s -k -X DELETE -u elastic:ElasticLabs2024! \
#   "https://localhost:9200/.ds-logs-custom*labs*"
```

> **Importante:** Mantener el generador de logs activo y ambos agentes operativos si se continuará con las prácticas posteriores. Los datos generados aquí serán utilizados en la Práctica 5 para crear dashboards operativos.

---

## Resumen

### Logros Alcanzados

En esta práctica se implementaron exitosamente dos métodos de recolección de logs en paralelo:

| Aspecto | Elastic Agent (Fleet) | Filebeat 7.17.22 (Legacy) |
|---------|----------------------|---------------------------|
| **Administración** | Centralizada vía Fleet UI | Archivo YAML por host |
| **Enrollment** | Token + Fleet Server | N/A (configuración directa) |
| **Campos ECS** | Completos (data_stream, ecs, elastic_agent) | Parciales (agent, host) |
| **Actualización** | Remota desde Fleet | Manual en cada host |
| **Índice destino** | Data stream: `logs-custom-labs.app-default` | Índice fijo: `filebeat-7.17.22-labs-app` |
| **Escalabilidad** | Alta (políticas compartidas) | Media (config por host) |

### Conceptos Clave Aplicados

- **Enrollment de Elastic Agent**: Proceso completo de registro usando tokens y Fleet Server
- **Políticas de agente**: Creación y asignación de integraciones Custom Logs
- **Configuración Filebeat**: Input tipo `log` con parsing JSON y output directo a Elasticsearch
- **Data streams vs índices**: Diferencia fundamental entre el modelo moderno (Agent) y legacy (Filebeat)
- **Comparación ECS**: Identificación de campos exclusivos de cada método de recolección

### Recursos Adicionales

- [Documentación Fleet y Elastic Agent](https://www.elastic.co/guide/en/fleet/8.14/index.html)
- [Integración Custom Logs](https://www.elastic.co/guide/en/fleet/8.14/elastic-agent-input-configuration.html)
- [Configuración Filebeat 7.17](https://www.elastic.co/guide/en/beats/filebeat/7.17/configuring-howto-filebeat.html)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
