# Identificar los componentes del entorno y validar el flujo de datos de una fuente de logs

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 34 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Aplicar |

## Descripción General

En esta práctica inaugural levantarás el entorno completo del Elastic Stack 8.14.1 mediante Docker Compose, inspeccionarás cada componente en ejecución (Elasticsearch, Kibana, Logstash, Fleet Server y Filebeat 7.17.22 legacy), verificarás la salud del clúster a través de la API REST y, finalmente, inyectarás un archivo de logs de muestra de Nginx para observar el flujo completo de datos desde la fuente hasta su visualización en Kibana. Al concluir, el entorno quedará operativo como base para todas las prácticas subsecuentes del curso.

## Objetivos de Aprendizaje

- [ ] Identificar y describir el rol de cada componente del Elastic Stack (Elasticsearch, Kibana, Logstash, Beats, Elastic Agent y Fleet Server) observando los contenedores desplegados en el entorno de laboratorio.
- [ ] Validar que el flujo de datos de una fuente de logs de muestra (nginx_access_sample.log) llega correctamente desde Logstash hasta su indexación en Elasticsearch y visualización en Kibana.
- [ ] Distinguir las diferencias arquitectónicas entre Filebeat 7.17.22 (legacy) y el modelo de Elastic Agent + Fleet en la versión 8.14.1 mediante observación directa del entorno.
- [ ] Documentar los puertos, credenciales y nombres de contenedores del entorno para referencia en prácticas posteriores.

## Prerrequisitos

### Conocimientos previos

- Comandos básicos de Linux (navegación de directorios, edición de archivos, lectura de logs).
- Uso de Docker: `docker ps`, `docker logs`, `docker exec`.
- Consultas HTTP básicas con `curl`.
- Concepto general de qué es un índice en Elasticsearch y qué es un pipeline en Logstash.

### Acceso requerido

- Sistema operativo Ubuntu 22.04 LTS con acceso de usuario con privilegios `sudo`.
- Docker Engine 26.1.4 y Docker Compose 2.27.1 instalados y funcionales.
- Repositorio del curso clonado en `~/elastic-labs/`.
- Conexión a Internet para descarga de imágenes Docker (primera ejecución).
- Puertos 9200, 9300, 5601, 5044, 8080, 8220 libres en el host.

## Entorno de Laboratorio

### Recursos de hardware mínimos

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 4 núcleos | 8 núcleos |
| RAM | 16 GB | 24 GB |
| Disco SSD | 60 GB libres | 80 GB libres |

### Software del entorno

| Componente | Versión | Contenedor |
|------------|---------|------------|
| Elasticsearch | 8.14.1 | `es01` |
| Kibana | 8.14.1 | `kibana01` |
| Logstash | 8.14.1 | `logstash01` |
| Fleet Server | 8.14.1 | `fleet-server01` |
| Filebeat (legacy) | 7.17.22 | `filebeat-legacy01` |
| Red Docker | bridge | `elastic-net` (172.20.0.0/24) |

### Credenciales del stack

| Usuario | Contraseña | Uso |
|---------|------------|-----|
| `elastic` | `ElasticLabs2024!` | Administrador de Elasticsearch y Kibana |
| `kibana_system` | `KibanaLabs2024!` | Comunicación interna Kibana ↔ Elasticsearch |

### Preparación inicial

Asegúrate de que el repositorio del curso está clonado y actualizado:

```bash
cd ~
ls elastic-labs/ || git clone https://github.com/elastic-training/elastic-labs.git elastic-labs
cd ~/elastic-labs
git pull origin main
```

Verifica que Docker está operativo:

```bash
docker --version
docker compose version
```

**Salida esperada:**

```
Docker version 26.1.4, build ...
Docker Compose version v2.27.1
```

---

## Paso a Paso

### Paso 1: Levantar el entorno con Docker Compose

**Objetivo:** Iniciar todos los servicios del Elastic Stack definidos en el archivo `docker-compose.yml` del repositorio y confirmar que los contenedores arrancan sin errores.

**Instrucciones:**

1. Navega al directorio de configuración del laboratorio:

```bash
cd ~/elastic-labs/config
```

2. Inicia los servicios en modo *detached*:

```bash
docker compose up -d
```

3. Espera aproximadamente 60–90 segundos para que Elasticsearch complete su inicialización (generación de certificados TLS, creación de usuarios internos). Monitorea el progreso:

```bash
docker compose logs -f es01 --since 1m
```

Presiona `Ctrl+C` cuando veas la línea que indica que el nodo está listo:

```
"message": "started ..."
```

4. Verifica que todos los contenedores están en estado `running`:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Salida esperada:**

```
NAMES               STATUS          PORTS
es01                Up X minutes    0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp
kibana01            Up X minutes    0.0.0.0:5601->5601/tcp
logstash01          Up X minutes    0.0.0.0:5044->5044/tcp, 0.0.0.0:8080->8080/tcp
fleet-server01      Up X minutes    0.0.0.0:8220->8220/tcp
filebeat-legacy01   Up X minutes    
```

**Verificación:** Los cinco contenedores deben aparecer con estado `Up`. Si alguno muestra `Restarting` o `Exited`, revisa sus logs con `docker logs <nombre_contenedor>`.

---

### Paso 2: Verificar la salud del clúster de Elasticsearch

**Objetivo:** Confirmar que el clúster de Elasticsearch está operativo y responde a consultas mediante la API REST, utilizando autenticación y TLS.

**Instrucciones:**

1. Consulta el endpoint `_cluster/health` con credenciales de administrador:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:9200/_cluster/health | jq .
```

> **Nota:** La opción `-k` omite la verificación del certificado TLS autofirmado. En producción se usaría el CA certificate.

2. Observa los campos clave de la respuesta:

**Salida esperada:**

```json
{
  "cluster_name": "docker-cluster",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 1,
  "number_of_data_nodes": 1,
  "active_primary_shards": ...,
  "active_shards": ...,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  ...
}
```

3. Verifica la versión del nodo:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:9200 | jq '{name: .name, version: .version.number, tagline: .tagline}'
```

**Salida esperada:**

```json
{
  "name": "es01",
  "version": "8.14.1",
  "tagline": "You Know, for Search"
}
```

4. Lista los índices del sistema para confirmar que el clúster tiene datos internos inicializados:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:9200/_cat/indices?v&h=index,health,status,docs.count | head -20
```

**Verificación:** El campo `"status"` debe ser `"green"` o `"yellow"` (yellow es aceptable en un clúster de un solo nodo porque las réplicas no pueden asignarse). La versión debe ser `8.14.1`.

---

### Paso 3: Explorar Kibana e identificar los componentes del stack

**Objetivo:** Acceder a la interfaz web de Kibana y localizar las secciones correspondientes a cada componente del Elastic Stack: Fleet, Stack Monitoring, Discover e Integrations.

**Instrucciones:**

1. Abre un navegador web y accede a:

```
https://localhost:5601
```

2. Acepta la advertencia del certificado autofirmado y autentícate con:
   - **Usuario:** `elastic`
   - **Contraseña:** `ElasticLabs2024!`

3. Una vez dentro del dashboard principal de Kibana, navega a las siguientes secciones y anota lo que observas:

   **a) Stack Monitoring:**
   - Menú lateral → **Management** → **Stack Monitoring**
   - Observa los nodos de Elasticsearch, instancias de Kibana y Logstash reportadas.

   **b) Fleet:**
   - Menú lateral → **Management** → **Fleet**
   - Identifica si Fleet Server está conectado (indicador verde).
   - Observa las políticas de agente disponibles.

   **c) Integrations:**
   - Menú lateral → **Management** → **Integrations**
   - Busca la integración de "Custom Logs" y "Nginx" para familiarizarte con el catálogo.

   **d) Discover:**
   - Menú lateral → **Analytics** → **Discover**
   - Nota que aún no hay data views configurados con datos de laboratorio (los crearemos más adelante).

4. Desde la terminal, verifica la conectividad de Kibana con Elasticsearch:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:5601/api/status | jq '.status.overall.level'
```

**Salida esperada:**

```json
"available"
```

**Verificación:** Kibana muestra estado `"available"`, Fleet Server aparece como conectado en la UI, y Stack Monitoring reporta al menos un nodo de Elasticsearch activo.

---

### Paso 4: Inspeccionar los contenedores y distinguir componentes legacy vs. modernos

**Objetivo:** Examinar los contenedores en ejecución para comprender la diferencia arquitectónica entre Filebeat 7.17.22 (modelo legacy de Beats individuales) y el modelo moderno de Elastic Agent + Fleet.

**Instrucciones:**

1. Inspecciona la versión y configuración del contenedor Filebeat legacy:

```bash
docker exec filebeat-legacy01 filebeat version
```

**Salida esperada:**

```
filebeat version 7.17.22 (amd64), libbeat 7.17.22 ...
```

2. Revisa la configuración de Filebeat legacy:

```bash
docker exec filebeat-legacy01 cat /usr/share/filebeat/filebeat.yml
```

Observa que la configuración es un archivo YAML estático que define inputs y outputs directamente en el archivo. Este es el modelo **legacy**: cada Beat se configura individualmente en cada host.

3. Ahora inspecciona Fleet Server (modelo moderno):

```bash
docker logs fleet-server01 --tail 20
```

Busca líneas que indiquen la conexión con Elasticsearch y la disponibilidad para recibir agentes:

```
"message":"fleet-server is running and available"
```

4. Compara ambos modelos documentando las diferencias observadas:

| Aspecto | Filebeat 7.17 (Legacy) | Elastic Agent + Fleet (8.14) |
|---------|------------------------|------------------------------|
| Configuración | Archivo YAML estático por host | Política centralizada en Fleet |
| Actualización | Manual en cada servidor | Push automático desde Kibana |
| Alcance | Solo logs (Filebeat) | Logs, métricas, seguridad (unificado) |
| Gestión | Individual | Centralizada vía Fleet Server |

**Verificación:** Puedes ejecutar ambos comandos de versión sin error y observar claramente la diferencia de modelo de gestión entre ambos componentes.

---

### Paso 5: Inspeccionar el pipeline preconfigurado de Logstash

**Objetivo:** Examinar la configuración del pipeline de Logstash que procesará los logs de muestra de Nginx, comprendiendo las secciones input, filter y output.

**Instrucciones:**

1. Revisa la configuración del pipeline de Logstash:

```bash
docker exec logstash01 cat /usr/share/logstash/pipeline/nginx-sample.conf
```

2. Identifica las tres secciones del pipeline:

   - **Input:** Define cómo Logstash recibe datos (en este caso, un input HTTP en el puerto 8080 o un input file).
   - **Filter:** Aplica parsing con `grok` para descomponer las líneas de log de Nginx en campos estructurados.
   - **Output:** Envía los eventos procesados a Elasticsearch al índice `labs-nginx-sample`.

   Ejemplo de la estructura esperada:

```ruby
input {
  http {
    port => 8080
    codec => "line"
  }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    index => "labs-nginx-sample"
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_verification => false
  }
}
```

3. Verifica que Logstash está procesando correctamente (sin errores en los logs):

```bash
docker logs logstash01 --tail 30 | grep -i "pipeline"
```

Busca una línea similar a:

```
[INFO ] Pipeline started successfully {:pipeline_id=>"main" ...}
```

**Verificación:** El pipeline está activo y en estado `started`. Las tres secciones (input, filter, output) están claramente definidas en el archivo de configuración.

---

### Paso 6: Inyectar el archivo de logs de muestra de Nginx

**Objetivo:** Enviar el archivo `nginx_access_sample.log` al pipeline de Logstash a través del input HTTP para generar eventos que se indexen en Elasticsearch.

**Instrucciones:**

1. Verifica que el archivo de muestra existe en el repositorio:

```bash
ls -la ~/elastic-labs/sample-data/nginx_access_sample.log
wc -l ~/elastic-labs/sample-data/nginx_access_sample.log
```

**Salida esperada:**

```
50 /home/usuario/elastic-labs/sample-data/nginx_access_sample.log
```

> El archivo contiene 50 líneas de logs de acceso Nginx en formato Combined Log.

2. Envía cada línea del archivo al input HTTP de Logstash en el puerto 8080:

```bash
while IFS= read -r line; do
  curl -s -X POST "http://localhost:8080" \
    -H "Content-Type: text/plain" \
    -d "$line" > /dev/null
done < ~/elastic-labs/sample-data/nginx_access_sample.log
echo "Inyección completada: $(wc -l < ~/elastic-labs/sample-data/nginx_access_sample.log) líneas enviadas."
```

**Salida esperada:**

```
Inyección completada: 50 líneas enviadas.
```

3. Espera 5–10 segundos para que Logstash procese los eventos y los envíe a Elasticsearch. Luego verifica que el índice fue creado:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:9200/_cat/indices/labs-nginx-sample?v
```

**Salida esperada:**

```
health status index              uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   labs-nginx-sample  xxxxxxxxxxxxxxxxxxxxxx   1   1         50            0     ...kb         ...kb
```

4. Consulta algunos documentos del índice para verificar que el parsing de grok funcionó correctamente:

```bash
curl -s -k -u elastic:ElasticLabs2024! \
  https://localhost:9200/labs-nginx-sample/_search?size=2 | jq '.hits.hits[]._source | {clientip, verb, request, response, bytes}'
```

**Salida esperada (ejemplo):**

```json
{
  "clientip": "192.168.1.100",
  "verb": "GET",
  "request": "/index.html",
  "response": "200",
  "bytes": "1024"
}
{
  "clientip": "10.0.0.55",
  "verb": "POST",
  "request": "/api/login",
  "response": "302",
  "bytes": "512"
}
```

**Verificación:** El índice `labs-nginx-sample` existe con 50 documentos indexados y los campos están correctamente parseados (clientip, verb, request, response, bytes separados del mensaje original).

---

### Paso 7: Visualizar los datos en Kibana Discover

**Objetivo:** Crear un Data View en Kibana para el índice `labs-nginx-sample` y explorar los eventos ingestados en la interfaz Discover.

**Instrucciones:**

1. En el navegador, accede a Kibana:

```
https://localhost:5601
```

2. Crea un Data View programáticamente desde la terminal (alternativa rápida):

```bash
curl -s -k -X POST "https://localhost:5601/api/data_views/data_view" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -u elastic:ElasticLabs2024! \
  -d '{
    "data_view": {
      "title": "labs-nginx-sample",
      "name": "Labs Nginx Sample",
      "timeFieldName": "@timestamp"
    }
  }' | jq '.data_view.id'
```

**Salida esperada:**

```json
"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

3. En Kibana, navega a **Analytics** → **Discover**.

4. En el selector de Data View (parte superior izquierda), selecciona **Labs Nginx Sample**.

5. Ajusta el rango de tiempo en el selector de fechas (esquina superior derecha) a **Last 7 days** o un rango que incluya las fechas de los logs de muestra.

6. Deberías ver los 50 eventos listados con sus campos parseados. Expande uno de los eventos para verificar que los campos `clientip`, `verb`, `request`, `response` y `bytes` están presentes.

**Verificación:** Los 50 eventos aparecen en Discover con campos estructurados correctamente parseados. El Data View `Labs Nginx Sample` está disponible para futuras consultas.

---

### Paso 8: Documentar el entorno para referencia futura

**Objetivo:** Crear un archivo de referencia rápida con todos los endpoints, puertos y credenciales del entorno de laboratorio que servirá como consulta para las prácticas siguientes.

**Instrucciones:**

1. Crea el archivo de referencia:

```bash
cat > ~/elastic-labs/ENTORNO_REFERENCIA.md << 'EOF'
# Referencia Rápida del Entorno de Laboratorio

## Endpoints de Servicios

| Servicio | URL | Puerto |
|----------|-----|--------|
| Elasticsearch | https://localhost:9200 | 9200 |
| Kibana | https://localhost:5601 | 5601 |
| Logstash (Beats input) | localhost:5044 | 5044 |
| Logstash (HTTP input) | http://localhost:8080 | 8080 |
| Fleet Server | https://localhost:8220 | 8220 |

## Credenciales

| Usuario | Contraseña | Propósito |
|---------|------------|-----------|
| elastic | ElasticLabs2024! | Admin general |
| kibana_system | KibanaLabs2024! | Servicio Kibana |

## Contenedores

| Nombre | Imagen | Versión |
|--------|--------|---------|
| es01 | elasticsearch | 8.14.1 |
| kibana01 | kibana | 8.14.1 |
| logstash01 | logstash | 8.14.1 |
| fleet-server01 | elastic-agent | 8.14.1 |
| filebeat-legacy01 | filebeat | 7.17.22 |

## Comandos Útiles

```bash
# Verificar estado del clúster
curl -s -k -u elastic:ElasticLabs2024! https://localhost:9200/_cluster/health | jq .

# Listar índices
curl -s -k -u elastic:ElasticLabs2024! https://localhost:9200/_cat/indices?v

# Reiniciar todo el stack
cd ~/elastic-labs/config && docker compose restart

# Detener todo el stack
cd ~/elastic-labs/config && docker compose down

# Ver logs de un contenedor
docker logs <nombre_contenedor> --tail 50
```

## Red Docker

- Nombre: elastic-net
- Subred: 172.20.0.0/24
- Driver: bridge
EOF
```

2. Verifica que el archivo se creó correctamente:

```bash
cat ~/elastic-labs/ENTORNO_REFERENCIA.md
```

**Verificación:** El archivo `ENTORNO_REFERENCIA.md` existe y contiene toda la información de endpoints, credenciales y comandos útiles del entorno.

---

## Resumen de Validaciones Finales

Ejecuta este bloque de comandos como verificación integral del laboratorio completado:

```bash
echo "=== Verificación Final del Lab 01-00-01 ==="
echo ""
echo "1. Contenedores activos:"
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "es01|kibana01|logstash01|fleet-server01|filebeat-legacy01"
echo ""
echo "2. Salud del clúster:"
curl -s -k -u elastic:ElasticLabs2024! https://localhost:9200/_cluster/health | jq '{status: .status, nodes: .number_of_nodes}'
echo ""
echo "3. Versión de Elasticsearch:"
curl -s -k -u elastic:ElasticLabs2024! https://localhost:9200 | jq '.version.number'
echo ""
echo "4. Estado de Kibana:"
curl -s -k -u elastic:ElasticLabs2024! https://localhost:5601/api/status | jq '.status.overall.level'
echo ""
echo "5. Documentos en labs-nginx-sample:"
curl -s -k -u elastic:ElasticLabs2024! https://localhost:9200/labs-nginx-sample/_count | jq '.count'
echo ""
echo "6. Filebeat legacy version:"
docker exec filebeat-legacy01 filebeat version 2>/dev/null | head -1
echo ""
echo "=== Verificación completada ==="
```

**Salida esperada:**

```
=== Verificación Final del Lab 01-00-01 ===

1. Contenedores activos:
es01                Up X minutes
kibana01            Up X minutes
logstash01          Up X minutes
fleet-server01      Up X minutes
filebeat-legacy01   Up X minutes

2. Salud del clúster:
{
  "status": "yellow",
  "nodes": 1
}

3. Versión de Elasticsearch:
"8.14.1"

4. Estado de Kibana:
"available"

5. Documentos en labs-nginx-sample:
50

6. Filebeat legacy version:
filebeat version 7.17.22 (amd64), libbeat 7.17.22 ...

=== Verificación completada ===
```

---

## Troubleshooting

| Problema | Causa probable | Solución |
|----------|---------------|----------|
| Contenedor `es01` en estado `Restarting` | Insuficiente memoria para la JVM | Ejecutar `sudo sysctl -w vm.max_map_count=262144` y reiniciar con `docker compose restart es01` |
| Kibana muestra "Kibana server is not ready yet" | Elasticsearch aún no completó su inicio | Esperar 60 segundos adicionales y recargar la página |
| `curl` a puerto 9200 retorna "Connection refused" | El contenedor aún está iniciando o el puerto no está mapeado | Verificar con `docker ps` que el contenedor está `Up` y el puerto está listado |
| El índice `labs-nginx-sample` no aparece | Los eventos no llegaron a Logstash o hubo error de parsing | Revisar `docker logs logstash01 --tail 50` buscando errores de grok o conexión |
| Fleet Server no aparece como conectado en Kibana | Fleet Server necesita más tiempo para registrarse | Esperar 2 minutos y refrescar la página de Fleet en Kibana |
| Error "max virtual memory areas vm.max_map_count [65530] is too low" | Configuración de kernel insuficiente para Elasticsearch | Ejecutar `sudo sysctl -w vm.max_map_count=262144` y añadir a `/etc/sysctl.conf` para persistencia |

---

## Limpieza (opcional)

Si necesitas liberar recursos al finalizar la práctica (no recomendado si continuarás con los siguientes laboratorios):

```bash
cd ~/elastic-labs/config
docker compose down -v
```

> **Advertencia:** El flag `-v` elimina los volúmenes de datos. Solo usar si deseas un reset completo del entorno.

---
