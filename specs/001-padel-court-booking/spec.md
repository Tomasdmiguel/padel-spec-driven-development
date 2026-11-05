# Feature Specification: Reserva de Canchas de Pádel

**Feature Branch**: `001-padel-court-booking`

**Created**: 2026-11-04

**Status**: Draft

**Input**: User description: "Autenticación de usuarios con correo y contraseña; exploración de las
5 canchas con calendario y grilla de 24 bloques de 1 hora; creación de reservas con re-validación
anti-colisión, solo bloques enteros y nunca en el pasado; panel Mis Reservas con reservas futuras,
historial y cancelación. Fuera de alcance: pagos, panel de administración, notificaciones externas,
reservas de varias horas en un clic y matchmaking."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Registro e inicio de sesión (Priority: P1)

Una persona que quiere jugar crea una cuenta con su correo electrónico y una contraseña, y luego
inicia sesión. Con la sesión activa puede acceder a la disponibilidad completa y a sus reservas.

**Why this priority**: Sin sesión activa no se puede ver la disponibilidad completa ni reservar.
Es la puerta de entrada a todo el resto de la funcionalidad.

**Independent Test**: Registrar una cuenta nueva, cerrar sesión, iniciar sesión con las mismas
credenciales y verificar el acceso a la grilla de disponibilidad. Intentar acceder sin sesión y
verificar que se pide iniciar sesión.

**Acceptance Scenarios**:

1. **Given** un correo no registrado, **When** la persona se registra con ese correo y una
   contraseña válida, **Then** la cuenta se crea y queda con sesión activa.
2. **Given** un correo ya registrado, **When** alguien intenta registrarse con él, **Then** el
   registro se rechaza con un mensaje amigable indicando que el correo ya está en uso.
3. **Given** una cuenta existente, **When** la persona ingresa correo y contraseña correctos,
   **Then** inicia sesión y accede a la disponibilidad.
4. **Given** una cuenta existente, **When** la persona ingresa una contraseña incorrecta,
   **Then** el acceso se rechaza con un mensaje amigable que no revela cuál dato falló.
5. **Given** una persona sin sesión activa, **When** intenta ver la grilla de horarios o
   reservar, **Then** el sistema le pide iniciar sesión.
6. **Given** una persona con sesión activa, **When** cierra sesión, **Then** pierde el acceso a
   la disponibilidad completa y a sus reservas hasta volver a iniciar sesión.

---

### User Story 2 - Consultar disponibilidad y reservar un bloque (Priority: P1)

Un usuario con sesión activa elige una de las 5 canchas, selecciona una fecha en el calendario,
ve la grilla de 24 bloques de 1 hora con su estado ("Disponible" o "Reservado") y confirma la
reserva de un bloque disponible.

**Why this priority**: Es el valor central del producto: conseguir un turno en una cancha sin
colisiones con otras personas.

**Independent Test**: Con un usuario sin reservas activas, elegir una cancha y una fecha futura,
reservar un bloque disponible y verificar que el bloque pasa a "Reservado" para cualquier usuario.

**Acceptance Scenarios**:

1. **Given** un usuario con sesión activa, **When** abre la exploración de canchas, **Then** ve
   exactamente estas 5 canchas: Cancha Laureles, Cancha El Poblado, Cancha Belén, Cancha Robledo
   y Cancha Envigado.
2. **Given** una cancha y una fecha seleccionadas, **When** se muestra la grilla, **Then** se ven
   24 bloques de 1 hora (00:00–01:00 hasta 23:00–24:00), cada uno marcado como "Disponible" o
   "Reservado".
3. **Given** un bloque "Disponible" en el futuro y un usuario sin reserva activa, **When** el
   usuario lo selecciona y confirma, **Then** la reserva se crea y el bloque queda "Reservado".
4. **Given** que otro usuario reservó el mismo bloque mientras el primero confirmaba, **When** el
   primero confirma, **Then** el sistema re-valida, rechaza la reserva y muestra el mensaje "La
   cancha ya fue reservada en este horario", y la grilla se actualiza.
5. **Given** un bloque cuyo horario de inicio ya pasó, **When** el usuario intenta reservarlo,
   **Then** el sistema no lo permite.
6. **Given** un usuario que ya tiene una reserva activa, **When** intenta reservar otro bloque,
   **Then** el sistema rechaza la reserva con un mensaje amigable indicando que ya tiene una
   reserva activa.

---

### User Story 3 - Ver y cancelar mis reservas (Priority: P2)

Un usuario con sesión activa abre su panel "Mis Reservas", ve su reserva futura y su historial de
reservas pasadas (cancha, fecha y hora), y puede cancelar una reserva futura confirmando la
acción.

**Why this priority**: Permite gestionar el turno propio y liberar la cancha, pero la reserva en
sí ya entrega valor sin este panel.

**Independent Test**: Con un usuario que tiene una reserva futura y al menos una pasada, abrir el
panel, verificar ambas listas, cancelar la futura y verificar que el bloque vuelve a "Disponible".

**Acceptance Scenarios**:

1. **Given** un usuario con reservas, **When** abre "Mis Reservas", **Then** ve separadas sus
   reservas futuras y su historial de reservas pasadas, cada una con nombre de la cancha, fecha y
   hora.
2. **Given** una reserva futura propia, **When** el usuario elige cancelarla, **Then** el sistema
   pide confirmación antes de cancelar.
3. **Given** que el usuario confirmó la cancelación, **When** termina la acción, **Then** la
   reserva desaparece de las futuras, el bloque vuelve a "Disponible" y el usuario puede hacer una
   nueva reserva.
4. **Given** que el usuario desiste en el paso de confirmación, **When** cierra la confirmación,
   **Then** la reserva se mantiene sin cambios.
5. **Given** una reserva pasada, **When** el usuario la ve en el historial, **Then** no tiene la
   opción de cancelarla.
6. **Given** una reserva de otro usuario, **When** alguien intenta verla o cancelarla, **Then** el
   sistema no lo permite.

### Edge Cases

- Dos usuarios confirman el mismo bloque de la misma cancha casi al mismo tiempo: solo una reserva
  se guarda; la otra recibe el mensaje de conflicto.
- El usuario abre la grilla, deja pasar el tiempo y el bloque elegido pasa a estar en el pasado
  antes de confirmar: la reserva se rechaza al confirmar.
- El usuario consulta el día actual: los bloques cuyo inicio ya pasó no se pueden reservar.
- El usuario selecciona una fecha pasada en el calendario: ningún bloque se puede reservar.
- La reserva activa de un usuario llega a su hora de fin: pasa al historial y el usuario ya puede
  reservar de nuevo.
- La sesión del usuario expira mientras está en la grilla: al confirmar se le pide iniciar sesión
  y no se guarda ninguna reserva.
- Una petición pide una cancha que no está en el catálogo o un horario que no es un bloque entero:
  se rechaza como petición inválida.
- Un usuario intenta cancelar una reserva cuyo bloque ya comenzó: no se permite.

## Requirements *(mandatory)*

### Functional Requirements

**Autenticación**

- **FR-001**: El sistema DEBE permitir registrarse con un correo electrónico y una contraseña.
- **FR-002**: El sistema DEBE rechazar el registro de un correo que ya tenga cuenta.
- **FR-003**: El sistema DEBE permitir iniciar y cerrar sesión con correo y contraseña.
- **FR-004**: El sistema DEBE exigir sesión activa para ver la grilla de disponibilidad, crear
  reservas, ver "Mis Reservas" y cancelar reservas; sin sesión DEBE responder como no autenticado.
- **FR-005**: El sistema DEBE permitir que cada usuario vea y cancele solo sus propias reservas.
- **FR-006**: El sistema NO DEBE guardar las contraseñas en texto legible.

**Exploración**

- **FR-007**: El sistema DEBE listar exactamente 5 canchas fijas: Cancha Laureles, Cancha El
  Poblado, Cancha Belén, Cancha Robledo y Cancha Envigado; no se pueden agregar, editar ni
  eliminar canchas.
- **FR-008**: El usuario DEBE poder elegir una fecha en un calendario para la cancha seleccionada.
- **FR-009**: Para la cancha y fecha elegidas, el sistema DEBE mostrar 24 bloques de 1 hora que
  cubren el día completo, cada uno marcado como "Disponible" o "Reservado".
- **FR-010**: La grilla NO DEBE mostrar a otros usuarios quién hizo una reserva.

**Reservas**

- **FR-011**: El usuario DEBE poder seleccionar un bloque disponible y confirmar su reserva.
- **FR-012**: Las reservas DEBEN corresponder a un único bloque entero de 1 hora que empieza en
  hora en punto (ej. 14:00–15:00).
- **FR-013**: Al confirmar, el sistema DEBE re-validar que el bloque siga disponible; si ya está
  tomado DEBE rechazar la reserva como conflicto y mostrar "La cancha ya fue reservada en este
  horario".
- **FR-014**: El sistema NO DEBE permitir, bajo ninguna circunstancia, dos reservas para la misma
  cancha en el mismo bloque, incluso ante confirmaciones simultáneas.
- **FR-015**: El sistema DEBE rechazar reservas de bloques cuyo inicio ya pasó.
- **FR-016**: Un usuario DEBE tener como máximo una reserva activa (futura o en curso, sin
  cancelar); el sistema DEBE rechazar una nueva reserva mientras tenga una activa.
- **FR-017**: El sistema DEBE rechazar como petición inválida cualquier reserva con cancha fuera
  del catálogo, fecha u hora mal formadas, o un horario que no sea un bloque entero.

**Mis Reservas**

- **FR-018**: El usuario DEBE tener un panel "Mis Reservas" con dos listas: reservas futuras e
  historial de reservas pasadas.
- **FR-019**: Cada elemento de las listas DEBE mostrar nombre de la cancha, fecha y hora.
- **FR-020**: El usuario DEBE poder cancelar una reserva propia cuyo bloque aún no comenzó, tras
  confirmar la acción.
- **FR-021**: Al cancelarse una reserva, el bloque DEBE volver a "Disponible" de inmediato.

**Errores**

- **FR-022**: Todo error DEBE mostrarse al usuario como un mensaje amigable; nunca como error
  técnico ni detalle interno.

### Key Entities *(include if feature involves data)*

- **Usuario**: persona registrada. Atributos: correo electrónico (único), contraseña protegida,
  fecha de registro. Tiene muchas reservas.
- **Cancha**: una de las 5 canchas fijas. Atributos: identificador y nombre. Catálogo inmutable.
- **Reserva**: un bloque de 1 hora de una cancha para un usuario. Atributos: usuario, cancha,
  fecha, hora de inicio, estado (activa o cancelada), fecha de creación. La combinación cancha +
  fecha + hora de inicio es única entre las reservas no canceladas.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Una persona nueva puede registrarse y completar su primera reserva en menos de
  2 minutos.
- **SC-002**: Un usuario con sesión activa ve la grilla de una cancha y fecha en menos de
  2 segundos después de elegirlas.
- **SC-003**: En pruebas con confirmaciones simultáneas del mismo bloque, el 100 % de los casos
  terminan con exactamente una reserva guardada y el resto rechazadas con el mensaje de conflicto.
- **SC-004**: Ninguna reserva se guarda para un bloque pasado, fuera del catálogo o de un usuario
  con otra reserva activa (0 casos en pruebas).
- **SC-005**: El 90 % de los usuarios completa una reserva en el primer intento sin ayuda.
- **SC-006**: Ningún mensaje visible para el usuario contiene errores técnicos ni trazas internas.
- **SC-007**: Un usuario puede cancelar su reserva futura en menos de 30 segundos y el bloque se ve
  como "Disponible" para otros usuarios en su siguiente consulta.

## Assumptions

- Todos los horarios usan la hora local del club (America/Bogota); "pasado" se evalúa con esa hora.
- Una reserva activa es una reserva no cancelada cuyo bloque aún no terminó; cuando termina pasa al
  historial.
- Sin sesión activa, una persona puede ver la lista de canchas pero no la grilla de disponibilidad
  ni reservar ("disponibilidad completa" requiere sesión).
- Las reservas canceladas no aparecen en el historial; el historial muestra solo reservas que se
  cumplieron.
- La cancelación está permitida hasta el inicio del bloque, sin plazo mínimo de anticipación.
- No hay límite de cuánto tiempo a futuro se puede reservar.
- La contraseña debe tener al menos 8 caracteres; no hay recuperación de contraseña ni
  verificación de correo en esta iteración (se necesitaría enviar correos, que está fuera de
  alcance).
- La sesión se mantiene hasta que el usuario la cierre o expire por inactividad.
- Fuera de alcance: pasarela de pagos (el pago es presencial en el club), panel de administración
  de canchas, notificaciones externas (correo, SMS, WhatsApp), reservas de varias horas en un
  solo clic (2 horas = 2 reservas, sujeto a la regla de una reserva activa) y matchmaking o
  torneos.
