# Data Model: Reserva de Canchas de Pádel

**Feature**: `001-padel-court-booking` | **Fecha**: 2026-11-05

Base de datos: SQLite local en `db/padel.db`. Esquema en `db/schema.sql`. Todas las fechas y horas
de reserva están en hora local del club (`America/Bogota`). Los timestamps técnicos
(`created_at`, `last_activity_at`) se guardan en ISO 8601 UTC.

## Entidades

### Usuario (`users`)

| Campo | Tipo | Reglas |
|---|---|---|
| `id` | INTEGER PK | autoincremental |
| `email` | TEXT NOT NULL UNIQUE | se guarda sin espacios y en minúsculas; formato de correo válido (FR-001, FR-002) |
| `password_hash` | TEXT NOT NULL | `scrypt$<sal>$<hash>`; nunca la contraseña en claro (FR-006) |
| `created_at` | TEXT NOT NULL | fecha de registro |

**Validación de entrada**: contraseña de al menos 8 caracteres (Assumptions).

### Sesión (`sessions`)

Soporte técnico de FR-003, FR-004 y FR-006a.

| Campo | Tipo | Reglas |
|---|---|---|
| `token_hash` | TEXT PK | SHA-256 del token enviado en la cookie `sid` |
| `user_id` | INTEGER NOT NULL FK → `users.id` | |
| `created_at` | TEXT NOT NULL | |
| `last_activity_at` | TEXT NOT NULL | se actualiza en cada petición autenticada |

**Regla**: si `ahora - last_activity_at > 24 h` la sesión es inválida (401) y se borra. Cerrar
sesión borra la fila.

### Cancha (`courts`)

| Campo | Tipo | Reglas |
|---|---|---|
| `id` | INTEGER PK | fijo, 1–5 |
| `name` | TEXT NOT NULL UNIQUE | |

**Semilla fija (inmutable, Principio I / FR-007)**:

| id | name |
|---|---|
| 1 | Cancha Laureles |
| 2 | Cancha El Poblado |
| 3 | Cancha Belén |
| 4 | Cancha Robledo |
| 5 | Cancha Envigado |

### Reserva (`reservations`)

| Campo | Tipo | Reglas |
|---|---|---|
| `id` | INTEGER PK | autoincremental |
| `user_id` | INTEGER NOT NULL FK → `users.id` | dueño de la reserva |
| `court_id` | INTEGER NOT NULL FK → `courts.id` | debe existir en el catálogo (FR-017) |
| `date` | TEXT NOT NULL | `YYYY-MM-DD`, fecha real del calendario |
| `start_hour` | INTEGER NOT NULL | `CHECK (start_hour BETWEEN 0 AND 23)`; bloque `start_hour:00–(start_hour+1):00` (FR-012) |
| `status` | TEXT NOT NULL | `CHECK (status IN ('active','cancelled'))` |
| `created_at` | TEXT NOT NULL | |
| `cancelled_at` | TEXT NULL | solo si `status = 'cancelled'` |

**Restricciones e índices**:

- `CREATE UNIQUE INDEX ux_reservation_slot ON reservations(court_id, date, start_hour) WHERE
  status = 'active';` Nunca dos reservas vigentes en la misma cancha y bloque (Principio III,
  FR-014).
- `CREATE INDEX ix_reservation_user ON reservations(user_id, status);` para "Mis Reservas" y el
  límite de reservas activas.

## Relaciones

- Usuario 1 — N Reserva
- Cancha 1 — N Reserva
- Usuario 1 — N Sesión

## Estados de una reserva

`status` guarda solo lo que el usuario decidió. Lo temporal se calcula contra la hora actual de
Bogotá (ver research R3).

```text
              confirmar (bloque libre, no comenzó, < 2 activas)
   (nada) ────────────────────────────────────────────────► FUTURA
                                                              │  │
               cancelar (antes del inicio, con confirmación)  │  │ llega la hora de inicio
   CANCELADA ◄────────────────────────────────────────────────┘  ▼
   (status='cancelled',                                       EN CURSO
    no aparece en ninguna lista,                                 │ llega la hora de fin
    libera el bloque)                                            ▼
                                                              HISTORIAL
```

| Estado derivado | Condición | ¿Cuenta para el límite de 2? | ¿Ocupa el bloque? | ¿Cancelable? | Lista en "Mis Reservas" |
|---|---|---|---|---|---|
| Futura | `active` y el bloque no comenzó | Sí | Sí | Sí | Futuras |
| En curso | `active` y el bloque comenzó pero no terminó | Sí | Sí | No | Futuras |
| Historial | `active` y el bloque terminó | No | Sí (bloque pasado) | No | Historial |
| Cancelada | `cancelled` | No | No | No | Ninguna |

## Reglas de validación al crear una reserva (en este orden)

1. Sesión activa, si no **401** `UNAUTHENTICATED` (Principio IV).
2. `courtId` entero existente en el catálogo, `date` con formato `YYYY-MM-DD` y fecha válida,
   `startHour` entero 0–23. Si no **400** `INVALID_REQUEST` (FR-017).
3. El bloque no comenzó. Si comenzó **400** `BLOCK_IN_PAST` (FR-015).
4. Dentro de una transacción `IMMEDIATE`:
   - El usuario tiene menos de 2 reservas activas. Si no **409** `ACTIVE_LIMIT_REACHED` (FR-016).
   - No hay reserva `active` en el mismo bloque. Si la hay **409** `SLOT_TAKEN` (FR-013).
   - Insertar. Si el índice único salta, **409** `SLOT_TAKEN` (FR-014).
