# Implementation Plan: Reserva de Canchas de Pádel

**Branch**: `001-padel-court-booking` | **Date**: 2026-11-05 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-padel-court-booking/spec.md`

## Summary

Aplicación web para reservar bloques de 1 hora en 5 canchas fijas. Los usuarios se registran con
correo y contraseña, consultan una grilla de 24 bloques por cancha y fecha, reservan (máximo 2
reservas activas) y gestionan sus reservas en "Mis Reservas".

Enfoque técnico: API REST en Express + TypeScript sobre SQLite (`db/padel.db`, better-sqlite3), con
sesiones propias por cookie que expiran a las 24 h de inactividad. La anti-colisión usa un índice
único parcial y una transacción `IMMEDIATE`. El frontend es React + TypeScript + Tailwind con Vite.
Detalle de las decisiones en [research.md](research.md).

## Technical Context

**Language/Version**: TypeScript 5.x sobre Node.js 22 LTS (backend) y navegador moderno (frontend)

**Primary Dependencies**: Backend: Express 5, better-sqlite3, cookie-parser. Frontend: React 19,
react-router-dom, Tailwind CSS, Vite. Hash con `crypto.scrypt` nativo.

**Storage**: SQLite local en `db/padel.db`, esquema en `db/schema.sql`, acceso con SQL puro vía
better-sqlite3 (sin ORM)

**Testing**: Vitest + supertest (API sobre SQLite `:memory:` con reloj simulado); Vitest + React
Testing Library (frontend)

**Target Platform**: Servidor Node.js local; navegadores de escritorio y móvil actuales

**Project Type**: Aplicación web (frontend + backend)

**Performance Goals**: Grilla visible en < 2 s tras elegir cancha y fecha (SC-002); consultas
indexadas por cancha + fecha

**Constraints**: 0 reservas duplicadas por bloque incluso con confirmaciones simultáneas (SC-003);
horario `America/Bogota`; sin errores técnicos visibles (SC-006); sesión con 24 h de inactividad
máxima

**Scale/Scope**: Un club, 5 canchas, 24 bloques por día, del orden de cientos de usuarios; 4
pantallas (login, registro, canchas/grilla, Mis Reservas)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio / regla | Cómo lo cumple el diseño | Estado |
|---|---|---|
| I. Catálogo cerrado | 5 canchas sembradas con ids fijos en `db/schema.sql`; sin endpoints de alta/edición/baja; `courtId` fuera del catálogo → 400 | ✅ |
| II. Bloques de 1 hora, 24 h | Reserva = `date` + `start_hour` entero 0–23 con `CHECK`; grilla siempre de 24 bloques | ✅ |
| III. Prevención de colisiones (NO NEGOCIABLE) | Validación dentro de transacción `IMMEDIATE` + índice único parcial `(court_id, date, start_hour) WHERE status='active'` | ✅ |
| IV. Autenticación obligatoria | Middleware `requireAuth` en disponibilidad y reservas; sin sesión → 401 | ✅ |
| V. Errores amigables y HTTP semántico | Formato `{error:{code,message}}`; 400 / 401 / 409 (+404 reserva ajena, 500 genérico); manejador final sin stack traces | ✅ |
| VI. Cero código sombra | Solo endpoints y pantallas que piden los FR; sin polling, admin, pagos ni notificaciones | ✅ |
| VII. Spec como fuente de verdad | El límite de 2 reservas y la sesión de 24 h vienen de `/speckit-clarify` y ya están en `spec.md` | ✅ |
| Stack | React + TS + Tailwind; Node + Express + TS; SQLite `padel.db`; better-sqlite3 con SQL puro | ✅ |
| Estructura plana | `/frontend`, `/backend`, `/db`; módulos por responsabilidad, sin capas de Clean Architecture | ✅ |
| Estilo | Funciones y componentes funcionales con hooks; sin clases; camelCase y PascalCase | ✅ |

**Nota sobre la ubicación de `padel.db`**: la constitución exige el archivo `padel.db` y la carpeta
`/db`; se ubica en `db/padel.db` (ignorado por git) junto a `db/schema.sql`.

**Re-check post-diseño (Fase 1)**: el modelo de datos, el contrato de la API y el quickstart
mantienen todos los puntos anteriores. El 404 para reservas ajenas o inexistentes complementa la
lista mínima de códigos de la constitución sin contradecirla. **Resultado: PASS**, sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/001-padel-court-booking/
├── plan.md              # Este archivo
├── research.md          # Fase 0: decisiones técnicas
├── data-model.md        # Fase 1: entidades, estados, validaciones
├── quickstart.md        # Fase 1: puesta en marcha y escenarios de validación
├── contracts/
│   └── api.md           # Fase 1: contrato REST
└── tasks.md             # Fase 2 (/speckit-tasks, aún no creado)
```

### Source Code (repository root)

```text
db/
├── schema.sql           # tablas, índices y semilla de las 5 canchas
└── padel.db             # generado al arrancar (git-ignored)

backend/
├── package.json
├── tsconfig.json
├── src/
│   ├── server.ts        # arranque: abre la base, aplica schema, escucha :3000
│   ├── app.ts           # crea la app Express (inyectable para pruebas)
│   ├── db.ts            # conexión better-sqlite3 y aplicación del esquema
│   ├── clock.ts         # "ahora" en America/Bogota (reemplazable en pruebas)
│   ├── errors.ts        # códigos, mensajes amigables y manejador de errores
│   ├── auth.ts          # hash de contraseña, sesiones, requireAuth, rutas /api/auth
│   ├── courts.ts        # rutas /api/courts y disponibilidad
│   ├── reservations.ts  # rutas /api/reservations (crear, mías, cancelar)
│   └── validation.ts    # validación de correo, fecha, hora y cancha
└── tests/
    ├── auth.test.ts
    ├── courts.test.ts
    └── reservations.test.ts

frontend/
├── package.json
├── vite.config.ts       # proxy /api -> http://localhost:3000
├── index.html
├── src/
│   ├── main.tsx
│   ├── App.tsx          # rutas y guarda de sesión
│   ├── api.ts           # fetch con credenciales y traducción de errores
│   ├── types.ts         # Court, Block, Reservation, User
│   ├── hooks/
│   │   └── useAuth.tsx  # contexto de sesión
│   ├── pages/
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   ├── CourtsPage.tsx         # selector de cancha, fecha y grilla
│   │   └── MyReservationsPage.tsx
│   └── components/
│       ├── BlockGrid.tsx
│       ├── ConfirmDialog.tsx
│       └── ErrorMessage.tsx
└── tests/
    ├── BlockGrid.test.tsx
    └── MyReservationsPage.test.tsx
```

**Structure Decision**: aplicación web con la estructura plana que exige la constitución:
`/frontend`, `/backend` y `/db` en la raíz, cada app con su propio `package.json`. El backend se
organiza por responsabilidad (un archivo por dominio) en lugar de capas de modelos, servicios y
controladores.

## Complexity Tracking

Sin violaciones de la constitución; no aplica.
