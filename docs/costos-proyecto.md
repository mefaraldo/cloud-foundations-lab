# Estimación de costos — cloud-foundations-lab

**Presupuesto mensual objetivo:** USD 20
**Región:** us-east-1
**Fecha:** 2026-07-07

Este archivo es la sección de costos del entregable del proyecto. **No es opción-múltiple** — cada respuesta requiere que hayas mirado el output, la arquitectura, y hayas decidido.

---

## Preguntas para el arranque (con el `services.json` de ejemplo)

Corré `python3 pricing.py` sobre el ejemplo tal como viene y respondé:

**Q1.** ¿Cuál es el costo mensual total on-demand?
> $61.93

**Q2.** Listá los top 3 servicios por costo, con % del total:
1. nat-gateway  $32.85 (53% del total)
2. rds-db-t3-micro  $12.41 (20% del total)
3. web-tier-ec2 $7.59 (12% del total)

**Q3.** De esos top 3, ¿cuántos son **compute**? ¿Cuántos son **storage** o **network**?
> nat-gateway - network
  rds-db-t3-micro   db
  web-tier-ec2 - compute

**Q4.** Aplicá Savings Plan y Spot: ¿cuánto ahorrás sobre el total?
> $3.73

**Q5.** ¿La optimización SP + Spot alcanza para entrar en tu budget de {{BUDGET}}?
> No — el optimizado sigue siendo $58.20, casi 3 veces el budget.

---

## Desafío 1 — Cambiar la arquitectura, no el descuento

**Contexto:** el NAT Gateway es de lejos el servicio más caro del ejemplo (~$32/mes solo por estar prendido 24/7). No hay Savings Plan ni Spot para NAT. Pero en el lab 07 vimos que **VPC endpoints** pueden reemplazarlo cuando el tráfico privado va sólo a AWS (ej. a S3).

**Q6.** Editá `services.json`: reemplazá la línea del `nat-gateway` por un **VPC endpoint para S3** (unit_price ~$0.01/hora * 730hs = ~$7.3/mes). Corré `pricing.py` de nuevo.

- Costo mensual total nuevo (optimizado): $32.65
- Ahorro vs. el original: $25.55
- ¿Ahora entra en el budget? No

**Q7.** ¿Qué tipo de tráfico **rompería** esta decisión? (pista: los VPC endpoints Gateway solo sirven para S3 y DynamoDB)
si la máquina privada necesita salir a internet para algo que no sea S3 ni DynamoDB — por ejemplo descargar actualizaciones, conectarse a una API externa, instalar paquetes — el VPC endpoint no sirve y necesitaría el NAT de vuelta.



## Desafío 2 — Ajustar a un budget agresivo

**Contexto:** el equipo te dice que el proyecto tiene que caber en **$25/mes**. Con lo que vimos, no hay forma con la arquitectura del ejemplo. Hay que decidir tradeoffs.

**Q8.** Diseñá 2 opciones para entrar en $25:

**Opción A — recortar servicios:**
- Qué sacás:worker-batch
- Qué se pierde en producto:los procesos batch no corren en este entorno
- Costo final estimado:$32.02

**Opción B — cambiar dimensionamiento:**
- Qué instance class bajás:ninguna — mantuvimos db.t3.micro pero redujimos las horas
- Qué storage class cambiás (Standard → IA/Glacier): S3 Standard → S3-IA (de $0.023 a $0.0125/GB)
- Qué uso mensual reducís:RDS de 730hs a 500hs (apagarlo de noche y fines de semana)
- Costo final estimado:$27.69

**Q9.** ¿Cuál elegirías y por qué? (una decisión, no las dos)
 Elegiría la Opción B porque mantiene todos los servicios funcionando. La Opción A sacó el worker-batch y perdiste funcionalidad. La Opción B solo ajusta cuánto corre cada servicio — la base apagada de noche, menos egress — sin perder ninguna capacidad. Para un entorno de desarrollo esto es completamente razonable.

---

## Desafío 3 — Escalar para producción

**Contexto:** el proyecto pasó a producción. Requerimientos: Multi-AZ en la DB, 3x el tráfico, 5x el storage en S3, ELB con health checks.

**Q10.** Escribí un `services.production.json` con las siguientes modificaciones sobre el ejemplo:
- `rds-db-t3-micro`: Multi-AZ (duplicar unit_price a $0.034)
- `s3-data-lake`: 500 GB
- `s3-requests`: 1500 k-req
- `data-egress`: 150 GB
- Agregar un `alb`: 730 hs * $0.0225/hs + 1 LCU/mes * $0.008/LCU-hs

**Q11.** ¿Qué budget mínimo necesitás para prod? $78.57 (on-demand) o $74.83 (optimizado) — con $80/mes estarías cubierto
**Q12.** ¿Cuánto más caro es prod vs dev? Xn/veces
Dev optimizado: $24.19 / Prod optimizado: $74.83 → prod es ~3x más caro que dev

---

## Tu proyecto real

**Q13.** Reemplazá los servicios del ejemplo por los del **stack real de tu proyecto final**. Ajustá `unit_price` con la [calculadora AWS oficial](https://calculator.aws/) y `monthly_usage` con la estimación real del equipo.

Salida final de `python3 pricing.py --budget {{BUDGET}}`:

```
_pegar aquí el output completo_



**Q14.** ¿Cumple el budget? Si sí, ¿con qué margen? Si no, ¿qué decidieron cambiar?
> _respuesta con justificación_

Pendiente — se completa junto con el armado del proyecto final. Los servicios reales de Matafuegos Focus en AWS se van a estimar con la calculadora oficial una vez definida la arquitectura final del entregable.


## Red de seguridad configurada

- [ ] `create-budget.sh` corrido contra AWS real
- [ ] Alerta al 80% ACTUAL confirmada (mail recibido de test)
- [ ] Alerta al 100% FORECASTED activa
- [ ] Mail del grupo (no default)

---

## Fuentes usadas

- [AWS Pricing Calculator](https://calculator.aws/)
- [EC2 pricing](https://aws.amazon.com/ec2/pricing/)
- [S3 pricing](https://aws.amazon.com/s3/pricing/)
- [NAT Gateway pricing](https://aws.amazon.com/vpc/pricing/)
- Otras: _completar_
