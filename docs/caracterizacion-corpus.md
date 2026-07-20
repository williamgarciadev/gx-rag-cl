# Caracterización del corpus — hallazgos

Entregable de la Fase 1 (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §8). A diferencia de los documentos de planeación en este mismo directorio, **este documento contiene datos extraídos**, no el plan para conocerlos. Se actualiza de forma incremental a medida que llegan las fuentes que faltan; no se cierra el gate de Fase 1 hasta que las cuatro fuentes oficiales tengan evidencia real (ver tabla de estado).

## 1. Estado por fuente

| Fuente | Estado | Motivo |
|---|---|---|
| Manual Instalador (PDF, 334p) | 🔴 Pendiente | No disponible en este entorno. Usuario lo subirá. |
| Manual de Usuario (PDF) | 🔴 Pendiente | No disponible en este entorno. Usuario lo subirá. |
| Wiki GeneXus (2 artículos de prueba) | 🟡 Bloqueada | Esta sesión no tiene salida de red permitida hacia `docs.genexus.com`/`wiki.genexus.com` (ver §4). Usuario subirá el HTML local de las dos páginas de prueba. |
| XPZ de la KB | 🔴 Pendiente | No hay ningún `.xpz` en este entorno. Usuario lo subirá. Herramientas de parseo ya listas (ver §5). |
| **Modelo de Datos Bantotal (MDU-99000)** | 🟢 Confirmada | Fuente real disponible en este entorno (skill `bantotal-model-docs`), caracterizada en §2. No es una de las cuatro fuentes oficiales de Fase 1, pero alimenta las mismas dos capas (catálogo + `cross_ref`) y aporta evidencia real para la tabla de prefijos. |

El gate de Fase 1 (cuatro fuentes oficiales caracterizadas) **no está cerrado**. Este documento registra el progreso posible con lo que hay disponible ahora mismo, sin inventar nada de lo pendiente.

---

## 2. Fuente confirmada: Modelo de Datos Bantotal (MDU-99000-GL-V3R1.11)

**Procedencia:** PDF oficial "Estructura del Modelo de Datos Bantotal" (De Larrobla & Asociados, 2001, actualizado 2025), 118 páginas, 80 tablas catalogadas. Disponible en este entorno vía la skill `bantotal-model-docs` (no está en este repo — activo de skill, licencia interna, no se copia al proyecto). Análisis 100% completado sobre las 118 páginas, según su propio índice de estado.

**Relación con las fuentes oficiales de Fase 1:** complementa pero **no sustituye** al Manual Instalador (334p) que sigue pendiente. El Manual Instalador describe programas/procesos (paneles `HCDT`/`PCDT`, batch `PNU`/`PDP`, rutinas `PRTE`); esta fuente describe el modelo de **tablas** (`FST`/`FSD`/`FSR`/...). Son dos vistas del mismo sistema de nomenclatura, no la misma fuente.

### 2.1 Tipología de contenido

Mixta, igual que se anticipó para el resto del corpus Bantotal:
- **Prosa arquitectónica** (fundamentos, motor GeneXus, filosofía de diseño, roles) → retrieval, `bt_docs`.
- **Contenido estructurado catalogable**: catálogo de tablas por prefijo, campos, tipos de dato, claves primarias/secundarias, relaciones explícitas → catálogo SQLite / `cross_ref`.

### 2.2 Patrones de identificación

- **Prefijo de tabla**: 3 letras + código numérico de 3 dígitos (`FST017`, `FSD011`, `FSR002`). Confirma y amplía el patrón que ya tenía el Manual Instalador para `FST`/`FSD`/`FSR`.
- **Sub-codificación de módulo operacional**: `FST003.XX` (ej. `FST003.20` Cuentas Corrientes, `FST003.30` Préstamos) — un nivel de granularidad no documentado todavía en la tabla de prefijos preliminar.
- **Patrón de nueve campos base** (hallazgo transversal, `Patron_9_Campos_Bantotal.md`): toda tabla transaccional repite los mismos 9 campos clave (`PGCOD`, `XXMOD`, `XXSUC`, `XXMDA`, `XXPAP`, `XXCTA`, `XXOPER`, `XXSBOP`, `XXTOPE`), donde `XX` es un prefijo de 2 letras que identifica el contexto (`AO`=Operaciones, `PP`=Pagos, `EV`=Eventos; `CC`/`CA`/`PF`/`DT`/`GR` quedan como hipótesis no confirmadas en el propio análisis). Esto es una regla de extracción reutilizable: permite reconocer relaciones entre tablas transaccionales **por convención de nombre de campo**, no solo por prosa explícita.

### 2.3 Formato físico

Texto nativo. El análisis fue producido con `pdf-processing-pro` sobre extracción directa de texto; no hay mención de OCR ni de figuras portadoras de información en las 118 páginas revisadas. Consistente con el hallazgo preliminar de la Fase 1 para el resto del corpus Bantotal (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §7): **no hay evidencia, en ninguna fuente Bantotal vista hasta ahora, que justifique introducir OCR**.

### 2.4 Relaciones explícitas (→ `cross_ref`)

Relaciones tabla↔tabla documentadas en prosa, extraíbles directamente de texto sin necesitar XPZ:
- `FSD011 ↔ FST005 / FST0010` (superordinada/subordinadas)
- `FST001 → FST017` (Sucursal → Empresa)
- `FST717 → FST017` (Árbol de empresas)
- `FST028 → FST001` (Calendarios → Sucursales)
- `FST013 ↔ FST069 ↔ FST068` (Países ↔ Regiones ↔ Localidades)
- `FST004 → FST003` (Tipos de Operación → Módulos), `FST111 → FST110 + FST003` (Módulos de Sistema)
- `FSD001` (Personas, maestro universal) `↔ FSD002/003/004` (Físicas/Jurídicas/Inst. Financieras), `↔ FSD005/006` (Domicilios), `↔ FSD008/009` (Cuentas/Grupos)
- `FST039 ↔ FSH010` (Códigos de excepción ↔ Registro histórico de excepciones)
- `FST005 ↔ FSH005` (Monedas maestras ↔ Cotizaciones diarias, patrón configuración-estática vs. histórico-dinámico que se repite: `FST034/035` ↔ transacciones, `FSD012 → FSE012/FSE111/FSR111`)

Además, el **patrón de 9 campos** (§2.2) es una segunda vía de extracción de relaciones: dos tablas comparten identidad operativa si sus campos `XXOPER` (con distinto prefijo de 2 letras) coinciden — visto en el ejemplo real `FSD010.AOOPER = FSD601.PPOPER`. Esto es una regla estructural, no solo relaciones citadas en prosa.

---

## 3. Tabla de prefijos — consolidada (parcial, en progreso)

Amplía la tabla preliminar de `docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §6. Las filas de tablas (`FST`/`FSD`/`FSR`/nuevas) quedan **validadas y ampliadas con evidencia real** de la fuente confirmada (§2); las filas de programas/paneles/rutinas siguen citando solo la exploración preliminar del Manual Instalador, que aún no está disponible en este entorno — se marcan como pendientes de re-validación cuando llegue ese PDF.

| Prefijo | Tipo de objeto | Ejemplos reales | Tabla destino | Procedencia |
|---|---|---|---|---|
| `FST` | Tabla de parametrización / sistema | FST017, FST001, FST717, FST028, FST013, FST069, FST068, FST003, FST110, FST111, FST005, FST024, FST034, FST039 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal |
| `FSD` | Tabla de datos (transaccional/operativa) | FSD001–FSD004, FSD005/006, FSD008/009, FSD010, FSD011, FSD012, FSD014, FSD016, FSD601/602 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal |
| `FSR` | Tabla de relación | FSR002, FSR003, FSR004, FSR005, FSR006, FSR008, FSR111 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal |
| `FSE` | Tabla de extensión (datos adicionales especializados) | FSE012 (Documentos), FSE111 (Cheques) | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal — **nuevo, no estaba en la tabla preliminar** |
| `FSH` | Tabla histórica (auditoría/cambios) | FSH005, FSH010, FSH013, FSH014, FSH015, FSH016, FSH017, FSH031, FSH205 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal — **nuevo, no estaba en la tabla preliminar** |
| `FSN` | Numerador / secuencia automática | FSN001, FSN002, FSN003 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal — **nuevo, no estaba en la tabla preliminar** |
| `FSX` | Tabla de texto (descripciones/comentarios) | (sin ejemplo numérico confirmado en lo leído) | catálogo tablas + `cross_ref` | 🟡 modelo-datos-bantotal (categoría citada, sin ejemplo aún) |
| `HCDT` | Panel / WebPanel del módulo CDT | HCDT0033, HCDT0048, HCDT0052, HCDT0001, HCDT0042 | `gx_syntax` (objeto Bantotal) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `PCDT` | Proceso / procedimiento del módulo CDT | PCDT0004, PCDT0011, PCDT0040, PCDT0043 | `gx_syntax` (objeto Bantotal) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `PNU` | Proceso batch | PNU00002 | `gx_syntax` (batch) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `PDP` | Proceso batch (depósitos a plazo) | PDP00001, PDP00004 | `gx_syntax` (batch) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `PRTE` / `PRTE…WEB` | Rutina de plazo fijo / variante web | PRTE310, PRTE256, PRTE006, PRTEPN11, PRTE310WEB | `gx_syntax` (rutina) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `HZ` / `HW` / `HCVC` | Utilitario / panel genérico | HZ999003, HZ999004, HW200, HW021, HCVCO001 | `gx_syntax` (utilitario) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `GIK` | Nomenclatura de objetos GeneXus/KB (validación) | (por confirmar) | regla de validación | 🔴 pendiente XPZ |

**Regla adicional encontrada (no era visible en la tabla preliminar):** dentro de `FST`, el campo `FST003.XX` sub-codifica el módulo operacional (ej. `.20` Cuentas Corrientes, `.21` Caja de Ahorros, `.22` Depósitos a Plazo Fijo, `.30` Préstamos, `.50` Cajas, `.75` Inversiones). Es una segunda capa de identificador dentro del prefijo de tabla, a tener en cuenta en el diseño del reconocedor de entidades.

---

## 4. Hallazgo transversal: acceso de red bloqueado a la wiki en esta sesión

Al intentar traer en vivo las dos páginas de prueba (`?24744,For+each+command` y `?26286,For+each+Optimizations`) vía `WebFetch`, ambas devolvieron `403`. Según `/root/.ccr/README.md`, un 403/407 del proxy de esta sesión significa **política de egress de la organización**, no un bloqueo de la wiki: *"Do not retry or route around it — report the blocked host"*. Confirmado con `WebSearch` (que sí funciona) — la conectividad general está bien, el host específico `docs.genexus.com` no está permitido para esta sesión.

**Implicación para fases posteriores:** el scraper productivo (`src/ingestion/gx_scraper.py`, Fase 4) necesita correr en un entorno con salida habilitada a `docs.genexus.com`/`wiki.genexus.com`, o la ingesta de wiki debe apoyarse en el modo "archivo local" que el pipeline unificado ya contempla (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §4.1). No es una decisión de arquitectura a resolver aquí — solo queda registrado como condicionante a validar cuando se diseñe el scraper real.

**Camino elegido:** el usuario subirá el HTML local de las dos páginas de prueba; se retoma su caracterización (tipología, clasificación por género, curación) en cuanto lleguen.

---

## 5. Herramientas ya listas para las fuentes pendientes

No se han copiado al repo (siguen siendo activos de skill), pero quedan identificadas para cuando lleguen los archivos:

- **XPZ de la KB** → `bantotal-xpz-analyzer/scripts/parse_xpz.py` y `genexus-xpz-analyzer/scripts/xpz_parser.py`. El primero ya reconoce por GUID los tipos de objeto GeneXus relevantes (Transaction, Procedure, SDT, WebPanel, DataView, DataProvider, DataSelector, Enum/Domain, Image, Theme, StyleSheet) y soporta salida `markdown|json|table|mermaid`, listado (`--list`), inspección de un objeto (`--object`) y validación de nomenclatura GIK (`--validate-gik`) — cubre directamente el prefijo `GIK` pendiente de la tabla de §3.
- **Manual Instalador / Manual de Usuario (PDF)** → misma vía que se usó para caracterizar la fuente confirmada de §2: extracción de texto nativo (sin necesidad de OCR salvo que la evidencia real diga lo contrario al abrir el archivo).

---

## 6. Próximos pasos (bloqueados por archivos, no por diseño)

1. Usuario sube HTML local de las dos páginas de prueba de wiki → caracterizar tipología/patrones/formato/relaciones de la fuente Wiki (§3 de `gx-bt-rag-fase1-caracterizacion-corpus.md`), validar clasificación por género (§5.6), producir los dos artefactos curados de muestra, y correr la prueba de idempotencia (reingestar dos veces, confirmar no-duplicado por `source_key`+`content_hash`).
2. Usuario sube el Manual de Usuario (PDF) → caracterizar como fuente funcional (semántica de negocio, transacciones por módulo).
3. Usuario sube el Manual Instalador (PDF, 334p) → re-validar con evidencia real las filas `HCDT`/`PCDT`/`PNU`/`PDP`/`PRTE`/`HZ`/`HW`/`HCVC` de la tabla de §3, hoy basadas solo en la exploración preliminar.
4. Usuario sube uno o más `.xpz` de la KB → caracterizar como fuente de firmas propias (§3 de `gx-bt-rag-fase1-caracterizacion-corpus.md`), correr `parse_xpz.py` / `xpz_parser.py`, y confirmar/descartar el prefijo `GIK`.

El gate de Fase 1 (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §8) se da por cumplido solo cuando estos cuatro puntos estén cerrados con evidencia real.
