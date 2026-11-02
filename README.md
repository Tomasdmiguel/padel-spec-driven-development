# Reservas de Pádel: Spec-Driven Development con IA

Aplicación web para reservar turnos en canchas de pádel, desarrollada con **Spec-Driven
Development (SDD)** y **desarrollo asistido por IA**, usando
[GitHub Spec Kit](https://github.com/github/spec-kit) y
[Claude Code](https://claude.com/claude-code).

## Estado actual

**Fase: constitución del proyecto.** Las reglas del proyecto ya están definidas. La especificación
y el código de la aplicación todavía no se han creado.

## Cómo se desarrolla

En SDD, la especificación se escribe antes que el código y es la fuente de la verdad:

1. **Constitución:** principios y reglas que no se negocian. ✅
2. **Especificación:** qué se construye y por qué.
3. **Plan:** cómo se construye técnicamente.
4. **Tareas:** trabajo ordenado por dependencias.
5. **Implementación:** el agente de IA construye solo lo documentado.

El punto de partida es la [constitución del proyecto](.specify/memory/constitution.md). Entre sus
reglas para el agente de IA:

- **Cero código sombra:** no se agregan funcionalidades que no estén en la especificación.
- **La spec manda:** ante una contradicción o una falla lógica, el agente frena y pide actualizar
  la especificación antes de tocar el código.

## Reglas de dominio

- Catálogo cerrado de 5 canchas: Laureles, El Poblado, Belén, Robledo y Envigado.
- Reservas en bloques de 1 hora, disponibles las 24 horas.
- Nunca dos reservas en la misma cancha y el mismo horario.
- Toda reserva requiere un usuario con sesión activa.

## Stack definido

- **Frontend:** React, TypeScript y Tailwind CSS.
- **Backend:** Node.js, Express y TypeScript.
- **Base de datos:** SQLite con better-sqlite3 o SQL puro.
