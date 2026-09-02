# Analizar muestras de logs y diseñar el flujo de ingestión, normalización, almacenamiento y retención

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 50 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

## Descripción General

En esta práctica analizarás cuatro tipos distintos de archivos de log (JSON estructurado, texto plano Combined Log Format, multilinea Java y syslog) utilizando herramientas de línea de comandos. A partir de ese análisis, diseñarás y documentarás el flujo de ingestión óptimo para cada fuente, justificando la elección de componentes del stack. Finalmente, implementarás una política ILM y un index template en Elasticsearch que servirán como base para las prácticas posteriores del curso.

## Objetivos de Aprendizaje

- [ ] Analizar archivos de log de distintos formatos identificando estructura, campos relevantes y requisitos de parsing mediante herramientas CLI (`cat`, `grep`, `awk`, `jq`, `wc`)
- [ ] Diseñar y documentar el flujo de ingestión recomendado para cada tipo de log, justificando la selección entre Elastic Agent, Logstash e Ingest Node
- [ ] Mapear campos de logs propietarios al Elastic Common Schema (ECS) para garantizar normalización
- [ ] Implementar una política ILM (`labs-policy`) con fases hot, warm y delete mediante la API REST de Elasticsearch
- [ ] Crear un index template (`labs-app-template`) asociado a la política ILM y verificar su correcta aplicación

## Prerrequisitos

### Conocimientos Previos

- Práctica 1 completada exitosamente: stack Elastic en ejecución con el índice `labs-nginx-sample` disponible
- Comprensión básica de los componentes del Elastic Stack (Elasticsearch, Kibana, Logstash, Beats, Elastic Agent)
- Familiaridad con formato JSON y capacidad de interpretar logs de texto plano
- Uso básico de terminal Linux y herramientas como `cat`, `grep`, `awk`

### Acceso Requerido

- Terminal con acceso al directorio `~/elastic-labs/`
- Conectividad al clúster Elasticsearch en `https://localhost:9200`
- Credenciales: `elastic` / `ElasticLabs2024!`
- Acceso a Kibana en `https://localhost:5601`

## Entorno del Laboratorio

### Software Necesario

| Componente | Versión | Puerto |
|------------|---------|--------|
| Elasticsearch | 8.14.1 | 9200 |
| Kibana | 8.14.1 | 5601 |
| curl | 7.81.0 | — |
| jq | 1.6 | — |
| Python | 3.12.3 | — |

### Verificación Inicial del Entorno

```bash
# Verificar que el stack está en ejecución
cd ~/elastic-labs/
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Verificar conectividad con Elasticsearch
curl -sk -u elastic:ElasticLabs2024! https://localhost:9200/_cluster/health | jq '.status'
```

**Salida esperada:**
```
"green"
```

---

## Paso 1: Preparar los Archivos de Log de Muestra

### Objetivo

Crear los cuatro archivos de log de muestra que servirán como base para el análisis de formatos, campos y requisitos de parsing.

### Instrucciones

1. Crear el directorio de logs de muestra:

```bash
mkdir -p ~/elastic-labs/logs/samples
cd ~/elastic-labs/logs/samples
```

2. Crear el archivo `app_structured.json` (logs JSON de una aplicación Node.js):

```bash
cat > app_structured.json << 'EOF'
{"@timestamp":"2025-07-15T10:01:02.123Z","level":"INFO","service":"order-service","trace_id":"tr-a1b2c3","message":"Pedido creado exitosamente","order_id":"ORD-10042","user_id":"usr-5521","duration_ms":142,"http":{"method":"POST","path":"/api/orders","response":{"status_code":201}}}
{"@timestamp":"2025-07-15T10:01:03.456Z","level":"WARN","service":"order-service","trace_id":"tr-d4e5f6","message":"Inventario bajo para producto SKU-8832","product_id":"SKU-8832","stock_remaining":3,"threshold":5}
{"@timestamp":"2025-07-15T10:01:05.789Z","level":"ERROR","service":"payment-service","trace_id":"tr-g7h8i9","message":"Timeout al conectar con proveedor de pagos","duration_ms":5032,"http":{"method":"POST","path":"/api/payments","response":{"status_code":504}},"error":{"type":"ConnectionTimeout","provider":"stripe"}}
{"@timestamp":"2025-07-15T10:01:06.012Z","level":"INFO","service":"notification-service","trace_id":"tr-j1k2l3","message":"Email de confirmación enviado","recipient":"cliente@example.com","template":"order_confirmation","delivery_time_ms":890}
{"@timestamp":"2025-07-15T10:01:07.345Z","level":"DEBUG","service":"order-service","trace_id":"tr-m4n5o6","message":"Cache hit para catálogo de productos","cache_key":"catalog:electronics:page1","ttl_remaining_s":245}
{"@timestamp":"2025-07-15T10:01:08.678Z","level":"ERROR","service":"inventory-service","trace_id":"tr-p7q8r9","message":"Fallo al actualizar stock en base de datos","error":{"type":"DatabaseException","code":"23505","detail":"duplicate key value violates unique constraint"},"product_id":"SKU-1120","attempted_quantity":-1}
{"@timestamp":"2025-07-15T10:01:10.901Z","level":"INFO","service":"api-gateway","trace_id":"tr-s1t2u3","message":"Request procesado","client_ip":"192.168.1.100","http":{"method":"GET","path":"/api/products","response":{"status_code":200}},"duration_ms":23,"user_agent":"Mozilla/5.0"}
{"@timestamp":"2025-07-15T10:01:12.234Z","level":"WARN","service":"payment-service","trace_id":"tr-v4w5x6","message":"Reintento de pago #2 para orden ORD-10039","retry_count":2,"max_retries":3,"order_id":"ORD-10039"}
EOF
```

3. Crear el archivo `apache_access.log` (Combined Log Format):

```bash
cat > apache_access.log << 'EOF'
192.168.1.45 - alice [15/Jul/2025:10:01:02 +0000] "GET /api/products HTTP/1.1" 200 1842 "https://shop.example.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
10.0.0.12 - - [15/Jul/2025:10:01:03 +0000] "POST /api/orders HTTP/1.1" 201 432 "https://shop.example.com/cart" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
192.168.1.78 - bob [15/Jul/2025:10:01:04 +0000] "GET /api/orders/ORD-10042 HTTP/1.1" 200 2156 "-" "curl/7.81.0"
203.0.113.50 - - [15/Jul/2025:10:01:05 +0000] "GET /admin/config HTTP/1.1" 403 128 "-" "python-requests/2.28.0"
10.0.0.15 - charlie [15/Jul/2025:10:01:06 +0000] "PUT /api/products/SKU-8832 HTTP/1.1" 200 0 "https://admin.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
192.168.1.45 - alice [15/Jul/2025:10:01:07 +0000] "DELETE /api/cart/items/99 HTTP/1.1" 204 0 "https://shop.example.com/cart" "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
172.16.0.5 - - [15/Jul/2025:10:01:08 +0000] "GET /health HTTP/1.1" 200 15 "-" "ELB-HealthChecker/2.0"
203.0.113.50 - - [15/Jul/2025:10:01:09 +0000] "POST /api/login HTTP/1.1" 401 64 "-" "python-requests/2.28.0"
10.0.0.20 - dave [15/Jul/2025:10:01:10 +0000] "GET /api/reports/sales?from=2025-07-01&to=2025-07-15 HTTP/1.1" 200 45230 "https://admin.example.com/reports" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
192.168.1.100 - - [15/Jul/2025:10:01:11 +0000] "GET /static/js/app.bundle.js HTTP/1.1" 304 0 "https://shop.example.com/" "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0)"
EOF
```

4. Crear el archivo `java_exception.log` (logs multilinea con stack traces):

```bash
cat > java_exception.log << 'EOF'
2025-07-15 10:01:02.123 INFO  [main] com.example.Application - Aplicación iniciada en puerto 8080
2025-07-15 10:01:05.456 ERROR [http-nio-8080-exec-3] com.example.PaymentProcessor - Excepción no controlada al procesar pago
java.lang.NullPointerException: Cannot invoke method getId() on null reference
    at com.example.PaymentProcessor.process(PaymentProcessor.java:87)
    at com.example.OrderService.checkout(OrderService.java:134)
    at com.example.controller.OrderController.createOrder(OrderController.java:56)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:897)
2025-07-15 10:01:06.789 WARN  [scheduler-1] com.example.cache.CacheManager - Cache expirado, recargando catálogo completo
2025-07-15 10:01:08.012 ERROR [http-nio-8080-exec-7] com.example.repository.ProductRepository - Error de conexión a base de datos
org.postgresql.util.PSQLException: Connection refused to host: db-primary, port: 5432
    at org.postgresql.core.v3.ConnectionFactoryImpl.openConnectionImpl(ConnectionFactoryImpl.java:315)
    at org.postgresql.core.ConnectionFactory.openConnection(ConnectionFactory.java:49)
    at org.postgresql.jdbc.PgConnection.<init>(PgConnection.java:223)
    at com.example.repository.ProductRepository.findById(ProductRepository.java:45)
    at com.example.service.ProductService.getProduct(ProductService.java:78)
Caused by: java.net.ConnectException: Connection refused
    at java.net.PlainSocketImpl.socketConnect(Native Method)
    at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)
2025-07-15 10:01:10.345 INFO  [http-nio-8080-exec-1] com.example.controller.HealthController - Health check OK, uptime: 3600s
2025-07-15 10:01:12.678 ERROR [http-nio-8080-exec-5] com.example.service.InventoryService - Fallo al reservar stock
java.lang.IllegalStateException: Stock insuficiente para SKU-1120: disponible=0, solicitado=2
    at com.example.service.InventoryService.reserve(InventoryService.java:92)
    at com.example.OrderService.checkout(OrderService.java:128)
    at com.example.controller.OrderController.createOrder(OrderController.java:56)
EOF
```

5. Crear el archivo `auth.log` (syslog estándar de Linux):

```bash
cat > auth.log << 'EOF'
Jul 15 10:01:02 webserver01 sshd[12345]: Accepted publickey for deploy from 10.0.0.5 port 52341 ssh2: RSA SHA256:abc123
Jul 15 10:01:03 webserver01 sshd[12346]: Failed password for invalid user admin from 203.0.113.50 port 44221 ssh2
Jul 15 10:01:04 webserver01 sshd[12347]: Failed password for invalid user admin from 203.0.113.50 port 44222 ssh2
Jul 15 10:01:05 webserver01 sshd[12348]: Failed password for invalid user root from 203.0.113.50 port 44223 ssh2
Jul 15 10:01:06 webserver01 sudo[12350]: deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
Jul 15 10:01:07 webserver01 sshd[12351]: Failed password for invalid user test from 203.0.113.51 port 33100 ssh2
Jul 15 10:01:08 webserver01 sshd[12352]: Disconnecting invalid user admin 203.0.113.50 port 44224: Too many authentication failures
Jul 15 10:01:09 dbserver01 sshd[8901]: Accepted password for dba from 10.0.0.10 port 60122 ssh2
Jul 15 10:01:10 webserver01 systemd-logind[445]: Session opened for user deploy by (uid=0)
Jul 15 10:01:11 webserver01 sshd[12355]: Failed password for invalid user postgres from 203.0.113.52 port 55443 ssh2
EOF
```

### Verificación

```bash
# Verificar que los cuatro archivos existen y tienen contenido
wc -l ~/elastic-labs/logs/samples/*
```

**Salida esperada:**
```
  8 /home/usuario/elastic-labs/logs/samples/app_structured.json
 10 /home/usuario/elastic-labs/logs/samples/apache_access.log
 10 /home/usuario/elastic-labs/logs/samples/auth.log
 22 /home/usuario/elastic-labs/logs/samples/java_exception.log
 50 total
```

---

## Paso 2: Analizar el Log JSON Estructurado

### Objetivo

Examinar `app_structured.json` para identificar su estructura, campos disponibles, niveles de severidad y características que determinan la estrategia de ingestión.

### Instrucciones

1. Verificar que cada línea es JSON válido y visualizar un evento formateado:

```bash
cd ~/elastic-labs/logs/samples

# Validar JSON y mostrar primer evento formateado
head -1 app_structured.json | jq .
```

**Salida esperada:**
```json
{
  "@timestamp": "2025-07-15T10:01:02.123Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "tr-a1b2c3",
  "message": "Pedido creado exitosamente",
  "order_id": "ORD-10042",
  "user_id": "usr-5521",
  "duration_ms": 142,
  "http": {
    "method": "POST",
    "path": "/api/orders",
    "response": {
      "status_code": 201
    }
  }
}
```

2. Extraer todos los campos de primer nivel presentes en el archivo:

```bash
# Listar todos los campos únicos de primer nivel
cat app_structured.json | jq -r 'keys[]' | sort -u
```

**Salida esperada:**
```
@timestamp
cache_key
client_ip
duration_ms
error
http
level
max_retries
message
order_id
product_id
recipient
retry_count
service
stock_remaining
template
threshold
trace_id
ttl_remaining_s
user_agent
user_id
attempted_quantity
delivery_time_ms
```

3. Analizar la distribución de niveles de severidad y servicios:

```bash
# Distribución de niveles
echo "=== Niveles de severidad ==="
cat app_structured.json | jq -r '.level' | sort | uniq -c | sort -rn

# Distribución de servicios
echo -e "\n=== Servicios ==="
cat app_structured.json | jq -r '.service' | sort | uniq -c | sort -rn
```

**Salida esperada:**
```
=== Niveles de severidad ===
      3 INFO
      2 ERROR
      2 WARN
      1 DEBUG

=== Servicios ===
      3 order-service
      2 payment-service
      1 notification-service
      1 inventory-service
      1 api-gateway
```

4. Calcular el tamaño promedio de evento:

```bash
# Tamaño promedio por evento en bytes
total_bytes=$(wc -c < app_structured.json)
total_lines=$(wc -l < app_structured.json)
echo "Tamaño total: ${total_bytes} bytes"
echo "Eventos: ${total_lines}"
echo "Promedio por evento: $((total_bytes / total_lines)) bytes"
```

### Verificación

Confirmar que el archivo contiene JSON válido línea por línea (NDJSON) con `@timestamp` presente en todos los eventos:

```bash
cat app_structured.json | jq -e '.["@timestamp"]' > /dev/null 2>&1 && echo "✓ Todos los eventos tienen @timestamp" || echo "✗ Falta @timestamp en algún evento"
```

**Salida esperada:**
```
✓ Todos los eventos tienen @timestamp
```

---

## Paso 3: Analizar el Log de Texto Plano (Combined Log Format)

### Objetivo

Examinar `apache_access.log` para identificar el patrón de formato, los campos extraíbles y la complejidad de parsing necesaria.

### Instrucciones

1. Inspeccionar la estructura del archivo:

```bash
cd ~/elastic-labs/logs/samples

# Mostrar las primeras 3 líneas
head -3 apache_access.log
```

2. Identificar las IPs de origen y su frecuencia:

```bash
# Extraer IPs de cliente (campo 1)
echo "=== IPs de origen ==="
awk '{print $1}' apache_access.log | sort | uniq -c | sort -rn
```

**Salida esperada:**
```
=== IPs de origen ===
      3 192.168.1.45
      2 203.0.113.50
      2 10.0.0.12
      1 172.16.0.5
      1 10.0.0.20
      1 192.168.1.100
```

3. Extraer los códigos de respuesta HTTP:

```bash
# Códigos de respuesta (campo 9 en formato combined)
echo "=== Códigos de respuesta HTTP ==="
awk '{print $9}' apache_access.log | sort | uniq -c | sort -rn
```

**Salida esperada:**
```
=== Códigos de respuesta HTTP ===
      5 200
      1 201
      1 204
      1 304
      1 401
      1 403
```

4. Extraer los métodos HTTP y las rutas:

```bash
# Métodos HTTP
echo "=== Métodos HTTP ==="
awk -F'"' '{print $2}' apache_access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# Rutas (sin query string)
echo -e "\n=== Rutas únicas ==="
awk -F'"' '{print $2}' apache_access.log | awk '{print $2}' | cut -d'?' -f1 | sort -u
```

5. Identificar el patrón grok necesario para parsear este formato:

```bash
# El patrón grok para Combined Log Format es:
echo "Patrón grok requerido:"
echo '%{IPORHOST:source.ip} %{DATA:apache.access.identity} %{DATA:user.name} \[%{HTTPDATE:@timestamp}\] "%{WORD:http.request.method} %{DATA:url.path} HTTP/%{NUMBER:http.version}" %{NUMBER:http.response.status_code} %{NUMBER:http.response.body.bytes} "%{DATA:http.request.referrer}" "%{DATA:user_agent.original}"'
```

### Verificación

```bash
# Confirmar que todas las líneas tienen exactamente el mismo formato (10 campos mínimos separados por espacio)
awk 'NF < 10 {print NR": línea con formato inesperado"}' apache_access.log
echo "✓ Todas las líneas tienen formato consistente (sin salida = OK)"
```

---

## Paso 4: Analizar el Log Multilinea Java

### Objetivo

Examinar `java_exception.log` para comprender el desafío de los eventos multilinea, identificar el patrón de inicio de evento y contar los eventos lógicos reales.

### Instrucciones

1. Inspeccionar el archivo completo:

```bash
cd ~/elastic-labs/logs/samples
cat -n java_exception.log
```

2. Identificar el patrón de inicio de cada evento lógico:

```bash
# Las líneas que inician un nuevo evento comienzan con una fecha YYYY-MM-DD
echo "=== Líneas de inicio de evento ==="
grep -n "^[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}" java_exception.log
```

**Salida esperada:**
```
1:2025-07-15 10:01:02.123 INFO  [main] com.example.Application - Aplicación iniciada en puerto 8080
2:2025-07-15 10:01:05.456 ERROR [http-nio-8080-exec-3] com.example.PaymentProcessor - Excepción no controlada al procesar pago
8:2025-07-15 10:01:06.789 WARN  [scheduler-1] com.example.cache.CacheManager - Cache expirado, recargando catálogo completo
9:2025-07-15 10:01:08.012 ERROR [http-nio-8080-exec-7] com.example.repository.ProductRepository - Error de conexión a base de datos
17:2025-07-15 10:01:10.345 INFO  [http-nio-8080-exec-1] com.example.controller.HealthController - Health check OK, uptime: 3600s
18:2025-07-15 10:01:12.678 ERROR [http-nio-8080-exec-5] com.example.service.InventoryService - Fallo al reservar stock
```

3. Contar eventos lógicos vs. líneas físicas:

```bash
# Líneas físicas totales
total_lineas=$(wc -l < java_exception.log)

# Eventos lógicos (líneas que comienzan con timestamp)
eventos_logicos=$(grep -c "^[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}" java_exception.log)

echo "Líneas físicas: ${total_lineas}"
echo "Eventos lógicos: ${eventos_logicos}"
echo "Ratio líneas/evento: $(echo "scale=1; ${total_lineas}/${eventos_logicos}" | bc)"
```

**Salida esperada:**
```
Líneas físicas: 22
Eventos lógicos: 6
Ratio líneas/evento: 3.6
```

4. Identificar los niveles de severidad y las clases Java involucradas:

```bash
# Niveles
echo "=== Niveles de severidad ==="
grep "^[0-9]\{4\}" java_exception.log | awk '{print $3}' | sort | uniq -c | sort -rn

# Clases
echo -e "\n=== Clases Java ==="
grep "^[0-9]\{4\}" java_exception.log | awk -F'] ' '{print $2}' | awk '{print $1}' | sort -u
```

5. Documentar la configuración multiline necesaria:

```bash
echo "Configuración multiline para Filebeat/Elastic Agent:"
echo "---"
echo "multiline.type: pattern"
echo "multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'"
echo "multiline.negate: true"
echo "multiline.match: after"
echo "---"
echo "Explicación: Cada nueva línea que NO comienza con un timestamp"
echo "se une a la línea anterior (el stack trace se une al ERROR)."
```

### Verificación

```bash
# Verificar que los stack traces contienen "at " o "Caused by"
echo "Líneas de stack trace encontradas: $(grep -c '^\s\+at\|^Caused by' java_exception.log)"
```

**Salida esperada:**
```
Líneas de stack trace encontradas: 14
```

---

## Paso 5: Analizar el Log Syslog

### Objetivo

Examinar `auth.log` para identificar la estructura syslog RFC 3164, extraer información de seguridad relevante y determinar el método de ingestión adecuado.

### Instrucciones

1. Inspeccionar la estructura:

```bash
cd ~/elastic-labs/logs/samples
head -3 auth.log
```

2. Identificar los hosts que generan logs:

```bash
echo "=== Hosts de origen ==="
awk '{print $4}' auth.log | sort | uniq -c | sort -rn
```

**Salida esperada:**
```
      9 webserver01
      1 dbserver01
```

3. Analizar los procesos y tipos de evento:

```bash
# Procesos
echo "=== Procesos ==="
awk '{print $5}' auth.log | sed 's/\[.*//;s/://' | sort | uniq -c | sort -rn

# Eventos de autenticación fallida
echo -e "\n=== Intentos fallidos por IP de origen ==="
grep "Failed password" auth.log | grep -oP 'from \K[0-9.]+' | sort | uniq -c | sort -rn
```

**Salida esperada:**
```
=== Procesos ===
      8 sshd
      1 sudo
      1 systemd-logind

=== Intentos fallidos por IP de origen ===
      4 203.0.113.50
      1 203.0.113.51
      1 203.0.113.52
```

4. Identificar patrones de seguridad (posible ataque de fuerza bruta):

```bash
# Contar intentos fallidos consecutivos desde la misma IP
echo "=== Análisis de seguridad ==="
failed_from_50=$(grep -c "203.0.113.50" auth.log)
echo "Eventos desde 203.0.113.50: ${failed_from_50} (posible fuerza bruta)"

# Usuarios inválidos intentados
echo -e "\n=== Usuarios inválidos intentados ==="
grep "invalid user" auth.log | grep -oP 'invalid user \K\w+' | sort | uniq -c | sort -rn
```

### Verificación

```bash
# Confirmar que todas las líneas siguen el formato syslog RFC 3164
# (comienzan con mes abreviado, día, hora)
invalid=$(grep -cvP '^[A-Z][a-z]{2}\s+[0-9]+\s[0-9]{2}:[0-9]{2}:[0-9]{2}' auth.log)
echo "Líneas con formato no-syslog: ${invalid} (debe ser 0)"
```

**Salida esperada:**
```
Líneas con formato no-syslog: 0 (debe ser 0)
```

---

## Paso 6: Documentar el Diseño del Flujo de Ingestión

### Objetivo

Crear un documento de diseño que especifique el flujo de ingestión, mapeo ECS y estrategia de retención para cada tipo de log analizado.

### Instrucciones

1. Crear la plantilla de diseño:

```bash
mkdir -p ~/elastic-labs/exports
cat > ~/elastic-labs/exports/diseno_flujo_ingestion.md << 'DESIGN_EOF'
# Diseño de Flujo de Ingestión — Laboratorio Elastic Stack

## Resumen de Análisis

| Archivo | Formato | Líneas | Eventos Lógicos | Tam. Promedio/Evento | Complejidad Parsing |
|---------|---------|--------|-----------------|---------------------|---------------------|
| app_structured.json | JSON (NDJSON) | 8 | 8 | ~320 bytes | Muy baja |
| apache_access.log | Combined Log Format | 10 | 10 | ~180 bytes | Media (grok) |
| java_exception.log | Texto multilinea | 22 | 6 | ~450 bytes | Alta (multiline+grok) |
| auth.log | Syslog RFC 3164 | 10 | 10 | ~120 bytes | Baja-Media |

---

## Flujo 1: app_structured.json (JSON Estructurado)

### Método de Ingestión Recomendado
**Elastic Agent con integración Custom Logs** (ingestión directa)

### Justificación
- El formato JSON no requiere parsing con grok ni transformaciones complejas
- Los campos ya están nombrados y tipados; solo necesitan renombrarse a ECS
- Elastic Agent ofrece gestión centralizada vía Fleet y actualizaciones sin intervención manual
- El ingest pipeline de Elasticsearch puede hacer el mapeo ECS con procesadores `rename`

### Mapeo a ECS

| Campo Original | Campo ECS | Tipo |
|---------------|-----------|------|
| @timestamp | @timestamp | date |
| level | log.level | keyword |
| service | service.name | keyword |
| trace_id | trace.id | keyword |
| message | message | text |
| duration_ms | event.duration (convertir a ns) | long |
| http.method | http.request.method | keyword |
| http.path | url.path | keyword |
| http.response.status_code | http.response.status_code | integer |
| client_ip | source.ip | ip |
| error.type | error.type | keyword |
| user_id | user.id | keyword |

### Procesamiento Requerido
- Ingest pipeline con procesador `json` (si se envía como texto) o directo si Elastic Agent parsea JSON
- Procesador `rename` para mapear campos propios a ECS
- Procesador `convert` para `duration_ms` → nanosegundos si se usa `event.duration`

---

## Flujo 2: apache_access.log (Combined Log Format)

### Método de Ingestión Recomendado
**Elastic Agent con integración Apache HTTP Server** (módulo preconfigurado)

### Justificación
- Elastic Agent tiene una integración nativa para Apache que incluye parseo, dashboards y mapeo ECS
- Evita escribir patrones grok manuales para un formato estándar bien conocido
- Alternativa: Si se requiere enriquecimiento avanzado (GeoIP, threat intelligence), usar Logstash como procesador intermedio

### Mapeo a ECS

| Campo Extraído | Campo ECS | Tipo |
|---------------|-----------|------|
| IP cliente | source.ip | ip |
| Usuario | user.name | keyword |
| Timestamp | @timestamp | date |
| Método | http.request.method | keyword |
| Ruta | url.path | keyword |
| Versión HTTP | http.version | keyword |
| Status code | http.response.status_code | integer |
| Bytes | http.response.body.bytes | long |
| Referrer | http.request.referrer | keyword |
| User agent | user_agent.original | text |

### Procesamiento Requerido
- Patrón grok: `%{COMBINEDAPACHELOG}` (integrado en la integración Apache)
- Procesador `user_agent` para parsear el campo user agent
- Procesador `geoip` para enriquecer `source.ip` con datos geográficos

---

## Flujo 3: java_exception.log (Multilinea)

### Método de Ingestión Recomendado
**Logstash con codec multiline** o **Elastic Agent con configuración multiline**

### Justificación
- Los logs multilinea requieren configuración explícita de agrupación antes del envío
- Logstash ofrece mayor flexibilidad para transformaciones complejas (extraer clase Java, nivel, thread)
- Si el volumen es bajo-medio, Elastic Agent con Custom Logs y multiline es suficiente
- Para entornos con muchas aplicaciones Java heterogéneas, Logstash centraliza la lógica de parsing

### Configuración Multiline
```
pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
negate: true
match: after
```

### Mapeo a ECS

| Campo Extraído | Campo ECS | Tipo |
|---------------|-----------|------|
| Timestamp | @timestamp | date |
| Nivel (INFO/WARN/ERROR) | log.level | keyword |
| Thread | process.thread.name | keyword |
| Clase Java | log.logger | keyword |
| Mensaje + stack trace | message | text |
| Tipo de excepción | error.type | keyword |
| Stack trace completo | error.stack_trace | text |

### Procesamiento Requerido
- Codec/configuración multiline para agrupar stack traces
- Grok: `%{TIMESTAMP_ISO8601:@timestamp} %{LOGLEVEL:log.level}\s+\[%{DATA:process.thread.name}\] %{JAVACLASS:log.logger} - %{GREEDYDATA:message}`
- Procesador condicional para extraer `error.stack_trace` de eventos ERROR

---

## Flujo 4: auth.log (Syslog RFC 3164)

### Método de Ingestión Recomendado
**Elastic Agent con integración System** (módulo auth)

### Justificación
- La integración System de Elastic Agent incluye parseo nativo de auth.log con mapeo ECS completo
- Detecta automáticamente eventos de autenticación (login, logout, failed) y los clasifica
- Para dispositivos de red que envían syslog por UDP/TCP, usar Logstash con input syslog
- En este caso (archivo local en servidor Linux), Elastic Agent es la opción más eficiente

### Mapeo a ECS

| Campo Extraído | Campo ECS | Tipo |
|---------------|-----------|------|
| Timestamp | @timestamp | date |
| Hostname | host.name | keyword |
| Proceso | process.name | keyword |
| PID | process.pid | long |
| IP de origen (SSH) | source.ip | ip |
| Puerto de origen | source.port | integer |
| Usuario | user.name | keyword |
| Resultado (accepted/failed) | event.outcome | keyword |
| Tipo de evento | event.action | keyword |

### Procesamiento Requerido
- Parsing syslog header (timestamp, host, process[pid])
- Grok específico por tipo de mensaje (sshd accepted, sshd failed, sudo)
- Clasificación de `event.outcome`: "success" o "failure"
- Enriquecimiento: GeoIP en `source.ip` para detectar origen geográfico de ataques

---

## Estrategia de Retención (ILM)

### Política: `labs-policy`

| Fase | Condición de transición | Configuración |
|------|------------------------|---------------|
| **Hot** | Hasta 7 días o 10 GB | 1 réplica, prioridad 100 |
| **Warm** | Después de hot, hasta 14 días | 0 réplicas, force merge a 1 segmento, prioridad 50 |
| **Delete** | 30 días desde creación | Eliminar índice |

### Justificación
- **Hot (7d):** Período de investigación activa; los logs más recientes se consultan con frecuencia
- **Warm (14d):** Retención para análisis histórico y compliance; reducción de recursos
- **Delete (30d):** Cumple requisitos de retención mínima sin acumular almacenamiento innecesario

### Aplicación
- Index template `labs-app-template` con patrón `labs-app-*`
- Component template `labs-ecs-mappings` con campos ECS comunes
- Todos los índices del laboratorio creados bajo el patrón `labs-app-*` heredarán la política automáticamente

DESIGN_EOF

echo "✓ Documento de diseño creado en ~/elastic-labs/exports/diseno_flujo_ingestion.md"
```

2. Verificar el documento:

```bash
wc -l ~/elastic-labs/exports/diseno_flujo_ingestion.md
head -5 ~/elastic-labs/exports/diseno_flujo_ingestion.md
```

**Salida esperada:**
```
155 /home/usuario/elastic-labs/exports/diseno_flujo_ingestion.md
# Diseño de Flujo de Ingestión — Laboratorio Elastic Stack

## Resumen de Análisis

| Archivo | Formato | Líneas | Eventos Lógicos | Tam. Promedio/Evento | Complejidad Parsing |
```

---

## Paso 7: Implementar la Política ILM `labs-policy`

### Objetivo

Crear la política de Index Lifecycle Management en Elasticsearch que define las fases hot, warm y delete para los índices del laboratorio.

### Instrucciones

1. Crear la política ILM mediante la API REST:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_ilm/policy/labs-policy" \
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
          "set_priority": {
            "priority": 50
          },
          "allocate": {
            "number_of_replicas": 0
          },
          "forcemerge": {
            "max_num_segments": 1
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

**Salida esperada:**
```json
{
  "acknowledged": true
}
```

2. Verificar que la política se creó correctamente:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/labs-policy" | jq '.["labs-policy"].policy.phases | keys'
```

**Salida esperada:**
```json
[
  "delete",
  "hot",
  "warm"
]
```

3. Verificar los detalles de cada fase:

```bash
# Verificar fase hot
echo "=== Fase Hot ==="
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/labs-policy" | jq '.["labs-policy"].policy.phases.hot'

# Verificar fase warm
echo -e "\n=== Fase Warm ==="
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/labs-policy" | jq '.["labs-policy"].policy.phases.warm'

# Verificar fase delete
echo -e "\n=== Fase Delete ==="
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/labs-policy" | jq '.["labs-policy"].policy.phases.delete'
```

### Verificación

```bash
# Confirmar que la política existe y tiene las 3 fases
phases=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/labs-policy" | jq '.["labs-policy"].policy.phases | length')
echo "Número de fases en labs-policy: ${phases} (esperado: 3)"
```

**Salida esperada:**
```
Número de fases en labs-policy: 3 (esperado: 3)
```

---

## Paso 8: Crear el Component Template de Mappings ECS

### Objetivo

Crear un component template reutilizable con los mappings ECS comunes que será referenciado por los index templates del laboratorio.

### Instrucciones

1. Crear el component template `labs-ecs-mappings`:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_component_template/labs-ecs-mappings" \
  -H "Content-Type: application/json" \
  -d '{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "message": {
          "type": "text"
        },
        "log.level": {
          "type": "keyword"
        },
        "service.name": {
          "type": "keyword"
        },
        "trace.id": {
          "type": "keyword"
        },
        "host.name": {
          "type": "keyword"
        },
        "source.ip": {
          "type": "ip"
        },
        "http.request.method": {
          "type": "keyword"
        },
        "http.response.status_code": {
          "type": "integer"
        },
        "url.path": {
          "type": "keyword"
        },
        "user.name": {
          "type": "keyword"
        },
        "event.outcome": {
          "type": "keyword"
        },
        "event.action": {
          "type": "keyword"
        },
        "event.duration": {
          "type": "long"
        },
        "error.type": {
          "type": "keyword"
        },
        "error.stack_trace": {
          "type": "text",
          "index": false
        },
        "process.name": {
          "type": "keyword"
        },
        "process.pid": {
          "type": "long"
        },
        "user_agent.original": {
          "type": "text"
        }
      }
    }
  }
}' | jq .
```

**Salida esperada:**
```json
{
  "acknowledged": true
}
```

2. Verificar el component template:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_component_template/labs-ecs-mappings" | jq '.component_templates[0].component_template.template.mappings.properties | keys'
```

**Salida esperada:**
```json
[
  "@timestamp",
  "error.stack_trace",
  "error.type",
  "event.action",
  "event.duration",
  "event.outcome",
  "host.name",
  "http.request.method",
  "http.response.status_code",
  "log.level",
  "message",
  "process.name",
  "process.pid",
  "service.name",
  "source.ip",
  "trace.id",
  "url.path",
  "user.name",
  "user_agent.original"
]
```

---

## Paso 9: Crear el Index Template `labs-app-template`

### Objetivo

Crear el index template que asocia la política ILM y el component template ECS a todos los índices que coincidan con el patrón `labs-app-*`.

### Instrucciones

1. Crear el index template:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_index_template/labs-app-template" \
  -H "Content-Type: application/json" \
  -d '{
  "index_patterns": ["labs-app-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "labs-policy",
      "index.lifecycle.rollover_alias": "labs-app"
    }
  },
  "composed_of": ["labs-ecs-mappings"],
  "priority": 200,
  "_meta": {
    "description": "Template para índices de aplicación del laboratorio Elastic Stack",
    "created_by": "lab-02-00-01"
  }
}' | jq .
```

**Salida esperada:**
```json
{
  "acknowledged": true
}
```

2. Verificar que el template se creó correctamente:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/labs-app-template" | jq '.index_templates[0].index_template'
```

3. Verificar que el template referencia correctamente la política ILM y el component template:

```bash
# Verificar ILM asociado
echo "=== Política ILM asociada ==="
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/labs-app-template" | \
  jq -r '.index_templates[0].index_template.template.settings.index.lifecycle.name'

# Verificar component templates compuestos
echo -e "\n=== Component templates ==="
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/labs-app-template" | \
  jq -r '.index_templates[0].index_template.composed_of[]'
```

**Salida esperada:**
```
=== Política ILM asociada ===
labs-policy

=== Component templates ===
labs-ecs-mappings
```

---

## Paso 10: Validar la Aplicación del Template con un Índice de Prueba

### Objetivo

Crear un índice de prueba que coincida con el patrón `labs-app-*` y verificar que hereda automáticamente los mappings ECS y la política ILM.

### Instrucciones

1. Crear un índice de prueba e insertar un documento:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:9200/labs-app-test/_doc" \
  -H "Content-Type: application/json" \
  -d '{
  "@timestamp": "2025-07-15T10:01:02.123Z",
  "message": "Documento de prueba para validar template",
  "log.level": "INFO",
  "service.name": "test-service",
  "source.ip": "192.168.1.100",
  "http.response.status_code": 200
}' | jq '{result: .result, _index: ._index}'
```

**Salida esperada:**
```json
{
  "result": "created",
  "_index": "labs-app-test"
}
```

2. Verificar que los mappings ECS se aplicaron:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-app-test/_mapping" | \
  jq '.["labs-app-test"].mappings.properties["source.ip"].type'
```

**Salida esperada:**
```json
"ip"
```

3. Verificar que la política ILM está asociada al índice:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-app-test/_settings" | \
  jq '.["labs-app-test"].settings.index.lifecycle'
```

**Salida esperada:**
```json
{
  "name": "labs-policy",
  "rollover_alias": "labs-app"
}
```

4. Verificar el estado ILM del índice:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-app-test/_ilm/explain" | \
  jq '.indices["labs-app-test"] | {managed: .managed, policy: .policy, phase: .phase}'
```

**Salida esperada:**
```json
{
  "managed": true,
  "policy": "labs-policy",
  "phase": "hot"
}
```

### Verificación

```bash
# Verificación integral: el índice existe, tiene mappings correctos y política ILM
echo "=== Verificación Final del Template ==="
echo -n "1. Índice existe: "
curl -sk -u elastic:ElasticLabs2024! -o /dev/null -w "%{http_code}" "https://localhost:9200/labs-app-test"
echo ""

echo -n "2. Campo source.ip es tipo 'ip': "
curl -sk -u elastic:ElasticLabs2024! "https://localhost:9200/labs-app-test/_mapping" | \
  jq -r '.["labs-app-test"].mappings.properties["source.ip"].type'

echo -n "3. Política ILM aplicada: "
curl -sk -u elastic:ElasticLabs2024! "https://localhost:9200/labs-app-test/_ilm/explain" | \
  jq -r '.indices["labs-app-test"].policy'
```

**Salida esperada:**
```
=== Verificación Final del Template ===
1. Índice existe: 200
2. Campo source.ip es tipo 'ip': ip
3. Política ILM aplicada: labs-policy
```

---

## Validación y Pruebas

Ejecuta el siguiente script de validación integral que confirma todos los entregables de la práctica:

```bash
#!/bin/bash
echo "╔══════════════════════════════════════════════════════════╗"
echo "║  VALIDACIÓN INTEGRAL — Práctica 2 (Lab 02-00-01)       ║"
echo "╠══════════════════════════════════════════════════════════╣"

PASS=0
FAIL=0

# Test 1: Archivos de muestra existen
echo -n "║ [1/7] Archivos de log de muestra creados............"
if [ -f ~/elastic-labs/logs/samples/app_structured.json ] && \
   [ -f ~/elastic-labs/logs/samples/apache_access.log ] && \
   [ -f ~/elastic-labs/logs/samples/java_exception.log ] && \
   [ -f ~/elastic-labs/logs/samples/auth.log ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

# Test 2: Documento de diseño existe
echo -n "║ [2/7] Documento de diseño de flujo creado..........."
if [ -f ~/elastic-labs/exports/diseno_flujo_ingestion.md ] && \
   [ $(wc -l < ~/elastic-labs/exports/diseno_flujo_ingestion.md) -gt 100 ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

# Test 3: Política ILM labs-policy existe
echo -n "║ [3/7] Política ILM 'labs-policy' existe............."
ilm_status=$(curl -sk -u elastic:ElasticLabs2024! -o /dev/null -w "%{http_code}" \
  "https://localhost:9200/_ilm/policy/labs-policy")
if [ "$ilm_status" = "200" ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

# Test 4: ILM tiene 3 fases
echo -n "║ [4/7] ILM tiene fases hot, warm, delete............."
phases=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_ilm/policy/labs-policy" | jq '.["labs-policy"].policy.phases | length')
if [ "$phases" = "3" ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

# Test 5: Component template existe
echo -n "║ [5/7] Component template 'labs-ecs-mappings' existe.."
ct_status=$(curl -sk -u elastic:ElasticLabs2024! -o /dev/null -w "%{http_code}" \
  "https://localhost:9200/_component_template/labs-ecs-mappings")
if [ "$ct_status" = "200" ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

# Test 6: Index template existe
echo -n "║ [6/7] Index template 'labs-app-template' existe......"
it_status=$(curl -sk -u elastic:ElasticLabs2024! -o /dev/null -w "%{http_code}" \
  "https://localhost:9200/_index_template/labs-app-template")
if [ "$it_status" = "200" ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

# Test 7: Índice de prueba con ILM aplicado
echo -n "║ [7/7] Índice labs-app-test gestionado por ILM......."
managed=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-app-test/_ilm/explain" | \
  jq -r '.indices["labs-app-test"].managed')
if [ "$managed" = "true" ]; then
  echo " ✓ ║"; ((PASS++))
else
  echo " ✗ ║"; ((FAIL++))
fi

echo "╠══════════════════════════════════════════════════════════╣"
echo "║  Resultado: ${PASS}/7 pruebas exitosas, ${FAIL}/7 fallidas       ║"
if [ $FAIL -eq 0 ]; then
  echo "║  ★ PRÁCTICA COMPLETADA EXITOSAMENTE ★                  ║"
else
  echo "║  ⚠ Revisa los pasos con fallos antes de continuar      ║"
fi
echo "╚══════════════════════════════════════════════════════════╝"
```

Guarda y ejecuta:

```bash
cat > ~/elastic-labs/scripts/validate_lab02.sh << 'SCRIPT_EOF'
# (pegar el script de arriba aquí)
SCRIPT_EOF
chmod +x ~/elastic-labs/scripts/validate_lab02.sh
bash ~/elastic-labs/scripts/validate_lab02.sh
```

---

## Solución de Problemas

### Problema 1: Error 403 o autenticación fallida al crear la política ILM

**Síntomas:**
```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [cluster:admin/ilm/put] is unauthorized for user [elastic]"
  },
  "status": 403
}
```

**Causa:** El contenedor de Elasticsearch puede haber perdido la configuración de seguridad o las credenciales cambiaron. Alternativamente, se está usando un usuario sin privilegios de superusuario.

**Solución:**

```bash
# Verificar que se usa el usuario correcto
curl -sk -u elastic:ElasticLabs2024! "https://localhost:9200/_security/_authenticate" | jq '.username, .roles'

# Si las credenciales no funcionan, resetear password del usuario elastic
docker exec es01 bin/elasticsearch-reset-password -u elastic -b -f

# Actualizar la variable de entorno si es necesario y reintentar
```

### Problema 2: El index template no aplica la política ILM al índice de prueba

**Síntomas:**
Al consultar `_ilm/explain`, el campo `managed` aparece como `false` o la política es `null`:
```json
{
  "managed": false,
  "policy": null
}
```

**Causa:** El índice fue creado antes de que el index template existiera, o el nombre del índice no coincide con el patrón `labs-app-*` definido en `index_patterns`.

**Solución:**

```bash
# 1. Verificar que el nombre del índice coincide con el patrón
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_index_template/labs-app-template" | \
  jq '.index_templates[0].index_template.index_patterns'

# 2. Si el índice ya existía, eliminarlo y recrearlo
curl -sk -u elastic:ElasticLabs2024! -X DELETE "https://localhost:9200/labs-app-test"

# 3. Recrear el índice (el template se aplicará automáticamente)
curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:9200/labs-app-test/_doc" \
  -H "Content-Type: application/json" \
  -d '{"@timestamp":"2025-07-15T10:00:00Z","message":"test"}' | jq .

# 4. Verificar nuevamente
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-app-test/_ilm/explain" | \
  jq '.indices["labs-app-test"].managed'
```

---

## Limpieza

El índice de prueba `labs-app-test` puede eliminarse ya que solo sirvió para validación. Sin embargo, **la política ILM, el component template y el index template deben conservarse** ya que serán utilizados en las Prácticas 3 y 4.

```bash
# Eliminar solo el índice de prueba
curl -sk -u elastic:ElasticLabs2024! \
  -X DELETE "https://localhost:9200/labs-app-test" | jq .

# NO eliminar estos recursos (necesarios para prácticas posteriores):
# - Política ILM: labs-policy
# - Component template: labs-ecs-mappings
# - Index template: labs-app-template
# - Archivos en ~/elastic-labs/logs/samples/
# - Documento de diseño en ~/elastic-labs/exports/

echo "✓ Limpieza completada. Recursos de ILM y templates conservados para Prácticas 3-4."
```

---

## Resumen

En esta práctica has completado el ciclo analítico-diseño-implementación para la gestión de logs:

| Actividad | Resultado |
|-----------|-----------|
| Análisis de 4 formatos de log | Identificación de estructura, campos, complejidad de parsing y eventos lógicos vs. líneas físicas |
| Diseño de flujos de ingestión | Documento con justificación de Elastic Agent vs. Logstash para cada fuente, mapeo ECS completo |
| Política ILM `labs-policy` | Fases hot (7d/10GB), warm (14d, 0 réplicas, forcemerge), delete (30d) |
| Component template `labs-ecs-mappings` | 19 campos ECS tipados reutilizables |
| Index template `labs-app-template` | Patrón `labs-app-*`, referencia a ILM y component template |

### Conceptos Clave Reforzados

- **JSON estructurado** tiene complejidad de parsing muy baja → ingestión directa con Elastic Agent
- **Texto plano con patrón fijo** requiere grok → usar integraciones preconfiguradas cuando existan
- **Logs multilinea** son los más complejos → requieren configuración explícita de agrupación
- **Syslog** es un estándar bien soportado → integraciones nativas disponibles
- **ILM** automatiza el ciclo de vida de los datos reduciendo costos de almacenamiento
- **Component templates** promueven la reutilización de mappings entre múltiples index templates

### Recursos Adicionales

- [Documentación ILM de Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/index-lifecycle-management.html)
- [Elastic Common Schema (ECS) Reference](https://www.elastic.co/guide/en/ecs/current/index.html)
- [Index Templates](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/index-templates.html)
- [Grok Pattern Reference](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/grok-processor.html)

---
