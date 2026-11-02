# Reservas de Pádel Constitution

## Core Principles

### I. Catálogo Cerrado de Canchas

El sistema DEBE ofrecer exactamente estas 5 canchas: Cancha Laureles, Cancha El Poblado,
Cancha Belén, Cancha Robledo y Cancha Envigado. NO DEBE aceptar reservas para ninguna otra cancha.

### II. Bloques de Una Hora

Toda reserva DEBE durar exactamente 1 hora. Los bloques DEBEN estar disponibles las 24 horas
del día.

### III. Prevención de Colisiones (NO NEGOCIABLE)

El sistema NO DEBE guardar una reserva sin validar antes que la cancha esté libre en ese bloque.
NO DEBEN existir dos reservas para la misma cancha en el mismo horario.

### IV. Autenticación Obligatoria

Toda reserva DEBE realizarla un usuario con sesión activa. El backend DEBE rechazar con 401 los
intentos de reserva sin sesión activa.

### V. Errores Amigables y HTTP Semántico

- La UI NO DEBE exponer errores crudos ni stack traces. Todo error técnico DEBE traducirse a un
  mensaje amigable, por ejemplo: "La cancha ya fue reservada en este horario".
- El backend DEBE usar códigos HTTP semánticos: 400 para petición inválida, 401 para usuario no
  autenticado y 409 para conflicto de reserva.

### VI. Cero Código Sombra

El agente de IA DEBE construir estrictamente lo documentado en `spec.md`. NO DEBE agregar
funcionalidades "por si acaso".

### VII. Spec como Fuente de la Verdad

Si una instrucción contradice esta constitución o existe una falla lógica, el agente de IA DEBE
frenar, avisar y pedir que se actualice `spec.md` antes de tocar el código.

## Alcance y Stack Tecnológico

El proyecto es una aplicación web para reservar turnos en canchas de pádel, en bloques de 1 hora.

- **Frontend**: DEBE usar React, TypeScript y Tailwind CSS.
- **Backend**: DEBE usar Node.js con Express y TypeScript.
- **Base de datos**: DEBE usar SQLite local, en el archivo `padel.db`.
- **Acceso a datos**: DEBE usar better-sqlite3 o SQL puro. NO DEBE usar ORMs pesados.

## Estructura y Estilo de Código

- **Estructura**: DEBE ser simple y plana: `/frontend`, `/backend` y `/db`. NO DEBE usar Clean
  Architecture ni patrones complejos.
- **Estilo**: DEBE seguir la programación funcional; en React, componentes funcionales y hooks.
  NO DEBE usar clases salvo que sea obligatorio.
- **Nomenclatura**: DEBE usar camelCase para funciones y variables, y PascalCase para interfaces
  y tipos.

## Governance

- **Precedencia**: esta constitución prevalece sobre `spec.md`, los planes y las tareas. Ante una
  contradicción se aplica el Principio VII.
- **Enmiendas**: toda enmienda DEBE ser aprobada de forma explícita por el responsable del
  proyecto y registrarse en este archivo, actualizando la versión y la fecha de última enmienda.
- **Versionado**: se aplica versionado semántico. MAJOR para eliminar o redefinir principios de
  forma incompatible; MINOR para agregar principios o secciones, o ampliar materialmente una
  regla; PATCH para aclaraciones y correcciones sin cambio semántico.
- **Cumplimiento**: cada plan, conjunto de tareas e implementación DEBE verificarse contra esta
  constitución antes de darse por terminado.

**Version**: 1.0.0 | **Ratified**: 2026-11-02 | **Last Amended**: 2026-11-02
