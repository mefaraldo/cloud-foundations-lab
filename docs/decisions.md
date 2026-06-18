# Decision log

## Formato

Decision:
Contexto:
Alternativas:
Tradeoff:
Resultado:

## Decisiones

### 001 - Laboratorios locales
Decision: usar Docker Compose, MinIO y LocalStack en lugar de cuentas AWS personales.
Contexto: evitar costos accidentales y reducir friccion de setup.
Tradeoff: no se practica consola AWS real en profundidad.
Resultado: los labs son reproducibles y reutilizables.

### 002 - Entorno de desarrollo
Decision: GitHub Codespaces.
Contexto: el grupo no tiene instalaciones homogéneas (mix de macOS, Windows y Linux). Codespaces ofrece el mismo entorno para todos sin configuración local.
Alternativas: Docker Desktop local, WSL2, máquina virtual.
Tradeoff: depende de conectividad y de los free-tier hours disponibles (60 hs/mes por cuenta). Con Docker local se trabaja offline y sin límite de tiempo.
Resultado: Codespaces para las clases, Docker local como fallback documentado en el README.

### 003 - Formato de eventos crudos
Decision: JSONL (JSON Lines) para data/raw/events.jsonl.
Contexto: los eventos se generan uno por vez. JSONL permite procesar con streaming sin cargar todo el archivo en memoria, y es fácil de appender.
Alternativas: JSON array, CSV, Parquet.
Tradeoff: JSONL no es legible de un vistazo como un JSON array formateado. Parquet sería más eficiente a escala, pero requiere dependencias externas.
Resultado: JSONL para raw. CSV para processed (compatibilidad analítica máxima).

### 004 - Pipeline de procesamiento
Decision: script Python (process_events.py) lee JSONL y escribe JSON filtrado.
Contexto: necesitamos filtrar un subconjunto de eventos GitHub Archive para análisis. El script es reproducible: misma entrada, misma salida, sin efectos secundarios.
Tradeoff: un script por transformación vs una sola función general. Elegimos un script por transformación: más legible, más fácil de testear.
Resultado: process_events.py → data/processed/push_events.json (filtra PushEvent)

### 005 - Identidad y credenciales en el lab
Decision: Usar roles con STS en lugar de access keys de larga duracion para acceso entre servicios.
Contexto: Las access keys no expiran y si se filtran dan acceso indefinido. Los roles con STS generan credenciales temporales con trazabilidad.
Alternativas: Access keys rotadas manualmente, vault o secret manager.
Tradeoff: Asumir un rol requiere mas configuracion inicial, pero las credenciales expiran solas y reducen el riesgo significativamente.
Resultado: app-role con politica de privilegio minimo sobre course-data-raw. Credenciales STS confirmadas con Expiration visible.

### 006 - Instance profile en lugar de access keys en la instancia
Decision: Usar instance profile (rol via STS) en lugar de access keys guardadas en la maquina virtual.
Contexto: Una instancia EC2 que necesita leer S3 puede usar una clave fija guardada en disco, o un rol IAM via instance profile que entrega credenciales temporales automaticamente.
Alternativas: Access keys de larga duracion en el archivo de configuracion de la maquina.
Tradeoff: El instance profile requiere mas configuracion inicial, pero las credenciales rotan solas y nunca quedan escritas en ningun archivo.
Resultado: app-instance-profile conectado a app-role asignado a la instancia EC2. Cierre del circuito IAM → EC2 → S3 sin credenciales fijas.

### 007 - course-data-lake como fuente durable del modulo
Decision: Separar course-data-raw (demo IAM) de course-data-lake (fuente de verdad de datos reales del curso).
Contexto: Necesitamos un lugar durable para Olist y GitHub Archive que sobreviva al ciclo de vida de cada lab. Mezclar con el bucket de demo IAM enmascara el proposito de cada uno.
Tradeoff: Dos buckets en lugar de uno. A favor: separacion clara de intencion, escalable a futuras clases.
Resultado: course-data-lake con versioning, BPA, SSE y bucket policy desde el dia 1.