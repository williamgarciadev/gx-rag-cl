# Stack consolidado: `gx-bt-rag-mcp`

MCP server de **retrieval semántico puro** (sin LLM propio) sobre documentación de GeneXus y Bantotal, con una **capa de validación estructurada** que verifica sintaxis y firmas contra un catálogo, no solo contra similitud semántica.

> Documento de planeación. Refleja las decisiones tomadas en diseño, incluidas las tres correcciones sobre la versión previa del stack: (1) SDK oficial `mcp` en vez de FastMCP standalone, (2) ingesta desde XPZ como fuente del catálogo, (3) tablas de auditoría y de método faltante en la Syntax DB.

---

## 1. Principio rector

Dos capas con garantías distintas, y no deben confundirse:

- **Retrieval** (semántico + BM25 + reranking): devuelve *texto parecido* con un score. **Mitiga** la alucinación.
- **Validación** (`verify_syntax`, `list_methods`, `get_signature` sobre SQLite): devuelve un *veredicto* contra un catálogo cerrado. **Corta** la alucinación.

El diferenciador del proyecto es la segunda capa. El ecosistema resuelve la primera; la segunda se construye a medida.

---

## 2. Stack tecnológico

| Capa | Tecnología | Justificación |
|---|---|---|
| Lenguaje | Python 3.11+ | Ecosistema RAG maduro; el SDK MCP es Python |
| MCP Framework | **SDK oficial `mcp` (v1.x, `mcp[cli]`)** | Mantenido por Anthropic, en modo mantenimiento (solo fixes críticos): dependencia estable y auditable, preferida en entornos regulados. Para stdio local no se necesita nada del FastMCP standalone 3.x |
| Transporte | stdio (local) | Cliente = proceso hijo controlado; sin OAuth por conexión. Compatible con Claude, Cursor, Cline |
| Vector DB | ChromaDB (local, persistente) | Self-hosted, backend SQLite, gratuito, híbrido con BM25 |
| Embeddings | sentence-transformers + `intfloat/multilingual-e5-large` | Local, español/portugués nativo |
| Reranker | `cross-encoder/ms-marco-MiniLM-L-6-v2` | Local, mejora precisión del retrieval |
| Hybrid search | BM25 (`rank-bm25`) + dense + reranker | Precisión máxima |
| Syntax DB | SQLite (built-in) + SQLAlchemy | Estructurado, consultas rápidas para validación |
| Chunking | `langchain-text-splitters` (MarkdownHeaderTextSplitter) | Subpaquete ligero, aware de estructura MD. No arrastra LangChain completo |
| Scraper GX | `httpx` + `beautifulsoup4` + `html2text` | Async, wiki pública |
| PDF Parser BT | PyMuPDF + `pdf2image` + `pytesseract` (OCR opcional) | Texto + imágenes/diagramas |
| Ingesta XPZ | Skill propia de análisis XPZ | **Fuente más fiable del catálogo**: objetos reales de la KB, no solo la wiki |
| Config | `pydantic-settings` + `pyyaml` | Validación tipada |
| Testing | `pytest` + `pytest-asyncio` | Unitario + integración + grounding |
| Container | Docker + docker-compose | Reproducibilidad |
| Observabilidad | `structlog` | Logging estructurado para debug y auditoría/compliance |

> **Nota de versionado:** fijar versiones exactas al momento de instalar. Los números del SDK `mcp` cambian con frecuencia; confirmar en PyPI antes de congelar `pyproject.toml`.

---

## 3. Decisión de framework MCP: por qué el SDK oficial y no FastMCP standalone

Existen dos proyectos homónimos:

- **SDK oficial `mcp`** (organización modelcontextprotocol / Anthropic): implementación de referencia. Incluye una clase FastMCP 1.0 embebida como interfaz de alto nivel. La v1.x está en modo mantenimiento. En la v2.0 beta la clase embebida pasa a llamarse `MCPServer` para diferenciarse del proyecto homónimo.
- **FastMCP standalone 3.x** (Jeremiah Lowin / PrefectHQ): más features, comunidad amplia, pero evoluciona rápido (cambios de modelo de auth entre versiones).

Para un servidor **stdio local en banca**, la ventaja principal del standalone (OAuth de primera clase para HTTP remoto) no aplica. Se elige el SDK oficial por estabilidad y auditabilidad. Si en el futuro se requiere HTTP remoto, se reevalúa.

---

## 4. Colecciones ChromaDB

| Colección | Contenido | Fuente |
|---|---|---|
| `gx_docs` | Documentación wiki GeneXus | Scraper HTML → Markdown |
| `bt_docs` | Documentación core Bantotal | PDF → Markdown (OCR opcional) |
| `bt_apis` | Especificaciones de APIs Bantotal | PDF / spec |
| `gx_syntax` | Snippets de código GeneXus validados | Wiki + **XPZ de la KB** |
| `cross_ref` | Relaciones GX ↔ BT (objeto GX ↔ tabla Bantotal) | Skills XPZ + modelo de datos |

---

## 5. Tools MCP

### Retrieval
- `search_docs(query, collection, top_k=5, min_score, version?, snippet_mode=true)` — búsqueda híbrida con score threshold y **filtro por versión**.
- `get_document(doc_id)` — documento completo con URL fuente.
- `search_cross(query_gx, query_bt, top_k=3)` — búsqueda cruzada GX + BT.

### Validación (capa propia, sobre SQLite)
- `list_methods(object_type, version?)` — métodos/propiedades válidos de un objeto GX.
- `get_signature(method_name, object_type, version?)` — firma exacta: parámetros, tipos, retorno, versión.
- `verify_syntax(snippet, language="gx"|"bt_api", version?)` — veredicto **VÁLIDO / INVÁLIDO / NO_VERIFICABLE** contra la Syntax DB. Registra cada veredicto (auditoría) y encola los INVÁLIDO/NO_VERIFICABLE (método faltante).

### Ingesta
- `add_from_url(url, collection, metadata)` — scraper de URLs para wiki GX.

> **min_score:** no fijar 0.85 a ciegas. Calibrar el default con `evaluate_retrieval` sobre el corpus real en español antes de congelarlo.

---

## 6. Syntax DB (SQLite)

### Tablas de catálogo
- **`gx_syntax`**: `name`, `object_type`, `category`, `signature` (JSON), `description`, `version_since`, `source` (`wiki` | `xpz`), `source_doc_id`, `kb_scope` (global GeneXus vs. específico de la KB).
- **`bt_apis`**: `endpoint`, `method`, `path`, `params` (JSON), `request_body` (JSON), `responses` (JSON), `auth_type`, `source_doc_id`.

### Tablas de gobierno (añadidas respecto a la versión previa)
- **`validation_log`**: `timestamp`, `tool`, `input_snippet`, `verdict`, `matched_entry_id`, `version_queried`, `source`. Rastro auditable de cada veredicto — relevante para compliance.
- **`missing_methods`**: `token`, `object_type`, `first_seen`, `hit_count`, `status`. Cola de curaduría priorizada por frecuencia: convierte los falsos negativos en la lista de trabajo del catálogo.

> **Procedencia y versión en cada entrada:** un veredicto no dice solo "válido", dice "válido según `xpz` de la KB / `wiki` 2024, versión X". Distingue lo global de GeneXus de lo específico de tu KB, evitando falsos INVÁLIDO.

---

## 7. Fuentes de ingesta

| Fuente | Pipeline | Alimenta |
|---|---|---|
| Wiki GeneXus | httpx → BeautifulSoup → html2text → chunking MD → embeddings | `gx_docs`, parte de `gx_syntax` |
| PDF Bantotal | PyMuPDF / pdf2image → OCR (`spa+por`) → Markdown → chunking → embeddings | `bt_docs`, `bt_apis` |
| **XPZ de la KB** | Skill de análisis XPZ → objetos, firmas, tipos → Syntax DB | `gx_syntax` (fuente fiable), `cross_ref` |
| Modelo de datos Bantotal | Skill de modelo de datos → tablas ↔ objetos | `cross_ref` |

**Premisa clave:** la wiki no es la única fuente de verdad. Parte de la sintaxis válida es específica de la KB (Business Components, SDTs, patterns) y solo aparece en los XPZ. La documentación puede además estar deprecada respecto a la V3R1.11; por eso cada entrada lleva procedencia y versión.

---

## 8. Estructura de carpetas

```
gx-bt-rag-mcp/
├── pyproject.toml
├── docker-compose.yml
├── Dockerfile
├── config/
│   ├── config.yaml
│   └── mcp.json                # Claude Desktop / Cursor / Cline
│
├── src/
│   ├── __init__.py
│   ├── server.py               # Entry point (SDK oficial mcp)
│   ├── config.py               # pydantic-settings
│   │
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── search.py           # search_docs, search_cross
│   │   ├── retrieve.py         # get_document
│   │   ├── validate.py         # verify_syntax, list_methods, get_signature
│   │   └── ingest.py           # add_from_url
│   │
│   ├── vectorstore/
│   │   ├── __init__.py
│   │   ├── chroma_client.py
│   │   ├── embeddings.py       # multilingual-e5-large
│   │   ├── retriever.py        # hybrid search + reranker + filtro versión
│   │   └── collections.py
│   │
│   ├── syntax_db/
│   │   ├── __init__.py
│   │   ├── models.py           # gx_syntax, bt_apis, validation_log, missing_methods
│   │   ├── gx_repository.py
│   │   ├── bt_repository.py
│   │   └── audit_repository.py # validation_log, missing_methods
│   │
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── gx_scraper.py
│   │   ├── bt_pdf_parser.py
│   │   ├── xpz_ingestor.py     # usa skill XPZ → Syntax DB
│   │   ├── chunker.py
│   │   └── loader.py
│   │
│   └── prompts/
│       ├── gx_grounded_coder.py
│       └── bt_integrator.py
│
├── data/                       # gitignored
│   ├── chroma/
│   ├── syntax.db
│   └── raw/
│
├── scripts/
│   ├── ingest_gx.py
│   ├── ingest_bt.py
│   └── ingest_xpz.py           # ingesta del catálogo desde XPZ
│
└── tests/
    ├── test_tools.py
    ├── test_retrieval.py
    └── test_grounding.py       # benchmark anti-alucinación
```

---

## 9. `config.yaml` (referencia)

```yaml
server:
  name: "gx-bt-rag-mcp"
  transport: "stdio"

vectorstore:
  path: "./data/chroma"
  embedding_model: "intfloat/multilingual-e5-large"
  reranker_model: "cross-encoder/ms-marco-MiniLM-L-6-v2"
  chunk_size: 1000
  chunk_overlap: 200
  min_score: 0.85            # calibrar con evaluate_retrieval antes de fijar

collections:
  gx_docs:   { description: "Documentación wiki GeneXus", source_url_prefix: "https://wiki.genexus.com" }
  bt_docs:   { description: "Documentación core Bantotal" }
  bt_apis:   { description: "Especificaciones APIs Bantotal" }
  gx_syntax: { description: "Sintaxis validada GeneXus" }
  cross_ref: { description: "Relaciones GX ↔ BT" }

syntax_db:
  path: "./data/syntax.db"

ingestion:
  gx_wiki_base_url: "https://wiki.genexus.com"
  bt_pdf_path: "./data/raw/bt"
  xpz_path: "./data/raw/xpz"
  ocr_enabled: true
  ocr_lang: "spa+por"

versions:
  default: "gx18"            # filtra retrieval y validación por versión de proyecto
  supported: ["gx16", "gx18"]
```

---

## 10. Dependencias (`pyproject.toml`, referencia)

```toml
[project]
name = "gx-bt-rag-mcp"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "mcp[cli]",                       # SDK oficial — fijar versión al instalar
  "chromadb>=0.5.0",
  "sentence-transformers>=3.0",
  "rank-bm25>=0.2.2",
  "langchain-text-splitters>=0.2",
  "httpx>=0.27",
  "beautifulsoup4>=4.12",
  "html2text>=2024.2.26",
  "pymupdf>=1.24",
  "pdf2image>=1.17",
  "pytesseract>=0.3.10",
  "sqlalchemy>=2.0",
  "pydantic>=2.0",
  "pydantic-settings>=2.0",
  "pyyaml>=6.0",
  "structlog>=24.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0", "pytest-asyncio>=0.23", "ruff>=0.5", "mypy>=1.10"]
```

---

## 11. Decisiones descartadas

| Alternativa | Por qué no |
|---|---|
| FastMCP standalone 3.x como base | Evoluciona rápido; riesgo de continuidad en banca. Su ventaja (OAuth remoto) no aplica a stdio local |
| LangChain / LlamaIndex como base | Frameworks pesados; no son MCP servers. Solo se usa `langchain-text-splitters` |
| RAGFlow como base | Dockerizado con UI; no es un MCP server simple. Se toman ideas de parsing |
| Haystack como base | Enterprise, overkill. Se toman ideas de arquitectura |
| Pinecone / Weaviate | Cloud/SaaS; se requiere 100% local por privacidad bancaria |
| OpenAI embeddings | Cloud, API keys, costo |
| Ollama en el server | El LLM corre en el agente, no en el server; el server es puro retrieval |

---

## 12. Decisión abierta

**Arquitectura: monolito vs. dos servidores MCP.**
Un solo servidor (retrieval + validación juntos) es más simple. Dos servidores (retrieval reutilizado + capa de validación independiente) desacoplan la lógica auditable —la que da veredictos— del código de retrieval de terceros, y permiten versionar y certificar cada parte por separado. En contexto regulado, la segunda opción tiene ventajas de auditoría. Resolver antes de congelar la estructura definitiva.

---

## 13. Pendientes de calibración / verificación

- Fijar versión exacta del SDK `mcp` desde PyPI.
- Calibrar `min_score` con `evaluate_retrieval` sobre corpus real en español.
- Confirmar qué framework MCP usa realmente el fork base (knowledge-rag) en su `requirements.txt`.
- Definir criterios de "hecho" medibles para el benchmark de grounding (precisión/recall objetivo de `verify_syntax`).
