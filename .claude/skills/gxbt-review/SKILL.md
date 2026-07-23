---
name: gxbt-review
description: Review open PRs in gx-rag-cl against their linked GitHub issue and required checks, then post a three-group verdict with gxbt-loop labels. Use when asked to run gxbt-loop's reviewer or review its PR queue. Designed for /loop; never merges or pushes code.
---

# gxbt-review

Adaptado de `finna/Finn-loop` (`skills/finn-review`, MIT, Alex Finn 2026): mismo proceso, issue vinculado y checks en **GitHub** en vez de Linear.

Una pasada = un PR revisado. Bajo `/loop`, cada iteración corre esta skill una vez.

Este entorno puede no tener `gh` CLI. Si no está disponible, usar las tools `mcp__github__*` equivalentes (`pull_request_read`, `list_pull_requests`, `issue_read`, `pull_request_review_write` con `add_comment_to_pending_review`/`submit_pending` no aplica aquí — este skill solo comenta y etiqueta, nunca aprueba formalmente, ver §5).

## 1. Encontrar un PR que necesite revisión

Listar PRs abiertos (no drafts). Para cada uno, buscar el último comentario cuya primera línea sea `gxbt-review of COMMIT_SHA`.

Saltar un PR cuando ese SHA registrado sea igual al `headRefOid` actual y ya tenga `loop-approved`, `loop-changes-requested`, o `needs-human-review`. Revisar de nuevo si llegaron commits después del SHA registrado. Si nada necesita revisión, decirlo y terminar la pasada.

## 2. Leer el contrato y el código

- Parsear el número de issue de `Closes #N` en el body del PR y traer el issue completo, con comentarios. Sin issue vinculado es un hallazgo must-fix.
- Leer el diff completo y cada archivo cambiado en contexto.
- Revisar solo contra el issue vinculado: huecos de acceptance criteria, defectos, flujo de datos roto, expansión de alcance innecesaria, problemas de seguridad, estados de carga/error faltantes, y código que a futuros agentes les va a costar modificar.
- No sugerir mejoras no relacionadas salvo que sean severas.

Todo hallazgo must-fix arranca con uno de:
- `[AC-N]` — el PR no satisface ese criterio de aceptación
- `[DEFECT]` — la implementación está rota dentro del alcance
- `[SECURITY]` — un problema de seguridad severo bloquea el merge
- `[CI]` — un check requerido de GitHub falló

Los non-goals son innegociables. Si arreglar un hallazgo requeriría comportamiento excluido por un `NG-N`, no prescribir código — registrar `[SCOPE-CONFLICT AC-N ↔ NG-N]` con la contradicción exacta y marcar el PR para escalamiento humano.

## 3. Chequear evidencia de merge

Inspeccionar el head actual del PR, mergeability, y los checks requeridos:
- Si los checks requeridos están pendientes o la mergeability es aún desconocida, reportar que el PR está esperando y terminar sin postear veredicto ni cambiar labels. Una pasada posterior lo reintenta.
- Checks requeridos fallidos son hallazgos must-fix `[CI]`.
- Un conflicto de merge es un hallazgo must-fix `[DEFECT]`.
- **Si el repo todavía no tiene checks requeridos configurados** (estado actual de `gx-rag-cl`, sin CI todavía), marcar el PR para escalamiento humano; no aplicar `loop-approved`. La ausencia de CI nunca se trata como verde.

Revisar el `headRefOid` exacto usado para esta evidencia. Volver a traerlo justo antes de postear. Si cambió, descartar la revisión y volver a empezar en una pasada futura.

## 4. Postear un veredicto

Postear un comentario con esta estructura:

```md
gxbt-review of COMMIT_SHA

CI: required checks passed | failed | not configured
Mergeability: clean | conflicting

## Review

Summary: una o dos oraciones en lenguaje simple sobre qué hace este PR.

## 1. Must fix before merge

None.

## 2. Should fix soon

None.

## 3. Safe to merge

Yes — automated review evidence is complete. A human still makes the merge decision.
```

Luego fijar labels según el veredicto, chequeando labels existentes antes de quitarlos para que un label ausente no falle el comando:
- Sin must-fix y sin escalamiento nuevo: agregar `loop-approved`; quitar `loop-changes-requested`. Preservar un `needs-human-review` preexistente porque puede representar un gate humano de alto riesgo separado.
- Con must-fix presente: agregar `loop-changes-requested`; quitar `loop-approved`.
- Conflicto de alcance o sin CI requerida: agregar `needs-human-review`; quitar tanto `loop-approved` como `loop-changes-requested`; poner "Safe to merge" en `No — human decision required.`

El camino de escalamiento deja la cola de reparación automática a propósito. Un humano debe resolver el motivo, cambiar el issue o la configuración del repo según corresponda, y quitar `needs-human-review` antes de que `gxbt-review` vuelva a revisar ese mismo commit.

## 5. Límites duros

- Nunca mergear ni activar auto-merge.
- Nunca pushear commits a la rama del PR.
- Nunca aprobar ni solicitar cambios vía review formal de GitHub. Usar un comentario más labels — el loop puede correr con el token del autor del PR y GitHub rechaza self-reviews.
- `loop-approved` es evidencia para un humano, no autorización de merge.
