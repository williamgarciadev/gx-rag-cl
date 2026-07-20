# Caracterización del corpus — hallazgos

Entregable de la Fase 1 (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §8). A diferencia de los documentos de planeación en este mismo directorio, **este documento contiene datos extraídos**, no el plan para conocerlos. Se actualiza de forma incremental a medida que llegan las fuentes que faltan; no se cierra el gate de Fase 1 hasta que las cuatro fuentes oficiales tengan evidencia real (ver tabla de estado).

## 1. Estado por fuente

| Fuente | Estado | Motivo |
|---|---|---|
| Manual Instalador (PDF, 334p) | 🔴 Pendiente | No disponible en este entorno. Usuario lo subirá. Parcialmente corroborado por evidencia indirecta real (§3). |
| Manual de Usuario (PDF) | 🔴 Pendiente | No disponible en este entorno. Usuario lo subirá. |
| Wiki GeneXus (2 artículos de prueba) | 🟡 Bloqueada | Esta sesión no tiene salida de red permitida hacia `docs.genexus.com`/`wiki.genexus.com` (ver §5). Usuario subirá el HTML local de las dos páginas de prueba. |
| XPZ de la KB | 🔴 Pendiente | No hay ningún `.xpz` en este entorno. Usuario lo subirá. Herramientas de parseo ya listas (ver §6). |
| **Modelo de Datos Bantotal (MDU-99000)** | 🟢 Confirmada | Fuente real disponible en este entorno (skill `bantotal-model-docs`), caracterizada en §2. No es una de las cuatro fuentes oficiales de Fase 1, pero alimenta las mismas dos capas (catálogo + `cross_ref`) y aporta evidencia real para la tabla de prefijos. |
| **Referencias Rápidas Bantotal (documento interno del usuario)** | 🟢 Confirmada | Subido por el usuario a esta sesión (`Bantotal_Rapidas.md`, generado desde `Rapidas_2.txt`). No es una de las cuatro fuentes oficiales, pero es evidencia real de producción (SQL Server) — caracterizada en §3. Contiene datos operativos sensibles; ver nota de sensibilidad en §3.5. |

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

## 3. Fuente confirmada: Referencias Rápidas Bantotal (documento interno del usuario)

**Procedencia:** `Bantotal_Rapidas.md`, subido directamente por el usuario a esta sesión. El propio documento se declara "generado a partir de `Rapidas_2.txt` — Bantotal Core Bancario": una recopilación operativa interna (no un manual oficial), con 30 secciones temáticas (geografía, usuarios, personas/clientes, cuentas, préstamos, plan de pagos, tasas, contabilidad, garantías, seguros, CDT, ahorros, ACH, servicios web BTI, cadena de cierre, reportes regulatorios, CIFIN, carpeta digital, control dual, Formiik, autorizaciones, impuestos, metas comerciales, corresponsal, caja, accesos rápidos, tablas de parámetros globales, queries analíticos, debug). No se copia al repo (ver §3.5).

**Naturaleza distinta a las demás fuentes:** no es un manual redactado para lectores (como el Manual Instalador/Usuario o el Modelo de Datos), sino consultas SQL Server reales + tablas de referencia + catálogos de programas, tal como los usa un equipo de soporte/operaciones. Es la fuente con la relación señal/ruido más alta vista hasta ahora para `cross_ref` y para firmas de programas, pero también la que menos se parece a "prosa explicativa" — casi no aporta a `bt_docs`.

### 3.1 Tipología de contenido

Casi enteramente estructurado/catalogable, muy poca prosa:
- **Bloques SQL** (`SELECT`/`JOIN`/`UPDATE`/`INSERT`) sobre tablas Bantotal reales — cada `JOIN` es una relación `cross_ref` ya expresada en código, no en prosa.
- **Tablas de referencia cortas** (código → significado): estados, tipos de cliente, tipos de evento, códigos de garantía — candidatas a tablas de dominio/enum en el catálogo SQLite, no a chunks de retrieval.
- **Catálogos de programas** (nombre de programa → función) repetidos por sección temática — coinciden en formato con la tabla de prefijos de programas de la Fase 1.
- **Procedimientos manuales narrados** (ACH, Corresponsal, ajuste de tasas) — la única prosa real del documento, y la que contiene los datos operativos sensibles (§3.5).

### 3.2 Patrones de identificación

- **Confirma con evidencia real la regla H = panel / P = proceso** de `docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §6: decenas de programas `H*` en las tablas "Programas de..." de cada sección (paneles de mantenimiento/consulta), y programas `P*` en contextos de proceso batch/contabilización (`PW103`, `PP006`, `PRG0010B`, `PJCCA123`–`PJCCA126`, `PDECO310`, `PDECO604`, `PDP09100`).
- **Confirma el prefijo `PDP`** de la tabla preliminar (`PDP09100`, en la sección de CDT — depósitos a plazo), que hasta ahora solo tenía respaldo de la exploración inicial del Manual Instalador, no de una segunda fuente.
- **Firmas de procedimientos extraíbles de texto, sin XPZ**: el documento cita literalmente la firma de llamada de varios procedimientos de contabilización, en sintaxis GeneXus `Call()`, ej. `PP006 Call(&Programa, &Pgcod, &Itsuc, &Itmod, &Ittran, &Itnrel, &Pmncod)`. Es evidencia directa de que **parte de `gx_syntax` para objetos Bantotal es extraíble de documentación operativa en texto plano**, no solo de XPZ — algo que no estaba confirmado en la Fase 1 original.
- **Familias de prefijos de tabla nuevas**, más allá de las `FS*` ya vistas en §2 — ver tabla consolidada en §4.

### 3.3 Formato físico

Markdown ya limpio (texto nativo, sin ruido de maquetación de PDF/wiki) — es el único formato de las fuentes vistas hasta ahora que llega prácticamente listo para curación mínima. No aplica la pregunta de OCR.

### 3.4 Relaciones explícitas (→ `cross_ref`)

Cada bloque `JOIN` del documento es, literalmente, una relación `cross_ref` ya resuelta. Ejemplos representativos (sin los valores de filtro específicos, ver §3.5):
- `FBC205 ↔ FBC206 ↔ FST811 ↔ FST001` (jerarquía Región → Zona → Oficina)
- `FSD001 ↔ FSD002/003/004` (persona → detalle físico/jurídico/institución financiera), `↔ FSR008` (persona → cuenta), `↔ SNGC60` (persona → actividad económica)
- `FSD010 ↔ FSD611` vía los 9 campos base con prefijos `AO`/`PP` (préstamo ↔ seguros del plan de pago) — confirma en un caso nuevo el patrón de 9 campos de §2.2
- `FSD011 ↔ FST111` (saldo de préstamo → módulo del sistema)
- `AUT000 ↔ FST039` (excepción por tasa/monto ↔ código de excepción)
- `FSH015 ↔ FSH016` y su par en línea `FSD015 ↔ FSD016` (cabezal/detalle contable, histórico y vigente respectivamente — mismo patrón estructural en dos capas temporales)
- `XWFD01/02/05/06/07/08/09` (documentos ↔ versiones ↔ instancias ↔ personas ↔ cuentas ↔ operación — Carpeta Digital)
- `BTI004 ↔ BTI012 ↔ BTI014 ↔ BTI019 ↔ BTI025/026` (servicio → canal → método → parámetros → SDT — Servicios Web Bantotal)

### 3.5 Nota de sensibilidad — a decidir con el humano, no se cierra sola

**Hallazgo que hay que traer al usuario, no una fuente más para catalogar sin más.** El documento mezcla, en las secciones narrativas (ACH §14, Corresponsal §25, y varios ejemplos de `SELECT`/`UPDATE` con filtros literales), **datos operativos que parecen reales**: IPs internas, una URL interna con hostname, nombres de personas (compañeros de operaciones citados por nombre de pila), y números de documento/cuenta usados como valores de filtro en los ejemplos SQL. Ninguno de esos valores se reproduce en este documento de caracterización ni se copiará al repo.

Esto no es solo un detalle de curación — condiciona cómo se debe tratar esta fuente en el flujo de curación asistida (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §5): antes de que cualquier parte de este documento entre a un artefacto Markdown curado o al catálogo, hay que decidir con el humano si:
1. Se cura conservando solo la estructura (nombres de tabla, joins, catálogos de programas, tablas código→significado) y se **descartan** los procedimientos narrados con datos operativos específicos, o
2. Se cura completo pero con esos valores **redactados/anonimizados** antes de convertir a artefacto versionado.

No se asume ninguna de las dos — se deja igual que las demás decisiones abiertas de `docs/gx-bt-rag-punto-de-partida-desarrollo.md` §4: se plantea, no se cierra sola.

---

## 4. Tabla de prefijos — consolidada (parcial, en progreso)

Amplía la tabla preliminar de `docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §6 con evidencia de las dos fuentes confirmadas (§2 y §3). Las filas de programas/paneles/rutinas del Manual Instalador pasan de "solo exploración preliminar" a **parcialmente corroboradas** por la fuente de §3 (evidencia real de una segunda fuente independiente, aunque no del Manual Instalador mismo); siguen pendientes de re-validación completa cuando llegue ese PDF.

| Prefijo | Tipo de objeto | Ejemplos reales | Tabla destino | Procedencia |
|---|---|---|---|---|
| `FST` | Parametrización / sistema | FST017, FST001, FST717, FST028, FST013, FST069, FST068, FST003, FST110, FST111, FST005, FST024, FST034, FST039 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal + rapidas-bantotal |
| `FSD` | Datos (transaccional/operativa) | FSD001–FSD004, FSD005/006, FSD008/009, FSD010, FSD011, FSD012, FSD014, FSD016, FSD601/602 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal + rapidas-bantotal |
| `FSR` | Relación | FSR002, FSR003, FSR004, FSR005, FSR006, FSR008, FSR111 | catálogo tablas + `cross_ref` | ✅ modelo-datos-bantotal + rapidas-bantotal |
| `FSE` | Extensión (datos adicionales especializados) | FSE001, FSE002, FSE012, FSE046, FSE071, FSE111, FSE201, FSE508 | catálogo tablas + `cross_ref` | ✅ ambas fuentes |
| `FSH` | Histórica (auditoría/cambios) | FSH005, FSH010, FSH012, FSH013, FSH014, FSH015, FSH016, FSH017, FSH031, FSH104, FSH205 | catálogo tablas + `cross_ref` | ✅ ambas fuentes |
| `FSN` | Numerador / secuencia automática | FSN001, FSN002, FSN003, FSN999 | catálogo tablas + `cross_ref` | ✅ ambas fuentes |
| `FSX` | Texto (descripciones/comentarios) | FSX001 (correos), FSX015 (detalle anulación) | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **confirma la categoría que en §2 estaba sin ejemplo** |
| `FSI` | Reportes normativos / campos de información | FSI001–FSI011 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `FSA` | Auditoría contable (asientos desiguales) | FSA030 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `FSL` | Límites | FSL001 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `FBC` | Geografía comercial (regiones/zonas) | FBC205, FBC206 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `SNG` (+ subfamilia `SNGC`) | Usuarios, asesores, datos complementarios de personas | SNG001/002/021/039/057/120/415/912, SNGC11/13/20/31/32/33/60/70, SNGCP4 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `JCC` (+ subfamilias `JCCA`/`JCCN`/`JCCY`/`JCCI`/`JCCM`) | Cartera/calificación, solicitudes, metas, centinela, Formiik | JCCA01/02/12/52/60, JCCN52/53/54, JCCY12/13, JCCI02, JCCM70 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `FPP` | Plan de pagos: comisiones/otros conceptos, simulador | FPP002, FPP003, FPP015, FPP026, FPP028, FPP040, FPP065, FPP190 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `DECO` | Reportes regulatorios / info financiera / corresponsalía | DECO21, DECO50, DECO60–63, DECO850 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `MPE` | Canales / ACH | MPE001–MPE011, MPE020, MPE024 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `BTI` | Servicios Web Bantotal (integraciones) | BTI001, BTI004, BTI007, BTI012, BTI014, BTI019, BTI025, BTI026 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `AUT` / `AUM` | Autorizaciones y excepciones | AUT000–AUT0004, AUM000–AUM006 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `CTD` | Control Dual | CTD000, CTD001, CTD006, CTD007 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `XWF` / `XWFD` | Carpeta Digital / workflow documental | XWF060/063/065/069/700, XWFD01/02/05–09, XWFDE2 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `FRI` / `FRNG` / `FRTASKS` | Garantías reales, reglas de negocio, hilos de cadena de cierre | FRI101, FRNG49, FRTASKS | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `CLE` / `MBC` / `XCR` / `REP` / `CAP` | Canje, cajas, corresponsalía, reporteador, paralelización | CLE101, MBC004, XCR060, REP001–004, CAP003 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `MOI` | Cuentas de ahorro | MOI000, MOI001 | catálogo tablas + `cross_ref` | ✅ rapidas-bantotal — **nuevo** |
| `HCDT` | Panel / WebPanel del módulo CDT | HCDT0033, HCDT0048, HCDT0052, HCDT0001, HCDT0042 | `gx_syntax` (objeto Bantotal) | 🟡 exploración preliminar + convención H confirmada indirectamente en §3.2 |
| `PCDT` | Proceso / procedimiento del módulo CDT | PCDT0004, PCDT0011, PCDT0040, PCDT0043 | `gx_syntax` (objeto Bantotal) | 🟡 exploración preliminar + convención P confirmada indirectamente en §3.2 |
| `PNU` | Proceso batch | PNU00002 | `gx_syntax` (batch) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `PDP` | Proceso batch (depósitos a plazo) | PDP00001, PDP00004, **PDP09100** (confirmado en rapidas-bantotal, sección CDT) | `gx_syntax` (batch) | ✅ **confirmado por segunda fuente independiente** (§3.2) |
| `PRTE` / `PRTE…WEB` | Rutina de plazo fijo / variante web | PRTE310, PRTE256, PRTE006, PRTEPN11, PRTE310WEB | `gx_syntax` (rutina) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `HZ` / `HW` / `HCVC` | Utilitario / panel genérico | HZ999003, HZ999004, HW200, HW021, HCVCO001 | `gx_syntax` (utilitario) | 🔴 solo exploración preliminar — pendiente Manual Instalador |
| `GIK` | Nomenclatura de objetos GeneXus/KB (validación) | (por confirmar) | regla de validación | 🔴 pendiente XPZ |
| `X054xxx`, `RRCO`, `DP05xx`/`DP00500` | Series técnicas internas — categoría exacta por confirmar | X054007/010/011/023, RRCO03, DP0501, DP0502, DP00500 | catálogo tablas (tentativo) | 🟡 rapidas-bantotal — nombres reales, propósito aún no caracterizado con certeza |

**Reglas adicionales encontradas:**
- `FST003.XX` sub-codifica el módulo operacional (§2.2), ya documentado.
- El **patrón de 9 campos** (§2.2) se confirma en un segundo caso independiente en §3.4 (`FSD010`↔`FSD611`), reforzando que es una regla de extracción reutilizable y no una particularidad de un solo par de tablas.
- La convención de programas **H = panel, P = proceso** (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §6) queda **corroborada por una fuente independiente** del Manual Instalador — reduce el riesgo de que fuera una regularidad casual de la muestra original de 47 identificadores.

---

## 5. Hallazgo transversal: acceso de red bloqueado en esta sesión

Confirmado con dos dominios distintos, ambos con **403 de la política de egress**, no de los sitios en sí (`/root/.ccr/README.md`: *"Do not retry or route around it — report the blocked host"*):
- `docs.genexus.com` / `wiki.genexus.com` — bloquea las dos páginas de prueba de la wiki GeneXus.
- `docs.workwithplus.com` — bloquea el artículo de referencia de WorkWithPlus (`?1822,Column+Tags+-+Grid+Objects`) que el usuario pasó como URL.

`WebSearch` sí funciona (conectividad general correcta) — es un bloqueo por host específico, no un problema de red general.

**Implicación para fases posteriores:** cualquier scraper productivo (`src/ingestion/gx_scraper.py`, Fase 4) que necesite `docs.genexus.com`, `wiki.genexus.com` o `docs.workwithplus.com` debe correr en un entorno con esa salida habilitada, o la ingesta de esos dominios debe apoyarse en el modo "archivo local" que el pipeline unificado ya contempla (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §4.1). No es una decisión de arquitectura a resolver aquí.

**Camino elegido para la wiki:** el usuario subirá el HTML local de las dos páginas de prueba. El artículo de WorkWithPlus queda sin caracterizar por ahora — no es una de las cuatro fuentes oficiales de Fase 1; si el usuario quiere incorporarlo, aplica el mismo camino (HTML local).

---

## 6. Herramientas ya listas para las fuentes pendientes

No se han copiado al repo (siguen siendo activos de skill), pero quedan identificadas para cuando lleguen los archivos:

- **XPZ de la KB** → `bantotal-xpz-analyzer/scripts/parse_xpz.py` y `genexus-xpz-analyzer/scripts/xpz_parser.py`. El primero ya reconoce por GUID los tipos de objeto GeneXus relevantes (Transaction, Procedure, SDT, WebPanel, DataView, DataProvider, DataSelector, Enum/Domain, Image, Theme, StyleSheet) y soporta salida `markdown|json|table|mermaid`, listado (`--list`), inspección de un objeto (`--object`) y validación de nomenclatura GIK (`--validate-gik`) — cubre directamente el prefijo `GIK` pendiente de la tabla de §4.
- **Manual Instalador / Manual de Usuario (PDF)** → misma vía que se usó para caracterizar la fuente confirmada de §2: extracción de texto nativo (sin necesidad de OCR salvo que la evidencia real diga lo contrario al abrir el archivo).

---

## 7. Próximos pasos (bloqueados por archivos o por decisión humana, no por diseño)

1. Usuario sube HTML local de las dos páginas de prueba de wiki → caracterizar tipología/patrones/formato/relaciones de la fuente Wiki (§3 de `gx-bt-rag-fase1-caracterizacion-corpus.md`), validar clasificación por género (§5.6), producir los dos artefactos curados de muestra, y correr la prueba de idempotencia (reingestar dos veces, confirmar no-duplicado por `source_key`+`content_hash`).
2. Usuario sube el Manual de Usuario (PDF) → caracterizar como fuente funcional (semántica de negocio, transacciones por módulo).
3. Usuario sube el Manual Instalador (PDF, 334p) → re-validar con evidencia real las filas `HCDT`/`PCDT`/`PNU`/`PDP`/`PRTE`/`HZ`/`HW`/`HCVC` de la tabla de §4, hoy corroboradas solo parcialmente por una fuente distinta.
4. Usuario sube uno o más `.xpz` de la KB → caracterizar como fuente de firmas propias (§3 de `gx-bt-rag-fase1-caracterizacion-corpus.md`), correr `parse_xpz.py` / `xpz_parser.py`, y confirmar/descartar el prefijo `GIK`.
5. **Decisión humana pendiente (§3.5):** qué hacer con los datos operativos sensibles de "Referencias Rápidas Bantotal" antes de curarlo — descartar los procedimientos narrados o redactar los valores específicos. No se avanza a producir un artefacto curado de esta fuente hasta resolverlo.

El gate de Fase 1 (`docs/gx-bt-rag-fase1-caracterizacion-corpus.md` §8) se da por cumplido solo cuando los puntos 1–4 estén cerrados con evidencia real.
