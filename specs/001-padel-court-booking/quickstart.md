# Quickstart: Reserva de Canchas de Pádel

**Feature**: `001-padel-court-booking` | **Fecha**: 2026-11-05

Guía para levantar la aplicación y validar de punta a punta que la feature cumple la spec.
Referencias: [contrato de la API](contracts/api.md) y [modelo de datos](data-model.md).

## Prerrequisitos

- Node.js 22 LTS y npm.
- Puertos libres: `3000` (backend) y `5173` (frontend).

## Puesta en marcha

```bash
# Backend: instala, crea db/padel.db desde db/schema.sql con las 5 canchas y levanta la API
cd backend && npm install && npm run dev        # http://localhost:3000/api

# Frontend (otra terminal): Vite con proxy /api -> :3000
cd frontend && npm install && npm run dev       # http://localhost:5173
```

## Pruebas automatizadas

```bash
cd backend && npm test      # API con supertest sobre SQLite en memoria y reloj simulado
cd frontend && npm test     # componentes con React Testing Library
```

Deben pasar todas. Las pruebas de backend cubren como mínimo los escenarios de abajo marcados
con ⚙️.

## Escenarios de validación

### 1. Autenticación (US1)

| # | Paso | Resultado esperado |
|---|---|---|
| 1.1 ⚙️ | Registrarse en `/registro` con un correo nuevo y contraseña de 8+ caracteres | Cuenta creada y sesión activa; redirige a `/canchas` |
| 1.2 ⚙️ | Registrarse otra vez con el mismo correo | Mensaje "Ese correo ya está en uso." (409) |
| 1.3 ⚙️ | Registrarse con contraseña de 7 caracteres | Mensaje de contraseña débil (400) |
| 1.4 ⚙️ | Cerrar sesión e iniciar con contraseña incorrecta | "Correo o contraseña incorrectos." (401), sin decir cuál falló |
| 1.5 ⚙️ | Sin sesión, `GET /api/courts/1/availability?date=…` o `POST /api/reservations` | 401 `UNAUTHENTICATED`; en la UI redirige a `/login` |
| 1.6 ⚙️ | Sesión con `last_activity_at` de hace más de 24 h (reloj simulado) | 401 y la sesión se borra (FR-006a) |

### 2. Disponibilidad y reserva (US2)

| # | Paso | Resultado esperado |
|---|---|---|
| 2.1 ⚙️ | `GET /api/courts` | Exactamente las 5 canchas del catálogo |
| 2.2 ⚙️ | Elegir cancha y fecha futura | 24 bloques (00:00–01:00 … 23:00–24:00), cada uno "Disponible" o "Reservado", sin datos de quién reservó |
| 2.3 ⚙️ | Reservar un bloque disponible | 201; el bloque pasa a "Reservado" para otro usuario |
| 2.4 ⚙️ | Dos usuarios confirman el mismo bloque en paralelo (`Promise.all`) | Exactamente 1 → 201 y el otro → 409 "La cancha ya fue reservada en este horario"; la grilla se refresca (SC-003) |
| 2.5 ⚙️ | Reservar un bloque de hoy cuya hora de inicio ya pasó, o de una fecha pasada | 400 `BLOCK_IN_PAST`; en la UI esos bloques no son seleccionables |
| 2.6 ⚙️ | Con 1 reserva activa, reservar el bloque siguiente | 201; queda con 2 activas consecutivas |
| 2.7 ⚙️ | Con 2 reservas activas, intentar una tercera (también 2 en paralelo con 1 activa) | 409 `ACTIVE_LIMIT_REACHED`; nunca más de 2 activas |
| 2.8 ⚙️ | `courtId: 9`, `startHour: 14.5`, `startHour: 24` o `date: "2026-13-01"` | 400 `INVALID_REQUEST` |
| 2.9 | Abrir la grilla, cancelar desde otra pestaña y no recargar | La grilla no cambia hasta cambiar cancha/fecha o hacer una acción (FR-010a) |

### 3. Mis Reservas (US3)

| # | Paso | Resultado esperado |
|---|---|---|
| 3.1 ⚙️ | Usuario con una reserva futura y una ya terminada abre `/mis-reservas` | Dos listas separadas (futuras e historial) con cancha, fecha y hora |
| 3.2 | Pulsar "Cancelar" en una futura y luego "Volver" | Se abre confirmación; al desistir la reserva sigue igual |
| 3.3 ⚙️ | Confirmar la cancelación | 204; desaparece de futuras, el bloque vuelve a "Disponible" y se libera un cupo |
| 3.4 ⚙️ | Reserva en curso o del historial | Sin botón "Cancelar"; el `DELETE` responde 400 `RESERVATION_STARTED` |
| 3.5 ⚙️ | Usuario B intenta `DELETE` la reserva de A | 404 `RESERVATION_NOT_FOUND`; `mine` de B no muestra reservas de A |
| 3.6 ⚙️ | Reserva cancelada | No aparece ni en futuras ni en historial |

### 4. Errores (FR-022, SC-006)

| # | Paso | Resultado esperado |
|---|---|---|
| 4.1 ⚙️ | Forzar un error interno (por ejemplo, base no disponible en una prueba) | 500 con `INTERNAL_ERROR` y mensaje genérico; sin stack trace ni texto de SQLite |
| 4.2 | Apagar el backend y reservar desde la UI | Mensaje amigable de error de conexión, nunca texto técnico |
