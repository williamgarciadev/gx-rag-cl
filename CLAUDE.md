# gx-bt-rag-mcp

MCP server de retrieval semántico puro (sin LLM propio) sobre documentación de GeneXus y Bantotal, con una capa de validación estructurada que verifica sintaxis y firmas contra un catálogo SQLite.

Especificación completa: @docs/gx-bt-rag-mcp-stack.md
Caracterización del corpus (Fase 1, plan): @docs/gx-bt-rag-fase1-caracterizacion-corpus.md
Caracterización del corpus (Fase 1, hallazgos — entregable vivo): @docs/caracterizacion-corpus.md
Punto de partida de esta iteración de desarrollo: @docs/gx-bt-rag-punto-de-partida-desarrollo.md
UI de gestión del corpus (fase posterior): @docs/gx-bt-rag-ui-gestion-corpus.md
Repo de referencia (solo lectura, NO copiar código): `~/referencia/knowledge-rag`

<!-- Este archivo son reglas operativas, no documentación. El porqué de cada decisión está en el spec. Mantener bajo ~200 líneas. -->

## Estado actual

No existe código fuente todavía: no hay `src/`, `pyproject.toml` ni entorno Python creados. La iteración activa es **Fase 1: caracterización del corpus** (ver `gx-bt-rag-punto-de-partida-desarrollo.md`), no el esqueleto del servidor MCP. Progreso real en `docs/caracterizacion-corpus.md`: dos fuentes confirmadas con evidencia (Modelo de Datos Bantotal, Referencias Rápidas Bantotal), las cuatro fuentes oficiales de Fase 1 (Manual Instalador, Manual de Usuario, Wiki, XPZ) siguen pendientes de archivos reales. Esta sesión no tiene salida de red hacia `docs.genexus.com`/`wiki.genexus.com`/`docs.workwithplus.com` (bloqueo de política de egress, no de los sitios). Hay una decisión de sensibilidad de datos pendiente con el humano (§3.5 del documento de hallazgos) antes de curar la fuente "Referencias Rápidas".

## Comandos

```bash
# Entorno
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# Calidad (correr antes de dar una fase por terminada)
ruff check src/ tests/
mypy src/
pytest -q

# Servidor (stdio)
python -m src.server

# Ingesta
python scripts/ingest_gx.py     # wiki GeneXus
python scripts/ingest_bt.py     # PDFs Bantotal
python scripts/ingest_xpz.py    # catálogo desde XPZ (fuente fiable)
```

## Reglas duras (no negociables)

- **Framework MCP: SDK oficial `mcp` (paquete `mcp[cli]`), NO FastMCP standalone.** Elegido por estabilidad y auditabilidad en banca.
- **Versiones: consultar PyPI antes de fijar. NUNCA inventar un número de versión de memoria.**
- **No inventar métodos, propiedades ni firmas de GeneXus/Bantotal.** El catálogo (SQLite) es la fuente de verdad. Si algo no está en el catálogo, es `NO_VERIFICABLE`, no se asume.
- **Dos capas con garantías distintas, no mezclar:**
  - Retrieval (semántico + BM25 + reranker) → devuelve texto con score. Mitiga alucinación.
  - Validación (`verify_syntax`, `list_methods`, `get_signature` sobre SQLite) → devuelve veredicto. Corta alucinación.
- **Todo local. Zero cloud, zero API keys.** No introducir dependencias de servicios externos.
- **Cada entrada del catálogo lleva procedencia (`wiki` | `xpz`) y versión (`gx16` | `gx18`).** Un veredicto dice "válido según X, versión Y", no solo "válido".
- **El catálogo NO sale solo de la wiki.** Los XPZ de la KB son la fuente más fiable para objetos propios (BC, SDT, patterns). La wiki puede estar deprecada respecto a Bantotal V3R1.11.

## Método de trabajo: fases con gates

Construir por fases. No implementar todo de una vez. No avanzar de fase sin que su gate pase.

0. **Caracterización del corpus (Fase 1 activa)** — documento de caracterización de las cuatro fuentes (PDF Bantotal, Manual de Usuario, wiki GeneXus, XPZ de la KB), flujo de curación asistida, tabla de prefijos, e identidad `source_key`/`content_hash` validada con reingesta real.
   Gate: criterio de "hecho" de `gx-bt-rag-fase1-caracterizacion-corpus.md` §8. No se escribe código de servidor ni de ingestor hasta cerrarla.
1. **Esqueleto MCP** — `server.py` con SDK oficial + una tool trivial (`ping`).
   Gate: un cliente MCP (Claude Desktop / Cursor) ve la tool y responde.
2. **Retrieval** — `search_docs` sobre ChromaDB con unos pocos docs de prueba.
   Gate: `evaluate_retrieval` da métricas razonables sobre casos reales.
3. **Validación** — `verify_syntax`, `list_methods`, `get_signature` sobre SQLite.
   Gate: veredictos correctos contra casos GeneXus conocidos; sin falsos INVÁLIDO en métodos reales.
4. **Ingesta** — scrapers GX/BT + ingestor XPZ.
   Gate: cobertura del catálogo medible; catálogo poblado desde XPZ, no solo wiki.

Usar plan mode antes de ediciones grandes. Leer el diff antes de aceptar.

## Arquitectura y convenciones

- Estructura de carpetas: ver sección 8 del spec. Respetarla.
- `src/tools/` una tool por dominio (search, retrieve, validate, ingest).
- `src/syntax_db/models.py` incluye tablas de gobierno: `validation_log` (auditoría de cada veredicto) y `missing_methods` (cola de curaduría). No omitirlas.
- Embeddings: `intfloat/multilingual-e5-large` (es/pt). Cambiar el modelo obliga a reindexar todo.
- `min_score`: NO fijar 0.85 a ciegas. Calibrar con `evaluate_retrieval` sobre corpus en español.
- Logging estructurado con `structlog` en cada tool de validación.
- UI de gestión del corpus (`gx-bt-rag-ui-gestion-corpus.md`): cliente aparte de la API del servidor, no vive dentro del MCP server. Es fase posterior a la Fase 1.

## Decisiones abiertas (confirmar con el humano antes de asumir)

- **Monolito vs. dos servidores MCP** (retrieval + validación juntos vs. desacoplados). Determina toda la estructura de carpetas.
- **Esquema SQLite del catálogo.** No se fija hasta cerrar la Fase 1 de caracterización.
- **Necesidad y alcance de OCR.** Evidencia preliminar: PDFs Bantotal tienen texto nativo. Condicionado a confirmar el resto de fuentes.

## Qué NO hacer

- No usar LangChain/LlamaIndex como base. Solo `langchain-text-splitters` para chunking.
- No copiar código de knowledge-rag; leerlo como referencia y escribir limpio.
- No añadir dependencias cloud ni LLM en el servidor.
- No declarar una fase terminada porque el código compila: el gate es funcional.
- No avanzar a Fase 1 (esqueleto MCP) ni posteriores sin cerrar el gate de caracterización del corpus.
