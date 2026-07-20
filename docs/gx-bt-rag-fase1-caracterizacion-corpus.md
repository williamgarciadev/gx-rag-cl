# Fase 1 — Caracterización del corpus

Documento de planeación. Define **qué** hay que conocer del corpus antes de diseñar el ingestor, el esquema del catálogo y las reglas de extracción. No contiene datos extraídos: es el plan para conocerlos.

> **Principio rector.** El diferenciador del proyecto es la capa de validación, y esa capa solo es tan buena como el catálogo que la alimenta. El catálogo solo es tan bueno como la comprensión de qué se puede extraer y de dónde. Por eso conocer el corpus es la Fase 1, y todo lo demás cuelga de ella. Si esta fase se hace bien, el ingestor casi se escribe solo, porque las reglas salen del inventario. Si se salta, el ingestor es adivinanza.

---

## 1. Objetivo de la fase

Producir un **documento de caracterización del corpus**: por cada fuente, una descripción estructurada de su contenido, sus patrones de identificación, su formato físico y sus relaciones extraíbles. Ese documento es el input directo para las fases siguientes (diseño del esquema SQLite y de las reglas del ingestor).

Esta fase reemplaza al "empezar por el esqueleto MCP" como verdadera Fase 1. No se escribe código de servidor hasta cerrarla.

---

## 2. Qué hay que conocer de cada fuente

Para cada fuente, responder cuatro preguntas:

1. **Tipología de contenido.** Qué contiene y cómo se reparte entre las dos capas: prosa (va a retrieval, `bt_docs`/`gx_docs`) vs. contenido estructurado (identificadores, firmas, relaciones → catálogo SQLite).
2. **Patrones de identificación.** Qué prefijos y convenciones de nomenclatura existen. Estos patrones *son* las reglas de extracción del futuro ingestor.
3. **Formato físico.** Texto nativo vs. escaneado; con o sin figuras portadoras de información. Decide si se necesita OCR.
4. **Relaciones explícitas.** Vínculos que el texto ya trae (programa↔tabla, transacción↔módulo) y que alimentan `cross_ref`.

---

## 3. Fuentes a caracterizar

| Fuente | Naturaleza | Método de exploración | Aporta principalmente a |
|---|---|---|---|
| Manual Instalador (PDF) | Técnico: programas, procesos, tablas, pasos de instalación, relaciones | Lectura de texto nativo (PyMuPDF/pdftotext) | `gx_syntax` (objetos Bantotal), `cross_ref` |
| Manual de Usuario (PDF) | Funcional: semántica de negocio, transacciones por módulo, flujos | Lectura de texto nativo | `bt_docs` (retrieval) |
| Wiki GeneXus | Lenguaje y objetos GeneXus base (transacciones, propiedades, métodos) | Skill `genexus-wiki-search` | `gx_syntax` (lenguaje), `gx_docs` |
| XPZ de la KB | Firmas reales de los objetos propios de la KB (BC, SDT, procedures) | Skill de análisis XPZ | `gx_syntax` (KB propia), `cross_ref` |

**Nota sobre las tres fuentes de sintaxis.** El catálogo se nutre de tres orígenes complementarios, no de uno:
- La **wiki** da el lenguaje GeneXus genérico (común a cualquier KB).
- El **Manual Instalador** da los objetos y procesos específicos de Bantotal.
- Los **XPZ** dan lo específico de *esta* KB, que ni la wiki ni los manuales conocen (métodos de un BC concreto, patterns aplicados).

Cada entrada del catálogo debe llevar su procedencia para distinguir estos tres niveles.

---

## 4. Ingesta unificada e idempotencia (requisito transversal)

Requisito que atraviesa todas las fuentes: la ingesta debe aceptar **entrada multiformato por un pipeline único** y **no duplicar** al reingestar.

### 4.1 Entrada multiformato, pipeline único

Tres formas de entrada convergen en el mismo flujo:
- **URL** (p. ej. `https://docs.genexus.com/en/wiki?45280,...`) → descarga → HTML.
- **HTML** (archivo local).
- **Markdown** (archivo local).

Todas se **normalizan a un formato interno común** (Markdown con metadata de encabezados) antes de trocear y embeber. Un solo chunker y un solo cargador para las tres, no tres caminos paralelos.

### 4.2 Identidad combinada: fuente + hash

Estrategia de deduplicación acordada: **fuente como clave, hash para detectar cambios**.

- **Clave de fuente** (`source_key`): la URL canónica normalizada o la ruta del archivo. Es la identidad estable del documento. Reingestar la misma fuente **actualiza** (upsert), no inserta de nuevo.
- **Hash de contenido** (`content_hash`): hash del texto normalizado. Detecta si el contenido cambió respecto a la última ingesta de esa misma fuente.

**Normalización de dominios de la wiki.** Decisión tomada: `docs.genexus.com` y `wiki.genexus.com` son dos hosts de la misma documentación GeneXus y se normalizan al **mismo `source_key`** cuando apuntan al mismo artículo (mismo identificador de página). Evita duplicar una página que exista en ambos dominios. El normalizador de URL debe reducir ambos hosts a una forma canónica única antes de calcular la clave.

Lógica de decisión en cada ingesta:

| Situación | `source_key` | `content_hash` | Acción |
|---|---|---|---|
| Fuente nueva | no existe | — | Insertar |
| Reingesta sin cambios | existe | igual | No-op (saltar) |
| Reingesta con cambios | existe | distinto | Reemplazar chunks de esa fuente |
| Mismo contenido, otra fuente | distinta | igual | Registrar (posible duplicado semántico; ver 4.4) |

Los IDs de chunk en ChromaDB se derivan de `(source_key + posición)` para que el upsert sea determinista: la segunda ingesta pisa los mismos IDs en vez de crear nuevos.

### 4.3 Alcance: página única

Para una URL —incluso si es un índice o tabla de contenidos (ToC)— la ingesta procesa **solo la página dada**. No sigue enlaces (no crawl). Si se quiere ingestar páginas enlazadas, se pasan como URLs individuales. Decisión tomada para mantener la ingesta predecible y evitar arrastre incontrolado de contenido.

### 4.4 Fuera de alcance de esta identidad

La deduplicación por fuente+hash resuelve el duplicado de *ingesta*. **No** resuelve el duplicado *semántico* en retrieval (dos documentos distintos que describen el mismo objeto). Eso se maneja en la capa de recuperación (dedup por score/fuente al devolver resultados) y es un problema separado, no de la ingesta.

---

## 5. Curación del corpus (el PDF no entra crudo)

Principio: **el PDF crudo es un mal insumo para un agente IA**. Trae ruido de maquetación (encabezados y pies repetidos por página, copyright, numeración, texto de figuras entremezclado, saltos de línea arbitrarios). Ingestar eso directo contamina el retrieval y hace que el agente razone sobre basura. La curación convierte el PDF en **conocimiento**: limpio, estructurado, con identificadores y relaciones explícitos. Este es el paso que de verdad aporta conocimiento al agente.

Parsear ≠ curar. Parsear extrae el texto; curar es lo que viene después.

### 5.1 Qué hace la curación

- Elimina ruido repetido (encabezados, pies, copyright, números de página).
- Reconstruye la estructura lógica (secciones y jerarquía a partir del índice).
- Normaliza los identificadores al patrón de prefijos (ver §6).
- Marca las relaciones explícitas programa↔tabla para `cross_ref`.
- Separa prosa (→ retrieval, `bt_docs`) de contenido catalogable (→ Syntax DB).

### 5.2 Enfoque: curación asistida

Decisión tomada: **reglas automáticas producen un borrador de Markdown limpio; un humano valida los puntos críticos** —sobre todo el catálogo de identificadores y las relaciones—. Ni puramente automática (deja errores en documentos irregulares) ni puramente manual (no escala a cientos de páginas). El agente IA puede *ayudar a curar* (proponer limpieza, extraer identificadores, marcar relaciones); la validación de lo crítico es humana.

### 5.3 El resultado curado es un artefacto versionado

Decisión tomada: el **Markdown curado se conserva como artefacto propio**, revisable y versionable, **antes** de ingestar. No es un paso interno efímero del pipeline. Beneficios: se puede revisar, versionar, y re-ingestar sin re-curar; y desacopla la curación (lenta, con validación humana) de la ingesta (rápida, repetible).

El artefacto curado es, además, la entrada del pipeline unificado de §4: es el "formato interno común" (Markdown con metadata) para la fuente PDF. El PDF no entra al pipeline; entra su Markdown curado.

### 5.4 La curación preserva la procedencia

Innegociable en banca: lo curado **conserva la trazabilidad al original** (qué PDF, qué sección, qué página). Curar no puede significar perder el vínculo con la fuente —poder afirmar "esto salió de la página 39 del Manual Instalador" es parte del valor y de la auditabilidad. Cada bloque curado lleva su referencia al origen, que se propaga como metadata hasta la atribución de fuente en el retrieval.

### 5.5 Nivel de curación por fuente

La curación no es uniforme; cada fuente necesita un grado distinto:

| Fuente | Nivel de curación | Motivo |
|---|---|---|
| PDF Bantotal | Fuerte | Ruido de maquetación, figuras, estructura a reconstruir |
| Wiki GeneXus | Media | Quitar navegación/UI, anclar al dominio, resolver homónimos |
| XPZ de la KB | Baja | Ya son estructurados; poca limpieza |

### 5.6 Curación de la wiki GeneXus: clasificación por género

La wiki mezcla dos géneros de artículo con destino distinto, y la curación debe **clasificarlos automáticamente y enrutarlos**:

| Género de artículo | Ejemplo (URL de prueba) | Destino | Qué se extrae |
|---|---|---|---|
| Referencia (comando/objeto/propiedad) | `For each command` (?24744) | `gx_syntax` (catálogo) | Sintaxis, cláusulas, parámetros → firma |
| Guía / optimización / concepto | `For each Optimizations` (?26286) | `gx_docs` (retrieval) | Prosa explicativa completa |

Decisión tomada: **clasificación automática por género**. Reglas detectan si el artículo es referencia o guía (por estructura, títulos, presencia de bloques de sintaxis) y enrutan a la capa correspondiente. Un curador que trate ambos igual desperdicia el artículo de referencia (no extrae la firma) o ensucia el catálogo con prosa.

Ruido propio de la wiki (distinto al del PDF): no es maquetación sino **navegación web** —menús, breadcrumbs, "edit this page", barras laterales, enlaces de pie—. La curación media se queda con el cuerpo del artículo y descarta el cascarón, más el anclaje al dominio GeneXus para resolver homónimos (transaction/property/method colisionan con Microsoft/COM+/lógica formal).

---

## 6. Tabla de prefijos identificados (punto de partida)

Taxonomía preliminar derivada de la exploración del Manual Instalador (334 páginas, texto nativo, 47 identificadores únicos detectados en una primera pasada). Es el punto de partida de las reglas de extracción, **sujeta a validación y ampliación** durante la fase.


| Prefijo | Tipo de objeto inferido | Ejemplos observados | Tabla destino |
|---|---|---|---|
| `FST` | Tabla de parametrización / sistema | FST500, FST098, FST198, FST038, FST040 | catálogo tablas + `cross_ref` |
| `FSD` | Tabla de datos | FSD851, FSD850, FSD380, FSD016 | catálogo tablas + `cross_ref` |
| `FSR` | Tabla de relación | FSR011 | catálogo tablas + `cross_ref` |
| `HCDT` | Panel / WebPanel del módulo CDT (prefijo H) | HCDT0033, HCDT0048, HCDT0052, HCDT0001, HCDT0042 | `gx_syntax` (objeto Bantotal) |
| `PCDT` | Proceso / procedimiento del módulo CDT (prefijo P) | PCDT0004, PCDT0011, PCDT0040, PCDT0043 | `gx_syntax` (objeto Bantotal) |
| `PNU` | Proceso batch | PNU00002 | `gx_syntax` (batch) |
| `PDP` | Proceso batch (depósitos a plazo) | PDP00001, PDP00004 | `gx_syntax` (batch) |
| `PRTE` | Rutina de plazo fijo | PRTE310, PRTE256, PRTE006, PRTEPN11 | `gx_syntax` (rutina) |
| `PRTE…WEB` | Variante web de rutina | PRTE310WEB | `gx_syntax` (rutina) |
| `HZ` | Utilitario / panel genérico | HZ999003, HZ999004 | `gx_syntax` (utilitario) |
| `HW` | Panel genérico | HW200, HW021 | `gx_syntax` (utilitario) |
| `HCVC` | Panel de archivo plano chequeras/certificados | HCVCO001 | `gx_syntax` (utilitario) |
| `GIK` | Nomenclatura de objetos GeneXus/KB (validación) | (por confirmar) | regla de validación |

**Lectura de la taxonomía:** el prefijo `H` denota paneles/WebPanels; el prefijo `P` denota procesos/procedimientos. El segundo bloque (CDT, RTE…) denota el módulo o dominio funcional. El sufijo numérico identifica el objeto concreto. Esta regla H/P + dominio + número es la base del reconocedor de entidades del ingestor.

---

## 7. Hallazgos preliminares que condicionan fases posteriores

Evidencia ya recogida en la exploración inicial, a confirmar y completar durante la fase:

- **Los manuales Bantotal tienen texto nativo**, no escaneado. → Probablemente **no se necesita OCR pesado** (tipo Chandra) para el corpus Bantotal. Decisión de OCR queda condicionada a confirmar el resto de fuentes.
- **El Manual Instalador trae relaciones programa↔tabla explícitas** (p. ej. un proceso que modifica una tabla concreta). → `cross_ref` es poblable desde el propio texto, no solo desde XPZ.
- **La wiki de GeneXus es prosa semi-estructurada** (una propiedad/objeto por artículo, con campos razonablemente consistentes), no una especificación formal. → La extracción de firmas desde la wiki requiere un parser que entienda el patrón de artículo, no scraping genérico.
- **Los términos GeneXus tienen homónimos** (transaction/property/method colisionan con Microsoft, COM+, lógica formal). → El retrieval y la ingesta deben anclarse al dominio de la fuente para evitar ruido.

---

## 8. Entregable de la fase y criterio de "hecho"

**Entregable:** documento de caracterización del corpus con, por cada fuente, sus cuatro dimensiones (tipología, patrones, formato, relaciones) y la tabla de prefijos validada y ampliada.

**Criterio de "hecho":**
- Las cuatro fuentes están caracterizadas (no solo el Manual Instalador).
- La tabla de prefijos cubre los identificadores reales encontrados y cada prefijo tiene tipo y tabla destino asignados.
- Está decidido, con evidencia, si se necesita OCR y para qué fuentes.
- Está documentado qué relaciones `cross_ref` son extraíbles de texto y cuáles requieren XPZ.
- Está definida la clave de identidad (`source_key` + `content_hash`) y validado, con un caso real (p. ej. reingestar la URL de prueba dos veces), que la reingesta no duplica.
- Está definido el flujo de curación asistida y producido al menos un artefacto Markdown curado de muestra (p. ej. una sección del Manual Instalador) que conserve procedencia al original.

Solo al cumplirse esto se pasa a la fase de diseño del esquema SQLite y las reglas del ingestor.

---

## 9. Lo que esta fase NO hace

- No extrae aún el catálogo completo de identificadores (eso es producto de la fase de ingesta, no de caracterización).
- No escribe código de servidor MCP ni de ingestor.
- No fija el esquema SQLite definitivo (depende del resultado de esta fase).
- No cierra la decisión monolito vs. dos servidores (pendiente en el `.md` del stack, sección 12).
