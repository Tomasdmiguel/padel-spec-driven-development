# Feature Specification: Creación de Reservas

**Feature Branch**: `001-creacion-reservas`

**Created**: 2026-11-04

**Status**: Draft

**Input**: User description: "Feature: Creación de Reservas. 1. Intención: Permitir a los usuarios autenticados agendar un bloque de tiempo, garantizando que el sistema no asigne el mismo turno a dos personas. 2. Comportamiento: El usuario selecciona un bloque marcado como "Disponible" y confirma la acción. El sistema asigna el bloque al usuario y actualiza la grilla en tiempo real. 3. Restricciones: Prevención de Colisión: El sistema DEBE validar la disponibilidad en la base de datos antes de guardar. Si dos usuarios intentan reservar el mismo bloque al mismo milisegundo, el segundo DEBE recibir un error de conflicto. Regla de Tiempo: No se pueden realizar reservas en fechas u horarios del pasado. Formato: Las reservas son estrictamente en bloques enteros de 1 hora. 4. No-Objetivos: No se programará pasarela de pagos integrada (se paga en el club). No se enviarán correos ni SMS de confirmación."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Reservar un bloque disponible (Priority: P1)

Un jugador con sesión activa ve la grilla de horarios de las canchas, elige un bloque de 1 hora
marcado como "Disponible" en una de las 5 canchas del club, confirma la acción y el bloque queda
reservado a su nombre. La grilla muestra de inmediato ese bloque como ocupado.

**Why this priority**: Es el valor central de la funcionalidad: sin poder reservar un bloque
libre no existe el producto.

**Independent Test**: Iniciar sesión, seleccionar un bloque "Disponible" futuro, confirmar y
verificar que la reserva queda registrada a nombre del usuario y que el bloque deja de mostrarse
como "Disponible".

**Acceptance Scenarios**:

1. **Given** un usuario con sesión activa y un bloque futuro "Disponible" en la Cancha Laureles,
   **When** selecciona el bloque y confirma, **Then** el sistema registra la reserva a su nombre
   y muestra un mensaje de éxito.
2. **Given** una reserva recién confirmada, **When** el usuario mira la grilla, **Then** el
   bloque aparece como no disponible sin necesidad de recargar la página.
3. **Given** un usuario que seleccionó un bloque, **When** cancela el paso de confirmación,
   **Then** no se crea ninguna reserva y el bloque sigue "Disponible".

---

### User Story 2 - Evitar la doble asignación de un turno (Priority: P1)

Dos jugadores intentan reservar el mismo bloque de la misma cancha prácticamente al mismo
tiempo. Solo uno obtiene la reserva; el otro recibe un mensaje claro de que la cancha ya fue
reservada en ese horario.

**Why this priority**: Es la garantía principal de la funcionalidad y un principio no negociable
de la constitución (Principio III). Un turno asignado a dos personas genera conflictos reales en
el club.

**Independent Test**: Enviar dos solicitudes de reserva simultáneas para el mismo bloque y la
misma cancha desde dos usuarios distintos, y verificar que existe exactamente una reserva y que
la otra solicitud recibe un error de conflicto.

**Acceptance Scenarios**:

1. **Given** un bloque "Disponible", **When** dos usuarios confirman la reserva del mismo bloque
   en el mismo instante, **Then** exactamente uno obtiene la reserva y el otro recibe un error de
   conflicto con el mensaje "La cancha ya fue reservada en este horario".
2. **Given** un usuario con la grilla abierta desde hace un rato, **When** intenta confirmar un
   bloque que otro usuario reservó mientras tanto, **Then** recibe el error de conflicto y la
   grilla se actualiza para mostrar el bloque como no disponible.
3. **Given** un bloque ya reservado, **When** cualquier usuario intenta reservarlo, **Then** el
   sistema rechaza la reserva aunque la grilla del usuario estuviera desactualizada.

---

### User Story 3 - Ver la grilla actualizada en tiempo real (Priority: P2)

Mientras un jugador mira la grilla, las reservas que hacen otros usuarios aparecen sin que tenga
que recargar la página, de modo que no intente reservar bloques que ya no están libres.

**Why this priority**: Mejora la experiencia y reduce intentos fallidos, pero la garantía contra
colisiones (US2) ya protege la integridad de los datos aunque la grilla no se actualice.

**Independent Test**: Abrir la grilla en dos sesiones de usuarios distintos, reservar un bloque
desde una y verificar que en la otra el bloque pasa a no disponible sin recargar.

**Acceptance Scenarios**:

1. **Given** dos usuarios viendo la grilla del mismo día, **When** uno reserva un bloque,
   **Then** el otro ve ese bloque como no disponible en pocos segundos, sin recargar la página.

---

### User Story 4 - Rechazar reservas inválidas (Priority: P2)

El sistema impide reservar bloques en el pasado, bloques que no sean de 1 hora entera, canchas
que no pertenecen al club o reservas hechas sin sesión activa, y explica el motivo con un mensaje
amigable.

**Why this priority**: Protege las reglas del negocio; complementa la funcionalidad principal.

**Independent Test**: Intentar reservar un bloque pasado, un bloque con duración o inicio no
válido, una cancha inexistente y una reserva sin sesión, y verificar que todas se rechazan con un
mensaje comprensible.

**Acceptance Scenarios**:

1. **Given** un bloque cuyo horario de inicio ya pasó, **When** un usuario intenta reservarlo,
   **Then** el sistema lo rechaza como petición inválida y la grilla no lo ofrece como
   "Disponible".
2. **Given** una solicitud de reserva con una duración distinta de 1 hora o con un inicio que no
   coincide con una hora en punto, **When** se envía, **Then** el sistema la rechaza como
   petición inválida.
3. **Given** una solicitud para una cancha que no es una de las 5 del club, **When** se envía,
   **Then** el sistema la rechaza como petición inválida.
4. **Given** un visitante sin sesión activa, **When** intenta reservar, **Then** el sistema
   rechaza la acción como no autenticado y le indica que debe iniciar sesión.

---

### Edge Cases

- **Bloque en curso**: un bloque cuya hora de inicio ya pasó (por ejemplo, son las 18:20 y el
  bloque es de 18:00 a 19:00) se considera pasado y no se puede reservar.
- **Bloque que pasa a ser pasado durante la confirmación**: si el usuario selecciona el bloque
  de las 18:00 a las 17:59 y confirma a las 18:00:01, la reserva se rechaza porque la validación
  se hace al momento de guardar.
- **Cambio de día**: los bloques que cruzan medianoche (23:00 a 00:00) se tratan como cualquier
  otro bloque de 1 hora del día en que empiezan.
- **Sesión expirada al confirmar**: si la sesión venció entre la selección y la confirmación, la
  reserva se rechaza como no autenticada y se pide iniciar sesión de nuevo.
- **Confirmación repetida**: si el usuario confirma dos veces seguidas el mismo bloque (por
  ejemplo, doble clic), se crea como máximo una reserva y no se muestra un error técnico.
- **Fallo inesperado**: ante cualquier error técnico, el usuario ve un mensaje amigable y nunca
  detalles técnicos internos.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE mostrar una grilla de bloques de 1 hora por cancha, indicando
  cuáles están "Disponible" y cuáles no.
- **FR-002**: El sistema DEBE ofrecer exactamente las 5 canchas del club: Cancha Laureles,
  Cancha El Poblado, Cancha Belén, Cancha Robledo y Cancha Envigado.
- **FR-003**: Los bloques DEBEN existir las 24 horas del día, cada uno empezando a una hora en
  punto y durando exactamente 1 hora.
- **FR-004**: Solo un usuario con sesión activa DEBE poder crear una reserva; el intento sin
  sesión DEBE rechazarse como no autenticado.
- **FR-005**: El usuario DEBE seleccionar un bloque "Disponible" y confirmar explícitamente
  antes de que se cree la reserva.
- **FR-006**: Al confirmar, el sistema DEBE verificar en los datos guardados, en ese mismo
  momento, que la cancha esté libre en el bloque antes de registrar la reserva.
- **FR-007**: El sistema NO DEBE permitir nunca dos reservas para la misma cancha en el mismo
  bloque, ni siquiera cuando las solicitudes llegan simultáneamente.
- **FR-008**: Cuando una reserva no se pueda crear porque el bloque ya está ocupado, el sistema
  DEBE responder con un error de conflicto y mostrar el mensaje "La cancha ya fue reservada en
  este horario".
- **FR-009**: El sistema DEBE rechazar como petición inválida cualquier reserva cuyo horario de
  inicio sea anterior o igual al momento actual.
- **FR-010**: El sistema DEBE rechazar como petición inválida cualquier reserva que no sea un
  bloque entero de 1 hora que empiece en una hora en punto, o que no corresponda a una de las 5
  canchas.
- **FR-011**: La grilla NO DEBE ofrecer como "Disponible" bloques del pasado ni bloques ya
  reservados.
- **FR-012**: Tras una reserva exitosa, el sistema DEBE asociar el bloque al usuario que la hizo
  y confirmárselo con un mensaje de éxito en pantalla.
- **FR-013**: Cuando se crea una reserva, todos los usuarios que estén viendo la grilla DEBEN ver
  el bloque como no disponible sin recargar la página.
- **FR-014**: Todos los errores mostrados al usuario DEBEN ser mensajes amigables, sin detalles
  técnicos internos.
- **FR-015**: El sistema DEBE distinguir tres tipos de rechazo: petición inválida, usuario no
  autenticado y conflicto de reserva.

### Key Entities

- **Cancha**: una de las 5 canchas fijas del club, identificada por su nombre.
- **Bloque**: franja de 1 hora en una cancha, definida por la cancha, la fecha y la hora de
  inicio (en punto). Su estado visible es "Disponible" o no disponible.
- **Reserva**: asignación de un bloque de una cancha a un usuario. Atributos: cancha, fecha,
  hora de inicio, usuario que reservó y momento de creación. Como máximo existe una reserva por
  cancha y bloque.
- **Usuario**: persona con sesión activa que puede crear reservas.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En pruebas de 100 intentos simultáneos sobre el mismo bloque, existe exactamente 1
  reserva y los 99 intentos restantes reciben un error de conflicto: 0 turnos asignados a dos
  personas.
- **SC-002**: Un usuario con sesión activa completa una reserva, desde ver la grilla hasta la
  confirmación, en menos de 30 segundos.
- **SC-003**: El 100% de los intentos de reservar bloques pasados, bloques que no son de 1 hora
  entera, canchas fuera del catálogo o reservas sin sesión son rechazados.
- **SC-004**: Las reservas hechas por otros usuarios aparecen en la grilla de quien la está
  viendo en menos de 3 segundos, sin recargar.
- **SC-005**: El 100% de los mensajes de error mostrados al usuario están en lenguaje claro, sin
  detalles técnicos.

## Assumptions

- **Zona horaria**: "pasado" se evalúa en la hora local del club (America/Bogota, por las
  canchas de Medellín), usando el reloj del sistema y no el del dispositivo del usuario.
- **Inicio de sesión ya existe**: el registro y el inicio de sesión no forman parte de esta
  funcionalidad; se asume que existe una forma de tener sesión activa e identificar al usuario.
- **Sin límites adicionales**: no hay tope de reservas por usuario ni límite de anticipación,
  porque no se pidieron y la constitución prohíbe agregar funcionalidades "por si acaso"
  (Principio VI).
- **Fuera de alcance**: cancelar o modificar reservas, ver un historial de "mis reservas", pagos
  en línea (se paga en el club) y notificaciones por correo o SMS.
- **Tiempo real**: "actualizar la grilla en tiempo real" se interpreta como que cualquier usuario
  que esté viendo la grilla ve los cambios sin recargar, no solo quien hizo la reserva.
- **Disponibilidad**: todo bloque futuro sin reserva está "Disponible"; no existen cierres,
  mantenimientos ni bloqueos administrativos en esta funcionalidad.
