# Diseño de dominio inicial — Orders API profesional

## Decisiones confirmadas

Con tus respuestas, estas son las decisiones actuales del dominio:

1. **Estado inicial de la orden:** `CONFIRMED`.
2. **Cálculo de montos:** el backend calcula `totalAmount`.
3. **Idempotencia:** por `Idempotency-Key`.
4. **Dependencias externas:** pagos e inventario.
5. **Sincronía/asincronía:** pagos síncronos; procesamiento de órdenes asíncrono.
6. **Consistencia:** fuerte para pagos; eventual para órdenes.

## Ajustes recomendados (enfoque senior)

Aunque son buenas decisiones, te propongo dos ajustes para robustecer producción:

### 1) Idempotencia: evitar clave global sin contexto

Si usas solo `Idempotency-Key` de forma global, puedes tener colisiones entre clientes.

**Recomendación:** almacenar unicidad por `(tenant_or_customer_id, idempotency_key)` y guardar también `request_hash`.

Comportamiento:
- misma llave + mismo hash → devolver misma respuesta.
- misma llave + hash distinto → `409 Conflict`.

### 2) Estado inicial `CONFIRMED` solo si pago + reserva inventario son exitosos

Si la orden nace `CONFIRMED`, entonces en `POST /orders` debes completar en flujo síncrono:
- autorización/captura de pago,
- validación/reserva de inventario.

Si cualquiera falla, no creas orden confirmada.

## Flujo propuesto de `POST /orders`

1. Validar DTO.
2. Verificar idempotencia.
3. Calcular `totalAmount` en backend.
4. Ejecutar pagos con timeout + retry controlado + circuit breaker.
5. Reservar inventario con timeout + retry controlado + circuit breaker.
6. Persistir orden `CONFIRMED`.
7. Publicar evento `order.confirmed` para procesamiento asíncrono (notificaciones, integraciones, etc.).
8. Guardar snapshot de respuesta para idempotencia.

## Preguntas de diseño siguientes

Para cerrar el diseño antes del código NestJS, faltan estas decisiones:

1. ¿`CONFIRMED` implica **pago autorizado** o **pago capturado**?
2. ¿Qué hacemos si pago OK pero inventario falla? (compensación/refund inmediato).
3. ¿Cuánto tiempo permitimos para timeout de pagos e inventario (p95 objetivo)?
4. ¿Cuál será política de reintentos por dependencia (máximo intentos y backoff)?
5. ¿Qué campos mínimos tendrá el evento `order.confirmed`?

## Estructura de proyecto sugerida (siguiente paso)

- `src/orders/domain` (entidades, value objects, invariantes)
- `src/orders/application` (casos de uso)
- `src/orders/infrastructure` (repositorios, clientes externos, cola)
- `src/orders/interfaces/http` (controllers y DTOs)
- `src/common` (errores globales, logging, filtros)

---

Cuando confirmes las preguntas pendientes, el siguiente entregable será el **skeleton real de NestJS** con:
- módulo `orders`,
- DTOs con validación,
- controller + service + repository,
- excepción global,
- logging estructurado base,
- contrato de idempotencia.
