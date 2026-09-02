# Crear un dashboard de errores y configurar una regla de monitoreo para logs de aplicaciones

## Metadatos

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear |

## Descripción General

En esta práctica integradora final, construirás un dashboard operativo completo en Kibana para monitorear errores de aplicaciones, utilizando los datos generados en las prácticas anteriores (1, 3 y 4). Explorarás los datos con KQL y ES|QL en Discover, crearás cinco visualizaciones con Lens y configurarás una regla de alerta que detecte picos de errores y notifique mediante un conector de tipo Server log. Validarás el funcionamiento completo generando un pico artificial de errores con un script Python.

## Objetivos de Aprendizaje

- [ ] Crear un dashboard operativo en Kibana con al menos cinco visualizaciones Lens (métrica, barras apiladas, tabla, mapa, líneas) que muestren distribución de errores, tendencias temporales y distribución geográfica.
- [ ] Escribir consultas KQL y ES|QL en Kibana Discover para filtrar y analizar patrones específicos de error en logs indexados.
- [ ] Configurar una regla de alerta de tipo "Elasticsearch query" con umbral de picos de errores y validar su disparo mediante generación controlada de datos.

## Prerrequisitos

### Conocimientos Previos

- Familiaridad con la interfaz de Kibana (Discover, Stack Management) adquirida en prácticas anteriores.
- Comprensión de campos ECS normalizados: `log.level`, `error.type`, `@timestamp`, `geoip.location`.
- Experiencia básica con KQL (sintaxis de filtros) y conceptos de ES|QL (tuberías, comandos `FROM`, `WHERE`, `STATS`).

### Acceso y Datos Requeridos

| Recurso | Estado Requerido |
|---------|-----------------|
| Índice `labs-logstash-app-*` (Práctica 4) | ≥ 200 documentos indexados |
| Data view `labs-logstash-app-*` | Creado y funcional |
| Data view `labs-elastic-agent` (Práctica 3) | Con datos disponibles |
| Data view `labs-nginx-sample` (Práctica 1) | Con datos de referencia |
| Stack Elastic (es01, kibana01, logstash01) | Contenedores en ejecución |

## Entorno de Laboratorio

### Software Utilizado

| Componente | Versión | Puerto |
|------------|---------|--------|
| Elasticsearch | 8.14.1 | 9200 |
| Kibana | 8.14.1 | 5601 |
| Logstash | 8.14.1 | 5044 / 8080 |
| Python | 3.12.3 | — |

### Verificación Inicial del Entorno

```bash
# Verificar que los contenedores están en ejecución
cd ~/elastic-labs/
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Verificar conectividad con Elasticsearch
curl -sk -u elastic:ElasticLabs2024! https://localhost:9200/_cluster/health?pretty | jq '{status, number_of_nodes}'

# Verificar que existen datos en los índices necesarios
curl -sk -u elastic:ElasticLabs2024! https://localhost:9200/labs-logstash-app-*/_count | jq '.count'
curl -sk -u elastic:ElasticLabs2024! https://localhost:9200/labs-nginx-sample/_count | jq '.count'
```

**Salida esperada:**

```json
{
  "status": "green",
  "number_of_nodes": 1
}
```

El conteo de `labs-logstash-app-*` debe ser ≥ 200.

---

## Paso 1: Explorar Datos con KQL en Discover

### Objetivo

Utilizar Kibana Discover con consultas KQL para identificar patrones de error en los logs indexados, familiarizándose con la distribución de campos antes de construir el dashboard.

### Instrucciones

1. Abre Kibana en el navegador: `https://localhost:5601`. Inicia sesión con `elastic` / `ElasticLabs2024!`.

2. Navega a **Discover** (menú lateral izquierdo → Analytics → Discover).

3. En el selector de data view (esquina superior izquierda), selecciona **`labs-logstash-app-*`**.

4. Ajusta el rango temporal a **Last 24 hours** usando el selector de tiempo en la esquina superior derecha.

5. En la barra de búsqueda KQL, escribe la siguiente consulta y presiona Enter:

```
log.level: "ERROR"
```

6. Observa el número de documentos coincidentes (hits) en la parte superior de la tabla de resultados.

7. En el panel lateral izquierdo, localiza el campo `error.type` y haz clic sobre él para ver los top valores. Identifica las clases Java más frecuentes.

8. Amplía la consulta para buscar errores de una clase específica:

```
log.level: "ERROR" AND error.type: "NullPointerException"
```

9. Agrega las columnas `@timestamp`, `log.level`, `error.type` y `message` a la tabla haciendo clic en el ícono `+` junto a cada campo en el panel lateral.

10. Guarda esta búsqueda: haz clic en **Save** (parte superior), nómbrala `Labs - Errores NPE` y confirma.

11. Ahora cambia el data view a **`labs-nginx-sample`** y ejecuta:

```
http.response.status_code >= 500
```

12. Observa la distribución de errores 5xx por `url.path` en el panel lateral de campos.

### Salida Esperada

- La consulta `log.level: "ERROR"` debe retornar documentos con el campo `log.level` conteniendo exactamente "ERROR".
- El campo `error.type` debe mostrar valores como `NullPointerException`, `IOException`, `TimeoutException` u otros similares generados por el script de prácticas anteriores.
- El data view `labs-nginx-sample` debe mostrar eventos con códigos 500, 502 o 503.

### Verificación

- [ ] Se visualizan documentos filtrados por `log.level: "ERROR"` en `labs-logstash-app-*`.
- [ ] El panel lateral muestra la distribución de `error.type` con al menos 2 tipos distintos.
- [ ] La búsqueda guardada `Labs - Errores NPE` aparece en la lista de búsquedas guardadas.

---

## Paso 2: Ejecutar Consultas Analíticas con ES|QL

### Objetivo

Utilizar ES|QL en Discover para realizar consultas agregadas que cuantifiquen errores por clase Java y por hora, generando insights tabulares que informen el diseño del dashboard.

### Instrucciones

1. En Discover, cambia al modo **ES|QL** haciendo clic en el botón de alternancia junto a la barra de búsqueda (o seleccionando "ES|QL" en el menú desplegable del lenguaje de consulta).

2. Ejecuta la siguiente consulta para contar errores por tipo en las últimas 24 horas:

```esql
FROM labs-logstash-app-*
| WHERE @timestamp >= NOW() - 24 hours AND log.level == "ERROR"
| STATS error_count = COUNT(*) BY error.type
| SORT error_count DESC
| LIMIT 10
```

3. Observa la tabla resultante. Anota los top 3 tipos de error y sus conteos.

4. Ejecuta una segunda consulta para analizar la distribución temporal de errores por hora:

```esql
FROM labs-logstash-app-*
| WHERE @timestamp >= NOW() - 24 hours AND log.level == "ERROR"
| EVAL hour = DATE_TRUNC(1 hour, @timestamp)
| STATS hourly_errors = COUNT(*) BY hour
| SORT hour ASC
```

5. Ejecuta una tercera consulta comparativa entre fuentes de datos:

```esql
FROM labs-logstash-app-*
| WHERE @timestamp >= NOW() - 24 hours
| STATS
    total = COUNT(*),
    errors = COUNT(*) WHERE log.level == "ERROR",
    warnings = COUNT(*) WHERE log.level == "WARN"
| EVAL error_pct = ROUND(errors * 100.0 / total, 2)
```

6. Regresa al modo KQL haciendo clic en el selector de lenguaje.

### Salida Esperada

La primera consulta debe producir una tabla similar a:

```
error.type              | error_count
------------------------|------------
NullPointerException    | 45
IOException             | 32
TimeoutException        | 28
...
```

La segunda consulta mostrará el conteo de errores agrupado por hora.

### Verificación

- [ ] Las consultas ES|QL se ejecutan sin errores de sintaxis.
- [ ] La tabla de resultados muestra datos agregados con columnas calculadas.
- [ ] Se identifican al menos 2 tipos de error distintos con sus conteos.

---

## Paso 3: Crear el Dashboard con Cinco Visualizaciones Lens

### Objetivo

Construir el dashboard `Labs - Application Error Monitor` con cinco visualizaciones Lens que cubran métricas, tendencias temporales, top errores, distribución geográfica y comparación entre fuentes.

### Instrucciones

#### 3.1 Crear el Dashboard

1. Navega a **Dashboard** (menú lateral → Analytics → Dashboard).

2. Haz clic en **Create dashboard**.

3. Antes de agregar visualizaciones, haz clic en el ícono de configuración (engranaje) o en **Save** y nombra el dashboard: `Labs - Application Error Monitor`. Marca "Store time with dashboard" y selecciona **Last 24 hours**. Guarda.

#### 3.2 Visualización 1: Métrica de Conteo Total de Errores (últimas 24h)

4. Haz clic en **Create visualization** (o el botón `+` → Lens).

5. En el editor Lens:
   - **Data view**: `labs-logstash-app-*`
   - **Tipo de visualización**: Selecciona **Metric** en el selector de tipo (parte superior del editor).
   - **Filtro de panel**: En la barra de filtro del panel, escribe: `log.level: "ERROR"`
   - **Métrica primaria**: Arrastra el campo `Records` al área de métrica (o configura `Count` como función).
   - **Título**: Haz clic en el título y escribe `Total Errores (24h)`.

6. Haz clic en **Save and return** para añadir la visualización al dashboard.

#### 3.3 Visualización 2: Barras Apiladas de Eventos por log.level en el Tiempo

7. Haz clic en **Create visualization** nuevamente.

8. En el editor Lens:
   - **Data view**: `labs-logstash-app-*`
   - **Tipo de visualización**: **Bar vertical stacked**.
   - **Eje horizontal**: `@timestamp` con intervalo automático (Date histogram).
   - **Eje vertical**: `Count` de registros.
   - **Break down by**: Campo `log.level` (arrastrar al área "Break down by" o configurar como Top values de `log.level`, top 5).
   - **Título**: `Eventos por Nivel de Log`

9. Haz clic en **Save and return**.

#### 3.4 Visualización 3: Tabla de Top 10 Errores por error.type

10. Crea una nueva visualización Lens.

11. Configuración:
    - **Data view**: `labs-logstash-app-*`
    - **Tipo de visualización**: **Table**.
    - **Filtro de panel**: `log.level: "ERROR"`
    - **Columna 1 (Rows)**: Top values de `error.type`, tamaño 10.
    - **Columna 2 (Metrics)**: Count de registros. Etiqueta: `Conteo`.
    - **Título**: `Top 10 Tipos de Error`

12. Ordena por la columna de conteo en orden descendente. **Save and return**.

#### 3.5 Visualización 4: Mapa de Distribución Geográfica

13. Crea una nueva visualización Lens.

14. Configuración:
    - **Data view**: `labs-logstash-app-*`
    - **Tipo de visualización**: Selecciona **Map** (Choropleth/Region map). Si el tipo Map no está disponible directamente en Lens, usa **Maps** como visualización embebida:
      - Alternativa: Haz clic en **Create visualization** → selecciona tipo **Maps** en lugar de Lens.
      - Agrega una capa **Clusters and grids** con fuente `labs-logstash-app-*`.
      - Campo de geolocalización: `geoip.location` (tipo `geo_point`).
      - Métrica: Count.
    - **Título**: `Distribución Geográfica de Eventos`

15. **Save and return**.

> **Nota**: Si el campo `geoip.location` no contiene datos suficientes, puedes usar el data view `labs-nginx-sample` como fuente alternativa para esta visualización, ya que los logs de Nginx suelen incluir IPs de clientes enriquecidas con GeoIP.

#### 3.6 Visualización 5: Líneas Comparando Volumen Logstash vs Elastic Agent

16. Crea una nueva visualización Lens.

17. Configuración:
    - **Tipo de visualización**: **Line**.
    - Necesitas combinar dos fuentes. Método recomendado:
      - **Capa 1**: Data view `labs-logstash-app-*`, función Count sobre `@timestamp` (Date histogram). Etiqueta la serie como `Logstash`.
      - **Capa 2**: Haz clic en el botón **+** (Add layer) → selecciona "New visualization layer" con data view `labs-elastic-agent`, función Count sobre `@timestamp`. Etiqueta como `Elastic Agent`.
    - **Título**: `Volumen de Logs: Logstash vs Elastic Agent`

18. **Save and return**.

#### 3.7 Organizar y Guardar el Dashboard

19. Reorganiza las visualizaciones en el dashboard arrastrándolas:
    - Fila superior: Métrica (ancho pequeño) + Barras apiladas (ancho amplio).
    - Fila media: Tabla de top errores (mitad) + Mapa geográfico (mitad).
    - Fila inferior: Gráfico de líneas comparativo (ancho completo).

20. Haz clic en **Save** para guardar el dashboard final.

### Salida Esperada

El dashboard `Labs - Application Error Monitor` debe mostrar:
- Un número grande con el total de errores en las últimas 24h.
- Un gráfico de barras con colores diferenciados por nivel (ERROR en rojo, WARN en amarillo, INFO en azul/verde).
- Una tabla con las clases de excepción más frecuentes.
- Un mapa con puntos o regiones coloreadas según volumen de eventos.
- Dos líneas temporales comparando el volumen de las dos fuentes.

### Verificación

- [ ] El dashboard contiene exactamente 5 visualizaciones.
- [ ] Cada visualización muestra datos reales (no "No results found").
- [ ] El título del dashboard es `Labs - Application Error Monitor`.
- [ ] El rango temporal por defecto es "Last 24 hours".

---

## Paso 4: Crear el Conector Server Log

### Objetivo

Configurar un conector de tipo "Server log" en Kibana que servirá como destino de notificación para la regla de alerta.

### Instrucciones

1. Navega a **Stack Management** → **Rules and Connectors** → pestaña **Connectors**.

2. Haz clic en **Create connector**.

3. Selecciona el tipo **Server log**.

4. Configura:
   - **Connector name**: `Labs - Server Log Connector`

5. Haz clic en **Save**. No requiere configuración adicional ya que escribe directamente en los logs de Kibana.

### Salida Esperada

El conector aparece en la lista de conectores con estado "Available" (ícono verde).

### Verificación

- [ ] El conector `Labs - Server Log Connector` aparece en la lista de conectores activos.

---

## Paso 5: Configurar la Regla de Alerta de Picos de Errores

### Objetivo

Crear una regla de alerta llamada `labs-error-spike-alert` de tipo "Elasticsearch query" que se dispare cuando el conteo de documentos con `log.level: ERROR` supere 10 en una ventana de 5 minutos.

### Instrucciones

1. Navega a **Stack Management** → **Rules** (o **Alerting** → **Rules** en el menú lateral bajo Management).

2. Haz clic en **Create rule**.

3. Configura los campos generales:
   - **Name**: `labs-error-spike-alert`
   - **Tags**: `labs`, `errors`
   - **Check every**: `1 minute`

4. En **Rule type**, selecciona **Elasticsearch query** (bajo la categoría "Stack Rules" o "Observability" dependiendo de la versión).

5. Configura la condición de la regla:
   - **Index**: Escribe `labs-logstash-app-*` (o selecciona el data view correspondiente si la UI lo permite).
   - **Time field**: `@timestamp`
   - **Define your query using**: **KQL**
   - **Query**: 
     ```
     log.level: "ERROR"
     ```
   - **Condition**: `IS ABOVE` → **Threshold**: `10`
   - **Time window**: `5 minutes`
   - **Size** (documentos a retornar en la alerta): `100`

6. En la sección **Actions**, haz clic en **Add action** → selecciona el conector `Labs - Server Log Connector`.

7. Configura el mensaje de la acción:
   - **Message**:
     ```
     [ALERTA] Se detectaron {{context.hits}} errores en los últimos 5 minutos. Umbral: 10. Timestamp: {{context.date}}. Documentos: {{context.conditions.resultsLink}}
     ```
   
   > **Nota**: Las variables de contexto disponibles pueden variar. Si `{{context.hits}}` no es reconocida, usa `{{context.value}}` o `{{context.title}}`. Consulta la documentación del template de la regla.

8. Haz clic en **Save**.

9. Verifica que la regla aparece en la lista con estado **Active** (verde).

### Salida Esperada

La regla `labs-error-spike-alert` aparece en Stack Management → Rules con:
- Estado: Active
- Última ejecución: muestra el timestamp de la última verificación
- Check interval: 1 minute

### Verificación

- [ ] La regla está creada con el nombre `labs-error-spike-alert`.
- [ ] El tipo de regla es "Elasticsearch query".
- [ ] El umbral está configurado en >10 documentos en 5 minutos.
- [ ] La acción apunta al conector `Labs - Server Log Connector`.

---

## Paso 6: Validar la Regla de Alerta con un Pico Artificial de Errores

### Objetivo

Generar un pico controlado de errores mediante un script Python para confirmar que la regla de alerta se activa correctamente.

### Instrucciones

1. Crea el script generador de picos de errores:

```bash
cat > ~/elastic-labs/scripts/error_spike_generator.py << 'EOF'
#!/usr/bin/env python3
"""
Genera un pico de errores para validar la regla de alerta labs-error-spike-alert.
Envía 20 documentos con log.level: ERROR al input HTTP de Logstash.
"""

import json
import time
import random
import requests
from datetime import datetime, timezone

LOGSTASH_HTTP_URL = "http://localhost:8080"
NUM_ERRORS = 20

error_types = [
    "NullPointerException",
    "IOException",
    "TimeoutException",
    "OutOfMemoryError",
    "IllegalStateException"
]

messages = [
    "Failed to process request: connection refused",
    "Null reference in OrderService.processPayment()",
    "Timeout waiting for database response after 30000ms",
    "Heap space exhausted during batch processing",
    "Invalid state transition from PENDING to COMPLETED"
]

def generate_error_event():
    error_type = random.choice(error_types)
    return {
        "@timestamp": datetime.now(timezone.utc).isoformat(),
        "log": {"level": "ERROR"},
        "message": random.choice(messages),
        "error": {"type": error_type},
        "service": {"name": "labs-spike-test"},
        "host": {"name": "spike-generator-01"},
        "geoip": {
            "location": {
                "lat": round(random.uniform(19.0, 51.0), 4),
                "lon": round(random.uniform(-122.0, -73.0), 4)
            }
        }
    }

def main():
    print(f"[*] Generando {NUM_ERRORS} eventos de error para validar alerta...")
    success_count = 0
    
    for i in range(NUM_ERRORS):
        event = generate_error_event()
        try:
            response = requests.post(
                LOGSTASH_HTTP_URL,
                json=event,
                headers={"Content-Type": "application/json"},
                timeout=5
            )
            if response.status_code == 200:
                success_count += 1
            else:
                print(f"  [!] Evento {i+1}: HTTP {response.status_code}")
        except requests.exceptions.RequestException as e:
            print(f"  [!] Evento {i+1}: Error de conexión - {e}")
        
        time.sleep(0.2)  # Pequeña pausa entre eventos
    
    print(f"[+] Enviados {success_count}/{NUM_ERRORS} eventos exitosamente.")
    print(f"[*] Esperando ~60-120 segundos para que la regla se evalúe...")
    print(f"[*] Verifica en Kibana: Stack Management > Rules > labs-error-spike-alert")

if __name__ == "__main__":
    main()
EOF

chmod +x ~/elastic-labs/scripts/error_spike_generator.py
```

2. Verifica que el input HTTP de Logstash está activo:

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"test": "connectivity"}'
```

**Salida esperada**: `200`

3. Instala la dependencia `requests` si no está disponible:

```bash
pip3 install requests 2>/dev/null || pip install requests
```

4. Ejecuta el script:

```bash
cd ~/elastic-labs/scripts/
python3 error_spike_generator.py
```

5. Espera 60-120 segundos para que la regla se evalúe (intervalo de 1 minuto).

6. Verifica que los documentos llegaron al índice:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_search?pretty" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [
          {"term": {"log.level": "ERROR"}},
          {"term": {"service.name": "labs-spike-test"}}
        ],
        "filter": [
          {"range": {"@timestamp": {"gte": "now-10m"}}}
        ]
      }
    },
    "size": 3
  }' | jq '.hits.total.value'
```

**Salida esperada**: `20` (o un número cercano).

7. Verifica el estado de la alerta en Kibana:
   - Navega a **Stack Management** → **Rules**.
   - Localiza `labs-error-spike-alert`.
   - Verifica que la columna **Last response** muestra `Active` (alerta disparada) en color naranja/rojo.
   - Haz clic en la regla para ver el historial de ejecuciones.

8. Confirma la notificación en los logs de Kibana:

```bash
docker logs kibana01 2>&1 | grep -i "ALERTA\|alert\|labs-error-spike" | tail -5
```

### Salida Esperada

```
[*] Generando 20 eventos de error para validar alerta...
[+] Enviados 20/20 eventos exitosamente.
[*] Esperando ~60-120 segundos para que la regla se evalúe...
[*] Verifica en Kibana: Stack Management > Rules > labs-error-spike-alert
```

En Kibana, la regla debe mostrar estado "Active" con el último disparo reciente.

### Verificación

- [ ] El script envió exitosamente 20 eventos de error.
- [ ] Los documentos aparecen en el índice `labs-logstash-app-*` con `service.name: "labs-spike-test"`.
- [ ] La regla `labs-error-spike-alert` cambió a estado "Active" (alerta disparada).
- [ ] Los logs de Kibana contienen la notificación del Server log connector.

---

## Validación Final y Testing

### Lista de Verificación Completa

Ejecuta las siguientes comprobaciones para confirmar que toda la práctica se completó correctamente:

```bash
# 1. Verificar que el dashboard existe
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/saved_objects/_find?type=dashboard&search_fields=title&search=Labs%20-%20Application%20Error%20Monitor" \
  -H "kbn-xsrf: true" | jq '.total'
```

**Esperado**: `1`

```bash
# 2. Verificar que la regla existe y está activa
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/alerting/rules/_find?search=labs-error-spike-alert" \
  -H "kbn-xsrf: true" | jq '.data[0] | {name, enabled, executionStatus: .execution_status.status}'
```

**Esperado**:
```json
{
  "name": "labs-error-spike-alert",
  "enabled": true,
  "executionStatus": "active"
}
```

```bash
# 3. Verificar que el conector existe
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/actions/connectors" \
  -H "kbn-xsrf: true" | jq '.[] | select(.name == "Labs - Server Log Connector") | {name, connector_type_id, is_preconfigured}'
```

**Esperado**:
```json
{
  "name": "Labs - Server Log Connector",
  "connector_type_id": ".server-log",
  "is_preconfigured": false
}
```

```bash
# 4. Verificar búsqueda guardada
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/saved_objects/_find?type=search&search_fields=title&search=Labs%20-%20Errores%20NPE" \
  -H "kbn-xsrf: true" | jq '.total'
```

**Esperado**: `1`

### Criterios de Aceptación

| Criterio | Cumplido |
|----------|----------|
| Dashboard con 5 visualizaciones funcionales | ☐ |
| Consultas KQL ejecutadas correctamente en Discover | ☐ |
| Consultas ES|QL con resultados agregados | ☐ |
| Regla de alerta configurada y validada | ☐ |
| Conector Server log funcional | ☐ |
| Pico de errores generado y detectado | ☐ |

---

## Solución de Problemas

### Problema 1: La regla de alerta no se dispara tras generar el pico

**Síntomas**: La regla permanece en estado "OK" (verde) después de enviar los 20 eventos de error y esperar más de 2 minutos.

**Causa**: El índice configurado en la regla no coincide con el índice real donde Logstash escribe los documentos. Esto ocurre si el patrón de índice en la regla es `labs-logstash-app-*` pero Logstash escribe con un formato de fecha diferente (por ejemplo, `labs-logstash-app-2025.07.10`), o si el campo `log.level` se indexó con mayúsculas/minúsculas diferentes.

**Solución**:

```bash
# Verificar el nombre exacto del índice donde se escribieron los eventos
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-logstash-app-*?v&s=index" 

# Verificar el mapping del campo log.level
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_mapping/field/log.level?pretty"

# Si el campo es log.level.keyword, ajustar la consulta KQL en la regla a:
# log.level.keyword: "ERROR"
# O si el valor se almacena en minúsculas:
# log.level: "error"
```

Edita la regla en Stack Management → Rules → `labs-error-spike-alert` → Edit, corrige el patrón de índice o la consulta KQL, y guarda. Vuelve a ejecutar el script.

---

### Problema 2: La visualización de mapa muestra "No results found"

**Síntomas**: La visualización 4 (mapa de distribución geográfica) no muestra datos, mientras las otras visualizaciones sí funcionan correctamente.

**Causa**: El campo `geoip.location` no tiene el tipo de mapping `geo_point` en el índice, o los documentos existentes (de prácticas anteriores) no contienen datos GeoIP porque el filtro GeoIP de Logstash no estaba configurado o las IPs eran privadas (127.0.0.1, 192.168.x.x) que no se pueden geolocalizar.

**Solución**:

```bash
# Verificar si el campo geoip.location existe y su tipo
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-logstash-app-*/_mapping/field/geoip.location?pretty"

# Si no existe como geo_point, actualizar el index template
curl -sk -u elastic:ElasticLabs2024! \
  -X PUT "https://localhost:9200/_index_template/labs-logstash-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["labs-logstash-app-*"],
    "template": {
      "mappings": {
        "properties": {
          "geoip": {
            "properties": {
              "location": {"type": "geo_point"}
            }
          }
        }
      }
    }
  }'

# Reindexar o regenerar datos con el script (que ya incluye coordenadas válidas)
python3 ~/elastic-labs/scripts/error_spike_generator.py
```

Alternativamente, cambia la fuente de la visualización al data view `labs-nginx-sample` si este contiene datos GeoIP válidos, y actualiza el filtro del panel para mostrar solo errores.

---

## Limpieza

Si deseas revertir los cambios realizados en esta práctica (opcional, solo si necesitas reiniciar):

```bash
# Eliminar los documentos de prueba del spike generator
curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:9200/labs-logstash-app-*/_delete_by_query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "term": {"service.name": "labs-spike-test"}
    }
  }'

# Desactivar la regla de alerta (sin eliminarla)
RULE_ID=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:5601/api/alerting/rules/_find?search=labs-error-spike-alert" \
  -H "kbn-xsrf: true" | jq -r '.data[0].id')

curl -sk -u elastic:ElasticLabs2024! \
  -X POST "https://localhost:5601/api/alerting/rule/${RULE_ID}/_disable" \
  -H "kbn-xsrf: true"

echo "Regla desactivada. Dashboard y búsquedas guardadas permanecen intactos."
```

> **Nota**: No se recomienda eliminar el dashboard ni las visualizaciones ya que representan el entregable principal de esta práctica.

---

## Resumen

En esta práctica integradora has completado el ciclo completo de observabilidad operativa sobre el Elastic Stack:

| Actividad | Herramienta | Resultado |
|-----------|-------------|-----------|
| Exploración de datos con filtros | Discover + KQL | Identificación de patrones de error |
| Análisis agregado tabular | Discover + ES|QL | Conteos por tipo y distribución temporal |
| Visualización operativa | Lens + Dashboard | 5 visualizaciones en dashboard integrado |
| Monitoreo proactivo | Alerting + Connectors | Regla automática con notificación |
| Validación end-to-end | Python + API | Confirmación de disparo de alerta |

### Conceptos Clave Reforzados

- **KQL** es ideal para filtrado rápido e interactivo; **ES|QL** extiende las capacidades con agregaciones y transformaciones en una sola consulta.
- **Lens** permite crear visualizaciones sin código, con soporte para múltiples capas y data views en un mismo gráfico.
- Las **reglas de alerta** de tipo "Elasticsearch query" evalúan consultas periódicamente y disparan acciones cuando se superan umbrales definidos.
- El flujo completo (ingestión → indexación → exploración → visualización → alerta) demuestra la integración de todos los componentes del Elastic Stack trabajando en conjunto.

### Recursos Adicionales

- [Documentación oficial de Kibana Discover](https://www.elastic.co/guide/en/kibana/8.14/discover.html)
- [Referencia de sintaxis KQL](https://www.elastic.co/guide/en/kibana/8.14/kuery-query.html)
- [Guía de ES|QL](https://www.elastic.co/guide/en/elasticsearch/reference/8.14/esql.html)
- [Configuración de reglas de alerta](https://www.elastic.co/guide/en/kibana/8.14/alerting-getting-started.html)
- [Lens visualization editor](https://www.elastic.co/guide/en/kibana/8.14/lens.html)
