# Feature Specification: Reserva de Canchas de Pádel

**Feature Branch**: `001-padel-court-booking`

**Created**: 2026-11-04

**Status**: Draft

**Input**: User description: "Autenticación de usuarios con correo y contraseña; exploración de las 5 canchas fijas con calendario y grilla de 24 bloques de 1 hora (Disponible/Reservado); creación de reservas con re-validación anti-colisión, sin reservas en el pasado y máximo una reserva activa por usuario; panel 'Mis Reservas' con reservas futuras, historial y cancelación. Fuera de alcance: pagos, panel de administración, notificaciones externas, reservas de más de 1 hora en un clic y matchmaking."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Reservar un bloque horario en una cancha (Priority: P1)

Un jugador con sesión activa elige una de las 5 canchas, selecciona una fecha en el calendario, ve la grilla de 24 bloques de 1 hora con su estado ("Disponible" o "Reservado"), elige un bloque disponible y confirma la reserva. Justo antes de guardar, el sistema vuelve a comprobar que el bloque siga libre; si otro usuario lo tomó en ese lapso, la reserva se rechaza con un mensaje claro.

**Why this priority**: Es el valor central del producto: sin la capacidad de reservar sin colisiones no existe la aplicación. La prevención de colisiones es no negociable según la constitución.

**Independent Test**: Con un usuario de prueba ya autenticado, reservar un bloque futuro libre y verificar que pasa a mostrarse como "Reservado" para cualquier otro usuario; luego intentar reservar el mismo bloque con otro usuario y verificar el rechazo.

**Acceptance Scenarios**:

1. **Given** un usuario con sesión activa y sin reservas activas, **When** selecciona "Cancha Belén", una fecha futura y el bloque 14:00–15:00 que figura como "Disponible" y confirma, **Then** la reserva queda registrada a su nombre y el bloque aparece como "Reservado" en la grilla.
2. **Given** dos usuarios ven el mismo bloque como "Disponible", **When** el primero confirma y luego el segundo confirma el mismo bloque, **Then** solo la reserva del primero se registra y el segundo recibe el mensaje "La cancha ya fue reservada en este horario" y ve la grilla actualizada.
3. **Given** un usuario con sesión activa, **When** consulta la grilla de la fecha de hoy, **Then** los bloques cuya hora de inicio ya pasó no pueden seleccionarse para reservar.
4. **Given** un usuario que ya tiene una reserva activa, **When** intenta confirmar una segunda reserva en cualquier cancha, **Then** el sistema la rechaza con un mensaje claro que indica que solo se permite una reserva activa a la vez.
5. **Given** una cancha y fecha seleccionadas, **When** el usuario observa la grilla, **Then** ve exactamente 24 bloques (00:00–01:00 hasta 23:00–24:00), cada uno marcado de forma inequívoca como "Disponible" o "Reservado".

---

### User Story 2 - Registrarse e iniciar sesión (Priority: P2)

Una persona nueva crea su cuenta con correo electrónico y contraseña, y luego inicia sesión. Solo con sesión activa puede ver la disponibilidad completa y reservar. Puede cerrar su sesión cuando termine.

**Why this priority**: Es requisito habilitante de la reserva (toda reserva pertenece a un usuario identificado), pero por sí sola no entrega el valor principal; puede probarse de forma aislada.

**Independent Test**: Registrar una cuenta nueva, cerrar sesión, iniciar sesión con las mismas credenciales y verificar que se accede a la grilla de disponibilidad; verificar que sin sesión no se accede.

**Acceptance Scenarios**:

1. **Given** un correo no registrado, **When** la persona se registra con ese correo y una contraseña válida, **Then** la cuenta se crea y la persona queda con sesión activa.
2. **Given** un correo ya registrado, **When** alguien intenta registrarse con ese mismo correo, **Then** el sistema rechaza el registro con un mensaje claro.
3. **Given** una cuenta existente, **When** el usuario ingresa correo y contraseña correctos, **Then** inicia sesión; **When** ingresa una contraseña incorrecta, **Then** recibe un mensaje genérico de credenciales inválidas sin revelar cuál dato falló.
4. **Given** una persona sin sesión activa, **When** intenta ver la grilla de disponibilidad o confirmar una reserva, **Then** el sistema no lo permite y la invita a iniciar sesión.
5. **Given** un usuario con sesión activa, **When** cierra sesión, **Then** deja de poder ver la disponibilidad o reservar hasta volver a iniciar sesión.

---

### User Story 3 - Consultar y cancelar mis reservas (Priority: P3)

El usuario abre su panel "Mis Reservas", donde ve por separado sus reservas futuras y su historial de reservas pasadas, cada una con nombre de la cancha, fecha y hora. Puede seleccionar una reserva futura y cancelarla tras confirmar la acción, liberando el bloque.

**Why this priority**: Aporta control y autogestión al usuario, y es la única forma de liberar su reserva activa para poder hacer otra; depende de que existan reservas (P1).

**Independent Test**: Con un usuario que tiene una reserva futura y una pasada, abrir el panel, verificar ambas listas, cancelar la futura y comprobar que desaparece de la lista y que el bloque vuelve a figurar como "Disponible".

**Acceptance Scenarios**:

1. **Given** un usuario con reservas futuras y pasadas, **When** abre "Mis Reservas", **Then** ve dos listas separadas (futuras e historial) y cada ítem muestra nombre de la cancha, fecha y hora.
2. **Given** un usuario con una reserva futura, **When** la selecciona, elige cancelar y confirma, **Then** la reserva se cancela, deja de figurar en sus reservas futuras y el bloque vuelve a mostrarse como "Disponible".
3. **Given** un usuario que elige cancelar una reserva, **When** desiste en el paso de confirmación, **Then** la reserva se mantiene sin cambios.
4. **Given** una reserva en el historial (pasada), **When** el usuario la consulta, **Then** no se ofrece la opción de cancelarla.
5. **Given** dos usuarios distintos, **When** cada uno abre "Mis Reservas", **Then** cada uno ve únicamente sus propias reservas y no puede cancelar reservas de otra persona.

---

### Edge Cases

- **Bloque en curso**: un bloque cuya hora de inicio ya pasó pero aún no termina (p. ej. son las 14:20 y se mira 14:00–15:00) se considera pasado y no puede reservarse.
- **Confirmación tardía**: si el usuario abre la grilla antes de que inicie un bloque y confirma después de su hora de inicio, la reserva se rechaza por estar en el pasado.
- **Fechas pasadas en el calendario**: las fechas anteriores a hoy no permiten reservar ningún bloque.
- **Reserva activa que ya ocurrió**: cuando la hora de fin de la reserva activa de un usuario pasa, esa reserva pasa al historial y el usuario puede volver a reservar.
- **Cancelación de una reserva que ya comenzó**: si el usuario intenta cancelar una reserva cuya hora de inicio ya pasó (p. ej. tenía el panel abierto), la cancelación se rechaza con un mensaje claro.
- **Cancelación y nueva reserva**: tras cancelar su reserva activa, el usuario puede crear inmediatamente otra.
- **Intento de reservar una cancha inexistente**: cualquier solicitud para una cancha fuera del catálogo de 5 se rechaza.
- **Bloque no entero**: cualquier solicitud de reserva que no corresponda exactamente a un bloque de hora entera (p. ej. 14:30–15:30 o de 2 horas) se rechaza.
- **Sesión expirada durante la reserva**: si la sesión ya no está activa al confirmar, la reserva se rechaza y se pide iniciar sesión de nuevo.
- **Errores inesperados**: ante cualquier falla técnica el usuario ve un mensaje amigable, nunca detalles técnicos internos.

## Requirements *(mandatory)*

### Functional Requirements

**Autenticación**

- **FR-001**: El sistema DEBE permitir registrar una cuenta con correo electrónico y contraseña.
- **FR-002**: El sistema DEBE impedir registrar dos cuentas con el mismo correo electrónico (sin distinguir mayúsculas de minúsculas).
- **FR-003**: El sistema DEBE validar que el correo tenga un formato válido y que la contraseña tenga al menos 8 caracteres.
- **FR-004**: El sistema DEBE permitir iniciar sesión con correo y contraseña, y ante credenciales incorrectas DEBE mostrar un mensaje genérico que no revele si el correo existe.
- **FR-005**: El sistema DEBE permitir cerrar la sesión activa.
- **FR-006**: El sistema DEBE almacenar las contraseñas de forma que no puedan leerse en texto plano.
- **FR-007**: El sistema DEBE impedir ver la grilla de disponibilidad, crear reservas, ver "Mis Reservas" y cancelar reservas a quien no tenga sesión activa, rechazando la operación incluso si se intenta sin pasar por la interfaz.

**Canchas y disponibilidad**

- **FR-008**: El sistema DEBE ofrecer exactamente 5 canchas fijas: Cancha Laureles, Cancha El Poblado, Cancha Belén, Cancha Robledo y Cancha Envigado, sin posibilidad de agregarlas, editarlas o eliminarlas desde la aplicación.
- **FR-009**: El sistema DEBE permitir al usuario elegir una cancha y una fecha mediante un calendario.
- **FR-010**: Para la cancha y fecha elegidas, el sistema DEBE mostrar 24 bloques de 1 hora, desde 00:00–01:00 hasta 23:00–24:00.
- **FR-011**: Cada bloque DEBE indicarse claramente como "Disponible" o "Reservado". Los bloques pasados DEBEN mostrarse como no reservables.
- **FR-012**: La grilla NO DEBE revelar la identidad de quien tiene reservado un bloque.

**Creación de reservas**

- **FR-013**: El usuario DEBE poder seleccionar un bloque disponible y confirmar su reserva; cada reserva cubre exactamente un bloque de 1 hora de inicio en hora entera.
- **FR-014**: Al confirmar, el sistema DEBE volver a validar, en el momento de guardar, que el bloque siga libre; si otro usuario lo reservó entretanto, DEBE rechazar la reserva con el mensaje "La cancha ya fue reservada en este horario".
- **FR-015**: El sistema NO DEBE permitir, bajo ninguna circunstancia (incluidas confirmaciones simultáneas), que existan dos reservas vigentes para la misma cancha en el mismo bloque.
- **FR-016**: El sistema DEBE rechazar reservas para bloques cuya hora de inicio ya haya pasado en el momento de confirmar.
- **FR-017**: El sistema DEBE permitir a cada usuario como máximo una reserva activa (futura y no cancelada) a la vez, considerando todas las canchas; un intento adicional DEBE rechazarse con un mensaje claro.
- **FR-018**: El sistema DEBE rechazar con un mensaje claro cualquier solicitud de reserva inválida (cancha inexistente, bloque que no es de hora entera, duración distinta de 1 hora).

**Mis Reservas**

- **FR-019**: El sistema DEBE ofrecer un panel "Mis Reservas" con dos listas: reservas futuras e historial de reservas pasadas del usuario.
- **FR-020**: Cada ítem DEBE mostrar nombre de la cancha, fecha y hora del bloque.
- **FR-021**: El usuario DEBE poder cancelar una reserva futura propia, previa confirmación explícita de la acción; al cancelarse, el bloque DEBE volver a quedar "Disponible".
- **FR-022**: El sistema NO DEBE permitir cancelar reservas cuya hora de inicio ya pasó.
- **FR-023**: Un usuario NO DEBE poder ver en su panel ni cancelar reservas de otros usuarios.

**Errores**

- **FR-024**: Todo error DEBE presentarse al usuario como un mensaje comprensible en español, sin detalles técnicos internos.

### Key Entities

- **Usuario**: persona que usa la aplicación. Atributos: correo electrónico (único), contraseña protegida, fecha de creación. Tiene cero o más reservas.
- **Cancha**: una de las 5 canchas fijas del catálogo. Atributos: identificador y nombre. Inmutable.
- **Reserva**: ocupación de una cancha por un usuario durante un bloque de 1 hora. Atributos: usuario, cancha, fecha, hora de inicio (hora entera; la hora de fin es inicio + 1 hora), estado (vigente o cancelada), fecha de creación. Regla: a lo sumo una reserva vigente por cancha y bloque; a lo sumo una reserva vigente futura por usuario.
- **Bloque horario** (concepto derivado, no almacenado): cada una de las 24 franjas de 1 hora de un día para una cancha; su estado (Disponible/Reservado/pasado) se deriva de las reservas vigentes y la hora actual.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un usuario con sesión activa puede completar una reserva (elegir cancha, fecha, bloque y confirmar) en menos de 1 minuto.
- **SC-002**: En pruebas con 10 usuarios confirmando el mismo bloque al mismo tiempo, exactamente 1 reserva se registra y los otros 9 reciben el mensaje de conflicto: 0 reservas duplicadas.
- **SC-003**: El 100 % de los intentos de reservar bloques pasados, de reservar sin sesión activa o de tener una segunda reserva activa son rechazados.
- **SC-004**: Una persona nueva puede registrarse e iniciar sesión en menos de 2 minutos.
- **SC-005**: La grilla de disponibilidad de una cancha y fecha se muestra en menos de 2 segundos y refleja una reserva o cancelación hecha por otro usuario en cuanto se vuelve a cargar.
- **SC-006**: El 90 % de los usuarios de prueba identifican correctamente en su primer intento qué bloques están disponibles y cuáles reservados.
- **SC-007**: Ningún mensaje mostrado al usuario contiene detalles técnicos internos (verificado en el 100 % de los casos de error probados).

## Assumptions

- **Zona horaria**: todas las fechas y horas (incluido qué se considera "pasado") se rigen por la hora local de Medellín, Colombia (UTC−5), donde están las canchas.
- **Reserva activa**: se entiende como una reserva no cancelada cuya hora de fin aún no ha pasado. Al terminar su bloque deja de ser activa y pasa al historial.
- **Acceso sin sesión**: una persona sin sesión puede ver el listado de las 5 canchas, pero no la grilla de disponibilidad ni reservar ("disponibilidad completa" requiere sesión).
- **Horizonte del calendario**: en esta iteración no se define un límite máximo de anticipación para reservar; se puede elegir cualquier fecha desde hoy en adelante.
- **Cancelación**: no hay antelación mínima; una reserva puede cancelarse en cualquier momento antes de su hora de inicio. Las reservas canceladas no aparecen en las listas de "Mis Reservas".
- **Cuentas**: no se incluye verificación de correo ni recuperación de contraseña, ya que requerirían notificaciones externas (fuera de alcance).
- **Idioma**: la interfaz y los mensajes están en español.
- **Canal**: aplicación web usada desde navegador de escritorio o móvil con conexión a internet.
- **Fuera de alcance** (Non-Goals): pagos integrados (se paga en el club), panel de administración de canchas, notificaciones externas (correo, SMS, WhatsApp), reservas de más de 1 hora en un solo paso (2 horas = 2 reservas, sujetas al límite de una reserva activa) y matchmaking o torneos.
