# Configurar acceso diferenciado para equipos de aplicaciones y operaciones

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Versión Stack** | Elastic Stack 8.14.1 |

## Descripción General

En este laboratorio implementarás una capa completa de seguridad y segregación de acceso sobre el entorno Elastic Stack existente. Partiendo de un stack funcional donde todos los componentes usan el superusuario `elastic`, configurarás certificados TLS con una CA interna, crearás API keys con privilegios mínimos para agentes de ingestión, definirás roles y usuarios diferenciados para equipos de aplicaciones y operaciones, y establecerás Kibana Spaces aislados para cada equipo. Al finalizar, cada equipo solo podrá acceder a los datos y funcionalidades correspondientes a su responsabilidad.

## Objetivos de Aprendizaje

- [ ] Generar una CA interna y certificados TLS con `elasticsearch-certutil` para cifrar comunicaciones entre todos los componentes del stack
- [ ] Crear API keys con privilegios mínimos para Elastic Agent y Logstash, eliminando el uso de credenciales de superusuario en configuraciones
- [ ] Definir roles personalizados (`app-team-role`, `ops-team-role`) con privilegios granulares sobre índices específicos
- [ ] Configurar Kibana Spaces separados con data views restringidos y asignar usuarios exclusivamente a su espacio correspondiente
- [ ] Verificar que la segregación de acceso funciona correctamente validando permisos positivos y negativos para cada usuario

## Prerrequisitos

### Conocimientos Requeridos

- Entorno del Lab 6 completado con flujo de ingestión operativo
- Comprensión de conceptos TLS: CA, certificados, claves privadas, cadena de confianza
- Familiaridad con el modelo de seguridad de Elasticsearch: realms, roles, usuarios, privilegios
- Experiencia básica con la API REST de Elasticsearch y `curl`

### Acceso Necesario

- Terminal con acceso al directorio `~/elastic-labs/`
- Credenciales del superusuario: `elastic` / `ElasticLabs2024!`
- Puertos accesibles: 9200 (Elasticsearch), 5601 (Kibana), 5044 (Logstash), 8220 (Fleet)
- Conectividad a todos los contenedores Docker en la red `elastic-net`

## Entorno del Laboratorio

### Contenedores Docker Activos

| Contenedor | Servicio | Puerto Host | Red Interna |
|------------|----------|-------------|-------------|
| `es01` | Elasticsearch 8.14.1 | 9200, 9300 | 172.20.0.0/24 |
| `kibana01` | Kibana 8.14.1 | 5601 | 172.20.0.0/24 |
| `logstash01` | Logstash 8.14.1 | 5044, 8080 | 172.20.0.0/24 |
| `fleet-server01` | Fleet Server 8.14.1 | 8220 | 172.20.0.0/24 |

### Verificación Inicial del Entorno

```bash
# Navegar al directorio de trabajo
cd ~/elastic-labs/

# Verificar que todos los contenedores están en ejecución
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Confirmar que Elasticsearch responde con el usuario elastic
curl -s -u elastic:ElasticLabs2024! \
  --cacert config/certs/ca/ca.crt \
  https://localhost:9200/_cluster/health?pretty | jq '.status'
```

**Salida esperada:** El estado del clúster debe ser `green` o `yellow` y todos los contenedores deben mostrar estado `Up`.

---

## Paso 1: Generar CA Interna y Certificados TLS con elasticsearch-certutil

### Objetivo

Crear una Autoridad Certificadora (CA) interna y generar certificados TLS individuales para cada componente del stack, reemplazando los certificados autogenerados existentes.

### Instrucciones

1. Crear el directorio para los nuevos certificados:

```bash
mkdir -p ~/elastic-labs/config/certs/new-ca
```

2. Generar la CA interna del laboratorio dentro del contenedor de Elasticsearch:

```bash
docker exec -it es01 bin/elasticsearch-certutil ca \
  --out /usr/share/elasticsearch/config/certs/new-ca/elastic-labs-ca.p12 \
  --pass "LabsCA2024!"
```

3. Extraer el certificado CA en formato PEM para uso en otros componentes:

```bash
docker exec -it es01 openssl pkcs12 -in /usr/share/elasticsearch/config/certs/new-ca/elastic-labs-ca.p12 \
  -clcerts -nokeys -out /usr/share/elasticsearch/config/certs/new-ca/ca.crt \
  -passin pass:LabsCA2024!

docker exec -it es01 openssl pkcs12 -in /usr/share/elasticsearch/config/certs/new-ca/elastic-labs-ca.p12 \
  -nocerts -nodes -out /usr/share/elasticsearch/config/certs/new-ca/ca.key \
  -passin pass:LabsCA2024!
```

4. Crear un archivo de instancias para generar certificados individuales:

```bash
cat > ~/elastic-labs/config/certs/instances.yml << 'EOF'
instances:
  - name: es01
    dns:
      - es01
      - localhost
    ip:
      - 127.0.0.1
      - 172.20.0.2
  - name: kibana01
    dns:
      - kibana01
      - localhost
    ip:
      - 127.0.0.1
      - 172.20.0.3
  - name: logstash01
    dns:
      - logstash01
      - localhost
    ip:
      - 127.0.0.1
      - 172.20.0.4
  - name: fleet-server01
    dns:
      - fleet-server01
      - localhost
    ip:
      - 127.0.0.1
      - 172.20.0.5
EOF
```

5. Copiar el archivo de instancias al contenedor y generar los certificados:

```bash
docker cp ~/elastic-labs/config/certs/instances.yml \
  es01:/usr/share/elasticsearch/config/certs/instances.yml

docker exec -it es01 bin/elasticsearch-certutil cert \
  --ca /usr/share/elasticsearch/config/certs/new-ca/elastic-labs-ca.p12 \
  --ca-pass "LabsCA2024!" \
  --in /usr/share/elasticsearch/config/certs/instances.yml \
  --out /usr/share/elasticsearch/config/certs/new-ca/certs.zip \
  --pass ""
```

6. Extraer los certificados generados:

```bash
docker exec -it es01 bash -c "cd /usr/share/elasticsearch/config/certs/new-ca && unzip -o certs.zip"
```

7. Copiar los certificados al host para distribución a otros contenedores:

```bash
docker cp es01:/usr/share/elasticsearch/config/certs/new-ca/ \
  ~/elastic-labs/config/certs/new-ca/
```

8. Verificar la estructura de certificados generados:

```bash
ls -la ~/elastic-labs/config/certs/new-ca/*/
```

### Salida Esperada

```
~/elastic-labs/config/certs/new-ca/es01/:
-rw-r--r-- 1 root root  ... es01.p12

~/elastic-labs/config/certs/new-ca/kibana01/:
-rw-r--r-- 1 root root  ... kibana01.p12

~/elastic-labs/config/certs/new-ca/logstash01/:
-rw-r--r-- 1 root root  ... logstash01.p12

~/elastic-labs/config/certs/new-ca/fleet-server01/:
-rw-r--r-- 1 root root  ... fleet-server01.p12
```

### Verificación

```bash
# Verificar que el certificado de la CA es válido
openssl x509 -in ~/elastic-labs/config/certs/new-ca/ca.crt -noout -subject -issuer -dates

# Verificar que un certificado de instancia está firmado por la CA
openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/es01/es01.p12 \
  -clcerts -nokeys -passin pass:"" 2>/dev/null | \
  openssl x509 -noout -subject -issuer
```

La salida debe mostrar que el `issuer` coincide con el `subject` de la CA.

---

## Paso 2: Aplicar Certificados TLS a Elasticsearch y Verificar Comunicación

### Objetivo

Configurar Elasticsearch para usar los nuevos certificados TLS generados con la CA interna y verificar que las comunicaciones cifradas funcionan correctamente.

### Instrucciones

1. Extraer certificados PEM del PKCS12 de Elasticsearch para mayor compatibilidad:

```bash
openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/es01/es01.p12 \
  -clcerts -nokeys -out ~/elastic-labs/config/certs/new-ca/es01/es01.crt \
  -passin pass:""

openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/es01/es01.p12 \
  -nocerts -nodes -out ~/elastic-labs/config/certs/new-ca/es01/es01.key \
  -passin pass:""
```

2. Actualizar el `docker-compose.yml` para montar los nuevos certificados en Elasticsearch. Añadir los volúmenes necesarios en el servicio `es01`:

```yaml
# En ~/elastic-labs/docker-compose.yml, sección es01 > volumes, agregar:
    volumes:
      - ./config/certs/new-ca/ca.crt:/usr/share/elasticsearch/config/certs/ca.crt:ro
      - ./config/certs/new-ca/es01/es01.crt:/usr/share/elasticsearch/config/certs/es01.crt:ro
      - ./config/certs/new-ca/es01/es01.key:/usr/share/elasticsearch/config/certs/es01.key:ro
      - esdata01:/usr/share/elasticsearch/data
```

3. Actualizar la configuración de seguridad de Elasticsearch. Crear o modificar el archivo de configuración:

```bash
cat > ~/elastic-labs/config/elasticsearch.yml << 'EOF'
cluster.name: elastic-labs-cluster
node.name: es01
network.host: 0.0.0.0

xpack.security.enabled: true

# TLS - Transport layer (entre nodos)
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.certificate: certs/es01.crt
xpack.security.transport.ssl.key: certs/es01.key
xpack.security.transport.ssl.certificate_authorities: certs/ca.crt

# TLS - HTTP layer (clientes externos)
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.certificate: certs/es01.crt
xpack.security.http.ssl.key: certs/es01.key
xpack.security.http.ssl.certificate_authorities: certs/ca.crt
EOF
```

4. Montar el archivo de configuración en el contenedor (actualizar `docker-compose.yml`):

```yaml
    volumes:
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - ./config/certs/new-ca/ca.crt:/usr/share/elasticsearch/config/certs/ca.crt:ro
      - ./config/certs/new-ca/es01/es01.crt:/usr/share/elasticsearch/config/certs/es01.crt:ro
      - ./config/certs/new-ca/es01/es01.key:/usr/share/elasticsearch/config/certs/es01.key:ro
      - esdata01:/usr/share/elasticsearch/data
```

5. Reiniciar Elasticsearch con la nueva configuración:

```bash
cd ~/elastic-labs/
docker compose up -d es01
docker compose logs -f es01 | head -50
```

6. Esperar a que Elasticsearch esté disponible (aproximadamente 30-60 segundos) y verificar:

```bash
# Verificar conectividad con el nuevo certificado CA
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  https://localhost:9200/_cluster/health?pretty
```

### Salida Esperada

```json
{
  "cluster_name" : "elastic-labs-cluster",
  "status" : "yellow",
  "number_of_nodes" : 1,
  "active_primary_shards" : ...,
  ...
}
```

### Verificación

```bash
# Verificar la cadena de certificados TLS
openssl s_client -connect localhost:9200 \
  -CAfile ~/elastic-labs/config/certs/new-ca/ca.crt \
  </dev/null 2>/dev/null | grep "Verify return code"
```

La salida debe mostrar: `Verify return code: 0 (ok)`

---

## Paso 3: Configurar TLS en Kibana y Logstash

### Objetivo

Actualizar la configuración de Kibana y Logstash para que se comuniquen con Elasticsearch usando los nuevos certificados TLS de la CA interna.

### Instrucciones

1. Extraer certificados PEM para Kibana:

```bash
openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/kibana01/kibana01.p12 \
  -clcerts -nokeys -out ~/elastic-labs/config/certs/new-ca/kibana01/kibana01.crt \
  -passin pass:""

openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/kibana01/kibana01.p12 \
  -nocerts -nodes -out ~/elastic-labs/config/certs/new-ca/kibana01/kibana01.key \
  -passin pass:""
```

2. Crear la configuración de Kibana con TLS:

```bash
cat > ~/elastic-labs/config/kibana.yml << 'EOF'
server.name: kibana01
server.host: "0.0.0.0"
server.port: 5601

# HTTPS para el servidor Kibana
server.ssl.enabled: true
server.ssl.certificate: /usr/share/kibana/config/certs/kibana01.crt
server.ssl.key: /usr/share/kibana/config/certs/kibana01.key

# Conexión a Elasticsearch con TLS
elasticsearch.hosts: ["https://es01:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "KibanaLabs2024!"
elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/config/certs/ca.crt"]
elasticsearch.ssl.verificationMode: certificate

xpack.security.encryptionKey: "algo-de-32-caracteres-minimo-ok!"
xpack.encryptedSavedObjects.encryptionKey: "algo-de-32-caracteres-minimo-ok!"
xpack.reporting.encryptionKey: "algo-de-32-caracteres-minimo-ok!"
EOF
```

3. Actualizar los volúmenes de Kibana en `docker-compose.yml`:

```yaml
  kibana01:
    volumes:
      - ./config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro
      - ./config/certs/new-ca/ca.crt:/usr/share/kibana/config/certs/ca.crt:ro
      - ./config/certs/new-ca/kibana01/kibana01.crt:/usr/share/kibana/config/certs/kibana01.crt:ro
      - ./config/certs/new-ca/kibana01/kibana01.key:/usr/share/kibana/config/certs/kibana01.key:ro
```

4. Extraer certificados PEM para Logstash y crear configuración:

```bash
openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/logstash01/logstash01.p12 \
  -clcerts -nokeys -out ~/elastic-labs/config/certs/new-ca/logstash01/logstash01.crt \
  -passin pass:""

openssl pkcs12 -in ~/elastic-labs/config/certs/new-ca/logstash01/logstash01.p12 \
  -nocerts -nodes -out ~/elastic-labs/config/certs/new-ca/logstash01/logstash01.key \
  -passin pass:""
```

5. Actualizar los volúmenes de Logstash en `docker-compose.yml`:

```yaml
  logstash01:
    volumes:
      - ./config/logstash/pipeline/:/usr/share/logstash/pipeline/:ro
      - ./config/certs/new-ca/ca.crt:/usr/share/logstash/config/certs/ca.crt:ro
      - ./config/certs/new-ca/logstash01/logstash01.crt:/usr/share/logstash/config/certs/logstash01.crt:ro
      - ./config/certs/new-ca/logstash01/logstash01.key:/usr/share/logstash/config/certs/logstash01.key:ro
```

6. Reiniciar Kibana y Logstash:

```bash
cd ~/elastic-labs/
docker compose up -d kibana01 logstash01
```

7. Esperar 60 segundos y verificar que Kibana responde por HTTPS:

```bash
# Verificar Kibana con HTTPS
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  https://localhost:5601/api/status | jq '.status.overall.level'
```

### Salida Esperada

```
"available"
```

### Verificación

```bash
# Verificar que Logstash se conecta correctamente a Elasticsearch
docker logs logstash01 2>&1 | grep -i "Elasticsearch pool URLs" | tail -1

# Verificar certificado TLS de Kibana
openssl s_client -connect localhost:5601 \
  -CAfile ~/elastic-labs/config/certs/new-ca/ca.crt \
  </dev/null 2>/dev/null | grep "Verify return code"
```

---

## Paso 4: Crear API Keys con Privilegios Mínimos

### Objetivo

Crear API keys específicas para Elastic Agent y Logstash con permisos restringidos, eliminando el uso del superusuario `elastic` en configuraciones de agentes de ingestión.

### Instrucciones

1. Crear la API key para Elastic Agent (escritura solo sobre `logs-*` y `metrics-*`):

```bash
curl -s -X POST "https://localhost:9200/_security/api_key" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "name": "elastic-agent-ingest-key",
    "expiration": "365d",
    "role_descriptors": {
      "agent_writer": {
        "cluster": ["monitor"],
        "indices": [
          {
            "names": ["logs-*", "metrics-*"],
            "privileges": ["write", "create_index", "auto_configure"]
          }
        ]
      }
    },
    "metadata": {
      "application": "elastic-agent",
      "environment": "labs"
    }
  }' | jq '.' | tee ~/elastic-labs/config/agent-api-key.json
```

2. Crear la API key para Logstash (permisos sobre `logs-python-app-*` y `labs-logstash-*`):

```bash
curl -s -X POST "https://localhost:9200/_security/api_key" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "name": "logstash-pipeline-key",
    "expiration": "365d",
    "role_descriptors": {
      "logstash_writer": {
        "cluster": ["monitor", "manage_ilm", "manage_index_templates"],
        "indices": [
          {
            "names": ["logs-python-app-*", "labs-logstash-app-*"],
            "privileges": ["write", "create_index", "manage", "auto_configure"]
          }
        ]
      }
    },
    "metadata": {
      "application": "logstash",
      "environment": "labs"
    }
  }' | jq '.' | tee ~/elastic-labs/config/logstash-api-key.json
```

3. Extraer el valor codificado de la API key de Logstash para usarlo en la configuración:

```bash
LOGSTASH_API_KEY=$(jq -r '.encoded' ~/elastic-labs/config/logstash-api-key.json)
echo "API Key de Logstash (encoded): $LOGSTASH_API_KEY"
```

4. Actualizar la pipeline de Logstash para usar la API key en lugar de usuario/contraseña:

```bash
cat > ~/elastic-labs/config/logstash/pipeline/main.conf << EOF
input {
  beats {
    port => 5044
    ssl_enabled => true
    ssl_certificate => "/usr/share/logstash/config/certs/logstash01.crt"
    ssl_key => "/usr/share/logstash/config/certs/logstash01.key"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca.crt"]
  }
  http {
    port => 8080
    codec => json
  }
}

filter {
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }
  mutate {
    add_field => { "[@metadata][target_index]" => "labs-logstash-app-%{+YYYY.MM.dd}" }
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    ssl_enabled => true
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca.crt"]
    api_key => "${LOGSTASH_API_KEY}"
    index => "%{[@metadata][target_index]}"
  }
}
EOF
```

5. Pasar la API key como variable de entorno al contenedor de Logstash. Actualizar `docker-compose.yml`:

```yaml
  logstash01:
    environment:
      - LOGSTASH_API_KEY=${LOGSTASH_API_KEY}
```

6. Exportar la variable y reiniciar Logstash:

```bash
export LOGSTASH_API_KEY=$(jq -r '.encoded' ~/elastic-labs/config/logstash-api-key.json)
cd ~/elastic-labs/
docker compose up -d logstash01
```

7. Verificar que Logstash se conecta correctamente con la API key:

```bash
sleep 20
docker logs logstash01 2>&1 | tail -20 | grep -i "pipeline\|error\|started"
```

### Salida Esperada

```json
// Ejemplo de respuesta de creación de API key
{
  "id": "abc123...",
  "name": "logstash-pipeline-key",
  "api_key": "xyz789...",
  "encoded": "YWJjMTIzLi4uOnh5ejc4OS4uLg=="
}
```

En los logs de Logstash:

```
[INFO ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
```

### Verificación

```bash
# Enviar un evento de prueba a Logstash para confirmar escritura con API key
curl -s -X POST "http://localhost:8080" \
  -H "Content-Type: application/json" \
  -d '{"message": "Test API key auth", "app": "test", "level": "INFO"}'

sleep 5

# Verificar que el documento llegó al índice
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_count" | jq '.count'
```

El conteo debe ser mayor a 0.

---

## Paso 5: Crear Roles Diferenciados para Equipos

### Objetivo

Definir el rol `app-team-role` con privilegios de lectura sobre índices de aplicación y el rol `ops-team-role` con privilegios de gestión completa sobre infraestructura e ILM.

### Instrucciones

1. Crear el rol `app-team-role` (equipo de aplicaciones — solo lectura):

```bash
curl -s -X PUT "https://localhost:9200/_security/role/app-team-role" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "cluster": ["monitor"],
    "indices": [
      {
        "names": ["logs-nginx-*", "logs-python-app-*", "logs-custom-labs*"],
        "privileges": ["read", "view_index_metadata"],
        "allow_restricted_indices": false
      }
    ],
    "applications": [
      {
        "application": "kibana-.kibana",
        "privileges": ["feature_discover.read", "feature_dashboard.read", "feature_visualize.read"],
        "resources": ["space:app-space"]
      }
    ]
  }' | jq '.'
```

2. Crear el rol `ops-team-role` (equipo de operaciones — gestión completa):

```bash
curl -s -X PUT "https://localhost:9200/_security/role/ops-team-role" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "cluster": ["monitor", "manage_ilm", "manage_index_templates", "read_ilm"],
    "indices": [
      {
        "names": ["logs-*", "metrics-*", "labs-*"],
        "privileges": ["read", "write", "manage", "view_index_metadata", "create_index"],
        "allow_restricted_indices": false
      }
    ],
    "applications": [
      {
        "application": "kibana-.kibana",
        "privileges": ["feature_discover.all", "feature_dashboard.all", "feature_visualize.all", "feature_indexPatterns.all", "feature_stackAlerts.all"],
        "resources": ["space:ops-space"]
      }
    ]
  }' | jq '.'
```

3. Crear el usuario `app-user-01` asignado al rol de aplicaciones:

```bash
curl -s -X POST "https://localhost:9200/_security/user/app-user-01" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "password": "AppTeam2024!",
    "roles": ["app-team-role"],
    "full_name": "Desarrollador - Equipo Aplicaciones",
    "email": "app-team@empresa.com",
    "metadata": {
      "team": "applications",
      "department": "engineering"
    }
  }' | jq '.'
```

4. Crear el usuario `ops-user-01` asignado al rol de operaciones:

```bash
curl -s -X POST "https://localhost:9200/_security/user/ops-user-01" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "password": "OpsTeam2024!",
    "roles": ["ops-team-role"],
    "full_name": "Ingeniero - Equipo Operaciones",
    "email": "ops-team@empresa.com",
    "metadata": {
      "team": "operations",
      "department": "infrastructure"
    }
  }' | jq '.'
```

5. Verificar la creación de roles y usuarios:

```bash
# Listar roles personalizados
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_security/role/app-team-role,ops-team-role" | jq 'keys'

# Listar usuarios creados
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_security/user/app-user-01,ops-user-01" | jq 'keys'
```

### Salida Esperada

```json
["app-team-role", "ops-team-role"]
["app-user-01", "ops-user-01"]
```

### Verificación

```bash
# Verificar que app-user-01 puede leer logs-nginx-* (si existe el índice)
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u app-user-01:AppTeam2024! \
  "https://localhost:9200/logs-nginx-*/_search?size=0" | jq '.hits.total'

# Verificar que app-user-01 NO puede leer metrics-*
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u app-user-01:AppTeam2024! \
  "https://localhost:9200/metrics-*/_search?size=0" | jq '.status'
```

El primer comando debe retornar un total (puede ser 0 si no hay datos), y el segundo debe retornar status `403`.

---

## Paso 6: Crear Kibana Spaces Separados

### Objetivo

Configurar dos Kibana Spaces aislados (`app-space` y `ops-space`) con data views restringidos a los índices correspondientes de cada equipo.

### Instrucciones

1. Crear el Space `app-space` para el equipo de aplicaciones:

```bash
curl -s -X POST "https://localhost:5601/api/spaces/space" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "id": "app-space",
    "name": "Aplicaciones",
    "description": "Espacio para el equipo de desarrollo de aplicaciones",
    "color": "#00BFB3",
    "initials": "AP",
    "disabledFeatures": ["infrastructure", "apm", "uptime", "siem", "fleet"]
  }' | jq '.'
```

2. Crear el Space `ops-space` para el equipo de operaciones:

```bash
curl -s -X POST "https://localhost:5601/api/spaces/space" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "id": "ops-space",
    "name": "Operaciones",
    "description": "Espacio para el equipo de operaciones e infraestructura",
    "color": "#DB1374",
    "initials": "OP",
    "disabledFeatures": []
  }' | jq '.'
```

3. Crear un Data View en `app-space` restringido a índices de aplicación:

```bash
curl -s -X POST "https://localhost:5601/s/app-space/api/data_views/data_view" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "data_view": {
      "title": "logs-app-*",
      "name": "Logs de Aplicaciones",
      "timeFieldName": "@timestamp"
    }
  }' | jq '.data_view.id'
```

4. Crear un Data View en `ops-space` con acceso a todos los logs y métricas:

```bash
curl -s -X POST "https://localhost:5601/s/ops-space/api/data_views/data_view" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "data_view": {
      "title": "logs-*",
      "name": "Todos los Logs",
      "timeFieldName": "@timestamp"
    }
  }' | jq '.data_view.id'

curl -s -X POST "https://localhost:5601/s/ops-space/api/data_views/data_view" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "data_view": {
      "title": "metrics-*",
      "name": "Métricas de Infraestructura",
      "timeFieldName": "@timestamp"
    }
  }' | jq '.data_view.id'
```

5. Verificar que los Spaces fueron creados correctamente:

```bash
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/spaces/space" \
  -H "kbn-xsrf: true" | jq '.[].id'
```

### Salida Esperada

```
"default"
"app-space"
"ops-space"
```

### Verificación

```bash
# Verificar que app-user-01 puede acceder a app-space
curl -s -o /dev/null -w "%{http_code}" \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u app-user-01:AppTeam2024! \
  "https://localhost:5601/s/app-space/api/data_views" \
  -H "kbn-xsrf: true"

# Verificar que app-user-01 NO puede acceder a ops-space
curl -s -o /dev/null -w "%{http_code}" \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u app-user-01:AppTeam2024! \
  "https://localhost:5601/s/ops-space/api/data_views" \
  -H "kbn-xsrf: true"
```

El primer comando debe retornar `200` y el segundo `403`.

---

## Paso 7: Validación Integral de Segregación de Acceso

### Objetivo

Ejecutar una batería completa de pruebas para confirmar que la segregación de acceso funciona correctamente en todos los niveles: índices, API keys, roles y Kibana Spaces.

### Instrucciones

1. Crear un script de validación completo:

```bash
cat > ~/elastic-labs/scripts/validate-access.sh << 'SCRIPT'
#!/bin/bash
CA_CERT=~/elastic-labs/config/certs/new-ca/ca.crt
ES_URL="https://localhost:9200"
KB_URL="https://localhost:5601"

PASS=0
FAIL=0

check() {
  local desc="$1"
  local expected="$2"
  local actual="$3"
  if [ "$actual" == "$expected" ]; then
    echo "  ✅ PASS: $desc (esperado: $expected, obtenido: $actual)"
    ((PASS++))
  else
    echo "  ❌ FAIL: $desc (esperado: $expected, obtenido: $actual)"
    ((FAIL++))
  fi
}

echo "═══════════════════════════════════════════════════"
echo " VALIDACIÓN DE SEGREGACIÓN DE ACCESO"
echo "═══════════════════════════════════════════════════"

echo ""
echo "── Test 1: app-user-01 - Acceso a índices de aplicación ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u app-user-01:AppTeam2024! "$ES_URL/logs-nginx-*/_search?size=0")
check "app-user-01 lee logs-nginx-*" "200" "$status"

echo ""
echo "── Test 2: app-user-01 - Denegación a índices de operaciones ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u app-user-01:AppTeam2024! "$ES_URL/metrics-*/_search?size=0")
check "app-user-01 NO lee metrics-*" "403" "$status"

echo ""
echo "── Test 3: ops-user-01 - Acceso a todos los logs ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u ops-user-01:OpsTeam2024! "$ES_URL/logs-*/_search?size=0")
check "ops-user-01 lee logs-*" "200" "$status"

echo ""
echo "── Test 4: ops-user-01 - Acceso a métricas ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u ops-user-01:OpsTeam2024! "$ES_URL/metrics-*/_search?size=0")
check "ops-user-01 lee metrics-*" "200" "$status"

echo ""
echo "── Test 5: ops-user-01 - Gestión de ILM ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u ops-user-01:OpsTeam2024! "$ES_URL/_ilm/policy")
check "ops-user-01 lee políticas ILM" "200" "$status"

echo ""
echo "── Test 6: app-user-01 - Denegación de gestión ILM ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u app-user-01:AppTeam2024! "$ES_URL/_ilm/policy")
check "app-user-01 NO gestiona ILM" "403" "$status"

echo ""
echo "── Test 7: Kibana Spaces - app-user-01 en app-space ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u app-user-01:AppTeam2024! \
  "$KB_URL/s/app-space/api/data_views" -H "kbn-xsrf: true")
check "app-user-01 accede app-space" "200" "$status"

echo ""
echo "── Test 8: Kibana Spaces - app-user-01 bloqueado en ops-space ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -u app-user-01:AppTeam2024! \
  "$KB_URL/s/ops-space/api/data_views" -H "kbn-xsrf: true")
check "app-user-01 NO accede ops-space" "403" "$status"

echo ""
echo "── Test 9: API Key de Logstash - Escritura permitida ──"
LOGSTASH_KEY=$(jq -r '.encoded' ~/elastic-labs/config/logstash-api-key.json)
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -H "Authorization: ApiKey $LOGSTASH_KEY" \
  -H "Content-Type: application/json" \
  -X POST "$ES_URL/labs-logstash-app-test/_doc" \
  -d '{"@timestamp":"2024-01-01T00:00:00Z","message":"validation test"}')
check "API key Logstash escribe en labs-logstash-app-*" "201" "$status"

echo ""
echo "── Test 10: API Key de Logstash - Lectura denegada en otros índices ──"
status=$(curl -s -o /dev/null -w "%{http_code}" --cacert $CA_CERT \
  -H "Authorization: ApiKey $LOGSTASH_KEY" \
  "$ES_URL/.security/_search?size=0")
check "API key Logstash NO lee .security" "403" "$status"

echo ""
echo "═══════════════════════════════════════════════════"
echo " RESULTADOS: $PASS pasaron, $FAIL fallaron"
echo "═══════════════════════════════════════════════════"
SCRIPT

chmod +x ~/elastic-labs/scripts/validate-access.sh
```

2. Ejecutar el script de validación:

```bash
~/elastic-labs/scripts/validate-access.sh
```

### Salida Esperada

```
═══════════════════════════════════════════════════
 VALIDACIÓN DE SEGREGACIÓN DE ACCESO
═══════════════════════════════════════════════════

── Test 1: app-user-01 - Acceso a índices de aplicación ──
  ✅ PASS: app-user-01 lee logs-nginx-* (esperado: 200, obtenido: 200)

── Test 2: app-user-01 - Denegación a índices de operaciones ──
  ✅ PASS: app-user-01 NO lee metrics-* (esperado: 403, obtenido: 403)

── Test 3: ops-user-01 - Acceso a todos los logs ──
  ✅ PASS: ops-user-01 lee logs-* (esperado: 200, obtenido: 200)

...

═══════════════════════════════════════════════════
 RESULTADOS: 10 pasaron, 0 fallaron
═══════════════════════════════════════════════════
```

### Verificación

Todos los tests deben mostrar `✅ PASS`. Si algún test falla, revisar los pasos anteriores correspondientes al componente que falla.

---

## Validación y Testing

### Prueba Funcional Completa

Realizar las siguientes verificaciones manuales adicionales:

```bash
# 1. Verificar que TLS está activo en todos los endpoints
echo "=== Certificado Elasticsearch ==="
echo | openssl s_client -connect localhost:9200 2>/dev/null | \
  openssl x509 -noout -subject -dates

echo "=== Certificado Kibana ==="
echo | openssl s_client -connect localhost:5601 2>/dev/null | \
  openssl x509 -noout -subject -dates

# 2. Listar API keys activas
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_security/api_key?pretty" | \
  jq '.api_keys[] | {name: .name, id: .id, invalidated: .invalidated}'

# 3. Verificar que Logstash procesa eventos con la API key
curl -s -X POST "http://localhost:8080" \
  -H "Content-Type: application/json" \
  -d '{"message":"Final validation event","level":"INFO","app":"validator"}'
sleep 5
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search?q=message:Final+validation&size=1" | \
  jq '.hits.hits[0]._source.message'
```

### Criterios de Aceptación

| Criterio | Método de Verificación | Resultado Esperado |
|----------|----------------------|-------------------|
| TLS activo en Elasticsearch | `openssl s_client -connect localhost:9200` | Certificado válido firmado por CA interna |
| TLS activo en Kibana | `openssl s_client -connect localhost:5601` | Certificado válido firmado por CA interna |
| API key Logstash funcional | Envío de evento y consulta en índice | Documento indexado correctamente |
| app-user-01 solo lee logs de app | Consulta a `logs-nginx-*` y `metrics-*` | 200 y 403 respectivamente |
| ops-user-01 gestiona infraestructura | Consulta a `logs-*`, `metrics-*` e ILM | 200 en todos |
| Kibana Spaces aislados | Acceso cruzado entre spaces | 403 al acceder space ajeno |

---

## Solución de Problemas

### Problema 1: Logstash no se conecta a Elasticsearch con la API key

**Síntomas:**
- Los logs de Logstash muestran: `[ERROR] Encountered a retryable error` o `401 Unauthorized`
- No se indexan nuevos documentos en `labs-logstash-app-*`
- El pipeline aparece como `running` pero los eventos se acumulan en el dead letter queue

**Causa:**
La variable de entorno `LOGSTASH_API_KEY` no se pasó correctamente al contenedor, o el valor codificado de la API key tiene caracteres especiales que fueron interpretados por el shell. Otra causa común es que la API key fue creada con `role_descriptors` que no incluyen el índice exacto al que Logstash intenta escribir.

**Solución:**

```bash
# 1. Verificar que la variable está disponible dentro del contenedor
docker exec logstash01 env | grep LOGSTASH_API_KEY

# 2. Si está vacía, verificar el valor en el host
echo $LOGSTASH_API_KEY

# 3. Si el valor es correcto pero Logstash no lo reconoce, 
#    hardcodear temporalmente en la pipeline para diagnóstico
docker exec -it logstash01 cat /usr/share/logstash/pipeline/main.conf | grep api_key

# 4. Verificar que la API key es válida
API_KEY=$(jq -r '.encoded' ~/elastic-labs/config/logstash-api-key.json)
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -H "Authorization: ApiKey $API_KEY" \
  "https://localhost:9200/_security/_authenticate" | jq '.username'

# 5. Si la API key no tiene permisos sobre el índice correcto, recrearla
#    con los nombres de índices exactos que usa la pipeline
```

### Problema 2: El usuario app-user-01 recibe error 403 al acceder a app-space en Kibana

**Síntomas:**
- Al hacer login en Kibana con `app-user-01`, el usuario ve un error "You do not have permission to access the requested page"
- La API de Elasticsearch permite la lectura de índices correctamente con el mismo usuario
- El Space `app-space` existe y es accesible con el usuario `elastic`

**Causa:**
El rol `app-team-role` no tiene configurados correctamente los privilegios de Kibana para el space `app-space`. Los privilegios de aplicación en Elasticsearch (`applications` en el role descriptor) deben coincidir exactamente con el identificador del space (`space:app-space`) y las features habilitadas deben incluir al menos `feature_discover.read` para poder acceder al space.

**Solución:**

```bash
# 1. Verificar la configuración actual del rol
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_security/role/app-team-role" | jq '.app-team-role.applications'

# 2. Si el campo applications está vacío o incorrecto, actualizar el rol
curl -s -X PUT "https://localhost:9200/_security/role/app-team-role" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{
    "cluster": ["monitor"],
    "indices": [
      {
        "names": ["logs-nginx-*", "logs-python-app-*", "logs-custom-labs*"],
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
  }'

# 3. Verificar que el usuario tiene el rol asignado
curl -s --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_security/user/app-user-01" | jq '.["app-user-01"].roles'

# 4. Probar el acceso nuevamente
curl -s -o /dev/null -w "%{http_code}" \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -u app-user-01:AppTeam2024! \
  "https://localhost:5601/s/app-space/api/data_views" \
  -H "kbn-xsrf: true"
```

---

## Limpieza

Si necesitas revertir los cambios realizados en este laboratorio (por ejemplo, para repetirlo desde cero):

```bash
cd ~/elastic-labs/

# Revocar las API keys creadas
curl -s -X DELETE "https://localhost:9200/_security/api_key" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{"name": "logstash-pipeline-key"}'

curl -s -X DELETE "https://localhost:9200/_security/api_key" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt \
  -d '{"name": "elastic-agent-ingest-key"}'

# Eliminar usuarios y roles
curl -s -X DELETE "https://localhost:9200/_security/user/app-user-01" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

curl -s -X DELETE "https://localhost:9200/_security/user/ops-user-01" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

curl -s -X DELETE "https://localhost:9200/_security/role/app-team-role" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

curl -s -X DELETE "https://localhost:9200/_security/role/ops-team-role" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

# Eliminar Kibana Spaces (esto elimina también los data views dentro)
curl -s -X DELETE "https://localhost:5601/api/spaces/space/app-space" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

curl -s -X DELETE "https://localhost:5601/api/spaces/space/ops-space" \
  -H "kbn-xsrf: true" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

# Eliminar índice de prueba
curl -s -X DELETE "https://localhost:9200/labs-logstash-app-test" \
  -u elastic:ElasticLabs2024! \
  --cacert ~/elastic-labs/config/certs/new-ca/ca.crt

# Eliminar archivos de certificados generados
rm -rf ~/elastic-labs/config/certs/new-ca/
rm -f ~/elastic-labs/config/agent-api-key.json
rm -f ~/elastic-labs/config/logstash-api-key.json

# Limpiar variable de entorno
unset LOGSTASH_API_KEY
```

> **Nota:** Si deseas conservar el entorno seguro para los laboratorios siguientes, NO ejecutes la limpieza. Los labs posteriores pueden depender de esta configuración de seguridad.

---

## Resumen

En este laboratorio has implementado una capa completa de seguridad y segregación de acceso en Elastic Stack:

| Componente | Configuración Aplicada |
|------------|----------------------|
| **TLS** | CA interna con `elasticsearch-certutil`, certificados PEM para ES, Kibana y Logstash |
| **API Keys** | `elastic-agent-ingest-key` (escritura logs/metrics), `logstash-pipeline-key` (escritura labs-logstash-app-*) |
| **Roles** | `app-team-role` (lectura app logs), `ops-team-role` (gestión completa infra + ILM) |
| **Usuarios** | `app-user-01` → app-team-role, `ops-user-01` → ops-team-role |
| **Kibana Spaces** | `app-space` (data view logs-app-*), `ops-space` (data views logs-* y metrics-*) |

**Principios clave aplicados:**

- **Mínimo privilegio:** Cada agente y usuario tiene solo los permisos estrictamente necesarios para su función
- **Defensa en profundidad:** TLS cifra las comunicaciones, API keys autentican servicios, roles limitan acceso a datos
- **Segregación de responsabilidades:** Cada equipo opera en su propio espacio sin interferencia ni visibilidad cruzada

### Recursos Adicionales

- [Elasticsearch Security API Reference](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/security-api.html)
- [Kibana Spaces Documentation](https://www.elastic.co/guide/en/kibana/8.14/xpack-spaces.html)
- [elasticsearch-certutil Reference](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/certutil.html)
- [API Keys Management](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/security-api-create-api-key.html)
