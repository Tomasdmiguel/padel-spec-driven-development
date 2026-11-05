# API Contract: Reserva de Canchas de Pádel

**Feature**: `001-padel-court-booking` | **Fecha**: 2026-11-05

- Base: `/api`. Cuerpos en JSON.
- Autenticación: cookie `sid` (`HttpOnly`, `SameSite=Lax`) que entrega el login o el registro.
- Fechas `YYYY-MM-DD` y horas enteras `0–23`, siempre en hora local del club (`America/Bogota`).

## Formato de error (todas las rutas)

```json
{ "error": { "code": "SLOT_TAKEN", "message": "La cancha ya fue reservada en este horario" } }
```

| HTTP | `code` | `message` (se muestra tal cual en la UI) |
|---|---|---|
| 400 | `INVALID_REQUEST` | "Los datos enviados no son válidos." |
| 400 | `WEAK_PASSWORD` | "La contraseña debe tener al menos 8 caracteres." |
| 400 | `BLOCK_IN_PAST` | "Ese horario ya pasó. Elige un bloque futuro." |
| 400 | `RESERVATION_STARTED` | "La reserva ya comenzó y no se puede cancelar." |
| 401 | `UNAUTHENTICATED` | "Tu sesión no está activa. Inicia sesión para continuar." |
| 401 | `INVALID_CREDENTIALS` | "Correo o contraseña incorrectos." |
| 404 | `RESERVATION_NOT_FOUND` | "No encontramos esa reserva." |
| 409 | `EMAIL_TAKEN` | "Ese correo ya está en uso." |
| 409 | `SLOT_TAKEN` | "La cancha ya fue reservada en este horario" |
| 409 | `ACTIVE_LIMIT_REACHED` | "Ya tienes 2 reservas activas. Cancela una o espera a que termine para reservar otra." |
| 500 | `INTERNAL_ERROR` | "Ocurrió un problema. Intenta de nuevo en unos minutos." |

Nunca se incluyen stack traces, mensajes de SQLite ni detalles internos (Principio V).

---

## Autenticación

### `POST /api/auth/register`

Body: `{ "email": string, "password": string }`

- **201** `{ "user": { "id": 1, "email": "ana@correo.com" } }` + cookie `sid` (queda con sesión
  activa).
- **400** `INVALID_REQUEST` (correo mal formado), `WEAK_PASSWORD`.
- **409** `EMAIL_TAKEN`.

### `POST /api/auth/login`

Body: `{ "email": string, "password": string }`

- **200** `{ "user": { "id": 1, "email": "ana@correo.com" } }` + cookie `sid`.
- **400** `INVALID_REQUEST`.
- **401** `INVALID_CREDENTIALS` (mismo mensaje si el correo no existe o la contraseña es
  incorrecta).

### `POST /api/auth/logout` *(requiere sesión)*

- **204** sin cuerpo. Borra la sesión y la cookie.
- **401** `UNAUTHENTICATED`.

### `GET /api/auth/me` *(requiere sesión)*

- **200** `{ "user": { "id": 1, "email": "ana@correo.com" } }`.
- **401** `UNAUTHENTICATED`.

---

## Canchas y disponibilidad

### `GET /api/courts` *(pública)*

- **200** `{ "courts": [ { "id": 1, "name": "Cancha Laureles" }, … ] }` — siempre las 5 canchas.

### `GET /api/courts/:courtId/availability?date=YYYY-MM-DD` *(requiere sesión)*

- **200**

```json
{
  "courtId": 1,
  "date": "2026-11-06",
  "blocks": [
    { "startHour": 0, "status": "available", "bookable": true },
    { "startHour": 14, "status": "reserved", "bookable": false }
  ]
}
```

  Siempre 24 elementos (`startHour` 0–23). `status`: `available` | `reserved`. `bookable` es
  `true` solo si está disponible y el bloque no comenzó. No incluye quién reservó (FR-010).
- **400** `INVALID_REQUEST` (cancha fuera de catálogo, fecha ausente o inválida).
- **401** `UNAUTHENTICATED`.

---

## Reservas

### `POST /api/reservations` *(requiere sesión)*

Body: `{ "courtId": number, "date": "YYYY-MM-DD", "startHour": number }`

- **201** `{ "reservation": Reservation }`
- **400** `INVALID_REQUEST`, `BLOCK_IN_PAST`.
- **401** `UNAUTHENTICATED`.
- **409** `ACTIVE_LIMIT_REACHED`, `SLOT_TAKEN`.

Orden de validación: ver [data-model.md](../data-model.md#reglas-de-validación-al-crear-una-reserva-en-este-orden).

### `GET /api/reservations/mine` *(requiere sesión)*

- **200**

```json
{
  "upcoming": [ Reservation ],
  "history":  [ Reservation ]
}
```

  `upcoming`: reservas activas futuras o en curso, orden ascendente por fecha y hora.
  `history`: reservas activas ya terminadas, orden descendente. Las canceladas no aparecen.
- **401** `UNAUTHENTICATED`.

### `DELETE /api/reservations/:id` *(requiere sesión)*

Cancela una reserva propia que aún no comenzó. La confirmación se pide en la UI antes de llamar.

- **204** sin cuerpo. El bloque queda disponible de inmediato (FR-021).
- **400** `RESERVATION_STARTED`.
- **401** `UNAUTHENTICATED`.
- **404** `RESERVATION_NOT_FOUND` (no existe, es de otro usuario o ya está cancelada).

---

## Tipo `Reservation`

```json
{
  "id": 42,
  "courtId": 1,
  "courtName": "Cancha Laureles",
  "date": "2026-11-06",
  "startHour": 14,
  "cancellable": true
}
```

`cancellable` es `true` solo si el bloque aún no comenzó.
