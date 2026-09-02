# Configurar, validar y presentar una solución de gestión centralizada de logs

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 51 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

Este laboratorio es el proyecto integrador final del curso. Partiendo de una especificación de requisitos, construirás desde cero una solución completa de gestión centralizada de logs que integra tres fuentes (Nginx, aplicación Python y systemd journal), aplica normalización ECS, implementa políticas de retención ILM, configura acceso diferenciado por equipos y valida el cumplimiento de 15 criterios de aceptación mediante un script automatizado. Además, ejecutarás un runbook operativo que simula escenarios de fallo y recuperación.

## Objetivos de Aprendizaje

- [ ] Diseñar e implementar un flujo completo de ingestión desde tres fuentes heterogéneas hasta Elasticsearch con campos ECS obligatorios
- [ ] Crear políticas ILM con retención diferenciada (30 días hot, 90 días warm, delete) y templates de índice con mappings ECS
- [ ] Configurar acceso diferenciado con roles, usuarios y Kibana Spaces para dos equipos (app-team y ops-team)
- [ ] Validar automáticamente el cumplimiento de criterios de aceptación de calidad, rendimiento y seguridad
- [ ] Ejecutar y documentar un runbook operativo con escenarios de fallo, pico de tráfico y recuperación

## Prerrequisitos

### Conocimientos Requeridos

- Comprensión completa de la arquitectura Elastic Stack (Elasticsearch, Kibana, Logstash, Elastic Agent, Fleet)
- Experiencia con ECS (Elastic Common Schema) y mapeo de campos
- Manejo de ILM, index templates y data streams
- Configuración de seguridad: roles, usuarios, API keys, Kibana Spaces
- Docker Compose para orquestación de servicios

### Acceso Requerido

- Entorno con laboratorios 6–9 completados (stack base funcional)
- Acceso root/sudo en el host Ubuntu 22.04 LTS
- Credenciales: `elastic` / `ElasticLabs2024!`
- Script `/opt/lab-scripts/validate-solution.py` disponible (provisto por el instructor)

## Entorno del Laboratorio

### Hardware Mínimo

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 4 núcleos | 8 núcleos |
| RAM | 16 GB | 32 GB |
| Disco | 60 GB SSD libres | 80 GB SSD |
| Red | Acceso localhost, puertos 5601/9200/5044/8220/8080 | Sin restricciones de firewall |

### Software

| Componente | Versión |
|------------|---------|
| Docker Engine | 26.1.4 |
| Docker Compose | 2.27.1 |
| Elasticsearch | 8.14.1 |
| Kibana | 8.14.1 |
| Logstash | 8.14.1 |
| Elastic Agent | 8.14.1 |
| Python | 3.12.3 |
| curl / jq | 7.81.0 / 1.6 |

### Configuración Inicial del Entorno

```bash
# Crear estructura de directorios del proyecto integrador
mkdir -p ~/elastic-labs/{config/certs,logs,scripts,exports,data}
mkdir -p /opt/lab-reports
mkdir -p /opt/lab-apps
mkdir -p /opt/lab-scripts

# Verificar que Docker está operativo
docker info --format '{{.ServerVersion}}'
docker compose version

# Verificar puertos disponibles
for port in 9200 9300 5601 5044 8080 8220; do
  ss -tlnp | grep -q ":${port} " && echo "ALERTA: Puerto $port en uso" || echo "OK: Puerto $port libre"
done
```

## Paso a Paso

---

### Paso 1: Diseñar la Arquitectura de la Solución

**Objetivo:** Documentar el diseño del flujo de ingestión, agentes, pipelines, data streams y políticas antes de implementar.

**Instrucciones:**

1. Crea el documento de diseño:

```bash
cat > /opt/lab-reports/lab10-design.md << 'EOF'
# Diseño de Solución - Gestión Centralizada de Logs

## Arquitectura General

```
┌─────────────────┐     ┌──────────────┐     ┌───────────────────┐
│  Nginx (Docker) │────▶│ Elastic Agent │────▶│                   │
└─────────────────┘     │  (Fleet)      │     │                   │
                        └──────────────┘     │                   │
┌─────────────────┐     ┌──────────────┐     │  Elasticsearch    │
│ systemd journal │────▶│ Elastic Agent │────▶│  (es01)           │
│   (host)        │     │  (Fleet)      │     │                   │
└─────────────────┘     └──────────────┘     │                   │
                                              │                   │
┌─────────────────┐     ┌──────────────┐     │                   │
│ Python App      │────▶│  Logstash    │────▶│                   │
│ (Docker)        │     │  (logstash01) │     └───────────────────┘
└─────────────────┘     └──────────────┘              │
                                                       ▼
                                              ┌───────────────────┐
                                              │  Kibana (kibana01) │
                                              │  - Spaces: app/ops │
                                              └───────────────────┘
```

## Decisiones de Diseño

| Decisión | Justificación |
|----------|---------------|
| Elastic Agent para Nginx | Integración nativa con módulo nginx, parsing automático |
| Elastic Agent para systemd | Integración system con journald, sin configuración adicional |
| Logstash para Python App | Requiere parsing grok personalizado y enriquecimiento ECS |
| ILM con hot 30d / warm 90d | Cumple requisito de retención con optimización de costes |
| Dos Kibana Spaces | Aislamiento visual y de permisos entre equipos |

## Campos ECS Obligatorios

- `@timestamp` - Marca temporal del evento
- `host.name` - Nombre del host origen
- `log.level` - Nivel de severidad (INFO, WARN, ERROR, DEBUG)
- `service.name` - Nombre del servicio emisor
- `event.dataset` - Dataset de origen

## Data Streams y Templates

| Fuente | Data Stream / Índice | Template |
|--------|---------------------|----------|
| Nginx | logs-final-nginx-default | logs-final-* |
| systemd | logs-final-system-default | logs-final-* |
| Python App | logs-final-pythonapp-default | logs-final-* |

## Política ILM: final-retention-policy

- Hot: 30 días (max_age) o 50 GB (max_primary_shard_size)
- Warm: 90 días adicionales (readonly, forcemerge 1 segment)
- Delete: después de warm

## Seguridad

| Equipo | Rol | Acceso |
|--------|-----|--------|
| app-team | app-team-role | Lectura logs-final-pythonapp-*, logs-final-nginx-* |
| ops-team | ops-team-role | Lectura/escritura todos los logs-final-*, cluster monitor |
EOF
```

2. Verifica que el documento se creó correctamente:

```bash
cat /opt/lab-reports/lab10-design.md | head -5
```

**Salida esperada:**

```
# Diseño de Solución - Gestión Centralizada de Logs

## Arquitectura General
```

**Verificación:**

```bash
test -f /opt/lab-reports/lab10-design.md && echo "✅ Documento de diseño creado" || echo "❌ Falta documento"
```

---

### Paso 2: Desplegar las Aplicaciones Fuente

**Objetivo:** Levantar los contenedores de Nginx y la aplicación Python que generarán los logs a ingestar.

**Instrucciones:**

1. Crea el generador de logs Python:

```bash
cat > /opt/lab-apps/log_generator.py << 'PYEOF'
#!/usr/bin/env python3
"""Generador de logs para la aplicación Python del proyecto integrador."""
import time
import random
import json
import sys
import os

LEVELS = ["DEBUG", "INFO", "INFO", "INFO", "WARN", "ERROR"]
SERVICES = ["payment-service", "order-service", "inventory-service"]
MESSAGES = [
    "Request processed successfully",
    "Database connection established",
    "Cache miss for key user_session",
    "Timeout waiting for upstream response",
    "Failed to validate payment token",
    "Order created with ID {order_id}",
    "Inventory check completed for SKU {sku}",
    "Rate limit exceeded for client {client}",
    "Health check passed",
    "Configuration reloaded"
]

EPS = int(os.environ.get("LOG_EPS", "500"))

def generate_log():
    level = random.choice(LEVELS)
    service = random.choice(SERVICES)
    msg = random.choice(MESSAGES).format(
        order_id=random.randint(10000, 99999),
        sku=f"SKU-{random.randint(100,999)}",
        client=f"10.0.{random.randint(1,254)}.{random.randint(1,254)}"
    )
    log_entry = (
        f"{time.strftime('%Y-%m-%d %H:%M:%S')} "
        f"[{level}] "
        f"[{service}] "
        f"- {msg}"
    )
    print(log_entry, flush=True)

if __name__ == "__main__":
    interval = 1.0 / EPS if EPS > 0 else 1.0
    print(f"Starting log generator at {EPS} EPS", file=sys.stderr, flush=True)
    while True:
        generate_log()
        time.sleep(interval)
PYEOF
chmod +x /opt/lab-apps/log_generator.py
```

2. Crea el `docker-compose.yml` para las aplicaciones fuente:

```bash
cat > /opt/lab-apps/docker-compose.yml << 'EOF'
version: "3.8"

services:
  nginx-app:
    image: nginx:1.25-alpine
    container_name: nginx-final
    ports:
      - "8888:80"
    volumes:
      - nginx-logs:/var/log/nginx
    networks:
      - elastic-net
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:80"]
      interval: 10s
      timeout: 5s
      retries: 3

  python-app:
    image: python:3.12-slim
    container_name: python-app-final
    working_dir: /app
    volumes:
      - ./log_generator.py:/app/log_generator.py:ro
      - python-logs:/var/log/python-app
    command: >
      bash -c "python /app/log_generator.py 2>&1 | tee /var/log/python-app/app.log"
    environment:
      - LOG_EPS=500
    networks:
      - elastic-net
    restart: unless-stopped

  traffic-generator:
    image: curlimages/curl:8.5.0
    container_name: traffic-gen-final
    depends_on:
      nginx-app:
        condition: service_healthy
    networks:
      - elastic-net
    entrypoint: /bin/sh
    command: >
      -c 'while true; do
        curl -s -o /dev/null -w "" http://nginx-app:80/;
        curl -s -o /dev/null -w "" http://nginx-app:80/api/health;
        curl -s -o /dev/null -w "" http://nginx-app:80/nonexistent;
        sleep 0.01;
      done'
    restart: unless-stopped

volumes:
  nginx-logs:
  python-logs:

networks:
  elastic-net:
    external: true
EOF
```

3. Asegúrate de que la red Docker externa existe y levanta los servicios:

```bash
docker network inspect elastic-net >/dev/null 2>&1 || \
  docker network create --subnet=172.20.0.0/24 elastic-net

cd /opt/lab-apps
docker compose up -d
```

4. Verifica que los contenedores están corriendo:

```bash
docker ps --filter "name=nginx-final" --filter "name=python-app-final" --format "table {{.Names}}\t{{.Status}}"
```

**Salida esperada:**

```
NAMES                STATUS
nginx-final          Up X seconds (healthy)
python-app-final     Up X seconds
```

**Verificación:**

```bash
# Verificar que Nginx genera logs
docker exec nginx-final cat /var/log/nginx/access.log | tail -3

# Verificar que Python app genera logs
docker exec python-app-final tail -3 /var/log/python-app/app.log
```

---

### Paso 3: Crear la Política ILM y el Index Template

**Objetivo:** Configurar la política de retención `final-retention-policy` y el template `logs-final-*` con mappings ECS.

**Instrucciones:**

1. Crea la política ILM:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_ilm/policy/final-retention-policy" \
  -H "Content-Type: application/json" -d '{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "30d",
            "max_primary_shard_size": "50gb"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "readonly": {},
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "delete": {
        "min_age": "120d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}' | jq .
```

2. Crea el component template con mappings ECS:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_component_template/final-ecs-mappings" \
  -H "Content-Type: application/json" -d '{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "host": {
          "properties": {
            "name": { "type": "keyword" }
          }
        },
        "log": {
          "properties": {
            "level": { "type": "keyword" }
          }
        },
        "service": {
          "properties": {
            "name": { "type": "keyword" }
          }
        },
        "event": {
          "properties": {
            "dataset": { "type": "keyword" }
          }
        },
        "message": { "type": "text" },
        "client": {
          "properties": {
            "ip": { "type": "ip" }
          }
        },
        "http": {
          "properties": {
            "request": {
              "properties": {
                "method": { "type": "keyword" }
              }
            },
            "response": {
              "properties": {
                "status_code": { "type": "long" }
              }
            }
          }
        }
      }
    }
  }
}' | jq .
```

3. Crea el index template compuesto:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_index_template/logs-final-template" \
  -H "Content-Type: application/json" -d '{
  "index_patterns": ["logs-final-*"],
  "data_stream": {},
  "composed_of": ["final-ecs-mappings"],
  "priority": 500,
  "template": {
    "settings": {
      "index.lifecycle.name": "final-retention-policy",
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }
}' | jq .
```

**Salida esperada:**

```json
{ "acknowledged": true }
```

**Verificación:**

```bash
# Verificar política ILM
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/final-retention-policy" | jq '.["final-retention-policy"].policy.phases | keys'

# Salida esperada: ["delete", "hot", "warm"]

# Verificar template
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/logs-final-template" | jq '.index_templates[0].index_template.index_patterns'

# Salida esperada: ["logs-final-*"]
```

---

### Paso 4: Configurar Logstash para la Aplicación Python

**Objetivo:** Crear el pipeline de Logstash que ingesta logs de la aplicación Python, los parsea con grok y los enriquece con campos ECS.

**Instrucciones:**

1. Crea el archivo de configuración del pipeline:

```bash
cat > ~/elastic-labs/config/logstash-final-pipeline.conf << 'EOF'
input {
  file {
    path => "/var/log/python-app/app.log"
    start_position => "beginning"
    sincedb_path => "/usr/share/logstash/data/sincedb_python"
    tags => ["python-app"]
  }
}

filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:log_timestamp} \[%{LOGLEVEL:log_level}\] \[%{DATA:service_name}\] - %{GREEDYDATA:log_message}"
    }
  }

  date {
    match => ["log_timestamp", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
    remove_field => ["log_timestamp"]
  }

  mutate {
    rename => {
      "log_level" => "[log][level]"
      "service_name" => "[service][name]"
      "log_message" => "message"
    }
    add_field => {
      "[event][dataset]" => "python-app.logs"
      "[host][name]" => "python-app-final"
    }
    uppercase => ["[log][level]"]
  }

  # Extraer client IP si existe en el mensaje
  grok {
    match => { "message" => "client %{IP:[client][ip]}" }
    tag_on_failure => []
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_verification => false
    data_stream => true
    data_stream_type => "logs"
    data_stream_dataset => "final-pythonapp"
    data_stream_namespace => "default"
  }
}
EOF
```

2. Actualiza el contenedor de Logstash para montar el pipeline y los logs de Python:

```bash
cat > ~/elastic-labs/config/docker-compose-logstash-final.yml << 'EOF'
version: "3.8"

services:
  logstash-final:
    image: docker.elastic.co/logstash/logstash:8.14.1
    container_name: logstash-final01
    volumes:
      - ~/elastic-labs/config/logstash-final-pipeline.conf:/usr/share/logstash/pipeline/pipeline.conf:ro
      - python-logs:/var/log/python-app:ro
    environment:
      - XPACK_MONITORING_ENABLED=false
      - LS_JAVA_OPTS=-Xmx1g -Xms512m
    networks:
      - elastic-net
    depends_on:
      - es01
    restart: unless-stopped

volumes:
  python-logs:
    external: true
    name: lab-apps_python-logs

networks:
  elastic-net:
    external: true
EOF

cd ~/elastic-labs/config
docker compose -f docker-compose-logstash-final.yml up -d
```

3. Espera 30 segundos y verifica la ingestión:

```bash
sleep 30
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_count" | jq .count
```

**Salida esperada:**

```
Un número mayor a 0 (dependiendo del tiempo transcurrido, típicamente >1000)
```

**Verificación:**

```bash
# Verificar campos ECS en un documento
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_search?size=1" | \
  jq '.hits.hits[0]._source | {timestamp: .["@timestamp"], host_name: .host.name, log_level: .log.level, service_name: .service.name, event_dataset: .event.dataset}'
```

Salida esperada (ejemplo):

```json
{
  "timestamp": "2024-XX-XXTXX:XX:XX.XXXZ",
  "host_name": "python-app-final",
  "log_level": "INFO",
  "service_name": "payment-service",
  "event_dataset": "python-app.logs"
}
```

---

### Paso 5: Configurar Elastic Agent vía Fleet para Nginx y Systemd

**Objetivo:** Crear la política `final-solution-policy` en Fleet con integraciones para Nginx y system (journald).

**Instrucciones:**

1. Crea la política de agente vía API de Fleet:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/api/fleet/agent_policies" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "name": "final-solution-policy",
    "namespace": "default",
    "monitoring_enabled": ["logs", "metrics"],
    "description": "Política del proyecto integrador final - Lab 10"
  }' | jq '{id: .item.id, name: .item.name, status: .item.status}'
```

Guarda el ID de la política:

```bash
POLICY_ID=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/fleet/agent_policies" \
  -H "kbn-xsrf: true" | jq -r '.items[] | select(.name=="final-solution-policy") | .id')
echo "Policy ID: $POLICY_ID"
```

2. Añade la integración de Nginx:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/api/fleet/package_policies" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d "{
    \"name\": \"nginx-final-integration\",
    \"namespace\": \"default\",
    \"policy_id\": \"${POLICY_ID}\",
    \"package\": {
      \"name\": \"nginx\",
      \"version\": \"1.25.0\"
    },
    \"inputs\": [
      {
        \"type\": \"logfile\",
        \"enabled\": true,
        \"streams\": [
          {
            \"enabled\": true,
            \"data_stream\": {
              \"type\": \"logs\",
              \"dataset\": \"nginx.access\"
            },
            \"vars\": {
              \"paths\": { \"value\": [\"/var/log/nginx/access.log\"] }
            }
          },
          {
            \"enabled\": true,
            \"data_stream\": {
              \"type\": \"logs\",
              \"dataset\": \"nginx.error\"
            },
            \"vars\": {
              \"paths\": { \"value\": [\"/var/log/nginx/error.log\"] }
            }
          }
        ]
      }
    ]
  }" | jq '{id: .item.id, name: .item.name}'
```

3. Añade la integración de System (journald):

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/api/fleet/package_policies" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d "{
    \"name\": \"system-final-integration\",
    \"namespace\": \"default\",
    \"policy_id\": \"${POLICY_ID}\",
    \"package\": {
      \"name\": \"system\",
      \"version\": \"1.54.0\"
    },
    \"inputs\": [
      {
        \"type\": \"logfile\",
        \"enabled\": true,
        \"streams\": [
          {
            \"enabled\": true,
            \"data_stream\": {
              \"type\": \"logs\",
              \"dataset\": \"system.syslog\"
            },
            \"vars\": {
              \"paths\": { \"value\": [\"/var/log/syslog\"] }
            }
          }
        ]
      },
      {
        \"type\": \"journald\",
        \"enabled\": true,
        \"streams\": [
          {
            \"enabled\": true,
            \"data_stream\": {
              \"type\": \"logs\",
              \"dataset\": \"system.journal\"
            }
          }
        ]
      }
    ]
  }" | jq '{id: .item.id, name: .item.name}'
```

4. Verifica la política y sus integraciones:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/fleet/agent_policies/${POLICY_ID}" \
  -H "kbn-xsrf: true" | jq '{name: .item.name, package_policies: [.item.package_policies[].name]}'
```

**Salida esperada:**

```json
{
  "name": "final-solution-policy",
  "package_policies": [
    "nginx-final-integration",
    "system-final-integration"
  ]
}
```

---

### Paso 6: Configurar Seguridad — Roles, Usuarios y Kibana Spaces

**Objetivo:** Crear acceso diferenciado para `app-team` (solo lectura de logs de aplicación) y `ops-team` (acceso completo operativo).

**Instrucciones:**

1. Crea el rol para app-team:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_security/role/app-team-role" \
  -H "Content-Type: application/json" -d '{
  "indices": [
    {
      "names": ["logs-final-pythonapp-*", "logs-nginx.*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["feature_discover.read", "feature_dashboard.read", "feature_visualize.read"],
      "resources": ["space:app-space"]
    }
  ]
}' | jq .
```

2. Crea el rol para ops-team:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_security/role/ops-team-role" \
  -H "Content-Type: application/json" -d '{
  "cluster": ["monitor", "manage_ilm", "manage_index_templates"],
  "indices": [
    {
      "names": ["logs-final-*", "logs-nginx.*", "logs-system.*"],
      "privileges": ["read", "write", "view_index_metadata", "manage"]
    }
  ],
  "applications": [
    {
      "application": "kibana-.kibana",
      "privileges": ["feature_discover.all", "feature_dashboard.all", "feature_visualize.all", "feature_indexPatterns.all", "feature_stackAlerts.all"],
      "resources": ["space:ops-space"]
    }
  ]
}' | jq .
```

3. Crea los usuarios:

```bash
# Usuario app-team
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_security/user/app-user" \
  -H "Content-Type: application/json" -d '{
  "password": "AppTeam2024!",
  "roles": ["app-team-role"],
  "full_name": "App Team User",
  "email": "app-team@lab.local"
}' | jq .

# Usuario ops-team
curl -s -k -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_security/user/ops-user" \
  -H "Content-Type: application/json" -d '{
  "password": "OpsTeam2024!",
  "roles": ["ops-team-role"],
  "full_name": "Ops Team User",
  "email": "ops-team@lab.local"
}' | jq .
```

4. Crea los Kibana Spaces:

```bash
# Space para app-team
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/api/spaces/space" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "id": "app-space",
    "name": "Application Team",
    "description": "Espacio para el equipo de aplicaciones",
    "color": "#00bfb3",
    "disabledFeatures": ["siem", "apm", "infrastructure", "uptime"]
  }' | jq .

# Space para ops-team
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/api/spaces/space" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "id": "ops-space",
    "name": "Operations Team",
    "description": "Espacio para el equipo de operaciones",
    "color": "#ee4035",
    "disabledFeatures": []
  }' | jq .
```

**Verificación:**

```bash
# Verificar que app-user puede leer logs de python-app
curl -s -k -u app-user:AppTeam2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_count" | jq .

# Verificar que app-user NO puede leer logs de system
curl -s -k -u app-user:AppTeam2024! \
  "https://localhost:9200/logs-system.*/_count" | jq .status
# Salida esperada: 403

# Verificar spaces
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/spaces/space" \
  -H "kbn-xsrf: true" | jq '.[].id'
```

---

### Paso 7: Crear Dashboards Operativos en Kibana

**Objetivo:** Crear data views y un dashboard de monitoreo en el space de ops-team.

**Instrucciones:**

1. Crea el data view para logs finales en el ops-space:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/s/ops-space/api/data_views/data_view" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "data_view": {
      "title": "logs-final-*",
      "name": "Final Solution Logs",
      "timeFieldName": "@timestamp"
    }
  }' | jq '{id: .data_view.id, title: .data_view.title}'
```

2. Crea un dashboard básico con una visualización de conteo por servicio mediante la API de saved objects:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/s/ops-space/api/saved_objects/dashboard" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "attributes": {
      "title": "Final Solution - Operational Dashboard",
      "description": "Dashboard operativo del proyecto integrador",
      "timeRestore": true,
      "timeTo": "now",
      "timeFrom": "now-1h",
      "refreshInterval": {
        "pause": false,
        "value": 30000
      },
      "panelsJSON": "[]",
      "kibanaSavedObjectMeta": {
        "searchSourceJSON": "{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"filter\":[]}"
      }
    }
  }' | jq '{id: .id, type: .type}'
```

3. Repite la creación de data view para el app-space:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/s/app-space/api/data_views/data_view" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -d '{
    "data_view": {
      "title": "logs-final-pythonapp-*",
      "name": "Python App Logs",
      "timeFieldName": "@timestamp"
    }
  }' | jq '{id: .data_view.id, title: .data_view.title}'
```

**Verificación:**

```bash
# Listar dashboards en ops-space
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:5601/s/ops-space/api/saved_objects/_find?type=dashboard" \
  -H "kbn-xsrf: true" | jq '.saved_objects[].attributes.title'
```

**Salida esperada:**

```
"Final Solution - Operational Dashboard"
```

---

### Paso 8: Ejecutar el Script de Validación Automática

**Objetivo:** Ejecutar el script de validación que verifica los 15 criterios de aceptación predefinidos.

**Instrucciones:**

1. Crea el script de validación (si no fue provisto por el instructor):

```bash
cat > /opt/lab-scripts/validate-solution.py << 'PYEOF'
#!/usr/bin/env python3
"""Script de validación automática - 15 criterios de aceptación."""
import requests
import json
import sys
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

ES_URL = "https://localhost:9200"
KB_URL = "https://localhost:5601"
AUTH = ("elastic", "ElasticLabs2024!")
VERIFY = False

results = []

def check(name, condition, detail=""):
    status = "✅ PASS" if condition else "❌ FAIL"
    results.append({"criterion": name, "status": status, "detail": detail})
    print(f"  {status}: {name} {detail}")

def es_get(path):
    r = requests.get(f"{ES_URL}{path}", auth=AUTH, verify=VERIFY)
    return r.json() if r.status_code == 200 else {}

def kb_get(path):
    r = requests.get(f"{KB_URL}{path}", auth=AUTH, verify=VERIFY, headers={"kbn-xsrf": "true"})
    return r.json() if r.status_code == 200 else {}

print("\n" + "="*60)
print("  VALIDACIÓN DE SOLUCIÓN - PROYECTO INTEGRADOR FINAL")
print("="*60 + "\n")

# 1. Cluster health
health = es_get("/_cluster/health")
check("1. Cluster health green/yellow", health.get("status") in ["green", "yellow"], f"(status: {health.get('status')})")

# 2. ILM policy exists
ilm = es_get("/_ilm/policy/final-retention-policy")
check("2. Política ILM 'final-retention-policy' existe", "final-retention-policy" in ilm)

# 3. Index template exists
tpl = es_get("/_index_template/logs-final-template")
check("3. Index template 'logs-final-template' existe", len(tpl.get("index_templates", [])) > 0)

# 4. Python app data stream has documents
count_py = es_get("/logs-final-pythonapp-default/_count")
py_count = count_py.get("count", 0)
check("4. Data stream python-app tiene documentos", py_count > 0, f"(count: {py_count})")

# 5. ECS field @timestamp present
sample = es_get("/logs-final-pythonapp-default/_search?size=1")
hits = sample.get("hits", {}).get("hits", [])
doc = hits[0]["_source"] if hits else {}
check("5. Campo @timestamp presente", "@timestamp" in doc)

# 6. ECS field host.name present
check("6. Campo host.name presente", "host" in doc and "name" in doc.get("host", {}))

# 7. ECS field log.level present
check("7. Campo log.level presente", "log" in doc and "level" in doc.get("log", {}))

# 8. ECS field service.name present
check("8. Campo service.name presente", "service" in doc and "name" in doc.get("service", {}))

# 9. ECS field event.dataset present
check("9. Campo event.dataset presente", "event" in doc and "dataset" in doc.get("event", {}))

# 10. Throughput check (>100 docs in last 60s as proxy for 500 EPS)
throughput = es_get("/logs-final-pythonapp-default/_count?q=@timestamp:[now-60s TO now]")
tp_count = throughput.get("count", 0)
check("10. Throughput mínimo sostenido", tp_count > 100, f"(últimos 60s: {tp_count} docs)")

# 11. Role app-team-role exists
role_app = es_get("/_security/role/app-team-role")
check("11. Rol 'app-team-role' existe", "app-team-role" in role_app)

# 12. Role ops-team-role exists
role_ops = es_get("/_security/role/ops-team-role")
check("12. Rol 'ops-team-role' existe", "ops-team-role" in role_ops)

# 13. User app-user exists
user_app = es_get("/_security/user/app-user")
check("13. Usuario 'app-user' existe", "app-user" in user_app)

# 14. User ops-user exists
user_ops = es_get("/_security/user/ops-user")
check("14. Usuario 'ops-user' existe", "ops-user" in user_ops)

# 15. Kibana spaces exist
spaces = kb_get("/api/spaces/space")
space_ids = [s["id"] for s in spaces] if isinstance(spaces, list) else []
check("15. Kibana Spaces (app-space, ops-space) existen",
      "app-space" in space_ids and "ops-space" in space_ids,
      f"(encontrados: {space_ids})")

# Summary
print("\n" + "="*60)
passed = sum(1 for r in results if "PASS" in r["status"])
total = len(results)
print(f"  RESULTADO: {passed}/{total} criterios cumplidos")
print("="*60)

# Save report
report_path = "/opt/lab-reports/validation-report.json"
with open(report_path, "w") as f:
    json.dump({"total": total, "passed": passed, "results": results}, f, indent=2)
print(f"\n  Reporte guardado en: {report_path}\n")

sys.exit(0 if passed == total else 1)
PYEOF
chmod +x /opt/lab-scripts/validate-solution.py
```

2. Ejecuta el script:

```bash
python3 /opt/lab-scripts/validate-solution.py
```

**Salida esperada:**

```
============================================================
  VALIDACIÓN DE SOLUCIÓN - PROYECTO INTEGRADOR FINAL
============================================================

  ✅ PASS: 1. Cluster health green/yellow (status: green)
  ✅ PASS: 2. Política ILM 'final-retention-policy' existe
  ✅ PASS: 3. Index template 'logs-final-template' existe
  ✅ PASS: 4. Data stream python-app tiene documentos (count: XXXX)
  ✅ PASS: 5. Campo @timestamp presente
  ✅ PASS: 6. Campo host.name presente
  ✅ PASS: 7. Campo log.level presente
  ✅ PASS: 8. Campo service.name presente
  ✅ PASS: 9. Campo event.dataset presente
  ✅ PASS: 10. Throughput mínimo sostenido (últimos 60s: XXX docs)
  ✅ PASS: 11. Rol 'app-team-role' existe
  ✅ PASS: 12. Rol 'ops-team-role' existe
  ✅ PASS: 13. Usuario 'app-user' existe
  ✅ PASS: 14. Usuario 'ops-user' existe
  ✅ PASS: 15. Kibana Spaces (app-space, ops-space) existen

============================================================
  RESULTADO: 15/15 criterios cumplidos
============================================================
```

**Verificación:**

```bash
cat /opt/lab-reports/validation-report.json | jq '{total, passed}'
```

---

### Paso 9: Ejecutar el Runbook Operativo

**Objetivo:** Simular escenarios reales de fallo y recuperación para validar la resiliencia de la solución.

**Instrucciones:**

1. **Escenario A — Simular fallo de Logstash y verificar recuperación:**

```bash
echo "=== Escenario A: Fallo de Logstash ==="

# Obtener conteo antes del fallo
COUNT_BEFORE=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_count" | jq .count)
echo "Documentos antes: $COUNT_BEFORE"

# Detener Logstash
docker stop logstash-final01
echo "Logstash detenido. Esperando 30 segundos..."
sleep 30

# Reiniciar Logstash
docker start logstash-final01
echo "Logstash reiniciado. Esperando recuperación (45 segundos)..."
sleep 45

# Verificar que los logs acumulados se ingestan
COUNT_AFTER=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_count" | jq .count)
echo "Documentos después: $COUNT_AFTER"

if [ "$COUNT_AFTER" -gt "$COUNT_BEFORE" ]; then
  echo "✅ PASS: Logstash se recuperó y continuó la ingestión"
else
  echo "❌ FAIL: No se detectó recuperación"
fi
```

2. **Escenario B — Rotación de log y continuidad:**

```bash
echo "=== Escenario B: Rotación de log ==="

# Simular rotación (truncar el archivo de log)
docker exec python-app-final sh -c "cp /var/log/python-app/app.log /var/log/python-app/app.log.1 && > /var/log/python-app/app.log"
echo "Log rotado. Esperando 20 segundos para nuevos logs..."
sleep 20

# Verificar que nuevos logs siguen llegando
NEW_COUNT=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_count?q=@timestamp:[now-20s TO now]" | jq .count)
echo "Nuevos documentos en últimos 20s: $NEW_COUNT"

if [ "$NEW_COUNT" -gt 0 ]; then
  echo "✅ PASS: Ingestión continuó tras rotación de log"
else
  echo "❌ FAIL: No se detectaron nuevos documentos tras rotación"
fi
```

3. **Escenario C — Pico de tráfico (2.000 EPS durante 60 segundos):**

```bash
echo "=== Escenario C: Pico de tráfico ==="

# Aumentar EPS temporalmente
docker exec python-app-final sh -c "kill 1" 2>/dev/null
docker stop python-app-final

# Reiniciar con EPS elevado
cd /opt/lab-apps
docker compose run -d --name python-app-burst -e LOG_EPS=2000 python-app

echo "Generando pico de 2000 EPS durante 60 segundos..."
sleep 60

# Detener el burst
docker stop python-app-burst && docker rm python-app-burst

# Restaurar servicio normal
docker compose up -d python-app

# Esperar ingestión
sleep 30

# Verificar que no hay pérdida significativa
BURST_COUNT=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_count?q=@timestamp:[now-3m TO now]" | jq .count)
echo "Documentos ingestados en últimos 3 minutos: $BURST_COUNT"

# Esperamos al menos 60000 docs (2000*60*~50% tolerancia)
if [ "$BURST_COUNT" -gt 30000 ]; then
  echo "✅ PASS: Pico de tráfico manejado sin pérdida significativa"
else
  echo "⚠️  WARN: Posible pérdida parcial durante el pico (${BURST_COUNT} docs)"
fi
```

4. **Escenario D — Detener Elasticsearch y verificar recuperación:**

```bash
echo "=== Escenario D: Fallo de Elasticsearch ==="

# Detener ES por 30 segundos (reducido de 2 min para el lab)
docker stop es01
echo "Elasticsearch detenido. Esperando 30 segundos..."
sleep 30

# Reiniciar
docker start es01
echo "Elasticsearch reiniciado. Esperando que el cluster se estabilice (60s)..."
sleep 60

# Verificar salud del cluster
HEALTH=$(curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cluster/health" | jq -r .status)
echo "Estado del cluster: $HEALTH"

if [ "$HEALTH" = "green" ] || [ "$HEALTH" = "yellow" ]; then
  echo "✅ PASS: Elasticsearch se recuperó correctamente"
else
  echo "❌ FAIL: Cluster en estado $HEALTH"
fi
```

**Verificación global del runbook:**

```bash
echo ""
echo "========================================="
echo "  RUNBOOK OPERATIVO COMPLETADO"
echo "========================================="
```

---

### Paso 10: Preparar la Presentación Técnica

**Objetivo:** Generar el documento final de presentación con arquitectura, decisiones y evidencias.

**Instrucciones:**

1. Genera el reporte final:

```bash
cat > /opt/lab-reports/lab10-presentation.md << 'EOF'
# Presentación Técnica - Solución de Gestión Centralizada de Logs

## 1. Resumen Ejecutivo

Solución implementada para la gestión centralizada de logs de tres fuentes
heterogéneas, cumpliendo requisitos de calidad de datos (ECS), rendimiento
(500 EPS sostenidos), retención (30d hot / 90d warm) y seguridad
(acceso diferenciado por equipos).

## 2. Arquitectura Implementada

- **Fuente 1 - Nginx**: Recolección vía Elastic Agent con integración nginx
- **Fuente 2 - systemd journal**: Recolección vía Elastic Agent con integración system
- **Fuente 3 - Python App**: Recolección vía Logstash con pipeline grok + ECS

## 3. Componentes Desplegados

| Componente | Contenedor | Función |
|------------|-----------|---------|
| Elasticsearch | es01 | Almacenamiento y búsqueda |
| Kibana | kibana01 | Visualización y gestión |
| Logstash | logstash-final01 | Parsing Python app |
| Fleet Server | fleet-server01 | Gestión de agentes |
| Nginx | nginx-final | Fuente de logs web |
| Python App | python-app-final | Fuente de logs aplicación |

## 4. Decisiones de Diseño Clave

1. **Logstash para Python App**: El formato de log requiere grok personalizado
   que no es soportado nativamente por las integraciones de Elastic Agent
2. **ILM con forcemerge en warm**: Reduce segmentos para optimizar búsquedas
   históricas y liberar recursos
3. **Réplicas=0 en lab**: Cluster single-node; en producción se usaría 1 réplica
4. **Kibana Spaces separados**: Aislamiento completo de dashboards entre equipos

## 5. Evidencias de Cumplimiento

### Validación Automática
EOF

# Añadir resultados de validación
echo '```json' >> /opt/lab-reports/lab10-presentation.md
cat /opt/lab-reports/validation-report.json >> /opt/lab-reports/lab10-presentation.md
echo '```' >> /opt/lab-reports/lab10-presentation.md

cat >> /opt/lab-reports/lab10-presentation.md << 'EOF'

### Runbook Operativo

| Escenario | Resultado |
|-----------|-----------|
| Fallo de Logstash | Recuperación automática tras reinicio |
| Rotación de log | Continuidad de ingestión verificada |
| Pico 2000 EPS | Manejado sin pérdida significativa |
| Fallo de Elasticsearch | Recuperación completa del cluster |

## 6. Mejoras para Producción

- Añadir nodos adicionales para alta disponibilidad
- Implementar dead letter queue en Logstash
- Configurar alertas en Watcher/Rules para umbrales de ingestión
- Habilitar snapshot repository para backup automatizado
- Implementar TLS mutual entre agentes y Elasticsearch

---
*Generado: $(date -Iseconds)*
EOF

echo "✅ Presentación generada en /opt/lab-reports/lab10-presentation.md"
```

2. Verifica los entregables finales:

```bash
echo "=== Entregables del Proyecto ==="
ls -la /opt/lab-reports/
echo ""
echo "Archivos:"
for f in /opt/lab-reports/*; do
  echo "  📄 $(basename $f) ($(wc -l < $f) líneas)"
done
```

**Salida esperada:**

```
=== Entregables del Proyecto ===
Archivos:
  📄 lab10-design.md (XX líneas)
  📄 lab10-presentation.md (XX líneas)
  📄 validation-report.json (XX líneas)
```

---

## Validación y Pruebas

Ejecuta esta secuencia final de validación completa:

```bash
echo "╔══════════════════════════════════════════════════════╗"
echo "║   VALIDACIÓN FINAL DEL PROYECTO INTEGRADOR          ║"
echo "╚══════════════════════════════════════════════════════╝"
echo ""

# 1. Verificar todos los contenedores
echo "1. Estado de contenedores:"
docker ps --format "   {{.Names}}: {{.Status}}" | grep -E "(es01|kibana|logstash-final|nginx-final|python-app)"
echo ""

# 2. Verificar salud del cluster
echo "2. Salud del cluster:"
curl -s -k -u elastic:ElasticLabs2024! "https://localhost:9200/_cluster/health" | \
  jq '"   Status: \(.status) | Nodes: \(.number_of_nodes) | Shards: \(.active_shards)"'
echo ""

# 3. Verificar data streams
echo "3. Data streams activos:"
curl -s -k -u elastic:ElasticLabs2024! "https://localhost:9200/_data_stream/logs-final-*" | \
  jq '.data_streams[] | "   \(.name): \(.status) (\(.indices | length) backing indices)"'
echo ""

# 4. Verificar ILM
echo "4. Política ILM:"
curl -s -k -u elastic:ElasticLabs2024! "https://localhost:9200/_ilm/policy/final-retention-policy" | \
  jq '."final-retention-policy".policy.phases | keys | "   Fases: \(.)"'
echo ""

# 5. Ejecutar validación completa
echo "5. Ejecutando validación automática..."
python3 /opt/lab-scripts/validate-solution.py
echo ""

# 6. Verificar entregables
echo "6. Entregables:"
for f in lab10-design.md lab10-presentation.md validation-report.json; do
  test -f "/opt/lab-reports/$f" && echo "   ✅ $f" || echo "   ❌ $f FALTANTE"
done
```

**Criterio de éxito:** Los 15 criterios de validación deben mostrar PASS y los tres archivos de entregables deben existir.

---

## Solución de Problemas

### Problema 1: Logstash no ingesta documentos de la aplicación Python

**Síntomas:**
- El conteo en `logs-final-pythonapp-default/_count` permanece en 0
- En los logs de Logstash aparece: `No sincedb_path set, using default`

**Causa:** El volumen de Docker `python-logs` no está correctamente montado en el contenedor de Logstash, o el nombre del volumen externo no coincide con el nombre real creado por Docker Compose en `/opt/lab-apps`.

**Solución:**

```bash
# Verificar el nombre real del volumen
docker volume ls | grep python

# El nombre suele ser: lab-apps_python-logs o opt_lab-apps_python-logs
# Ajustar en docker-compose-logstash-final.yml:
REAL_VOLUME=$(docker volume ls --format '{{.Name}}' | grep python-logs)
echo "Volumen encontrado: $REAL_VOLUME"

# Editar el compose y actualizar el nombre del volumen externo
sed -i "s/name: lab-apps_python-logs/name: ${REAL_VOLUME}/" \
  ~/elastic-labs/config/docker-compose-logstash-final.yml

# Reiniciar Logstash
cd ~/elastic-labs/config
docker compose -f docker-compose-logstash-final.yml down
docker compose -f docker-compose-logstash-final.yml up -d

# Verificar logs de Logstash
docker logs logstash-final01 --tail 20 2>&1 | grep -E "(Pipeline|ERROR|file)"
```

---

### Problema 2: El script de validación falla en el criterio de throughput (criterio 10)

**Síntomas:**
- El criterio 10 muestra `❌ FAIL: Throughput mínimo sostenido (últimos 60s: 0 docs)`
- Los demás criterios ECS pasan correctamente

**Causa:** Existe un desfase temporal (clock skew) entre el host y el contenedor de la aplicación Python, o Logstash tiene un backlog acumulado y los documentos recién indexados tienen timestamps del pasado, no de los últimos 60 segundos.

**Solución:**

```bash
# Verificar la hora del host vs el timestamp más reciente en ES
echo "Hora del host: $(date -u +%Y-%m-%dT%H:%M:%S)"
curl -s -k -u elastic:ElasticLabs2024! \
  "https://localhost:9200/logs-final-pythonapp-default/_search?size=1&sort=@timestamp:desc" | \
  jq '.hits.hits[0]._source["@timestamp"]'

# Si hay desfase, sincronizar NTP en el host
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd

# Verificar que el generador Python usa la hora correcta
docker exec python-app-final date -u

# Si el backlog es el problema, esperar a que Logstash procese todo
# y luego re-ejecutar la validación
sleep 60
python3 /opt/lab-scripts/validate-solution.py
```

---

## Limpieza

```bash
# Detener contenedores de aplicaciones
cd /opt/lab-apps
docker compose down -v

# Detener Logstash del proyecto
cd ~/elastic-labs/config
docker compose -f docker-compose-logstash-final.yml down

# Eliminar data streams del proyecto (OPCIONAL - solo si deseas limpiar datos)
curl -s -k -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/_data_stream/logs-final-pythonapp-default"

# Eliminar política ILM y template (OPCIONAL)
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_ilm/policy/final-retention-policy"
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_index_template/logs-final-template"
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_component_template/final-ecs-mappings"

# Eliminar usuarios y roles (OPCIONAL)
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_security/user/app-user"
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_security/user/ops-user"
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_security/role/app-team-role"
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/_security/role/ops-team-role"

# Eliminar Kibana Spaces (OPCIONAL)
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:5601/api/spaces/space/app-space" -H "kbn-xsrf: true"
curl -s -k -u elastic:ElasticLabs2024! -X DELETE "https://localhost:5601/api/spaces/space/ops-space" -H "kbn-xsrf: true"

# Los reportes se conservan en /opt/lab-reports/ como evidencia
echo "✅ Limpieza completada. Reportes preservados en /opt/lab-reports/"
```

---

## Resumen

En este laboratorio integrador has completado el ciclo completo de un proyecto de gestión centralizada de logs:

| Fase | Logro |
|------|-------|
| **Diseño** | Documentaste la arquitectura, flujos de ingestión y decisiones de diseño |
| **Implementación** | Desplegaste tres fuentes, configuraste Logstash con parsing ECS, creaste ILM y templates |
| **Seguridad** | Implementaste acceso diferenciado con roles, usuarios y Kibana Spaces |
| **Validación** | Ejecutaste validación automática de 15 criterios de aceptación |
| **Operación** | Probaste resiliencia ante fallos de componentes, rotación de logs y picos de tráfico |
| **Documentación** | Generaste presentación técnica con evidencias de cumplimiento |

### Conceptos Clave Aplicados

- **ECS (Elastic Common Schema)**: Normalización de campos para correlación entre fuentes
- **ILM (Index Lifecycle Management)**: Retención diferenciada con fases hot/warm/delete
- **Fleet y Elastic Agent**: Gestión centralizada de recolección de logs
- **Logstash Pipelines**: Parsing personalizado con grok y enriquecimiento de campos
- **RBAC y Kibana Spaces**: Aislamiento de acceso por equipo funcional

### Recursos Adicionales

- [Elastic Common Schema Reference](https://www.elastic.co/guide/en/ecs/current/index.html)
- [ILM Policy Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/ilm-policy-definition.html)
- [Fleet and Elastic Agent Guide](https://www.elastic.co/guide/en/fleet/8.14/index.html)
- [Kibana Spaces](https://www.elastic.co/guide/en/kibana/8.14/xpack-spaces.html)
