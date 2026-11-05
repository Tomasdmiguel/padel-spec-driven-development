# Research: Reserva de Canchas de Pádel

**Feature**: `001-padel-court-booking` | **Fecha**: 2026-11-05

El stack lo fija la constitución (React + TypeScript + Tailwind, Node.js + Express + TypeScript,
SQLite en `padel.db` con better-sqlite3). Esta investigación resuelve los puntos abiertos dentro de
ese marco. No quedan elementos `NEEDS CLARIFICATION`.

## R1. Prevención de colisiones (Principio III, FR-013, FR-014)

- **Decision**: Doble barrera.
  1. Índice único parcial en SQLite:
     `UNIQUE (court_id, date, start_hour) WHERE status = 'active'`.
  2. La creación de la reserva corre dentro de una transacción `IMMEDIATE` de better-sqlite3
     que primero verifica que el bloque esté libre y el límite de reservas activas, y luego
     inserta. Si el índice único igual salta (`SQLITE_CONSTRAINT_UNIQUE`), se traduce a 409
     `SLOT_TAKEN`.
- **Rationale**: better-sqlite3 es síncrono y la transacción `IMMEDIATE` toma el lock de escritura
  antes de leer, así que dos confirmaciones simultáneas se serializan. El índice único es la
  garantía final a nivel de datos: aunque haya un error en el código, la base no acepta dos
  reservas activas en el mismo bloque. Al ser parcial, una reserva cancelada no bloquea el bloque.
- **Alternatives considered**: Solo validar en código (no cumple el "NO NEGOCIABLE" ante errores
  de lógica); índice único sin filtro (impediría volver a reservar un bloque cancelado); locks en
  memoria (no protege a nivel de datos).

## R2. Límite de 2 reservas activas (FR-016)

- **Decision**: Contar dentro de la misma transacción `IMMEDIATE` de R1 las reservas del usuario
  con `status = 'active'` cuyo bloque no terminó. Si son 2 o más, se rechaza con 409
  `ACTIVE_LIMIT_REACHED`.
- **Rationale**: Al estar en la misma transacción serializada, dos confirmaciones simultáneas
  desde dos pestañas no pueden superar el límite (edge case de la spec). No se puede expresar con
  un índice porque depende de la hora actual.
- **Alternatives considered**: Trigger SQL (depende de "ahora", más difícil de probar); columna
  contador en usuario (se desincroniza al terminar los bloques).

## R3. Manejo de horario y "pasado" (FR-015, Assumptions)

- **Decision**: La reserva se guarda como `date` (`YYYY-MM-DD`) + `start_hour` (entero 0–23), en
  hora local del club `America/Bogota`. El backend calcula "ahora" en esa zona con
  `Intl.DateTimeFormat` (`timeZone: 'America/Bogota'`) en un único módulo `clock.ts`, que se puede
  reemplazar en las pruebas.
  - Bloque **reservable / cancelable** (no comenzó): `date > hoy` o (`date = hoy` y
    `start_hour > horaActual`).
  - Reserva **activa** (no terminó): `status = 'active'` y (`date > hoy` o (`date = hoy` y
    `start_hour >= horaActual`)).
  - **Historial**: `status = 'active'` y el bloque ya terminó. Las canceladas no se muestran.
- **Rationale**: Bogotá no tiene horario de verano, así que fecha + hora entera es inequívoca y
  hace natural la validación de "bloque entero". Centralizar el reloj permite probar los casos
  límite de tiempo sin esperar.
- **Alternatives considered**: Timestamps UTC (obliga a convertir en todas partes para mostrar la
  grilla por día local); una librería de fechas (innecesaria con `Intl`).

## R4. Sesiones con expiración por inactividad de 24 h (FR-003, FR-004, FR-006a)

- **Decision**: Sesiones propias en la tabla `sessions` de SQLite. Al iniciar sesión se genera un
  token aleatorio (`crypto.randomBytes(32)`), se guarda su hash SHA-256 y se envía el token en una
  cookie `sid` `HttpOnly`, `SameSite=Lax` (y `Secure` en producción). Un middleware `requireAuth`
  busca la sesión, la rechaza con 401 si `last_activity_at` tiene más de 24 h (y la borra), y si
  es válida actualiza `last_activity_at`. Cerrar sesión borra la fila y la cookie.
- **Rationale**: Cubre exactamente la regla de 24 h deslizantes con una tabla y un middleware. No
  necesita dependencias de sesión ni un store adicional. Guardar el hash evita que una copia de la
  base permita suplantar sesiones.
- **Alternatives considered**: `express-session` con store SQLite (dependencia extra y otra capa de
  configuración para lo mismo); JWT (no se puede invalidar al cerrar sesión sin lista negra y la
  expiración por inactividad es incómoda).

## R5. Hash de contraseñas (FR-006)

- **Decision**: `crypto.scrypt` nativo de Node con sal aleatoria de 16 bytes; se guarda
  `scrypt$<sal>$<hash>`. La comparación usa `crypto.timingSafeEqual`.
- **Rationale**: Algoritmo resistente a fuerza bruta sin dependencias nativas extra.
- **Alternatives considered**: `bcrypt` (módulo nativo adicional), `argon2` (idem).

## R6. Formato de errores y códigos HTTP (Principio V, FR-022)

- **Decision**: Toda respuesta de error usa `{ "error": { "code": "<CODIGO>", "message": "<texto
  amigable en español>" } }`. Códigos HTTP: 400 petición inválida (formato, cancha fuera de
  catálogo, bloque pasado o ya comenzado), 401 no autenticado o credenciales incorrectas, 404
  reserva inexistente o ajena, 409 conflicto (bloque tomado, límite de reservas, correo en uso).
  Un manejador de errores final de Express convierte cualquier excepción no prevista en 500 con un
  mensaje genérico y deja el detalle solo en el log del servidor. El frontend muestra
  `error.message` y, si la respuesta no trae ese formato o falla la red, un mensaje genérico.
- **Rationale**: Cumple el Principio V. Responder 404 para reservas ajenas evita revelar que
  existen (FR-005).
- **Alternatives considered**: Responder 403 para reservas ajenas (revela que el id existe).

## R7. Actualización de la grilla (FR-010a)

- **Decision**: El frontend pide la disponibilidad al montar la vista, al cambiar cancha o fecha y
  después de reservar, cancelar o recibir 409 `SLOT_TAKEN`. Sin polling ni WebSockets.
- **Rationale**: Es exactamente lo que fija la clarificación; la re-validación del backend (R1)
  garantiza la consistencia.

## R8. Frontend: build, navegación y calendario

- **Decision**: Vite + React + TypeScript + Tailwind CSS. Navegación con `react-router-dom`
  (rutas `/login`, `/registro`, `/canchas`, `/mis-reservas`). Calendario con
  `<input type="date">` nativo. Confirmación de cancelación con un modal propio. En desarrollo,
  el proxy de Vite redirige `/api` al backend para que la cookie sea del mismo origen.
- **Rationale**: Es la configuración estándar y mínima del stack obligatorio. El input de fecha
  nativo evita una librería de calendario.
- **Alternatives considered**: Librería de calendario (dependencia innecesaria); navegación por
  estado sin router (dificulta redirigir a login al expirar la sesión).

## R9. Pruebas

- **Decision**: Vitest en backend y frontend. Backend: pruebas de API con `supertest` contra una
  base SQLite en memoria (`:memory:`) y el reloj reemplazado. Frontend: Vitest + React Testing
  Library para la grilla, el formulario de reserva y "Mis Reservas".
- **Rationale**: Un solo runner para todo el proyecto, compatible con TypeScript sin
  configuración extra. La base en memoria hace las pruebas rápidas y aisladas.
- **Alternatives considered**: Jest (más configuración con TS y ESM).

## R10. Esquema y semilla de canchas (Principio I, FR-007)

- **Decision**: `db/schema.sql` crea las tablas con `IF NOT EXISTS` e inserta las 5 canchas con
  `INSERT OR IGNORE` e ids fijos (1–5). El backend lo ejecuta al arrancar sobre `db/padel.db`. No
  hay endpoints para crear, editar ni borrar canchas.
- **Rationale**: Catálogo cerrado garantizado por datos y por la API. Sin herramienta de
  migraciones: es el primer y único esquema.
