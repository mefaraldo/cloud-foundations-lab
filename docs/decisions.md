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
Contexto: Una instancia EC2 puede usar una clave fija guardada en disco, o un rol IAM via instance profile que entrega credenciales temporales automaticamente.
Alternativas: Access keys de larga duracion en el archivo de configuracion de la maquina.
Tradeoff: El instance profile requiere mas configuracion inicial, pero las credenciales rotan solas y nunca quedan escritas en ningun archivo.
Resultado: app-instance-profile conectado a app-role asignado a la instancia EC2. Cierre del circuito IAM → EC2 → S3 sin credenciales fijas.

### 007 - course-data-lake como fuente durable del modulo
Decision: Separar course-data-raw (demo IAM) de course-data-lake (fuente de verdad de datos reales del curso).
Contexto: Necesitamos un lugar durable para Olist y GitHub Archive que sobreviva al ciclo de vida de cada lab.
Tradeoff: Dos buckets en lugar de uno. A favor: separacion clara de intencion, escalable a futuras clases.
Resultado: course-data-lake con versioning, BPA, SSE y bucket policy desde el dia 1.

### 008 - VPC endpoint en lugar de NAT para trafico a S3
Decision: usar VPC endpoint Gateway para que la subred privada llegue a S3, en lugar de un NAT gateway.
Contexto: una EC2 privada puede ir a S3 por NAT gateway (mas caro, expone egress) o por VPC endpoint (gratis, trafico privado).
Tradeoff: VPC endpoint solo cubre S3 y DynamoDB. Para otros servicios habria que sumar PrivateLink.
Resultado: VPC endpoint para S3 en la route table privada, sin NAT.

### 009 - Postgres en docker para dev, RDS en prod
Decision: usar docker postgres para desarrollo local y RDS managed para produccion. No usar postgres-on-EC2.
Contexto: postgres-on-EC2 da toda la carga operativa de self-managed sin las garantias de RDS ni la simplicidad de docker para dev.
Tradeoff: docker no es produccion (sin HA, sin backups automaticos). RDS cuesta plata.
Resultado: dev=docker, prod=RDS.

### 010 - Credencial en Secrets Manager, nunca en el codigo
Decision: la password se guarda en Secrets Manager. La app la lee en runtime con su rol.
Contexto: credenciales en codigo son el vector de incidente mas comun.
Tradeoff: una dependencia mas. A favor: rotacion automatica soportada, acceso auditado, control via IAM.
Resultado: app/db en Secrets Manager. Mismo codigo de conexion para las 3 opciones.

### 011 - IaC declarativa con OpenTofu en lugar de scripts de AWS CLI
Decision: usar OpenTofu (HCL declarativo) para la infra en lugar de scripts imperativos con aws CLI o boto3.
Contexto: scripts imperativos requieren manejar idempotencia a mano y no muestran el diff antes de aplicar.
Alternativas: Terraform (mismo HCL, licencia BSL desde 2023), CloudFormation (AWS-only), AWS CDK, Pulumi.
Tradeoff: hay que aprender HCL y el modelo de state. A favor: diff antes de aplicar, destroy/apply idempotente, portabilidad entre clouds.
Resultado: iac/ en HCL, ejecutable con tofu o terraform indistinto. Backend local en este lab; remoto en el proyecto final.

### 012 - Estimar costos antes de tocar infra, y monitorearlos con Budget
Decision: estimar el costo mensual con pricing.py + services.json antes de tocar la infra, y configurar un AWS Budget con alertas al 80% ACTUAL y 100% FORECASTED por mail.
Contexto: sin estimacion, el costo es una sorpresa a fin de mes. Sin budget con alerta, un recurso olvidado puede correr semanas generando factura sin que nadie lo note.
Alternativas: Cost Explorer manual, tags + reportes semanales, herramientas third-party (Infracost, Cloudability).
Tradeoff: los descuentos aplicados son referenciales. Los reales dependen de region, familia, periodo y commitment.
Resultado: estimacion ejecutada en Q1-Q14 del workbook. Budget de $25/mes configurado con alertas al mail del grupo.