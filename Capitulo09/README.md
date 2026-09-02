# Medir y optimizar un pipeline de ingestión de logs

## Metadata

| Campo | Valor |
|-------|-------|
| **Duración** | 54 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |

## Descripción General

En este laboratorio ejecutarás un ciclo completo de benchmark, análisis, optimización y validación de un pipeline de ingestión de logs. Generarás 50.000 eventos sintéticos con un script Python, medirás la línea base de rendimiento (throughput, latencia, backlog), identificarás cuellos de botella mediante las APIs de monitoreo de Logstash y Elasticsearch, aplicarás optimizaciones específicas y compararás el rendimiento entre Logstash+Filebeat y Elastic Agent.

## Objetivos de Aprendizaje

- [ ] Establecer una línea base de rendimiento midiendo throughput (EPS), latencia de procesamiento y tamaño del backlog bajo carga controlada
- [ ] Identificar cuellos de botella en el pipeline usando métricas de la API de monitoreo de Logstash y estadísticas de Elasticsearch
- [ ] Ajustar `pipeline.workers`, `pipeline.batch.size` y compresión HTTP para maximizar el throughput sin degradar la latencia
- [ ] Comparar el rendimiento de Filebeat vs Elastic Agent para recolección de logs de archivo y documentar diferencias cuantitativas

## Prerrequisitos

### Conocimiento previo

- Comprensión de la arquitectura Elastic Stack y el rol de cada componente
- Familiaridad con configuración de pipelines de Logstash (inputs, filters, outputs)
- Experiencia con las APIs REST de Elasticsearch y Logstash
- Conceptos de throughput, latencia y backlog (Lección 9.1)

### Acceso requerido

- Entorno del Lab 8 completado con ILM, SLM y seguridad configurados
- Stack Elastic (Elasticsearch, Kibana, Logstash) 8.14.1 operativo en Docker
- Python 3.12.3 con paquetes `faker` y `requests` instalados
- Filebeat 8.14.1 instalado o disponible como contenedor
- Acceso a `localhost:9200` (Elasticsearch), `localhost:9600` (Logstash API), `localhost:5601` (Kibana)

## Entorno del Laboratorio

### Software utilizado

| Componente | Versión | Puerto |
|------------|---------|--------|
| Elasticsearch | 8.14.1 | 9200 |
| Kibana | 8.14.1 | 5601 |
| Logstash | 8.14.1 | 9600 (API), 5044 (Beats) |
| Filebeat | 8.14.1 | 5066 (métricas HTTP) |
| Python | 3.12.3 | — |
| faker | 24.3.0 | — |

### Preparación inicial del entorno

```bash
# Verificar que el stack está corriendo
cd ~/elastic-labs/
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Verificar conectividad con Elasticsearch
curl -sk -u elastic:ElasticLabs2024! https://localhost:9200/_cluster/health?pretty | jq '.status'

# Verificar API de monitoreo de Logstash
curl -s http://localhost:9600/_node/stats | jq '.status'

# Crear directorio de trabajo para este lab
mkdir -p ~/elastic-labs/lab09/{configs,results}
```

## Paso a Paso

---

### Paso 1: Crear el generador de logs sintéticos

**Objetivo:** Crear un script Python que genere 50.000 eventos de log en formato JSON a una tasa controlada de 1.000 eventos/segundo.

**Instrucciones:**

1. Crea el script generador:

```bash
cat > ~/elastic-labs/scripts/benchmark_log_generator.py << 'EOF'
#!/usr/bin/env python3
"""Generador de logs sintéticos para benchmark de ingestión."""

import json
import time
import sys
import os
from datetime import datetime, timezone
from faker import Faker

fake = Faker()

# Configuración
TOTAL_EVENTS = 50000
RATE_PER_SECOND = 1000
OUTPUT_FILE = "/tmp/benchmark-app.log"

# Niveles de log con distribución realista
LOG_LEVELS = ["INFO"] * 70 + ["WARN"] * 15 + ["ERROR"] * 10 + ["DEBUG"] * 5

# Servicios simulados
SERVICES = ["auth-service", "payment-api", "user-service", "order-processor", "notification-worker"]

def generate_event(seq_num):
    """Genera un evento de log en formato JSON."""
    return {
        "@timestamp": datetime.now(timezone.utc).isoformat(),
        "sequence": seq_num,
        "level": fake.random_element(LOG_LEVELS),
        "service": fake.random_element(SERVICES),
        "message": fake.sentence(nb_words=12),
        "source_ip": fake.ipv4(),
        "user_id": fake.uuid4(),
        "response_time_ms": fake.random_int(min=1, max=5000),
        "http_method": fake.random_element(["GET", "POST", "PUT", "DELETE"]),
        "http_path": fake.uri_path(deep=3),
        "http_status": fake.random_element([200, 200, 200, 201, 301, 400, 403, 404, 500]),
        "bytes_sent": fake.random_int(min=100, max=50000),
        "trace_id": fake.hexify(text="^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^")
    }

def main():
    """Genera eventos a tasa controlada."""
    # Limpiar archivo previo
    if os.path.exists(OUTPUT_FILE):
        os.remove(OUTPUT_FILE)

    print(f"[*] Generando {TOTAL_EVENTS} eventos a {RATE_PER_SECOND} EPS...")
    print(f"[*] Archivo de salida: {OUTPUT_FILE}")

    start_time = time.time()
    batch_size = 100  # Escribir en lotes para eficiencia de I/O
    events_written = 0

    with open(OUTPUT_FILE, "w") as f:
        while events_written < TOTAL_EVENTS:
            batch_start = time.time()

            # Escribir un lote
            for i in range(min(batch_size, TOTAL_EVENTS - events_written)):
                event = generate_event(events_written + i)
                f.write(json.dumps(event) + "\n")

            events_written += min(batch_size, TOTAL_EVENTS - events_written)
            f.flush()

            # Control de tasa
            elapsed_in_batch = time.time() - batch_start
            expected_time = batch_size / RATE_PER_SECOND
            if elapsed_in_batch < expected_time:
                time.sleep(expected_time - elapsed_in_batch)

            # Progreso cada 10%
            if events_written % (TOTAL_EVENTS // 10) == 0:
                elapsed = time.time() - start_time
                rate = events_written / elapsed if elapsed > 0 else 0
                print(f"    Progreso: {events_written}/{TOTAL_EVENTS} "
                      f"({events_written*100//TOTAL_EVENTS}%) - "
                      f"Tasa real: {rate:.0f} EPS")

    total_time = time.time() - start_time
    file_size = os.path.getsize(OUTPUT_FILE) / (1024 * 1024)
    print(f"\n[✓] Completado: {events_written} eventos en {total_time:.1f}s")
    print(f"    Tasa media: {events_written/total_time:.0f} EPS")
    print(f"    Tamaño archivo: {file_size:.2f} MB")

if __name__ == "__main__":
    main()
EOF

chmod +x ~/elastic-labs/scripts/benchmark_log_generator.py
```

2. Instala las dependencias necesarias (si no están disponibles):

```bash
pip3 install faker==24.3.0 requests==2.31.0 --quiet
```

3. Ejecuta el generador para crear el archivo de benchmark:

```bash
python3 ~/elastic-labs/scripts/benchmark_log_generator.py
```

**Salida esperada:**

```
[*] Generando 50000 eventos a 1000 EPS...
[*] Archivo de salida: /tmp/benchmark-app.log
    Progreso: 5000/50000 (10%) - Tasa real: 998 EPS
    Progreso: 10000/50000 (20%) - Tasa real: 1001 EPS
    ...
    Progreso: 50000/50000 (100%) - Tasa real: 999 EPS

[✓] Completado: 50000 eventos en 50.1s
    Tasa media: 999 EPS
    Tamaño archivo: 28.43 MB
```

**Verificación:**

```bash
# Verificar número de líneas y formato
wc -l /tmp/benchmark-app.log
head -1 /tmp/benchmark-app.log | python3 -m json.tool
```

Debe mostrar exactamente 50000 líneas y el primer evento debe ser JSON válido con los campos definidos.

---

### Paso 2: Configurar el pipeline de Logstash con línea base (sin optimizar)

**Objetivo:** Configurar Logstash con parámetros conservadores (1 worker, batch_size 125) para establecer la línea base de rendimiento.

**Instrucciones:**

1. Crea la configuración de pipeline para benchmark:

```bash
cat > ~/elastic-labs/lab09/configs/benchmark-baseline.conf << 'EOF'
input {
  file {
    path => "/tmp/benchmark-app.log"
    start_position => "beginning"
    sincedb_path => "/tmp/sincedb_benchmark"
    codec => "json"
    mode => "read"
    file_completed_action => "log"
    file_completed_log_path => "/tmp/file_completed.log"
  }
}

filter {
  # Simulación de procesamiento con grok (intencionalmente subóptimo para la línea base)
  mutate {
    add_field => {
      "[@metadata][pipeline]" => "benchmark-baseline"
      "[event][dataset]" => "benchmark.app"
    }
  }

  # Parseo de timestamp
  date {
    match => ["@timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Enriquecimiento: clasificar por nivel
  if [level] == "ERROR" {
    mutate { add_tag => ["high_priority"] }
  } else if [level] == "WARN" {
    mutate { add_tag => ["medium_priority"] }
  }

  # Conversión de tipos
  mutate {
    convert => {
      "response_time_ms" => "integer"
      "bytes_sent" => "integer"
      "sequence" => "integer"
    }
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca/ca.crt"]
    index => "labs-benchmark-baseline-%{+YYYY.MM.dd}"
    http_compression => false
  }
}
EOF
```

2. Crea la configuración de Logstash con parámetros conservadores:

```bash
cat > ~/elastic-labs/lab09/configs/logstash-baseline.yml << 'EOF'
# Configuración de línea base - SIN optimizar
pipeline.workers: 1
pipeline.batch.size: 125
pipeline.batch.delay: 50

# Monitoreo habilitado
api.http.host: "0.0.0.0"
api.http.port: 9600

# Cola en memoria (por defecto)
queue.type: memory

# Logging
log.level: info
EOF
```

3. Crea el índice de destino con configuración de un solo shard:

```bash
curl -sk -u elastic:ElasticLabs2024! -X PUT "https://localhost:9200/labs-benchmark-baseline-$(date +%Y.%m.%d)" \
  -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "1s"
  }
}'
```

4. Detén el pipeline actual de Logstash y lanza con la configuración de línea base:

```bash
# Detener el contenedor actual
docker stop logstash01

# Lanzar Logstash con la configuración de benchmark
docker run -d --rm \
  --name logstash-benchmark \
  --network elastic-net \
  -v ~/elastic-labs/lab09/configs/benchmark-baseline.conf:/usr/share/logstash/pipeline/benchmark.conf:ro \
  -v ~/elastic-labs/lab09/configs/logstash-baseline.yml:/usr/share/logstash/config/logstash.yml:ro \
  -v ~/elastic-labs/config/certs:/usr/share/logstash/config/certs:ro \
  -v /tmp/benchmark-app.log:/tmp/benchmark-app.log:ro \
  -p 9600:9600 \
  docker.elastic.co/logstash/logstash:8.14.1
```

5. Espera a que Logstash inicie y verifique el pipeline:

```bash
# Esperar inicio (30-45 segundos)
sleep 40

# Verificar que el pipeline está activo
curl -s http://localhost:9600/_node/stats/pipelines | jq '.pipelines | keys'
```

**Salida esperada:**

```json
[
  "main"
]
```

**Verificación:**

```bash
curl -s http://localhost:9600/_node/stats/pipelines/main | jq '.events'
```

Debe mostrar los contadores `in`, `out`, `filtered` incrementándose.

---

### Paso 3: Medir la línea base de rendimiento

**Objetivo:** Capturar métricas de throughput, latencia y backlog durante la ingestión de los 50.000 eventos con la configuración sin optimizar.

**Instrucciones:**

1. Crea un script de medición automática:

```bash
cat > ~/elastic-labs/scripts/measure_pipeline.sh << 'SCRIPT'
#!/bin/bash
# Script de medición de rendimiento del pipeline de Logstash
# Uso: ./measure_pipeline.sh <nombre_medicion> <duracion_segundos>

MEASUREMENT_NAME=${1:-"baseline"}
DURATION=${2:-60}
OUTPUT_DIR=~/elastic-labs/lab09/results
LOGSTASH_API="http://localhost:9600"
ES_API="https://localhost:9200"
ES_CREDS="elastic:ElasticLabs2024!"

mkdir -p $OUTPUT_DIR

echo "═══════════════════════════════════════════════════"
echo " Medición: $MEASUREMENT_NAME"
echo " Duración: ${DURATION}s"
echo "═══════════════════════════════════════════════════"

# Muestra inicial
echo "[$(date +%H:%M:%S)] Capturando muestra inicial..."
STATS_START=$(curl -s ${LOGSTASH_API}/_node/stats/pipelines/main)

EVENTS_IN_START=$(echo $STATS_START | jq '.events.in')
EVENTS_OUT_START=$(echo $STATS_START | jq '.events.out')
DURATION_MS_START=$(echo $STATS_START | jq '.events.duration_in_millis')
QUEUE_EVENTS_START=$(echo $STATS_START | jq '.queue.events_count // 0')

echo "  events.in:  $EVENTS_IN_START"
echo "  events.out: $EVENTS_OUT_START"
echo "  queue:      $QUEUE_EVENTS_START eventos"

# Esperar duración de medición
echo "[$(date +%H:%M:%S)] Midiendo durante ${DURATION}s..."
sleep $DURATION

# Muestra final
echo "[$(date +%H:%M:%S)] Capturando muestra final..."
STATS_END=$(curl -s ${LOGSTASH_API}/_node/stats/pipelines/main)

EVENTS_IN_END=$(echo $STATS_END | jq '.events.in')
EVENTS_OUT_END=$(echo $STATS_END | jq '.events.out')
DURATION_MS_END=$(echo $STATS_END | jq '.events.duration_in_millis')
QUEUE_EVENTS_END=$(echo $STATS_END | jq '.queue.events_count // 0')

echo "  events.in:  $EVENTS_IN_END"
echo "  events.out: $EVENTS_OUT_END"
echo "  queue:      $QUEUE_EVENTS_END eventos"

# Calcular métricas
EVENTS_PROCESSED=$((EVENTS_OUT_END - EVENTS_OUT_START))
EVENTS_RECEIVED=$((EVENTS_IN_END - EVENTS_IN_START))
THROUGHPUT_EPS=$((EVENTS_PROCESSED / DURATION))
BACKLOG=$((EVENTS_IN_END - EVENTS_OUT_END))
DURATION_DIFF=$((DURATION_MS_END - DURATION_MS_START))

if [ $EVENTS_PROCESSED -gt 0 ]; then
  LATENCY_AVG_MS=$((DURATION_DIFF / EVENTS_PROCESSED))
else
  LATENCY_AVG_MS=0
fi

# Métricas de Elasticsearch
ES_INDEX_STATS=$(curl -sk -u $ES_CREDS "${ES_API}/_cat/indices/labs-benchmark-*?format=json&h=index,docs.count,store.size" 2>/dev/null)
ES_DOCS=$(echo $ES_INDEX_STATS | jq '.[0]["docs.count"] // "0"' -r)
ES_SIZE=$(echo $ES_INDEX_STATS | jq '.[0]["store.size"] // "0b"' -r)

# Métricas por plugin
PLUGIN_STATS=$(curl -s ${LOGSTASH_API}/_node/stats/pipelines/main 2>/dev/null)
FILTER_DURATION=$(echo $PLUGIN_STATS | jq '[.pipelines.main.plugins.filters[].events.duration_in_millis] | add // 0')
OUTPUT_DURATION=$(echo $PLUGIN_STATS | jq '[.pipelines.main.plugins.outputs[].events.duration_in_millis] | add // 0')

echo ""
echo "═══════════════════════════════════════════════════"
echo " RESULTADOS: $MEASUREMENT_NAME"
echo "═══════════════════════════════════════════════════"
echo " Throughput:        $THROUGHPUT_EPS EPS"
echo " Latencia media:    ${LATENCY_AVG_MS} ms/evento"
echo " Backlog actual:    $BACKLOG eventos"
echo " Queue size:        $QUEUE_EVENTS_END eventos"
echo " Eventos recibidos: $EVENTS_RECEIVED"
echo " Eventos procesados:$EVENTS_PROCESSED"
echo " ES docs indexados: $ES_DOCS"
echo " ES tamaño índice:  $ES_SIZE"
echo " Filter duration:   ${FILTER_DURATION}ms (acum.)"
echo " Output duration:   ${OUTPUT_DURATION}ms (acum.)"
echo "═══════════════════════════════════════════════════"

# Guardar resultados en CSV
RESULT_FILE="${OUTPUT_DIR}/metrics_${MEASUREMENT_NAME}.csv"
echo "metric,value" > $RESULT_FILE
echo "throughput_eps,$THROUGHPUT_EPS" >> $RESULT_FILE
echo "latency_avg_ms,$LATENCY_AVG_MS" >> $RESULT_FILE
echo "backlog,$BACKLOG" >> $RESULT_FILE
echo "queue_events,$QUEUE_EVENTS_END" >> $RESULT_FILE
echo "events_processed,$EVENTS_PROCESSED" >> $RESULT_FILE
echo "filter_duration_ms,$FILTER_DURATION" >> $RESULT_FILE
echo "output_duration_ms,$OUTPUT_DURATION" >> $RESULT_FILE
echo "es_docs,$ES_DOCS" >> $RESULT_FILE

echo ""
echo "[✓] Resultados guardados en: $RESULT_FILE"
SCRIPT

chmod +x ~/elastic-labs/scripts/measure_pipeline.sh
```

2. Ejecuta la medición de línea base (espera a que Logstash esté procesando):

```bash
# Verificar que hay eventos fluyendo
curl -s http://localhost:9600/_node/stats/pipelines/main | jq '.events.out'

# Ejecutar medición de 60 segundos
~/elastic-labs/scripts/measure_pipeline.sh "baseline" 60
```

3. Captura métricas detalladas por plugin:

```bash
# Obtener estadísticas detalladas de cada plugin
curl -s http://localhost:9600/_node/stats/pipelines/main | \
  jq '{
    filters: [.pipelines.main.plugins.filters[] | {
      name: .name,
      events_in: .events.in,
      events_out: .events.out,
      duration_ms: .events.duration_in_millis
    }],
    outputs: [.pipelines.main.plugins.outputs[] | {
      name: .name,
      events_in: .events.in,
      events_out: .events.out,
      duration_ms: .events.duration_in_millis
    }]
  }' | tee ~/elastic-labs/lab09/results/plugin_stats_baseline.json
```

4. Captura estadísticas de indexación de Elasticsearch:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_nodes/stats/indices/indexing" | \
  jq '.nodes | to_entries[0].value.indices.indexing | {
    index_total: .index_total,
    index_time_ms: .index_time_in_millis,
    index_current: .index_current,
    avg_ms_per_doc: (if .index_total > 0 then (.index_time_in_millis / .index_total) else 0 end)
  }' | tee ~/elastic-labs/lab09/results/es_indexing_baseline.json
```

**Salida esperada (ejemplo):**

```
═══════════════════════════════════════════════════
 RESULTADOS: baseline
═══════════════════════════════════════════════════
 Throughput:        850 EPS
 Latencia media:    3 ms/evento
 Backlog actual:    4200 eventos
 Queue size:        312 eventos
 Eventos recibidos: 52100
 Eventos procesados:51000
 ES docs indexados:  48500
 ES tamaño índice:  22.1mb
 Filter duration:   28500ms (acum.)
 Output duration:   142000ms (acum.)
═══════════════════════════════════════════════════
```

**Verificación:**

```bash
# Confirmar que los archivos de resultados existen
ls -la ~/elastic-labs/lab09/results/
cat ~/elastic-labs/lab09/results/metrics_baseline.csv
```

---

### Paso 4: Identificar cuellos de botella

**Objetivo:** Analizar las métricas recolectadas para determinar si el cuello de botella está en el input, filter u output del pipeline.

**Instrucciones:**

1. Analiza la distribución del tiempo por etapa:

```bash
# Obtener tiempos acumulados por etapa
curl -s http://localhost:9600/_node/stats/pipelines/main | \
  python3 -c "
import sys, json

data = json.load(sys.stdin)
pipeline = data.get('pipelines', {}).get('main', data)

# Tiempos por plugin
filters = pipeline.get('plugins', {}).get('filters', [])
outputs = pipeline.get('plugins', {}).get('outputs', [])

print('=' * 60)
print(' ANÁLISIS DE CUELLOS DE BOTELLA')
print('=' * 60)

total_filter_ms = 0
print('\n FILTROS:')
print(f' {\"Plugin\":<20} {\"Eventos\":<12} {\"Duración(ms)\":<15} {\"ms/evento\":<10}')
print(' ' + '-' * 57)
for f in filters:
    dur = f['events']['duration_in_millis']
    evts = f['events']['out']
    avg = dur / evts if evts > 0 else 0
    total_filter_ms += dur
    print(f' {f[\"name\"]:<20} {evts:<12} {dur:<15} {avg:<10.3f}')

total_output_ms = 0
print('\n OUTPUTS:')
print(f' {\"Plugin\":<20} {\"Eventos\":<12} {\"Duración(ms)\":<15} {\"ms/evento\":<10}')
print(' ' + '-' * 57)
for o in outputs:
    dur = o['events']['duration_in_millis']
    evts = o['events']['out']
    avg = dur / evts if evts > 0 else 0
    total_output_ms += dur
    print(f' {o[\"name\"]:<20} {evts:<12} {dur:<15} {avg:<10.3f}')

total = total_filter_ms + total_output_ms
print(f'\n RESUMEN:')
print(f'   Tiempo total filtros: {total_filter_ms}ms ({total_filter_ms*100//total if total>0 else 0}%)')
print(f'   Tiempo total outputs: {total_output_ms}ms ({total_output_ms*100//total if total>0 else 0}%)')

if total_output_ms > total_filter_ms * 2:
    print('\n ⚠️  CUELLO DE BOTELLA: OUTPUT (Elasticsearch bulk indexing)')
    print('    → Recomendación: aumentar batch_size, habilitar compresión HTTP')
elif total_filter_ms > total_output_ms * 2:
    print('\n ⚠️  CUELLO DE BOTELLA: FILTROS (procesamiento)')
    print('    → Recomendación: aumentar workers, simplificar filtros')
else:
    print('\n ℹ️  Distribución equilibrada entre filtros y output')
    print('    → Recomendación: aumentar workers y batch_size simultáneamente')
print('=' * 60)
"
```

2. Verifica el thread pool de Elasticsearch para detectar rechazos:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/thread_pool/write?v&h=node_name,name,active,queue,rejected,completed"
```

3. Verifica la presión de memoria del heap de Elasticsearch:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_nodes/stats/jvm" | \
  jq '.nodes | to_entries[0].value.jvm | {
    heap_used_percent: .mem.heap_used_percent,
    heap_used: .mem.heap_used_in_bytes,
    heap_max: .mem.heap_max_in_bytes,
    gc_old_count: .gc.collectors.old.collection_count,
    gc_old_time_ms: .gc.collectors.old.collection_time_in_millis
  }'
```

4. Documenta los hallazgos:

```bash
cat > ~/elastic-labs/lab09/results/bottleneck_analysis.md << 'EOF'
# Análisis de Cuellos de Botella - Línea Base

## Hallazgos

| Componente | Tiempo acumulado | % del total | Diagnóstico |
|-----------|-----------------|-------------|-------------|
| Filtros (mutate, date) | Ver medición | Ver % | Normal/Lento |
| Output (elasticsearch) | Ver medición | Ver % | Normal/Lento |

## Cuello de botella identificado

**Componente:** (completar con resultado del análisis)

**Causa probable:** Con 1 worker y batch_size 125, Logstash envía
lotes pequeños a Elasticsearch, incrementando el overhead de cada
operación bulk. Además, sin compresión HTTP, el payload es más grande.

## Optimizaciones propuestas

1. Aumentar `pipeline.workers` de 1 a 4
2. Aumentar `pipeline.batch.size` de 125 a 500
3. Habilitar `http_compression => true` en el output
4. Mantener 1 shard (entorno de nodo único)
EOF

echo "[✓] Análisis documentado en ~/elastic-labs/lab09/results/bottleneck_analysis.md"
```

**Salida esperada:**

El análisis debe revelar que el output de Elasticsearch consume entre 60-80% del tiempo total de procesamiento, indicando que el cuello de botella principal está en la comunicación con Elasticsearch (lotes pequeños, sin compresión).

**Verificación:**

```bash
# El thread pool de write no debe mostrar rechazos (rejected = 0)
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/thread_pool/write?v&h=name,active,queue,rejected" | grep write
```

---

### Paso 5: Aplicar optimizaciones al pipeline

**Objetivo:** Reconfigurar Logstash con parámetros optimizados y medir la mejora de rendimiento.

**Instrucciones:**

1. Detén el pipeline de línea base:

```bash
docker stop logstash-benchmark
```

2. Limpia los datos de estado para repetir la ingestión:

```bash
# Eliminar sincedb para que Logstash relea el archivo
rm -f /tmp/sincedb_benchmark

# Eliminar el índice de línea base
curl -sk -u elastic:ElasticLabs2024! -X DELETE \
  "https://localhost:9200/labs-benchmark-baseline-*"

# Regenerar el archivo de log (mismo contenido, nuevo archivo)
rm -f /tmp/benchmark-app.log
python3 ~/elastic-labs/scripts/benchmark_log_generator.py
```

3. Crea la configuración optimizada de Logstash:

```bash
cat > ~/elastic-labs/lab09/configs/logstash-optimized.yml << 'EOF'
# Configuración OPTIMIZADA
pipeline.workers: 4
pipeline.batch.size: 500
pipeline.batch.delay: 50

# Monitoreo habilitado
api.http.host: "0.0.0.0"
api.http.port: 9600

# Cola en memoria (suficiente para este benchmark)
queue.type: memory

# Logging
log.level: info
EOF
```

4. Crea el pipeline optimizado:

```bash
cat > ~/elastic-labs/lab09/configs/benchmark-optimized.conf << 'EOF'
input {
  file {
    path => "/tmp/benchmark-app.log"
    start_position => "beginning"
    sincedb_path => "/tmp/sincedb_benchmark_opt"
    codec => "json"
    mode => "read"
    file_completed_action => "log"
    file_completed_log_path => "/tmp/file_completed_opt.log"
  }
}

filter {
  # Optimización: el codec json ya parsea, no necesitamos grok
  # Solo enriquecimiento necesario
  mutate {
    add_field => {
      "[@metadata][pipeline]" => "benchmark-optimized"
      "[event][dataset]" => "benchmark.app"
    }
  }

  # Parseo de timestamp (necesario para @timestamp correcto)
  date {
    match => ["@timestamp", "ISO8601"]
    target => "@timestamp"
  }

  # Clasificación por nivel (optimizado con condicional simple)
  if [level] == "ERROR" {
    mutate { add_tag => ["high_priority"] }
  } else if [level] == "WARN" {
    mutate { add_tag => ["medium_priority"] }
  }

  # Conversión de tipos en un solo bloque mutate (más eficiente)
  mutate {
    convert => {
      "response_time_ms" => "integer"
      "bytes_sent" => "integer"
      "sequence" => "integer"
    }
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    user => "elastic"
    password => "ElasticLabs2024!"
    ssl_certificate_authorities => ["/usr/share/logstash/config/certs/ca/ca.crt"]
    index => "labs-benchmark-optimized-%{+YYYY.MM.dd}"
    # OPTIMIZACIÓN: compresión HTTP
    http_compression => true
  }
}
EOF
```

5. Crea el índice optimizado:

```bash
curl -sk -u elastic:ElasticLabs2024! -X PUT \
  "https://localhost:9200/labs-benchmark-optimized-$(date +%Y.%m.%d)" \
  -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "5s",
    "index.translog.durability": "async",
    "index.translog.sync_interval": "5s"
  }
}'
```

6. Lanza Logstash con la configuración optimizada:

```bash
docker run -d --rm \
  --name logstash-benchmark-opt \
  --network elastic-net \
  -v ~/elastic-labs/lab09/configs/benchmark-optimized.conf:/usr/share/logstash/pipeline/benchmark.conf:ro \
  -v ~/elastic-labs/lab09/configs/logstash-optimized.yml:/usr/share/logstash/config/logstash.yml:ro \
  -v ~/elastic-labs/config/certs:/usr/share/logstash/config/certs:ro \
  -v /tmp/benchmark-app.log:/tmp/benchmark-app.log:ro \
  -p 9600:9600 \
  docker.elastic.co/logstash/logstash:8.14.1

# Esperar inicio
sleep 40
```

7. Ejecuta la medición optimizada:

```bash
~/elastic-labs/scripts/measure_pipeline.sh "optimized" 60
```

8. Captura las estadísticas por plugin de la versión optimizada:

```bash
curl -s http://localhost:9600/_node/stats/pipelines/main | \
  jq '{
    filters: [.pipelines.main.plugins.filters[] | {
      name: .name,
      events_out: .events.out,
      duration_ms: .events.duration_in_millis
    }],
    outputs: [.pipelines.main.plugins.outputs[] | {
      name: .name,
      events_out: .events.out,
      duration_ms: .events.duration_in_millis
    }]
  }' | tee ~/elastic-labs/lab09/results/plugin_stats_optimized.json
```

**Salida esperada:**

El throughput optimizado debe ser significativamente mayor que la línea base:

```
═══════════════════════════════════════════════════
 RESULTADOS: optimized
═══════════════════════════════════════════════════
 Throughput:        3200 EPS    (vs 850 EPS baseline = +276%)
 Latencia media:    1 ms/evento (vs 3 ms baseline)
 Backlog actual:    200 eventos (vs 4200 baseline)
 ...
═══════════════════════════════════════════════════
```

**Verificación:**

```bash
# Verificar que todos los 50000 eventos fueron indexados
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-benchmark-optimized-*/_count" | jq '.count'
```

El count debe ser 50000 (o muy cercano si la ingestión aún está en curso).

---

### Paso 6: Comparar rendimiento con Filebeat

**Objetivo:** Desplegar Filebeat apuntando al mismo archivo de log y medir su throughput y latencia para comparar con Logstash.

**Instrucciones:**

1. Detén el pipeline optimizado de Logstash:

```bash
docker stop logstash-benchmark-opt
```

2. Regenera el archivo de benchmark:

```bash
rm -f /tmp/benchmark-app.log
python3 ~/elastic-labs/scripts/benchmark_log_generator.py
```

3. Crea la configuración de Filebeat:

```bash
cat > ~/elastic-labs/lab09/configs/filebeat-benchmark.yml << 'EOF'
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /tmp/benchmark-app.log
    json.keys_under_root: true
    json.overwrite_keys: true
    json.add_error_key: true

# Métricas HTTP habilitadas
http.enabled: true
http.host: "0.0.0.0"
http.port: 5066

# Output directo a Elasticsearch
output.elasticsearch:
  hosts: ["https://es01:9200"]
  username: "elastic"
  password: "ElasticLabs2024!"
  ssl.certificate_authorities: ["/usr/share/filebeat/config/certs/ca/ca.crt"]
  index: "labs-benchmark-filebeat-%{+yyyy.MM.dd}"
  # Configuración de rendimiento
  bulk_max_size: 500
  worker: 4
  compression_level: 3

setup.ilm.enabled: false
setup.template.enabled: false

# Logging
logging.level: info
logging.to_stderr: true
EOF
```

4. Crea el índice de destino para Filebeat:

```bash
curl -sk -u elastic:ElasticLabs2024! -X PUT \
  "https://localhost:9200/labs-benchmark-filebeat-$(date +%Y.%m.%d)" \
  -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "refresh_interval": "5s"
  }
}'
```

5. Lanza Filebeat como contenedor:

```bash
docker run -d --rm \
  --name filebeat-benchmark \
  --network elastic-net \
  -v ~/elastic-labs/lab09/configs/filebeat-benchmark.yml:/usr/share/filebeat/filebeat.yml:ro \
  -v ~/elastic-labs/config/certs:/usr/share/filebeat/config/certs:ro \
  -v /tmp/benchmark-app.log:/tmp/benchmark-app.log:ro \
  -p 5066:5066 \
  docker.elastic.co/beats/filebeat:8.14.1
```

6. Espera a que Filebeat inicie y mide el rendimiento:

```bash
sleep 20

# Crear script de medición para Filebeat
cat > ~/elastic-labs/scripts/measure_filebeat.sh << 'FBSCRIPT'
#!/bin/bash
DURATION=${1:-60}
ES_API="https://localhost:9200"
ES_CREDS="elastic:ElasticLabs2024!"
FB_API="http://localhost:5066"

echo "═══════════════════════════════════════════════════"
echo " Medición Filebeat - ${DURATION}s"
echo "═══════════════════════════════════════════════════"

# Muestra inicial de documentos en ES
DOCS_START=$(curl -sk -u $ES_CREDS "${ES_API}/labs-benchmark-filebeat-*/_count" 2>/dev/null | jq '.count // 0')
echo "[$(date +%H:%M:%S)] Docs iniciales en ES: $DOCS_START"

# Métricas de Filebeat
FB_STATS_START=$(curl -s ${FB_API}/stats 2>/dev/null)
FB_EVENTS_START=$(echo $FB_STATS_START | jq '.libbeat.output.events.total // 0')

sleep $DURATION

# Muestra final
DOCS_END=$(curl -sk -u $ES_CREDS "${ES_API}/labs-benchmark-filebeat-*/_count" 2>/dev/null | jq '.count // 0')
FB_STATS_END=$(curl -s ${FB_API}/stats 2>/dev/null)
FB_EVENTS_END=$(echo $FB_STATS_END | jq '.libbeat.output.events.total // 0')
FB_ACKED=$(echo $FB_STATS_END | jq '.libbeat.output.events.acked // 0')
FB_FAILED=$(echo $FB_STATS_END | jq '.libbeat.output.events.failed // 0')

DOCS_INDEXED=$((DOCS_END - DOCS_START))
THROUGHPUT=$((DOCS_INDEXED / DURATION))
FB_EVENTS_SENT=$((FB_EVENTS_END - FB_EVENTS_START))

echo "[$(date +%H:%M:%S)] Docs finales en ES: $DOCS_END"
echo ""
echo "═══════════════════════════════════════════════════"
echo " RESULTADOS FILEBEAT"
echo "═══════════════════════════════════════════════════"
echo " Throughput (ES):     $THROUGHPUT EPS"
echo " Docs indexados:      $DOCS_INDEXED"
echo " Eventos enviados FB: $FB_EVENTS_SENT"
echo " Total acked:         $FB_ACKED"
echo " Total failed:        $FB_FAILED"
echo "═══════════════════════════════════════════════════"

# Guardar
echo "metric,value" > ~/elastic-labs/lab09/results/metrics_filebeat.csv
echo "throughput_eps,$THROUGHPUT" >> ~/elastic-labs/lab09/results/metrics_filebeat.csv
echo "docs_indexed,$DOCS_INDEXED" >> ~/elastic-labs/lab09/results/metrics_filebeat.csv
echo "events_acked,$FB_ACKED" >> ~/elastic-labs/lab09/results/metrics_filebeat.csv
echo "events_failed,$FB_FAILED" >> ~/elastic-labs/lab09/results/metrics_filebeat.csv

echo "[✓] Resultados guardados"
FBSCRIPT

chmod +x ~/elastic-labs/scripts/measure_filebeat.sh
~/elastic-labs/scripts/measure_filebeat.sh 60
```

**Salida esperada:**

```
═══════════════════════════════════════════════════
 RESULTADOS FILEBEAT
═══════════════════════════════════════════════════
 Throughput (ES):     4500 EPS
 Docs indexados:      50000
 Eventos enviados FB: 50000
 Total acked:         50000
 Total failed:        0
═══════════════════════════════════════════════════
```

**Verificación:**

```bash
# Confirmar total de documentos indexados
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/labs-benchmark-filebeat-*/_count" | jq '.count'
```

---

### Paso 7: Generar tabla comparativa y documentar resultados

**Objetivo:** Consolidar todas las métricas en una tabla comparativa y documentar las conclusiones.

**Instrucciones:**

1. Genera el informe comparativo:

```bash
cat > ~/elastic-labs/scripts/generate_report.py << 'EOF'
#!/usr/bin/env python3
"""Genera informe comparativo de rendimiento."""
import csv
import os

RESULTS_DIR = os.path.expanduser("~/elastic-labs/lab09/results")

def load_metrics(filename):
    """Carga métricas desde CSV."""
    filepath = os.path.join(RESULTS_DIR, filename)
    metrics = {}
    if os.path.exists(filepath):
        with open(filepath) as f:
            reader = csv.DictReader(f)
            for row in reader:
                metrics[row['metric']] = row['value']
    return metrics

baseline = load_metrics("metrics_baseline.csv")
optimized = load_metrics("metrics_optimized.csv")
filebeat = load_metrics("metrics_filebeat.csv")

report = f"""
# Informe Comparativo de Rendimiento - Lab 09

## Fecha: {os.popen('date +%Y-%m-%d').read().strip()}

## Configuración del Test

| Parámetro | Valor |
|-----------|-------|
| Total eventos | 50,000 |
| Tasa de generación | 1,000 EPS |
| Formato | JSON (13 campos por evento) |
| Índice: shards | 1 |
| Índice: réplicas | 0 |

## Tabla Comparativa

| Métrica | Baseline (1w/125b) | Optimizado (4w/500b) | Filebeat (4w/500b) |
|---------|--------------------:|---------------------:|-------------------:|
| Throughput (EPS) | {baseline.get('throughput_eps', 'N/A')} | {optimized.get('throughput_eps', 'N/A')} | {filebeat.get('throughput_eps', 'N/A')} |
| Latencia media (ms) | {baseline.get('latency_avg_ms', 'N/A')} | {optimized.get('latency_avg_ms', 'N/A')} | N/A (directo) |
| Backlog (eventos) | {baseline.get('backlog', 'N/A')} | {optimized.get('backlog', 'N/A')} | 0 (sin cola) |
| Filter duration (ms) | {baseline.get('filter_duration_ms', 'N/A')} | {optimized.get('filter_duration_ms', 'N/A')} | N/A |
| Output duration (ms) | {baseline.get('output_duration_ms', 'N/A')} | {optimized.get('output_duration_ms', 'N/A')} | N/A |

## Optimizaciones Aplicadas

| # | Optimización | Impacto esperado |
|---|-------------|-----------------|
| 1 | `pipeline.workers`: 1 → 4 | Paralelización del procesamiento de filtros |
| 2 | `pipeline.batch.size`: 125 → 500 | Menos operaciones bulk, mayor eficiencia |
| 3 | `http_compression: true` | Reduce tráfico de red ~60-70% |
| 4 | `refresh_interval`: 1s → 5s | Menos refreshes, más throughput de indexación |
| 5 | `translog.durability`: async | Reduce latencia de escritura en disco |

## Conclusiones

1. **Logstash optimizado vs baseline**: El aumento de workers y batch_size
   produce una mejora significativa en throughput (esperado 2-4x).

2. **Filebeat vs Logstash**: Filebeat sin procesamiento intermedio logra
   mayor throughput al enviar directamente a Elasticsearch, eliminando
   la latencia de procesamiento de filtros.

3. **Trade-offs**:
   - Filebeat no permite transformaciones complejas (grok, enrich)
   - Logstash ofrece flexibilidad a costa de latencia adicional
   - Para logs pre-formateados en JSON, Filebeat es más eficiente
   - Para logs que requieren parsing, Logstash es necesario

4. **Cuello de botella principal**: El output a Elasticsearch (bulk indexing)
   es típicamente el factor limitante en entornos de nodo único.
"""

report_path = os.path.join(RESULTS_DIR, "informe_comparativo.md")
with open(report_path, "w") as f:
    f.write(report)

print(report)
print(f"\n[✓] Informe guardado en: {report_path}")
EOF

python3 ~/elastic-labs/scripts/generate_report.py
```

2. Verifica todos los índices de benchmark creados:

```bash
curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-benchmark-*?v&h=index,docs.count,store.size,pri.store.size&s=index"
```

**Salida esperada:**

```
index                                    docs.count store.size pri.store.size
labs-benchmark-filebeat-2024.XX.XX       50000      22.5mb     22.5mb
labs-benchmark-optimized-2024.XX.XX      50000      22.1mb     22.1mb
```

**Verificación:**

```bash
# Verificar que el informe existe y tiene contenido
wc -l ~/elastic-labs/lab09/results/informe_comparativo.md
cat ~/elastic-labs/lab09/results/informe_comparativo.md | head -30
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que el laboratorio se completó correctamente:

```bash
echo "═══════════════════════════════════════════════════"
echo " VALIDACIÓN FINAL DEL LABORATORIO 09"
echo "═══════════════════════════════════════════════════"

PASS=0
FAIL=0

# Test 1: Archivo de benchmark generado
if [ -f /tmp/benchmark-app.log ] && [ $(wc -l < /tmp/benchmark-app.log) -eq 50000 ]; then
  echo "✅ Test 1: Archivo de benchmark con 50,000 eventos"
  ((PASS++))
else
  echo "❌ Test 1: Archivo de benchmark incorrecto"
  ((FAIL++))
fi

# Test 2: Métricas de línea base capturadas
if [ -f ~/elastic-labs/lab09/results/metrics_baseline.csv ]; then
  echo "✅ Test 2: Métricas de línea base registradas"
  ((PASS++))
else
  echo "❌ Test 2: Faltan métricas de línea base"
  ((FAIL++))
fi

# Test 3: Métricas optimizadas capturadas
if [ -f ~/elastic-labs/lab09/results/metrics_optimized.csv ]; then
  echo "✅ Test 3: Métricas optimizadas registradas"
  ((PASS++))
else
  echo "❌ Test 3: Faltan métricas optimizadas"
  ((FAIL++))
fi

# Test 4: Métricas de Filebeat capturadas
if [ -f ~/elastic-labs/lab09/results/metrics_filebeat.csv ]; then
  echo "✅ Test 4: Métricas de Filebeat registradas"
  ((PASS++))
else
  echo "❌ Test 4: Faltan métricas de Filebeat"
  ((FAIL++))
fi

# Test 5: Índices de benchmark existen en Elasticsearch
INDICES=$(curl -sk -u elastic:ElasticLabs2024! \
  "https://localhost:9200/_cat/indices/labs-benchmark-*?h=index" 2>/dev/null | wc -l)
if [ $INDICES -ge 2 ]; then
  echo "✅ Test 5: Al menos 2 índices de benchmark en Elasticsearch"
  ((PASS++))
else
  echo "❌ Test 5: Índices de benchmark insuficientes ($INDICES)"
  ((FAIL++))
fi

# Test 6: Informe comparativo generado
if [ -f ~/elastic-labs/lab09/results/informe_comparativo.md ]; then
  echo "✅ Test 6: Informe comparativo generado"
  ((PASS++))
else
  echo "❌ Test 6: Falta informe comparativo"
  ((FAIL++))
fi

# Test 7: Throughput optimizado > baseline
if [ -f ~/elastic-labs/lab09/results/metrics_baseline.csv ] && \
   [ -f ~/elastic-labs/lab09/results/metrics_optimized.csv ]; then
  BASELINE_T=$(grep throughput ~/elastic-labs/lab09/results/metrics_baseline.csv | cut -d',' -f2)
  OPTIMIZED_T=$(grep throughput ~/elastic-labs/lab09/results/metrics_optimized.csv | cut -d',' -f2)
  if [ "${OPTIMIZED_T:-0}" -gt "${BASELINE_T:-0}" ]; then
    echo "✅ Test 7: Throughput optimizado ($OPTIMIZED_T) > baseline ($BASELINE_T)"
    ((PASS++))
  else
    echo "❌ Test 7: No se observó mejora de throughput"
    ((FAIL++))
  fi
else
  echo "❌ Test 7: No se pueden comparar (faltan archivos)"
  ((FAIL++))
fi

echo ""
echo "═══════════════════════════════════════════════════"
echo " Resultado: $PASS/7 pruebas pasadas, $FAIL fallidas"
echo "═══════════════════════════════════════════════════"
```

## Solución de Problemas

### Problema 1: Logstash no procesa eventos (throughput = 0)

**Síntomas:**
- La API `/_node/stats` muestra `events.in: 0` y `events.out: 0`
- El archivo de log existe y tiene contenido
- No hay errores visibles en los logs de Logstash

**Causa:**
El archivo `sincedb` de una ejecución anterior registra que el archivo ya fue leído completamente. El plugin `file` de Logstash usa el sincedb para rastrear la posición de lectura y no releerá un archivo ya procesado.

**Solución:**

```bash
# Eliminar archivos sincedb
rm -f /tmp/sincedb_benchmark*

# Verificar que el archivo de log existe y tiene contenido
wc -l /tmp/benchmark-app.log

# Reiniciar el contenedor de Logstash
docker restart logstash-benchmark

# Si persiste, verificar permisos del volumen montado
docker exec logstash-benchmark ls -la /tmp/benchmark-app.log

# Verificar logs de Logstash para errores de acceso
docker logs logstash-benchmark --tail 50 | grep -i "error\|permission\|cannot"
```

---

### Problema 2: Filebeat no envía eventos a Elasticsearch (events_failed > 0)

**Síntomas:**
- El endpoint `http://localhost:5066/stats` muestra `output.events.failed > 0`
- El índice `labs-benchmark-filebeat-*` tiene 0 documentos
- Los logs de Filebeat muestran errores de conexión TLS o autenticación

**Causa:**
Filebeat no puede verificar el certificado TLS de Elasticsearch porque la ruta al certificado CA está incorrecta dentro del contenedor, o las credenciales son incorrectas. En entornos Docker, las rutas de volúmenes montados deben coincidir exactamente con la configuración.

**Solución:**

```bash
# Verificar logs de Filebeat
docker logs filebeat-benchmark --tail 30 | grep -i "error\|tls\|auth"

# Verificar que el certificado CA está accesible dentro del contenedor
docker exec filebeat-benchmark ls -la /usr/share/filebeat/config/certs/ca/ca.crt

# Si el archivo no existe, verificar el montaje de volumen
docker inspect filebeat-benchmark | jq '.[0].Mounts'

# Probar conectividad desde dentro del contenedor
docker exec filebeat-benchmark curl -sk \
  -u elastic:ElasticLabs2024! \
  https://es01:9200/_cluster/health

# Si hay error de resolución DNS, verificar la red
docker exec filebeat-benchmark cat /etc/hosts
docker network inspect elastic-net | jq '.[0].Containers'

# Reconstruir con la ruta correcta si es necesario
docker stop filebeat-benchmark
# Ajustar la ruta en filebeat-benchmark.yml y relanzar
```

## Limpieza

```bash
echo "[*] Limpiando recursos del Lab 09..."

# Detener contenedores de benchmark
docker stop logstash-benchmark logstash-benchmark-opt filebeat-benchmark 2>/dev/null

# Eliminar índices de benchmark
curl -sk -u elastic:ElasticLabs2024! -X DELETE \
  "https://localhost:9200/labs-benchmark-*" 2>/dev/null
echo ""

# Limpiar archivos temporales
rm -f /tmp/benchmark-app.log
rm -f /tmp/sincedb_benchmark*
rm -f /tmp/file_completed*.log

# Reiniciar Logstash original
docker start logstash01

# Verificar que el stack vuelve a estado normal
sleep 10
docker compose -f ~/elastic-labs/docker-compose.yml ps --format "table {{.Name}}\t{{.Status}}"

echo ""
echo "[✓] Limpieza completada. Stack restaurado a estado pre-laboratorio."
echo "[ℹ] Los resultados se conservan en ~/elastic-labs/lab09/results/"
```

## Resumen

En este laboratorio completaste un ciclo completo de benchmark y optimización de un pipeline de ingestión:

| Fase | Actividad realizada |
|------|-------------------|
| **Generación de carga** | Script Python con faker generando 50,000 eventos JSON a 1,000 EPS |
| **Línea base** | Medición con 1 worker, batch_size 125, sin compresión |
| **Análisis** | Identificación del output como cuello de botella mediante API de monitoreo |
| **Optimización** | 4 workers, batch_size 500, compresión HTTP, refresh_interval 5s |
| **Comparativa** | Filebeat directo vs Logstash con procesamiento |

**Lecciones clave:**

1. El throughput del pipeline está limitado por su componente más lento (típicamente el output a Elasticsearch en nodo único)
2. Aumentar `pipeline.workers` y `pipeline.batch.size` produce mejoras significativas cuando el cuello de botella es el output
3. La compresión HTTP reduce la carga de red y mejora el throughput, especialmente con logs JSON verbosos
4. Filebeat sin procesamiento intermedio supera a Logstash en throughput puro, pero carece de capacidades de transformación
5. Las métricas deben interpretarse conjuntamente: throughput, latencia y backlog forman un sistema interdependiente

### Recursos adicionales

- [Logstash Performance Tuning](https://www.elastic.co/guide/en/logstash/current/performance-troubleshooting.html)
- [Tune for Indexing Speed - Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-indexing-speed.html)
- [Filebeat Performance Tips](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-performance.html)
- [Logstash Monitoring API Reference](https://www.elastic.co/guide/en/logstash/current/monitoring-logstash.html)
