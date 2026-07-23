---
name: gxbt-build
description: Claim the next agent-ready GitHub issue in gx-rag-cl, implement it, and open a PR — or fix review feedback on an existing PR. Use when asked to run gxbt-loop's builder, work the approved queue, or fix gxbt-review feedback. Designed for /loop; one pass does one unit of work.
---

# gxbt-build

Adaptado de `finna/Finn-loop` (`skills/finn-build`, MIT, Alex Finn 2026): mismo ciclo, sin Linear (issues y labels en **GitHub**), y con un guardrail propio de este repo (paso 4.5) que hace cumplir el gate de Fase 1 de `CLAUDE.md`.

Una pasada = una unidad de trabajo: arreglar el feedback de review de un PR existente, o construir un issue de punta a punta. Bajo `/loop`, cada iteración corre esta skill una vez.

Este entorno puede no tener `gh` CLI (p. ej. Claude Code on the web). Si no está disponible, usar las tools `mcp__github__*` equivalentes (`list_issues`, `issue_write`, `pull_request_read`, `create_pull_request`, etc.) en vez de los comandos `gh` de ejemplo.

## 0. Preflight

Antes de tocar GitHub, ramas o archivos:

- Confirmar que este es el repo correcto (`williamgarciadev/gx-rag-cl`) y que `origin` es alcanzable.
- Detectar la rama por defecto real (`gh repo view --json defaultBranchRef` o `mcp__github__get_me`/lectura del repo); no asumir que es `main`.
- Exigir árbol de trabajo limpio (`git status --porcelain` vacío). Si está sucio, reportar los paths y terminar la pasada. Nunca hacer stash, reset, sobrescribir o commitear trabajo ajeno.

## 1. Feedback de review primero

Listar PRs abiertos con la etiqueta `loop-changes-requested`. Saltar los que tengan `needs-human-review` — salieron de la cola automática hasta que un humano lo resuelva.

Si hay alguno, elegir el actualizado menos recientemente. Leer el issue de GitHub vinculado (`Closes #N` en el body del PR) y el último veredicto `gxbt-review of COMMIT_SHA`. Hacer checkout de su rama, arreglar solo los ítems "Must fix before merge", correr los checks relevantes, pushear, quitar `loop-changes-requested`, comentar qué se cambió. Terminar la pasada.

Si un fix propuesto cruza un non-goal del issue o requiere una decisión de producto, no implementarlo. Comentar el conflicto exacto, agregar `needs-human-review`, quitar `loop-changes-requested`, terminar la pasada.

## 2. Elegir

Listar issues de GitHub que cumplan todo:
- etiqueta `agent-ready`
- sin asignar
- sin etiqueta `blocked`
- sin relación de bloqueo sin resolver

Ordenar por prioridad, luego por más antiguo primero. Si la cola está vacía, decirlo y terminar la pasada. No inventar trabajo ni tomar un issue bloqueado.

## 3. Reclamar (lock cooperativo)

Autoasignarse el issue y moverlo a "In Progress" (o equivalente) si el repo usa ese estado. Reclamar antes de leer a fondo o escribir código. Volver a leer el issue justo después; si está bloqueado, asignado a otra persona, o ya no es `agent-ready`, no trabajarlo y volver al paso 2.

## 4. Leer

Traer el issue completo con comentarios. Implementar solo sus criterios de aceptación. Los non-goals son innegociables. Comparar cada `AC-N` contra cada `NG-N` antes de editar. Sin cambios no relacionados ni refactors oportunistas.

Si un criterio es ambiguo, choca con un non-goal, o depende de un bloqueo sin resolver, ir al paso 8. Nunca adivinar.

## 4.5. Gate de Fase 1 (regla propia de este repo)

Si el issue implica escribir o modificar código bajo `src/`, `scripts/ingest_*.py`, o cualquier código de servidor MCP o de ingestor:

1. Leer `docs/caracterizacion-corpus.md` §1 (tabla de estado por fuente).
2. Si **cualquiera** de las cuatro fuentes oficiales de Fase 1 (Manual Instalador, Manual de Usuario, Wiki, XPZ) sigue en 🔴 o 🟡, **no implementar**. Ir al paso 8 (Bloqueado) citando textualmente la regla de `CLAUDE.md`: *"No avanzar a Fase 1 (esqueleto MCP) ni posteriores sin cerrar el gate de caracterización del corpus."*
3. Si el issue es de documentación, caracterización o curación de corpus (no toca `src/` ni ingestores), este paso no aplica — seguir normalmente.

Este chequeo no lo tiene el Finn-loop original porque es específico de las reglas de fase de este proyecto.

## 5. Construir

- Traer la última rama por defecto de `origin` y crear o retomar una rama `gh-N-slug-corto`, usando el número real del issue.
- Implementar los criterios de aceptación con el estilo, arquitectura y nomenclatura ya existentes en el repo.
- Agregar o actualizar tests cuando el cambio afecte lógica, flujo de datos, permisos o comportamiento visible.
- Preservar el comportamiento fuera del contrato del issue.

## 6. Verificar

Correr lint, typecheck, build y los tests más acotados que apliquen (`ruff check`, `mypy`, `pytest -q` según `CLAUDE.md`, cuando ya exista `src/`). Todos los checks atribuibles a este cambio deben pasar antes de abrir PR. Revisar `git diff` y `git status` antes de publicar — detenerse si el diff trae trabajo no relacionado o secretos generados.

## 7. Publicar

Pushear y abrir un PR. La descripción debe incluir:
- Qué cambió y por qué
- `Closes #N`, con el número real del issue de GitHub
- Un ledger de alcance: una línea de evidencia por `AC-N`, una línea de preservación por `NG-N`, y `Other behavior changes: None`
- Pasos de test manual numerados, acordes a lo realmente construido
- Checks automatizados corridos y su resultado
- Riesgo: Bajo / Medio / Alto

Si `Other behavior changes: None` no es cierto, detenerse y pedir que se ajuste el issue antes de abrir el PR.

Comentar la URL del PR en el issue. Nunca mergear ni activar auto-merge. Terminar la pasada.

## 8. Bloqueado

Comentar una pregunta específica que un humano pueda responder de forma asíncrona, aplicar la etiqueta `blocked`, y desasignarse. Dejar `agent-ready` puesto: la consulta de selección excluye explícitamente `blocked`, así que el issue vuelve a aparecer solo después de que un humano responda y quite esa etiqueta.

Nunca usar "esto no está claro" como la pregunta. Indicar la decisión exacta, las opciones disponibles, y qué criterio de aceptación afecta. Terminar la pasada para que la siguiente iteración tome otro trabajo.
