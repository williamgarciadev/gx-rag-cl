# UI de gestión del corpus

Documento de planeación. Define una interfaz de administración para **gestionar los documentos ingestados** (no para consultar el RAG). Justificada por la escala del corpus (~5.400 HTML de wiki + manuales Bantotal + otros), donde gestionar por línea de comandos es inviable.

> **Distinción clave.** Esto NO es una UI de consulta —Claude Code y los clientes MCP ya consultan el RAG—. Es un **panel de administración del corpus**: ver, buscar, curar, re-ingestar, aprobar. Es gestión de datos, no chat.

---

## 1. Principio de frontera

La UI es un **cliente** del sistema, no parte del servidor MCP. Se comunica con el servidor de retrieval a través de su API; no vive dentro de él. Esto preserva el principio de "servidor puro retrieval" y evita que el servidor se hinche.

**Riesgo a vigilar:** las UIs se comen los proyectos. Mantenerla deliberadamente simple. Un CRUD honesto cubre el 90% del valor; el resto es tentación. Es **fase posterior**, no Fase 1.

---

## 2. Alcance (decisión tomada: los tres niveles)

### 2.1 CRUD de documentos
- **Listar** documentos ingestados (con paginación —hay miles).
- **Ver detalle** de un documento: fuente (`source_key`), procedencia, capa destino (retrieval/catálogo), género (referencia/guía), fecha de ingesta, `content_hash`.
- **Buscar / filtrar** por fuente, tipo, estado, género, texto.
- **Re-ingestar** un documento (dispara curación + upsert idempotente).
- **Eliminar** un documento del corpus.

### 2.2 Estado de curación y errores
- Estado de cada documento: crudo / curado-borrador / curado-validado / ingestado / **fallido**.
- **Bandeja de errores de ingesta**: qué falló, por qué, con enlace al origen para reintentar.
- Para el corpus grande (5.400 HTML): vista de progreso del batch y de la muestra de validación (gate 6a de la Fase 1).
- Marcado de documentos que la clasificación automática dejó como **dudosos** (para revisión por excepción).

### 2.3 Bandeja de aprobación + métricas
- **Bandeja de aprobación del crecimiento controlado**: sugerencias de fuentes nuevas (del feedback loop de método faltante) para aprobar / rechazar / ajustar. Registro de rechazos para no re-sugerir.
- **Aprobación de entradas al catálogo** (`gx_syntax`): la compuerta humana que el resto de documentos exige vive aquí, operativamente.
- **Métricas de uso**: qué se consulta más, tasa de VÁLIDO/INVÁLIDO/NO_VERIFICABLE, cobertura del catálogo, cola de método faltante priorizada.

---

## 3. Cómo se conecta con lo ya diseñado

| Función de la UI | Se apoya en (documento) |
|---|---|
| Ver `source_key`, `content_hash`, re-ingesta idempotente | Fase 1, §4 (idempotencia) |
| Estado de curación, validado/borrador | Fase 1, §5 (curación asistida) |
| Género referencia/guía, capa destino | Fase 1, §5.6 (clasificación wiki) |
| Bandeja de aprobación de fuentes nuevas | Crecimiento controlado (todo el doc) |
| Aprobación de entradas a catálogo | Crecimiento controlado, §4; Calidad, §1-2 |
| Métricas: cola de método faltante, cobertura | Calidad, §8 y §10 |

La UI no inventa conceptos: es la **cara operativa** de mecanismos ya definidos. Casi todo lo que muestra o acciona ya existe en el diseño; la UI lo hace usable.

---

## 4. Stack sugerido (a confirmar)

- **[HIPÓTESIS]** UI web ligera sobre la API del servidor de retrieval, siguiendo las guías de diseño frontend propias del usuario. Sin framework pesado si no hace falta.
- **[ABIERTA]** ¿La API que consume la UI la expone el propio servidor MCP (además de stdio) o un servicio aparte? Conecta con la decisión monolito vs. dos servidores (stack, §12).
- **[ABIERTA]** ¿Autenticación? Aunque hoy sea un usuario, en banca el acceso a gestión suele requerir control. A definir según entorno.

---

## 5. Qué NO debe ser esta UI

- No una interfaz de chat / consulta del RAG (ya la dan Claude Code y clientes MCP).
- No parte del servidor MCP (es cliente de su API).
- No un panel con features infinitas: CRUD + estado + aprobación + métricas, y parar ahí.
- No Fase 1: se construye después de que el corpus y la ingesta batch funcionen.

---

## 6. Preguntas abiertas

- **[ABIERTA]** ¿La UI permite editar directamente un artefacto curado, o solo re-disparar la curación? (Editar a mano rompe la reproducibilidad; preferible re-curar.)
- **[ABIERTA]** ¿Cómo se visualiza la procedencia hasta la línea/página del original?
- **[ABIERTA]** ¿La eliminación es física o lógica (soft-delete con historial, mejor para auditoría en banca)?
- **[ABIERTA]** ¿Vista de diff entre dos versiones de un artefacto curado (por el versionado de §5 de Fase 1)?
