---
name: gxbt-spec
description: Interview the user to turn a raw idea into a build-ready GitHub issue with acceptance criteria for gx-rag-cl. Use when asked to run gxbt-loop's spec skill, scope new work for this repo, or turn a request into an issue before building it.
---

# gxbt-spec

Adaptado de `finna/Finn-loop` (`skills/finn-spec`, MIT, Alex Finn 2026), sin Linear: este proyecto lleva el trabajo en **GitHub Issues**, no en Linear. El resto del proceso se conserva.

## Proceso

**1. Investigar primero.** Antes de preguntar nada, revisar `CLAUDE.md` y los documentos relevantes de `docs/` (especialmente `docs/gx-bt-rag-mcp-stack.md` y, si el pedido toca el corpus, `docs/caracterizacion-corpus.md`) para no preguntar algo que ya está decidido en el repo. Las preguntas deben ser sobre decisiones de producto reales, no sobre hechos que ya están en el código o en los docs.

**2. Entrevistar por rondas.** 1 a 4 preguntas por ronda, con opciones concretas y una recomendación primero. Enfocarse en las bifurcaciones reales: variaciones de comportamiento, límites de alcance, casos borde, implicancias de datos. Después de cada ronda, aplicar el test de confianza: ¿dos ingenieros que lean este spec construirían el mismo comportamiento? Seguir hasta pasar ese test; no hay límite de preguntas.

**3. Redactar el borrador** con estas secciones:
- Problem statement
- Acceptance Criteria (resultados observables y verificables, con etiquetas estables `AC-N`)
- Non-goals (exclusiones explícitas, etiquetas `NG-N`)
- Archivos relevantes, con justificación
- Expectativas de test
- Pasos de verificación

Dimensionar el issue a un día de trabajo de agente o menos. Trabajo más grande se parte en una cadena de issues chicos.

**Regla dura de este repo, no negociable:** si el issue implica escribir código de servidor o de ingestor (`src/`, `scripts/ingest_*.py` o equivalente) mientras el gate de Fase 1 (`docs/caracterizacion-corpus.md` §1) no está cerrado, el borrador debe decirlo explícitamente como Non-goal y limitarse a lo que sí se puede hacer en esta fase (caracterización, curación, documentación). No prometer en el spec algo que `gxbt-build` va a rechazar.

**4. Confirmar y archivar.** Mostrar el borrador completo para aprobación del usuario. Al confirmar, archivar el issue en GitHub (`mcp__github__issue_write` o `gh issue create` si `gh` está disponible en el entorno) y reportar número y URL.

## Reglas clave

- Nunca adivinar decisiones de producto — el usuario es el cerebro de producto.
- Nunca aplicar la etiqueta `agent-ready` — la aplica el humano después de revisar el issue archivado.
- Ningún criterio de aceptación puede requerir algo que un non-goal excluye.
