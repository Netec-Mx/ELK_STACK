# Elastic Stack para gestión centralizada de logs

Curso práctico orientado a la recolección, procesamiento, normalización, visualización, alertamiento y diagnóstico de logs de aplicaciones en servicios cloud, utilizando Elasticsearch, Kibana, Logstash, Beats, Elastic Agent y Fleet en entornos 7.17 y 9.3.

## Estructura

- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [Identificar los componentes del entorno y validar el flujo de datos de una fuente de logs.](Capitulo01/README.md#identificar-los-componentes-del-entorno-y-validar-el-flujo-de-datos-de-una-fuente-de-logs)
  - Descripción: Identificar los componentes de Elastic Stack presentes en el entorno y validar el flujo de datos de una fuente de logs, considerando Elasticsearch, Kibana, Logstash, Beats, Elastic Agent y Fleet.
  - Duración estimada: 34 min

### Capítulo 2

- [Analizar muestras de logs y diseñar el flujo de ingestión, normalización, almacenamiento y retención.](Capitulo02/README.md#analizar-muestras-de-logs-y-diseñar-el-flujo-de-ingestión-normalización-almacenamiento-y-retención)
  - Descripción: Analizar muestras de logs considerando sus fuentes, formatos, volumen y requisitos, y diseñar el flujo de ingestión, normalización, almacenamiento y retención mediante agente directo, Logstash o ingest node.
  - Duración estimada: 50 min

### Capítulo 3

- [Incorporar una fuente de logs mediante Elastic Agent o Filebeat y validar su recepción.](Capitulo03/README.md#incorporar-una-fuente-de-logs-mediante-elastic-agent-o-filebeat-y-validar-su-recepción)
  - Descripción: Incorporar una fuente de logs mediante Elastic Agent o Filebeat, administrarla con Fleet cuando corresponda y validar su recepción, considerando la compatibilidad y las diferencias entre las versiones 7.17 y 9.3.
  - Duración estimada: 54 min

### Capítulo 4

- [Construir y depurar un pipeline de Logstash para logs de aplicaciones.](Capitulo04/README.md#construir-y-depurar-un-pipeline-de-logstash-para-logs-de-aplicaciones)
  - Descripción: Construir y depurar un pipeline de Logstash para recibir, transformar, normalizar y enviar logs de aplicaciones, aplicando inputs, filters, outputs, codecs, ECS y mecanismos de manejo de errores.
  - Duración estimada: 54 min

### Capítulo 5

- [Crear un dashboard de errores y configurar una regla de monitoreo para logs de aplicaciones.](Capitulo05/README.md#crear-un-dashboard-de-errores-y-configurar-una-regla-de-monitoreo-para-logs-de-aplicaciones)
  - Descripción: Crear en Kibana un dashboard de errores para logs de aplicaciones y configurar una regla de monitoreo, utilizando Discover, data views, filtros, KQL, ES|QL, Lens y las capacidades disponibles según licenciamiento.
  - Duración estimada: 54 min

### Capítulo 6

- [Corregir un flujo de logs con errores de conectividad, permisos y parsing.](Capitulo06/README.md#corregir-un-flujo-de-logs-con-errores-de-conectividad-permisos-y-parsing)
  - Descripción: Diagnosticar y corregir un flujo de logs con errores de conectividad, permisos y parsing, validando además timestamps, multiline, mappings, ECS y eventos rechazados.
  - Duración estimada: 54 min

### Capítulo 7

- [Configurar acceso diferenciado para equipos de aplicaciones y operaciones.](Capitulo07/README.md#configurar-acceso-diferenciado-para-equipos-de-aplicaciones-y-operaciones)
  - Descripción: Configurar acceso diferenciado para equipos de aplicaciones y operaciones mediante autenticación, TLS, API keys, usuarios, roles, privilegios y Kibana Spaces para proteger los logs y segregar ambientes.
  - Duración estimada: 54 min

### Capítulo 8

- [Aplicar una política de retención y validar la recuperación de datos de logs.](Capitulo08/README.md#aplicar-una-política-de-retención-y-validar-la-recuperación-de-datos-de-logs)
  - Descripción: Aplicar una política de retención mediante ILM, data streams y data tiers, y validar la recuperación de datos de logs utilizando mecanismos de continuidad como snapshots, restauración, colas y reintentos.
  - Duración estimada: 54 min

### Capítulo 9

- [Medir y optimizar un pipeline de ingestión de logs.](Capitulo09/README.md#medir-y-optimizar-un-pipeline-de-ingestión-de-logs)
  - Descripción: Medir throughput, latencia y backlog de un pipeline de ingestión de logs y optimizar Elastic Agent, Beats o Logstash mediante batch size, workers, queues y diseño de pipelines.
  - Duración estimada: 54 min

### Capítulo 10

- [Configurar, validar y presentar una solución de gestión centralizada de logs.](Capitulo10/README.md#configurar-validar-y-presentar-una-solución-de-gestión-centralizada-de-logs)
  - Descripción: Configurar, validar y presentar una solución integral que recolecte, procese, almacene, visualice, proteja y permita diagnosticar logs de aplicaciones, verificando criterios de aceptación, calidad, rendimiento y pruebas operativas.
  - Duración estimada: 51 min

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.
