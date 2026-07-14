# Revisión Well-Architected — {{ Matafuegos Focus}}
2026-07-14

Revisión corta contra los 6 pilares del [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/), aplicada al stack del proyecto.

**Método:** para cada pilar, 1 pregunta guía + respuesta corta con evidencia (link a decisions.md, código, o descripción). Los pilares están ordenados por prioridad para este proyecto — no todos pesan igual, y eso es una decisión explícita.

---

## Pilar 1 — Reliability (confiabilidad)

**Pregunta:** Si mañana se cae una AZ, ¿cuánto tarda tu proyecto en volver, y cuántos datos perdés?

**RTO objetivo:** _minutos / horas / días_
**RPO objetivo:** _segundos / minutos / horas_

**Hallazgo:**
> _dónde estás hoy respecto a esos objetivos. ¿Multi-AZ? ¿Single-AZ + backup restaurable? ¿Solo backup manual?_

**Decisión / próximo paso:**
> _qué vas a hacer al respecto en el proyecto. Puede ser "aceptar el gap por costo" — pero explícito._

---
Pilar 1 — Reliability

RTO objetivo: 1 hora
RPO objetivo: 24 horas
Hallazgo: Single-AZ en dev (MultiAZ: false en rds/rds_config.json). Backup retention de 7 días configurado — recuperable pero no automático.
Decisión: Aceptar el gap por costo en dev. En prod activar Multi-AZ (decisión 009 en decisions.md).

## Pilar 2 — Operational Excellence (excelencia operativa)

**Pregunta:** Si en este momento sube el CPU de todos tus servidores al 95%, ¿cómo te enterás?

**Hallazgo:**
> _¿Tenés observabilidad? ¿La alarma del lab está activa? ¿Alguien recibe el mail del topic SNS?_

**Decisión / próximo paso:**
> _al menos 1 alarma accionable configurada. Idealmente 2-3._

---
Pilar 2 — Operational Excellence

Hallazgo: Alarma rds-conexiones-altas-matafuegos activa en CloudWatch (ver monitoring/mi-alarma.json). Topic SNS project-alerts creado. Sin subscribers reales todavía.
Decisión: Agregar mail real al topic SNS antes de ir a producción.



## Pilar 3 — Security (seguridad)

**Pregunta:** Si un atacante obtiene las credenciales del EC2 web-tier, ¿qué puede hacer?

**Hallazgo:**
> _¿Rol de instancia con privilegio mínimo? ¿Access keys en código? ¿SG de la app privada acepta solo del SG público (lab 07)?_

**Decisión / próximo paso:**
> _mirando la evidencia del lab 04-07: qué queda por cerrar._

---
Pilar 3 — Security

Hallazgo: Rol de instancia con privilegio mínimo s3:GetObject sobre raw/* (ver iam/s3_read_policy.json). Credenciales en Secrets Manager, no en código (decisión 010). SG privado acepta solo desde SG público (decisión 008). Sin access keys hardcodeadas.
Decisión: Cubierto para dev. Agregar MFA para el usuario admin en prod.

## Pilar 4 — Cost Optimization

**Pregunta:** ¿El costo estimado del proyecto entra en el budget del equipo, y hay una alerta si lo excede?

**Costo estimado:** USD _/mes (de `finops/pricing.py`)
**Budget:** USD _/mes
**Alerta configurada:** sí / no (link a `finops/create-budget.sh` corrido)

**Hallazgo:**
> _link a `docs/costos-proyecto.md` (workbook del lab 10) si ya está, sino "pendiente"._

**Decisión / próximo paso:**
> _cambios de arquitectura que ya se aplicaron para bajar el costo._

---
Pilar 4 — Cost Optimization

Costo estimado: $24.19/mes (optimizado)
Budget: $25/mes
Alerta configurada: sí (ver finops/budget.json y finops/notify.json)
Hallazgo: Ver docs/costos-proyecto.md. NAT reemplazado por VPC endpoint ahorró $25.55/mes (decisión 008).
Decisión: Cubierto. RDS apagado de noche en dev para reducir horas.

## Pilar 5 — Performance Efficiency

**Pregunta:** Si el tráfico se duplica de un día para el otro, ¿tu stack lo absorbe solo o hay que intervenir?

**Hallazgo:**
> _¿ASG configurado con target tracking? ¿Instance type sobre-dimensionado (leftover del "por las dudas")? ¿Cuellos de botella en la DB?_

**Decisión / próximo paso:**
> _al menos `monitoring/scaling-policy.json` con target tracking listo, aunque no se aplique todavía._

---
Pilar 5 — Performance Efficiency

Hallazgo: Sin ASG todavía — instancia única t3.micro. Política de scaling declarada en monitoring/scaling-policy.json con target tracking al 50% CPU.
Decisión: Implementar ASG multi-AZ en prod. En dev instancia única es suficiente.

## Pilar 6 — Sustainability

**Pregunta:** Si el proyecto queda encendido 24/7 pero solo se usa 8h/día laborales, ¿estás pagando por ciclos que no aportan?

**Hallazgo:**
> _¿Los entornos de dev se apagan de noche? ¿Los datos están en S3 Standard cuando podrían estar en IA? ¿Hay snapshots viejos que nunca se borraron?_

**Decisión / próximo paso:**
> _al menos 1 optimización identificada aunque no se implemente todavía._

---
Pilar 6 — Sustainability

Hallazgo: RDS configurado a 400hs/mes en dev (apagado de noche). S3 en clase IA para datos poco accedidos (decisión en docs/costos-proyecto.md).
Decisión: Revisar snapshots viejos antes de ir a prod. Lifecycle rules en S3 para mover datos viejos a Glacier.

## Resumen de decisiones

| Pilar | Estado | Decisión pendiente |
|---|---|---|
| Reliability | _en progreso / cubierto / gap explícito_ | _resumen 1 línea_ |
| Operational Excellence | | |
| Security | | |
| Cost Optimization | | |
| Performance Efficiency | | |
| Sustainability | | |

**Los 3 pilares más críticos para este proyecto (con justificación):**
1. _pilar_ — porque _razón_
2. _pilar_ — porque _razón_
3. _pilar_ — porque _razón_

Los otros 3 se documentan pero no se priorizan en la iteración actual.

---
Los 3 pilares más críticos:

Security — porque manejamos datos de clientes y comprobantes de pago de Focus Matafuegos
Reliability — porque si la base se cae, el negocio no puede operar
Cost Optimization — porque es un proyecto de una PyME con presupuesto limitado

## Cómo se hizo esta revisión

- Fecha: {{FECHA}}
- Alcance: stack del proyecto al momento de la revisión
- Método: 1 pregunta por pilar, hallazgo con evidencia
- Repositorio de decisiones: `docs/decisions.md`

**Fuentes:**
- [Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Reliability pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
- [Operational Excellence pillar](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/welcome.html)
