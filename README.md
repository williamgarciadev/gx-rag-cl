# gx-bt-rag-mcp

MCP server de retrieval semántico puro (sin LLM propio) sobre documentación de GeneXus y Bantotal, con una capa de validación estructurada que verifica sintaxis y firmas contra un catálogo SQLite.

Dos capas con garantías distintas:
- **Retrieval** (semántico + BM25 + reranker sobre ChromaDB) — mitiga la alucinación.
- **Validación** (`verify_syntax`, `list_methods`, `get_signature` sobre SQLite) — corta la alucinación con un veredicto contra un catálogo cerrado.

Todo local: zero cloud, zero API keys.

## Estado actual

En desarrollo. Iteración activa: **Fase 1 — caracterización del corpus y flujo de curación** (aún no hay `src/`, `pyproject.toml` ni servidor MCP).

## Documentación

- [`CLAUDE.md`](./CLAUDE.md) — reglas duras de desarrollo y método de trabajo por fases (gates).
- [`docs/gx-bt-rag-mcp-stack.md`](./docs/gx-bt-rag-mcp-stack.md) — stack tecnológico, tools MCP, esquema de la Syntax DB, estructura de carpetas.
- [`docs/gx-bt-rag-fase1-caracterizacion-corpus.md`](./docs/gx-bt-rag-fase1-caracterizacion-corpus.md) — qué se conoce de cada fuente del corpus, idempotencia de ingesta, curación asistida.
- [`docs/gx-bt-rag-punto-de-partida-desarrollo.md`](./docs/gx-bt-rag-punto-de-partida-desarrollo.md) — alcance de la iteración actual y decisiones pendientes de confirmar con el humano.
- [`docs/gx-bt-rag-ui-gestion-corpus.md`](./docs/gx-bt-rag-ui-gestion-corpus.md) — panel de administración del corpus (fase posterior).
