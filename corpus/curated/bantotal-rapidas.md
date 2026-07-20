---
título: Bantotal — Consultas y Referencias Rápidas (curado)
fuente_original: Bantotal_Rapidas.md, subido por el usuario a esta sesión el 2026-07-20
procedencia_declarada_por_el_autor: "Documento generado a partir de `Rapidas_2.txt` — Bantotal Core Bancario"
tipo_fuente: documento interno de referencia operativa (no manual oficial)
nivel_de_curación: bajo — solo redacción de datos operativos sensibles; estructura, tablas, SQL y catálogos de programas se conservan íntegros
fecha_de_curación: 2026-07-20
decisión_de_curación: "Redactar y conservar todo" (confirmado por el humano el 2026-07-20, ver docs/caracterizacion-corpus.md §3.5)
caracterización: docs/caracterizacion-corpus.md §3
---

# Nota de curación

Este artefacto es la versión curada de `Bantotal_Rapidas.md` (subido directamente por el usuario, no está en `docs/` porque contenía datos operativos sin redactar). Se aplicó **solo redacción**, no reescritura ni resumen: todo el contenido original se conserva, incluida su numeración de secciones. Se reemplazaron por placeholders los siguientes tipos de dato, identificados en `docs/caracterizacion-corpus.md` §3.5:

| Tipo de dato redactado | Placeholder | Seccion(es) afectadas |
|---|---|---|
| Nombres de personas (contactos internos) | `[contacto de negocio]` / `[operaciones]` | §8 Ajuste de Tasas, §14 Canales y ACH, §25 Corresponsal |
| IPs / URLs internas | `[URL interna de ...]` | §14 Canales y ACH |
| Números de documento de identificación usados como filtro de ejemplo | `<NRO_DOCUMENTO>` | §3 Personas y Clientes, §5 Préstamos y Cartera |
| Números de cuenta/operación usados como filtro de un caso real | `<NRO_CUENTA>` / `<NRO_OPERACION>` | §5 Préstamos y Cartera, §6 Plan de Pagos, §9 Contabilidad |

No se redactaron códigos de dominio de negocio (módulo, tipo de operación, país, moneda, estado), nombres de tabla, nombres de programa, ni nombres de oficina/ciudad usados como ejemplo de configuración (ej. altas de sucursal) — no identifican a una persona ni exponen infraestructura, y son justamente el contenido con valor de catálogo/`cross_ref` que esta fuente aporta (ver `docs/caracterizacion-corpus.md` §3.4).

---

# 📚 Bantotal — Consultas y Referencias Rápidas

> **Base:** SQL Server · **Sistema:** Bantotal Core Bancario
> **Scope:** Tablas, programas, guías de proceso, procedimientos operativos
> **Última revisión:** 2026

---

## Tabla de Contenido

1. [Geografía y Estructura Organizacional](#1-geografía-y-estructura-organizacional)
2. [Usuarios y Asesores](#2-usuarios-y-asesores)
3. [Personas y Clientes](#3-personas-y-clientes)
4. [Cuentas Cliente](#4-cuentas-cliente)
5. [Préstamos y Cartera](#5-préstamos-y-cartera)
6. [Plan de Pagos](#6-plan-de-pagos)
7. [Tasas e Índices](#7-tasas-e-índices)
8. [Ajuste de Tasas (Decreto 455)](#8-ajuste-de-tasas-decreto-455)
9. [Contabilidad](#9-contabilidad)
10. [Garantías](#10-garantías)
11. [Seguros](#11-seguros)
12. [CDT — Depósitos a Plazo](#12-cdt--depósitos-a-plazo)
13. [Cuentas de Ahorro](#13-cuentas-de-ahorro)
14. [Canales y ACH](#14-canales-y-ach)
15. [Servicios Web Bantotal (BTI)](#15-servicios-web-bantotal-bti)
16. [Cadena de Cierre](#16-cadena-de-cierre)
17. [Reportes y Regulatorio](#17-reportes-y-regulatorio)
18. [CIFIN — Centrales de Riesgo](#18-cifin--centrales-de-riesgo)
19. [Carpeta Digital](#19-carpeta-digital)
20. [Control Dual](#20-control-dual)
21. [Formiik](#21-formiik)
22. [Autorizaciones y Excepciones](#22-autorizaciones-y-excepciones)
23. [Impuestos y GMF](#23-impuestos-y-gmf)
24. [Metas Comerciales](#24-metas-comerciales)
25. [Corresponsal](#25-corresponsal)
26. [Operaciones de Caja](#26-operaciones-de-caja)
27. [Accesos Rápidos — Programas Bantotal](#27-accesos-rápidos--programas-bantotal)
28. [Tablas de Parámetros Globales](#28-tablas-de-parámetros-globales)
29. [Queries Analíticos Complejos](#29-queries-analíticos-complejos)
30. [Debug y Diagnóstico](#30-debug-y-diagnóstico)

---

## 1. Geografía y Estructura Organizacional

### Tablas maestras

| Tabla | Descripción |
|-------|-------------|
| `FBC205` | Regiones |
| `FBC206` | Zonas |
| `FST811` | Relación Zona ↔ Oficina |
| `FST001` | Sucursales / Oficinas |
| `FST198` | Guías Especiales de Proceso |

### Consulta jerárquica: Región → Zona → Oficina

```sql
-- Región base
SELECT TOP 10 * FROM dbo.FBC205
WHERE BC205Id1D IN ('Codigo Zona','Código Zona')
ORDER BY BC205Id1D;

-- Región + Zona
SELECT TOP 10 * FROM dbo.FBC205 RE
    INNER JOIN dbo.FBC206 ZO ON RE.BC205Emp = ZO.BC205Emp AND RE.BC205Cod = ZO.BC205Cod
WHERE RE.BC205Cod IN (811, 812, 813, 814, 815)
ORDER BY RE.BC205Id1D;

-- Jerarquía completa: Región → Zona → Oficina (versión ordenada)
SELECT
    RE.BC205Emp, RE.BC205Cod, RE.BC205Dsc,
    ZO.BC206Id1, ZO.BC206Chr1,
    OFI.Sucurs, OFI.Scnom
FROM dbo.FBC205 RE
    INNER JOIN dbo.FBC206 ZO  ON RE.BC205Emp = ZO.BC205Emp AND RE.BC205Cod = ZO.BC205Cod
    INNER JOIN dbo.FST811 RZO ON ZO.BC205Emp = RZO.Pgcod AND ZO.BC206Id1 = RZO.RegCod
    INNER JOIN dbo.FST001 OFI ON RZO.Pgcod = OFI.Pgcod AND RZO.OfiCod = OFI.Sucurs
WHERE RE.BC205Emp = 1 AND RE.BC205Cod IN (811, 812, 813, 814, 815)
ORDER BY RE.BC205Emp, RE.BC205Cod, ZO.BC206Id1, OFI.Scnom;

-- Jerarquía completa desde Guía Especial
SELECT
    RE.BC205Emp, RE.BC205Cod, RE.BC205Dsc,
    ZO.BC206Chr1, RZO.RegCod,
    OFI.Sucurs, OFI.Scnom, OFI.Scciud, OFI.Scdept
FROM dbo.FBC205 RE
    INNER JOIN dbo.FBC206 ZO  ON RE.BC205Emp = ZO.BC205Emp AND RE.BC205Cod = ZO.BC205Cod
    INNER JOIN dbo.FST811 RZO ON ZO.BC205Emp = RZO.Pgcod AND ZO.BC206Id1 = RZO.RegCod
    INNER JOIN dbo.FST001 OFI ON RZO.Pgcod = OFI.Pgcod AND RZO.OfiCod = OFI.Sucurs
    INNER JOIN dbo.FST198 GRE ON GRE.Tp1nro1 = RE.BC205Cod
        AND GRE.Tp1cod = 1 AND GRE.Tp1cod1 = 81013
        AND GRE.Tp1corr1 = 10 AND GRE.Tp1corr2 = 20 AND GRE.Tp1corr3 <> 0;
```

### Consulta LEFT JOIN (oficinas sin región asignada incluidas)

```sql
SELECT * FROM dbo.FST001 SUC
LEFT JOIN (
    SELECT ZO.BC205Emp, RE.BC205Cod, RE.BC205Dsc, ZO.BC206Id1, ZO.BC206Chr1, RZO.OfiCod
    FROM dbo.FBC205 RE
        INNER JOIN dbo.FBC206 ZO  ON RE.BC205Emp = ZO.BC205Emp AND RE.BC205Cod = ZO.BC205Cod
        INNER JOIN dbo.FST811 RZO ON ZO.BC205Emp = RZO.Pgcod AND ZO.BC206Id1 = RZO.RegCod
    WHERE RE.BC205Id1D IN ('Codigo Zona','Código Zona') AND ZO.BC205Emp = 1
) RZO ON RZO.BC205Emp = SUC.Pgcod AND RZO.OfiCod = SUC.Sucurs
WHERE SUC.Pgcod = 1;
```

### Parametrización de Códigos de Regiones

```sql
-- Guía Especial 81013: Códigos de región
SELECT * FROM dbo.FST198
WHERE Tp1cod = 1 AND Tp1cod1 = 81013
    AND Tp1corr1 = 10 AND Tp1corr2 = 20 AND Tp1corr3 <> 0;

-- Guía Especial 81016: Reportes por región (zonificación)
SELECT * FROM FST198 WHERE TP1COD1 = 81016 AND TP1CORR1 = 10;
-- tp1nro1: quien puede ver (1=Todo / 5=Solo Oficina)
-- 10 = Reporte de mora | 30 = Reporte comparativo de cartera
```

### Geografía Colombia

```sql
SELECT TOP 10 * FROM dbo.FST068 WHERE Pais = 169  -- Departamentos
SELECT TOP 10 * FROM dbo.FST013 WHERE Pais = 169  -- País
SELECT TOP 10 * FROM dbo.FST070 WHERE Pais = 169  -- Ciudades
SELECT TOP 10 * FROM dbo.DECO50 WHERE DECO50PAIS = 169  -- Centro Poblado
SELECT TOP 10 * FROM dbo.FST071 WHERE Fst071Pai = 169   -- Barrios
SELECT TOP 10 * FROM dbo.FSE071                         -- Sector Barrio
```

---

## 2. Usuarios y Asesores

```sql
SELECT TOP 10 * FROM dbo.SNGAS2                                    -- Código asesor
SELECT TOP 10 * FROM dbo.FST746                                    -- Nombre usuario asesor
SELECT TOP 10 * FROM dbo.FST046                                    -- Asesor sucursal / Módulos permitidos
SELECT TOP 10 * FROM dbo.FSE046                                    -- Correo usuario asesor
SELECT TOP 10 * FROM dbo.FSE046 WHERE Atributo = 'RELACIONCTA'    -- Cuenta del usuario
SELECT TOP 10 * FROM dbo.SNG057                                    -- Parametrización de cargos
SELECT TOP 10 * FROM dbo.JCCN22                                    -- Barrios asignados a asesores
SELECT TOP 10 * FROM dbo.FST846                                    -- Barra menús usuarios
SELECT TOP 10 * FROM dbo.PRFU00                                    -- Perfil de usuarios
SELECT TOP 10 * FROM dbo.FST047                                    -- Módulo-Usuarios
SELECT TOP 10 * FROM dbo.FST048                                    -- Transacción-Usuarios
SELECT TOP 10 * FROM dbo.FPP190                                    -- Relación préstamos con asesor
```

### Programas de administración de usuarios

| Programa | Función |
|----------|---------|
| `Hprf093` | Consulta permisos por módulo y transacción |
| `Hprf080` | Consulta programa donde entra |
| `Hprf075` | Consulta usuario programas |
| `Hprf072` | Consulta menú que programas tiene |
| `Hprf071` | Consulta programa en perfiles |
| `Hprf070` | Consulta perfiles que programas tiene |
| `Hstd0001` | Cambio de clave |
| `HPRF172` | Mantenimiento permisos por perfil/módulo/transacción |
| `HMNU001` | Mantenimiento de menús |
| `HTRT746` | Perfiles |
| `HPRF000` | Mantenimiento de perfiles |
| `hprf042` | Mantenimiento de perfiles - Usuarios |
| `hsngu02` | Panel asesores cuenta cliente / Suplentes |
| `hmbc009` | Habilitación/Inhabilitación de Usuarios Cajeros |

---

## 3. Personas y Clientes

### Maestro de personas

```sql
SELECT TOP 10 * FROM dbo.FSD001   -- Maestro de personas (Alta inconclusa: Pefbaj = '1830-01-01')
SELECT TOP 10 * FROM dbo.FSD002   -- Detalle personas natural/física (nombres, fecha nacimiento)
SELECT TOP 10 * FROM dbo.FSD003   -- Detalle personas jurídicas
SELECT TOP 10 * FROM dbo.FSR003   -- PJ - Representante Legal
SELECT TOP 10 * FROM dbo.FSD004   -- Detalle de empresas
SELECT TOP 10 * FROM dbo.FSD703   -- PJ extendido
SELECT TOP 10 * FROM dbo.FSD704   -- PJ extendido
```

### Datos complementarios

```sql
SELECT TOP 10 * FROM dbo.FSE001                    -- Extensión datos persona
SELECT TOP 10 * FROM dbo.FSE002                    -- Extensión datos persona / País: Nacionalidad
SELECT TOP 10 * FROM dbo.FSE201                    -- Documentos electrónicos
SELECT TOP 10 * FROM dbo.SNGC11                    -- Estrato / Lugar nacimiento / sngc11Dat1: Fecha expedición
SELECT TOP 10 * FROM dbo.SNGC11 WHERE sngc11Cmb2 = 1  -- Clientes marca PEP
SELECT TOP 10 * FROM dbo.SNGC81                    -- Clientes PEP - Información adicional
SELECT TOP 10 * FROM dbo.SNGC31                    -- Fecha expedición cédula
SELECT TOP 10 * FROM dbo.SNGC13 WHERE sngc13Corr = 1  -- Direcciones (Docod: 2=Personal / 4=Empresa)
SELECT TOP 10 * FROM dbo.SNGC32                    -- Direcciones por cuenta cliente
SELECT TOP 10 * FROM dbo.SNGC33                    -- Direcciones por cédula
SELECT TOP 10 * FROM dbo.FSD005                    -- Domicilios activos por personas
SELECT TOP 10 * FROM dbo.FSD006                    -- Domicilios activos por cuenta cliente
SELECT TOP 10 * FROM dbo.FSD212                    -- Categoría de clientes
SELECT TOP 10 * FROM dbo.FSR002 WHERE Rpccyg = 25  -- Relación personas con cónyuge
SELECT TOP 10 * FROM dbo.FSR008                    -- Integración Cuenta/Personas
SELECT TOP 10 * FROM dbo.FSE508                    -- Marcas cliente (FSE508Vic: Víctima)
SELECT TOP 10 * FROM dbo.DECO850                   -- Información financiera
```

### Actividad económica e información SARLAFT

```sql
SELECT TOP 10 * FROM dbo.SNGC60                                 -- Actividad CIIU
SELECT TOP 10 * FROM dbo.SNGC20 WHERE SNGC20CTA = 219159       -- Segmentación actual (campo SNGC20NM1)
SELECT TOP 10 * FROM dbo.SNGC70                                 -- Información adicional
-- sngc70Atr = 'HSNGCPF1_CHECK_1': Tratamiento datos
-- sngc70Atr = 'HSNGCPF1_CHECK_3': Autoriza envío correo
-- sngc70Atr = 'HSNGCPF1_CMBAUX4': Fondeador PN (val='3')
-- sngc70Atr = 'HSNGCPJ1_CHECK_1': Fondeador PJ (val='S')
SELECT TOP 10 * FROM dbo.SNG039  -- JOIN con SNGC70 (SNG039ValC = sngc70Val)
SELECT TOP 10 * FROM dbo.SNG036  -- JOIN con SNG039 (SNG036LtCo = SNG039LtCo)
```

### Correos electrónicos — consolidado en una línea

```sql
-- Versión STRING_AGG (SQL Server 2017+)
SELECT Pepais, Petdoc, Pendoc, Txcod,
    STRING_AGG(TRIM(Pextxt), ';')
FROM dbo.FSX001 WHERE Txcod = 0
GROUP BY Pepais, Petdoc, Pendoc, Txcod;

-- Versión STUFF + FOR XML (compatible versiones anteriores)
SELECT FX001.Pepais, FX001.Petdoc, FX001.Pendoc, FX001.Txcod,
    STUFF((
        SELECT TRIM(Pextxt), CASE WHEN Pextxt = '' THEN '' ELSE '|' END
        FROM dbo.FSX001 IFX001
        WHERE Pepais = 169 AND Txcod = 0
            AND IFX001.Pepais = FX001.Pepais
            AND FX001.Petdoc = IFX001.Petdoc
            AND IFX001.Pendoc = FX001.Pendoc
        FOR XML PATH('')
    ), 1, 0, '') CORREO
FROM dbo.FSX001 FX001
WHERE FX001.Pepais = 169 AND Txcod = 0
GROUP BY FX001.Pepais, FX001.Petdoc, FX001.Pendoc, FX001.Txcod;
```

### Tablas de referencia — Personas

```sql
SELECT TOP 10 * FROM dbo.FST014  -- Tipo de documento
SELECT TOP 10 * FROM dbo.FST020  -- Vínculos / Tipos de relaciones
SELECT TOP 10 * FROM dbo.FST023  -- Homologación géneros (sexo)
SELECT TOP 10 * FROM dbo.FST009  -- Estados civil
SELECT TOP 10 * FROM dbo.FST015  -- Tipos de domicilio
SELECT TOP 10 * FROM dbo.FST104  -- Sector
SELECT * FROM dbo.FST116         -- Códigos ocupación
SELECT * FROM dbo.FST114         -- Códigos nivel educativo
SELECT * FROM dbo.FST115         -- Códigos profesión
SELECT * FROM dbo.FST750         -- Códigos actividad económica
SELECT * FROM dbo.FST752         -- Códigos tipo actividad económica
SELECT * FROM dbo.SNGCP4         -- Tipos de identificación / longitud
SELECT TOP 10 * FROM dbo.FSR005  -- Teléfono personas
SELECT TOP 10 * FROM dbo.FSR006 WHERE Docod = 4 AND Doord = 1  -- Teléfono empresas
```

### Consulta completa de persona natural

```sql
SELECT DISTINCT
    B.Pftdoc AS TIPO_DOCUMENTO,
    A.PENDOC AS NUMERO_DOCUMENTO,
    A.Pepais AS PAIS_RESIDENCIA,
    G.sngc11Dpto AS DEP_NACIMIENTO,
    P.DepNom AS DEPARTAMENTO_NAC,
    G.SNGC11PROV AS CIUDAD_NACIMIENTO,
    O.LocNom AS NOMBRE_NACIM,
    B.Pfpnac AS PAIS_NACIMIENTO,
    G.sngc11Dat1 AS FECHA_EXP,
    J.SNGC31AUN1 AS DEP_EXP,
    Q.DepNom AS DEPARTENTO_EXP,
    J.SNGC31AUN2 AS CIUDAD_EXP,
    A.PETIPO AS TIPO_PERSONA,
    C.CTNRO AS CUENTA,
    B.PFAPE1 AS APELLIDO1, B.PFAPE2 AS APELLIDO2,
    B.PFNOM1 AS NOMBRE1,  B.PFNOM2 AS NOMBRE2,
    B.Pfcant AS SEXO,
    H.SNGC13PDOC AS PAIS_RESIDENCIA,
    H.sngc13Dpto AS DEPARTAMENTO,
    R.DepNom AS DEPARTAMENTO_NOM,
    H.sngc13Prov AS CIUDAD,
    M.LocNom AS NOMBRE_CIUDAD,
    H.DOCOD AS TIPO_DOMICILIO,
    H.SNGC13DIR AS DIRECCIÓN,
    I.SNGC60OCUP AS OCUPACIÓN,
    I.SNGC60NOME AS NOMBRE_NEGOCIO,
    I.SNGC60FINI AS FECHA_INIC_NEGOCIO,
    I.SNGC60AUX1 AS ORIGEN_RECURSOS,
    I.SNGC60TIPA AS TIPO_ACT,
    I.SNGC60ACTE AS CIIU,
    E.PEXTXT AS CORREO_ELECTRONICO
FROM FSD001 AS A
LEFT JOIN FSD002 AS B ON A.PETDOC = B.PFTDOC AND A.PENDOC = B.PFNDOC
LEFT JOIN FSR008 AS C ON A.PETDOC = C.PETDOC AND A.PENDOC = C.PENDOC
LEFT JOIN SNGC60 AS I ON A.PETDOC = I.SNGC60Tdoc AND A.PENDOC = I.SNGC60Ndoc
LEFT JOIN DECO850 AS K ON A.PETDOC = K.DECO850TDC AND A.PENDOC = K.DECO850NDC
LEFT JOIN SNGC11 AS G ON A.Pendoc = G.sngc11Ndoc
LEFT JOIN SNGC31 AS J ON A.PeTdoc = J.sngc31Tdoc AND A.PENDOC = J.SNGC31NDoc
LEFT JOIN SNGC13 AS H ON A.PeTdoc = H.sngc13Tdoc AND A.Pendoc = H.sngc13Ndoc
LEFT JOIN FSD006 AS D ON C.CTNRO = D.CTNRO
LEFT JOIN FSX001 AS E ON A.PETDOC = E.PETDOC AND A.PENDOC = E.PENDOC
LEFT JOIN FST070 AS M ON H.sngc13Prov = M.LOCCOD
LEFT JOIN FST070 AS O ON G.sngc11Prov = O.LOCCOD
LEFT JOIN FST068 AS P ON H.sngc13Dpto = P.DepCod
LEFT JOIN FST068 AS Q ON G.sngc11Dpto = Q.depcod
LEFT JOIN FST068 AS R ON H.sngc13Dpto = R.depcod
WHERE B.pfndoc = '<NRO_DOCUMENTO>'
    AND E.pexren = 1 AND E.txcod = 0
    AND G.sngc11Dat1 <> '1753-01-01 00:00:00.000'
    AND H.DOCOD IN (2,3,4)
ORDER BY A.Pendoc;
```

### Programas de alta de personas

| Programa | Función |
|----------|---------|
| `HSNGCA01` | Alta de persona |
| `HSNGCPF1` | Alta de persona física |
| `HSNGCPJ1` | Alta de persona jurídica |
| `HSNGC22` | Alta de domicilios |
| `HSNGC50` | Alta de cuenta cliente |
| `HSNGC48A` | Integración de cuentas |
| `HSNGCT14` | Condición ante impuestos |
| `HSNGCRD2` | Documentos adicionales |
| `HSNGCCHD` | Cambio de documento de persona |
| `HSNGC47` | Mantenimiento de domicilios |
| `HIF001A` | Alta de Instituciones Financieras / Creación cuentas bancos |

---

## 4. Cuentas Cliente

```sql
SELECT TOP 10 * FROM dbo.FSD008   -- Maestro cuentas cliente
SELECT TOP 10 * FROM dbo.FSR008   -- Integración Cuenta/Personas (ctccli: 1=Nuevo, 2=Antiguo, 3=Preferencial)
SELECT TOP 10 * FROM dbo.FST049   -- Tipo cliente FSD008
```

### Clasificación de clientes (FSR008.ctccli)

| Valor | Significado |
|-------|-------------|
| `1` | Cliente Nuevo |
| `2` | Cliente Antiguo |
| `3` | Cliente Preferencial |

---

## 5. Préstamos y Cartera

### Tablas maestras de préstamos

```sql
SELECT TOP 10 * FROM dbo.FSD010  -- Maestro de préstamos
SELECT TOP 10 * FROM dbo.FSD011  -- Saldos contables / Contable de préstamos
SELECT TOP 10 * FROM dbo.FSD014  -- Datos rubros
SELECT TOP 10 * FROM dbo.FSR014  -- Rubros
SELECT TOP 10 * FROM dbo.FST042  -- Relación de rubros
SELECT TOP 10 * FROM dbo.FSR012  -- Relcod=88: Codeudor / Relcod=77: Agentes
SELECT TOP 10 * FROM dbo.FRI101  -- Garantía Real
SELECT TOP 10 * FROM dbo.SNG912  -- Días de mora (SNG912DM)
-- Campos tablón: SNG912Vc=Cuota / SNG912iC=IVA / SNG912Cc=Comisión / SNG912sG=Seguro
```

### Estados de los créditos (FSD010.aostat)

| Código | Descripción |
|--------|-------------|
| `0` | Normal |
| `33` | Castigado |
| `60` | Reestructurado o Reprogramado |
| `61` | Renegociado o Refinanciado |
| `62` | Renovado |
| `63` | Cobro Administrativo |
| `64` | Cobro Judicial |
| `99` | Cancelado |

### Calificación de cartera

```sql
SELECT TOP 10 * FROM dbo.JCCA01  -- Calificación, Frecuencia
SELECT TOP 10 * FROM dbo.JCCA02  -- Calificación
SELECT TOP 10 * FROM dbo.JCCA60  -- Tasa determinación
-- Conteo por estado:
SELECT JCCA60EST, COUNT(*) FROM jcca60 GROUP BY JCCA60EST;
```

### Análisis de una operación específica

```sql
-- Reemplazar <NRO_OPERACION> con el número de operación
SELECT * FROM fsd601 WHERE ppoper = <NRO_OPERACION> ORDER BY Ppfval;
SELECT * FROM fsd602 WHERE ppoper = <NRO_OPERACION> ORDER BY Ppfpag;
SELECT * FROM fsd611 WHERE ppoper = <NRO_OPERACION> ORDER BY Ppfpag;
SELECT * FROM fsd612 WHERE ppoper = <NRO_OPERACION> ORDER BY Ppfpag;
SELECT * FROM fsd010 WHERE aooper = <NRO_OPERACION>;
SELECT * FROM fsd011 WHERE Scoper = <NRO_OPERACION> AND SCMOD = 113;
SELECT * FROM X054023 WHERE xllaooper = <NRO_OPERACION>;
SELECT * FROM fpp002 WHERE ppoper = <NRO_OPERACION> ORDER BY Ppfpag;
```

### Tablas por cliente (template)

```sql
-- Reemplazar <NRO_DOCUMENTO> y cuentas según el caso
SELECT * FROM FSD001 WHERE PENDOC = '<NRO_DOCUMENTO>'           -- Maestro persona
SELECT * FROM FSD002 WHERE PFNDOC = '<NRO_DOCUMENTO>'           -- Datos persona
SELECT * FROM FSR008 WHERE PENDOC = '<NRO_DOCUMENTO>'           -- Doc → cuenta
SELECT * FROM FSR002 WHERE rpndoc = '<NRO_DOCUMENTO>'           -- Relaciones
SELECT * FROM FSE001 WHERE d511ndoc = '<NRO_DOCUMENTO>'         -- Extensión
SELECT * FROM FSE002 WHERE pfxndoc = '<NRO_DOCUMENTO>'          -- Extensión datos
SELECT * FROM FSD008 WHERE ctnro IN (<NRO_CUENTA_1>, <NRO_CUENTA_2>)     -- Cuentas
SELECT * FROM fsd010 WHERE aocta IN (<NRO_CUENTA_1>, <NRO_CUENTA_2>)     -- Préstamos
SELECT * FROM fsd012 WHERE aocta IN (<NRO_CUENTA_1>, <NRO_CUENTA_2>)     -- Eventos
SELECT * FROM fsd011 WHERE sccta IN (<NRO_CUENTA_1>, <NRO_CUENTA_2>)     -- Saldos
```

### Solicitudes de crédito

```sql
SELECT TOP 10 * FROM dbo.SNG001   -- Solicitud de crédito / Instancia de préstamo
SELECT TOP 10 * FROM dbo.SNG002   -- Solicitud (detalle)
SELECT TOP 10 * FROM dbo.SNG021   -- Usuario que creó la solicitud (Formiik)
SELECT TOP 10 * FROM dbo.SNG120   -- Estados de las solicitudes de crédito
```

### Solicitudes rechazadas (JCCN52)

```sql
SELECT jccn52tdo, jccn52ndo, jccn52nom, JCCN52Ape, jccn52cas,
       jccn52sol, jccn52fan, jccn52cet, jccn52cmo, jccn52obs
FROM jccn52
WHERE jccn52fan BETWEEN '2026-01-01 00:00:00.000' AND '2026-01-13 00:00:00.000';
-- jccn52cet = Etapa | jccn52cmo = Motivo
SELECT * FROM jccn53  -- Etapas
SELECT * FROM jccn54  -- Motivos
```

### Préstamos anulados

```sql
SELECT TOP 10 * FROM dbo.JCCN52  -- Préstamos anulados (KnockOut)
SELECT TOP 10 * FROM dbo.JCCN54  -- Condiciones de anulación
```

### Módulos con saldo

```sql
SELECT DISTINCT FT111.Modulo
FROM dbo.FSD011 FD11
    INNER JOIN dbo.FST111 FT111
        ON FD11.Pgcod = FD11.Pgcod
        AND FT111.Modulo = FD11.Scmod
        AND FT111.Dscod = 50
WHERE FD11.Pgcod = 1 AND FD11.Sccta BETWEEN 0 AND 999999999;
```

---

## 6. Plan de Pagos

```sql
SELECT TOP 10 * FROM dbo.FSD601  -- Plan de pago: Capital / Interés
SELECT TOP 10 * FROM dbo.FSD602  -- Pago Capital / Interés
SELECT TOP 10 * FROM dbo.FSD611  -- Seguros / Intereses
SELECT TOP 10 * FROM dbo.FSD612  -- Pago Seguros/Intereses (SUM desde PP1IMP16 hasta PP1IMP20)
SELECT TOP 10 * FROM dbo.FPP002  -- Comisión / Otros conceptos
SELECT TOP 10 * FROM dbo.FPP003  -- Pago Comisión/Otros conceptos (PRESTCONC: 3=Mora / 6=Pago)
```

### Cambio de estado de cuota a pagada

```sql
UPDATE fsd602 SET pp1stat = 'T'
FROM fsd602
WHERE pgcod = 1 AND ppsuc = 65 AND ppcta = <NRO_CUENTA> AND ppoper = <NRO_OPERACION>
    AND d602co = 'S'
    AND ppfpag BETWEEN '2024-09-11' AND '2024-11-11'
    AND pptipo = 'F'
    AND pp1nump IN (10, 11, 12)
    AND pp1fech = '2024-06-19';
```

---

## 7. Tasas e Índices

### IBR

```sql
SELECT * FROM FSD024
WHERE CLTCOD IN ('20','21','22','23','24','25','63','65','61','62','64')
AND TGFDES = '2023-01-27';
```

| Código | Descripción |
|--------|-------------|
| `20` | IBR Mes Nominal |
| `23` | IBR Mes Efectiva |
| `21` | IBR Trimestre Nominal |
| `24` | IBR Trimestre Efectiva |
| `22` | IBR Semestre Nominal |
| `25` | IBR Semestre Efectiva |
| `65` | IBR Diaria Nominal |
| `63` | IBR Diaria Efectiva |
| `61` | DTF Nominal |
| `62` | DTF Efectiva |
| `64` | DTF Trimestre |

### TRM

```sql
SELECT * FROM FSH005 WHERE MONEDA = 101 AND cofdes = '2022-10-19';
```

### UVR

```sql
SELECT * FROM FST098 WHERE TPCOD = 2823;
SELECT * FROM Fsh205 WHERE PAPEL = 111 AND PRFDES = '2021-06-01 00:00:00.000';
SELECT * FROM FST144 WHERE COECOD IN (111, 112);
```

### IPC

```sql
SELECT * FROM FSI002
WHERE pgcod = 1 AND cicpo = 'INFLACIO'
ORDER BY cifech;
```

### Orden de carga de tasas

```sql
SELECT * FROM FST205   -- Índices y Especies
SELECT * FROM FSE205
SELECT * FROM FSFIAJ
SELECT * FROM FBC201
SELECT * FROM FBC202
SELECT * FROM FBC203
SELECT * FROM FBC204
SELECT * FROM FSFICN
```

---

## 8. Ajuste de Tasas (Decreto 455)

### EVTIPO — Tipos de evento en FSD012

| Código | Descripción |
|--------|-------------|
| `3` | Cambio de Tasa Normal (corriente) |
| `4` | Cambio de Tasa de Mora |
| `6` | Cambio de Tasa de Corte Normal |
| `11` | Reprogramación o diferimiento |
| `31` | Cambio en condiciones de negociación |
| `50` | Abono a capital |
| `88` | Nueva tasa de costo |

### Procedimiento MORA

```
1. Recibir información de [contacto de negocio]
2. Preparar datos para el query:
   SELECT * FROM fsd012 WHERE aooper IN (...) → EVCORR (el más alto) → bajar a Excel
   SELECT * FROM fsd010 WHERE aooper IN (...)  → bajar a Excel
3. Abrir nueva hoja: copiar títulos de FSD012 desde PGCOD hasta AOOPER de FSD010
4. Completar campos:
   - EVCORR:  el definido arriba
   - EVTIPO:  4 (mora)
   - EVFVAL:  fecha dada por [contacto de negocio]
   - EVFVTO:  campo Default de GeneXus
   - EVIMP:   0
   - EVTTAS:  1
   - EVTASA:  proporcionada por [contacto de negocio]
   - EVCAP→EVCD01: 0
   - EVCD02:  vacío
   - EVINV:   999999999 - EVCORR
   - EVPER→EVARB1: 0
   - EVMD / EVMD1: vacío
   - EVPRE→EV012RE: 0
   - D012FC:  fecha dada por [contacto de negocio] (fecha aplicación en Prod)
   - D012OR / D012SB: 0
   - D012CO:  'S'
5. Cuadrar cantidad de registros
6. Cuadrar que registros de una tasa sean la misma cantidad
```

### Procedimiento CORRIENTE

```
(Igual que MORA, excepto:)
   - EVTIPO:  3 (intereses corrientes)
   - D012FC:  fecha Default de GeneXus
   - Se sube como FSD012
```

### Ajuste tasas lineales x efectiva

```sql
SELECT * FROM fsd010  WHERE aofval >= '2025-02-03' AND aomod = 113 AND aottas = 2;
SELECT * FROM x054023 WHERE xllfvalor >= '2025-02-03' AND xllaomod = 113 AND xlltipotas = 2;
SELECT * FROM x054007 WHERE xllOAOfval >= '2025-02-03' AND xllOaomod = 113 AND xllOAOTTAS = 2;
```

### Abono a capital

```sql
SELECT * FROM fsd012
WHERE aomod = 113 AND evtipo = 50
    AND D012fc >= '2025-10-01 00:00:00.000'
    AND AOMOD <> 70 AND D012TR = 78;
```

---

## 9. Contabilidad

```sql
SELECT TOP 10 * FROM dbo.FSH012  -- Contabilidad histórica por día
SELECT TOP 10 * FROM dbo.FSH014  -- Contabilidad histórica por año (capturado mensual)
SELECT TOP 10 * FROM dbo.FSD016  -- Contabilidad diaria
SELECT TOP 10 * FROM dbo.FSH016  -- Contabilidad histórica
SELECT TOP 10 * FROM dbo.FSH015  -- Asientos (cabezal)
```

### Join cabezal + detalle contable

```sql
SELECT TOP 100 *
FROM dbo.FSD015 FD15
    INNER JOIN dbo.FSD016 FD16
        ON FD16.Pgcod = FD15.Pgcod AND FD15.Itmod = FD16.Itmod
        AND FD15.Itsuc = FD16.Itsuc AND FD15.Ittran = FD16.Ittran
        AND FD15.Itnrel = FD16.Itnrel
WHERE FD16.Itoper = <NRO_OPERACION>;
```

### Asientos anulados

```sql
-- Htpoas='A' | Hccorr='A' | 1=Debe | 2=Haber

-- Diarios (FSH015 / FSH016)
SELECT Htpoas,Hccorr,hsucor,hcmod,htran,hnrel,hfcon,Hccaja
FROM fsh015
WHERE hsucor=58 AND hcmod=22 AND htran=60 AND hfcon='2024-03-21' AND Hccorr=99
UNION
SELECT Htpoas,Hccorr,hsucor,hcmod,htran,hnrel,hfcon,Hccaja
FROM fsh015
WHERE hsucor=58 AND hcmod=22 AND htran=60 AND hfcon='2024-03-21' AND Htpoas='A';

-- Históricos (FSD015 / FSD016)
SELECT ittpoas,itcorr,itsuc,itmod,ittran,Itnrel,itfcon,Itcaja
FROM fsd015
WHERE itsuc=73 AND itmod=50 AND ittran=750 AND itfcon='2025-07-18' AND ittpoas='A' OR itcorr=99;
```

### Errores contables

```sql
-- Registro queda en E: fsd016.itafgt <> 'C' y fsd015.itcont = 'E'
SELECT TOP 10 * FROM dbo.FSH104   -- Errores contables
SELECT TOP 10 * FROM dbo.FSD515   -- Relación asientos anulados
SELECT TOP 10 * FROM dbo.FSX015   -- Detalle anulación (Txcod=9: Hora / 709: Permanencia anterior / 749: Observaciones)
SELECT TOP 10 * FROM dbo.FSA030   -- Asientos desiguales
```

### Programas de contabilización

| Programa | Función |
|----------|---------|
| `PW103` | Contabiliza con preformato |
| `PP006` | Contabiliza |
| `PRG0010B` | Graba preformato / Crea cabezal FSD015 |

---

## 10. Garantías

```sql
SELECT * FROM FRI101             -- Garantía Real
SELECT * FROM jcca12             -- Tabla Garantías
SELECT * FROM ppg000             -- Atributo con texto
SELECT * FROM ppg001             -- Atributo con texto
SELECT * FROM ppg002             -- Atributo con relación / fecha
SELECT * FROM ppg010             -- Atributo
SELECT * FROM ppg011             -- Atributo x tipo de dato
SELECT * FROM ppg012             -- Atributo x Obligatorio y Habilitado
SELECT * FROM ppg008 WHERE ppg008cta = 5058  -- Tabla x Garantías
```

### WII112 — Valorización de Garantías

```sql
SELECT * FROM fst198 WHERE tp1cod1 = 56663  -- Define los atributos
SELECT * FROM fst101 WHERE pbproc IN ('PJCCA124','PJCCA125')
```

| Programa | Función |
|----------|---------|
| `PJCCA123` | Control del carga de archivo |
| `PJCCA124` | Valorización anual |
| `PJCCA125` | Diario - Avalúos que cumplen 3 años (reporte) |
| `PJCCA126` | Control adicional |

### Módulo y tipo de operación (Garantías)

| Módulo | Tipo | Descripción |
|--------|------|-------------|
| `70` | `30` | Hipotecario |
| `70` | `31` | Vehículo |

---

## 11. Seguros

```sql
SELECT TOP 10 * FROM dbo.FSD611  -- Seguros / Intereses (plan de pago)
SELECT TOP 10 * FROM dbo.FSD612  -- Pago Seguros/Intereses
SELECT * FROM dbo.FST300         -- Tipos de Seguros
SELECT * FROM JCCA52             -- Uno a uno de las pólizas
SELECT * FROM X054011 WHERE sgcod IN (123,...,139)  -- Seguros nuevos / Seteo Producto
SELECT * FROM FST098 WHERE tpcod = 13429            -- Seguro Voluntario y Obligatorio
SELECT * FROM FPP065 WHERE PP065sgcod IN (...)      -- Pizarras
SELECT * FROM FST301                                -- Tipo de Seguros
```

### Consulta seguros por operación

```sql
SELECT Aomod, Aosuc, Aocta, Aooper, Aotope, Aofval, Aofvto, Aopzo, Aoimp, Aoperiod,
       Aofinc, Ppfpag, Ppimp11, Ppimp12, Ppimp13
FROM FSD010
LEFT JOIN FSD611
    ON FSD010.Pgcod = FSD611.Pgcod AND FSD010.Aomod = FSD611.Ppmod
    AND FSD010.Aomda = FSD611.Ppmda AND FSD010.Aosuc = FSD611.Ppsuc
    AND FSD010.Aosbop = FSD611.Ppsbop AND FSD010.Aooper = FSD611.Ppoper
    AND FSD010.Aopap = FSD611.Pppap AND FSD010.Aocta = FSD611.Ppcta
    AND FSD010.Aotope = FSD611.Pptope
WHERE Aofval >= '2024-06-20' AND Aomod IN (111, 103, 113)
    AND aostat <> 99 AND Pptipo <> 'K'
ORDER BY AOCTA, AOOPER, PPFPAG;
```

---

## 12. CDT — Depósitos a Plazo

### Estados del CDT (FST500 / FSD850)

| Código | Descripción |
|--------|-------------|
| `2` | Activado |
| `3` | Pendiente |
| `4` | Entregado |
| `7` | Custodia |
| `17` | Pendiente de Activar |
| `98` | Repuesto |
| `99` | Anulado |

### Pizarras CDT

```sql
SELECT * FROM FST053                                               -- Tipos de pizarra
SELECT * FROM FSD025 WHERE Pgcod=1 AND Tamod=22 AND Tpizar IN (1,3,8,9)  -- CDT monto
SELECT * FROM FSR025 WHERE Pgcod=1 AND Tamod=22 AND Tpizar IN (1,3,8,9)  -- CDT rango
```

### Guías CDT

```sql
SELECT * FROM FST198 WHERE Tp1cod1 = 70100  -- Calendario
SELECT * FROM FST198 WHERE Tp1cod1 = 57     -- Plazos y montos
```

### Tablas simulador CDT (actualización)

```sql
SELECT * FROM FPP015 WHERE pp010prd=22 AND pp015cod=12   -- Lista códigos numéricos
SELECT * FROM FPP026 WHERE pp010prd=22 AND pp026top=12   -- Lista códigos numéricos x Producto
SELECT * FROM FPP028 WHERE pp010prd=22 AND pp028top=12   -- Valores parámetro simple nivel Producto
SELECT * FROM FPP040 WHERE Pp028Mod=22 AND pp028top=12   -- Visibilidad parámetro
SELECT * FROM X054010 WHERE xpremod=22 AND xpretope=12   -- Productos (Módulos / Tipo op)
SELECT * FROM DP0501 WHERE DP0501MOD=22 AND DP0501TOP=12
SELECT * FROM DP0502 WHERE DP0502MOD=22 AND DP0502TOP=12
SELECT * FROM fst053 WHERE tpizar=12
SELECT * FROM DP00500 WHERE DP00500MOD=22 AND DP00500TOP=12
```

> **Último paso:** Ejecutar desde el llamador de programas el proceso `PDP09100`

---

## 13. Cuentas de Ahorro

### Tablas maestras

```sql
SELECT * FROM moi000  -- Estados de cuentas de ahorro
SELECT * FROM moi001  -- Detalle (Moi000cod = código de estado)
SELECT * FROM MPE001  -- Preformato Ctas de Ahorro
```

### Pizarras Ahorros

```sql
SELECT * FROM FSD025 WHERE Pgcod=1 AND Tamod=21  -- Ahorros
SELECT * FROM FSR025 WHERE Pgcod=1 AND Tamod=21  -- Ahorros
```

### Devengamiento

```sql
SELECT * FROM FST101 WHERE pbproc = 'PCC00003'
SELECT * FROM fsd130 WHERE devccmod = 21
SELECT * FROM fst004 WHERE modulo=21 AND totope=7
```

### Cuenta de Ahorros Niños / Junior

```sql
SELECT * FROM fst036 WHERE trmod=21 AND trnro=901   -- Rubro para alta de cuenta
SELECT * FROM FST098 WHERE TPCOD=1836                -- Módulo y Tipo de Operación
SELECT * FROM FST098 WHERE TPCOD=6065               -- Parámetros Titular y Apoderado
-- Corr1=Titular: Edad mínima (ValEsp) / Edad máxima (ImpEsp)
-- Corr2=Apoderado: igual estructura
SELECT * FROM fst098 WHERE tpcod=1543               -- Tipos de documento por alta
```

### FATCA

```sql
SELECT * FROM rep001 WHERE rep001cod IN (180,181)           -- Formulario PN y PJ
SELECT * FROM SNGDP1 WHERE sngdp1mod=22 AND sngdp1top=1    -- CDT FATCA
SELECT * FROM fst198 WHERE tp1cod1=14801 AND tp1corr1=4    -- Cuenta de Ahorros FATCA
```

### Programas

| Programa | Función |
|----------|---------|
| `hpap001` | Mantenimiento de Especies e Índices |
| `HPAP001C` | Confirmación modificaciones Cotizaciones de Especies |
| `hpp9040` | Administrador de Préstamos / Condonación / Pago de Créditos |
| `hsip570` | Alta de créditos reducida |
| `hsip571` | Alta reducida de Crédito |
| `hfsr601` | Cuentas para Cobranza de Préstamos |
| `hjcca060` | Determinación de la tasa para el crédito |

---

## 14. Canales y ACH

### Tablas de Cámaras

```sql
SELECT * FROM MPE001  -- Transaccional
SELECT * FROM MPE002  -- Transaccional
SELECT * FROM MPE005  -- Servicio ACH
SELECT * FROM MPE006  -- Cámara
SELECT * FROM MPE007  -- Ciclos
SELECT * FROM MPE008  -- Archivos ciclos
SELECT * FROM MPE009  -- Ciclos procesados
SELECT * FROM MPE010  -- Ciclos ID Módulo y Transacción
SELECT * FROM MPE011  -- Configuración Módulos y Transacciones
SELECT * FROM MPE020  -- Bancos
SELECT * FROM MPE024  -- Mensajes
```

### Procesos en ejecución

```sql
SELECT * FROM MPE009 ORDER BY 1 DESC
SELECT * FROM MPE005 ORDER BY 1 DESC
SELECT * FROM MPE006
```

### Ajuste ACH

```sql
SELECT * FROM MPE001 WHERE MPE001FET='2025-02-03' AND MPE001IDL IN (20983,20984);
-- Campo MPE001EST = PP
SELECT * FROM MPE002 WHERE MPE002IDL IN (20983,20984);
-- MPE002EST = 0
```

### Programas ACH

| Nro | Programa | Función |
|-----|----------|---------|
| 1 | `HARQ001` | Publicación de Servicios Bantotal |
| 2 | `HBTI025` | SDTs - Estructuras de datos |
| 3 | `HMPE0050` | Mantenimiento de Cámaras (MPE005 / MPE006) |
| 4 | `HMPE0056` | Operaciones de Cámara |
| 5 | `HIF00154` | Formatos de Cuenta en Interfaces |
| 6 | `HMPE0053` | Ciclos de cámaras |
| 7 | `HMPE0280` | Panel de Consultas de Transferencias ACH |
| 8 | `hbtsbt1t` | Mantenimiento de Transacciones por Servicio (BTSBT1) |
| 9 | `hrep001` | Reportes |

### Proceso creación entidad financiera ACH

```
1. Comunicado a operaciones ([operaciones])
2. Operaciones coloca un GLPI
3. Generar script MPE020
4. Descargar Test Fly (verificar versión) — eliminar producción y token antes
5. Activar cliente con cuenta activa, saldo y token
6. Hacer transferencias por portal con token de la app
7. URL pruebas: [URL interna de pruebas]
8. Probar descarga y carga del NACHAN a Integra:
   BT / Menú Cadena de Cierre / Agregar y Descargar Archivos (PMPE0030 - ACH)
   → Solo subir, no abrir
9. Integra ACH: [URL interna de Integra ACH]
```

### Generación de UID

```
1. Ingresar a HBTSBT2 — Panel de Generación De Identificador Único (UID)
2. Generar identificador → seleccionar operaciones
3. Módulo 21 / Sucursal 64 / Moneda 0 / Cuenta 200198
4. Generar UID
```

---

## 15. Servicios Web Bantotal (BTI)

### Consulta de servicio

```sql
DECLARE @service NVARCHAR(MAX) = 'BTNotificaEmbargo'

SELECT * FROM dbo.BTI004 WHERE BTISrvNom = @service   -- Servicios por Interfaz
SELECT * FROM dbo.BTI007 WHERE BTISrvNom = @service   -- Usuarios habilitados por Servicio/Método-Canal
SELECT * FROM dbo.BTI012 WHERE BTISrvNom = @service   -- Servicios/Métodos por Canal
SELECT * FROM dbo.BTI014 WHERE BTISrvNom = @service   -- Interfaz Servicios - Métodos
SELECT * FROM dbo.BTI019 WHERE BTISrvNom = @service   -- Parámetros de Métodos de Servicios
SELECT * FROM dbo.BTI025 WHERE BTISDTNom = 'SdtNotificaEmbargo_NotificaEmbargoItem'  -- SDT
SELECT * FROM dbo.BTI026 WHERE BTISDTNom = 'SdtNotificaEmbargo_NotificaEmbargoItem'  -- Elementos SDT
```

### Eliminar servicio (usar con precaución)

```sql
-- Descomentar solo cuando sea necesario eliminar
--DELETE FROM dbo.BTI004 WHERE BTISrvNom = @service
--DELETE FROM dbo.BTI007 WHERE BTISrvNom = @service
--DELETE FROM dbo.BTI012 WHERE BTISrvNom = @service
--DELETE FROM dbo.BTI014 WHERE BTISrvNom = @service
--DELETE FROM dbo.BTI019 WHERE BTISrvNom = @service
--DELETE FROM dbo.BTI025 WHERE BTISDTNom = 'SdtNotificaEmbargo_NotificaEmbargoItem'
--DELETE FROM dbo.BTI026 WHERE BTISDTNom = 'SdtNotificaEmbargo_NotificaEmbargoItem'
```

### Consulta autorización de servicios

```sql
SELECT * FROM BTI012  -- Permiso Canal/Servicio
SELECT * FROM BTI007  -- Permisos Usuario/Servicio/Método
SELECT * FROM BTI004  WHERE BTINom='BANTOTAL' AND BTISrvNom='CreditosDECO'
SELECT * FROM BTI014  WHERE BTINom='BANTOTAL' AND BTISrvNom='CreditosDECO'
SELECT * FROM BTI019  WHERE BTINom='BANTOTAL' AND BTISrvNom='CreditosDECO'
SELECT * FROM BTI012  WHERE BTICanNom='BTDIGITAL' AND BTINom='BANTOTAL' AND BTISrvNom='CreditosDECO'
SELECT * FROM BTI001  WHERE BTICanNom='BTDIGITAL'
```

### SDTs

```sql
SELECT * FROM BTI019 WHERE BTISrvNom='BTDepositosAPlazo' AND BTIMtdNom='ObtenerProductosHabilitados'
SELECT * FROM BTI025 WHERE BTISDTNom='SdtsBTProductoDepositoAPlazo'
SELECT * FROM BTI026 WHERE BTISDTNom='SdtsBTProductoDepositoAPlazo'
```

### Programas Bantotal (llamadas)

```
PW103      Call(&Programa, &PgCod, &ItMod, &Ittran, &Itnrel, &Pgfape, &MnCod)    -- Contabiliza con preformato
PRg0010    Call(&Programa, &Pgcod, &Itsuc, &Itmod, &Ittran, &Itnrel)             -- Siguiente Relación
PP006      Call(&Programa, &Pgcod, &Itsuc, &Itmod, &Ittran, &Itnrel, &Pmncod)   -- Contabiliza
PRG0010B   Call(&Programa, &Pgcod, &Itsuc, &Itmod, &Ittran, &Itnrel)             -- Graba Preformato
```

### Generación de UID

```
1. Ingresar a HBTSBT2 — Panel de Generación De Identificador Único (UID)
2. Generar identificador → seleccionar operaciones
3. Módulo 21 / Sucursal 64 / Moneda 0 / Cuenta 200198
4. Generar UID
```

---

## 16. Cadena de Cierre

```sql
SELECT * FROM dbo.FST101   -- Tareas cadena de cierre
SELECT * FROM dbo.FSR101   -- Threads tareas cadena de cierre
SELECT * FROM dbo.CAP003   -- SQL tareas cadena de cierre
SELECT * FROM dbo.FST998   -- Tipos de numerador
SELECT * FROM dbo.FSN999   -- Numeradores
```

### Programas de Cadena

| Programa | Función |
|----------|---------|
| `hfst101` | Cadena de Cierre |
| `hcap003` | Mantenimiento de Paralelización |
| `HFRPRCCONSOLE` | Consola de Procesos |
| `HCONSOL` | Consola de Procesos |
| `hrgap001` | Trabajar con Procesos |

### Análisis por hilos (FRTASKS)

```sql
SELECT TOP 100
    DATEDIFF(MINUTE, CONVERT(DATETIME, FRTskTimSt), CONVERT(DATETIME, FRTskTimEn)) AS diff, *
FROM dbo.FRTASKS WHERE FRPrcId = 2200 ORDER BY diff DESC;

SELECT TOP 100 * FROM dbo.FRTASKS
WHERE FRTskDsc LIKE '%Cartera%' ORDER BY FRTskTimCr DESC;
```

---

## 17. Reportes y Regulatorio

### Reporteador

```sql
SELECT TOP 10 * FROM dbo.REP001
SELECT TOP 10 * FROM dbo.REP002
SELECT TOP 10 * FROM dbo.REP003
SELECT TOP 10 * FROM dbo.REP004
```

### Reportes Normativos

```sql
SELECT * FROM fsi001  -- Campos de Información
SELECT * FROM fsi002  -- Valores Campo
SELECT * FROM fsi003  -- Integración de Informe
SELECT * FROM fsi004  -- Informes
SELECT * FROM fsi005  -- Campos Totalizadores
SELECT * FROM fsi006  -- Integración de Campo
SELECT * FROM fsi008  -- Agrupación de Campo
SELECT * FROM fsi010  -- Tipos de Campo
SELECT * FROM fsi011  -- Tipo de Importe
SELECT * FROM fsl001  -- Límite global por cuenta
```

### RACT — Reporte Anual de Costos Totales

```sql
SELECT * FROM dbo.DECO60   -- Parametrización de conceptos
SELECT * FROM dbo.DECO61   -- Parametrización de rubros - conceptos
SELECT * FROM dbo.DECO62   -- Parametrización de productos - conceptos
SELECT TOP 10 * FROM dbo.DECO63  -- Ejecución diaria

-- Programa en Cadena de Cierre:
SELECT * FROM fst101 WHERE PBPROC = 'PDECO310'
SELECT * FROM fst101 WHERE PBPROC = 'PDECO604'
SELECT * FROM fsr101 WHERE PBNSEC = 77007
SELECT * FROM fsr101 WHERE PBNSEC IN (1233, 1303)
SELECT * FROM fst198 WHERE tp1cod1 = 33301
```

### Consulta de desembolsos regulatorio

```sql
SELECT
    A.Hsucor AS SUCURSAL, A.HCMOD AS MODULO, A.HTRAN AS TRANSACCION,
    A.HNREL AS RELACION, B.HMODUL AS TIPO_DE_CREDITO,
    A.HFCON AS FECHA_DESEMBOLSO, A.HUSING AS USUARIO_DESEMBOLSO,
    B.HCTA AS CUENTA, B.HOPER AS OPERACION,
    C.PETDOC AS TIPO_IDENTIFICACION, C.PENDOC AS NUMERO_DOCUMENTO,
    E.PFAPE1 AS APELLIDO1, E.PFAPE2 AS APELLIDO2,
    E.PFNOM1 AS NOMBRE1,  E.PFNOM2 AS NOMBRE2,
    E.Pfcant AS SEXO,
    D.sngc11Dpto AS DEP_NACIMIENTO, D.SNGC11PROV AS CIUDAD_NACIMIENTO,
    D.sngc11Dat1 AS FECHA_EXP,
    G.SNGC60FINI AS FECHA_INICIO_NEGOCIO,
    F.PEXTXT AS CORREO_ELECTRONICO,
    G.SNGC60OCUP AS OCUPACION, G.SNGC60nome AS NOMBRE_NEGOCIO,
    G.SNGC60UBIC AS UBICACION_NEGOCIO, G.SNGC60RZSO AS RAZON_SOCIAL
FROM fsh015 AS A
INNER JOIN FSH016 AS B ON A.PGCOD=B.PgCod AND A.Hsucor=B.Hsucor AND A.Hcmod=B.Hcmod
    AND A.Htran=B.Htran AND A.HNREL=B.Hnrel AND A.HFCON=B.Hfcon
INNER JOIN FSr008 AS C ON B.HCTA=C.CTNRO
INNER JOIN SNGC11 AS D ON C.PETDOC=D.SNGC11TDOC AND C.PENDOC=D.SNGC11NDOC
INNER JOIN FSD002 AS E ON C.PETDOC=E.PFTDOC AND C.PENDOC=E.PFNDOC
INNER JOIN FSX001 AS F ON C.PETDOC=F.PETDOC AND C.PENDOC=F.PENDOC
INNER JOIN SNGC60 AS G ON C.PETDOC=G.SNGC60TDOC AND C.PENDOC=G.SNGC60NDOC
WHERE A.PgCod=1 AND A.Hcmod=30
    AND A.Htran IN (5,30,33,34,35,40,43,45,981,982)
    AND A.hfcon BETWEEN '2024-03-19' AND '2024-03-19'
    AND A.Hccorr <> 99 AND A.Htpoas <> 'A'
    AND B.Hcord=10 AND F.TXCOD=0 AND G.SNGC60CORR=0
ORDER BY B.Hoper;
```

---

## 18. CIFIN — Centrales de Riesgo

```sql
SELECT * FROM FST050 WHERE COCOD = 205
SELECT * FROM fsp026 WHERE cocod = 205 ORDER BY cofech
SELECT * FROM fst098 WHERE tpdesc LIKE '%cifin%'
SELECT * FROM fst198 WHERE tp1desc LIKE '%cifin%'
SELECT * FROM FST098 WHERE Pgcod=1 AND Tpcod=13600 AND Tp1corr1=205  -- % Centrales de riesgo
SELECT * FROM FST098 WHERE Pgcod=1 AND Tpcod=13404 AND Tpcorr IN (122,123)  -- Papelería/Centrales
SELECT * FROM FST098 WHERE Pgcod=1 AND Tpcod=81020  -- Calificación cliente
```

### CIFIN — Creación oficina

```sql
SELECT * FROM FST001  -- Oficina
SELECT * FROM FBC205  -- Región
SELECT * FROM FBC206  -- Zona
SELECT * FROM FST810  -- Zona
SELECT * FROM FST811  -- Zona y Oficina
SELECT * FROM FST098 WHERE tpcod = 3967  -- Código CIFIN

INSERT INTO CIFINCONSULTA.CONSULTACIFIN_OFICINA
    (CODIGOTIPOPRODUCTO, SECUENCIALOFICINA, NOMBREOFICINA, CODIGOOFICINACIFIN, CODIGOBANTOTAL, ESTAACTIVO)
VALUES ('PSLT', 1, 'Zipaquira', '85', '84', 1);

INSERT INTO FST098 (Pgcod, Tpcod, Tpcorr, Tpnro, Tpdesc, Tpimp)
VALUES (1, 3967, 103, 501, N'Pasto MiPyme                  ', 51.00);
```

---

## 19. Carpeta Digital

```sql
SELECT * FROM XWF060  -- Tipos de documento
SELECT * FROM XWF065  -- Mapeo Tipos de Documentos-CacTYP
SELECT * FROM XWF069  -- Perfiles por Tipo de Documento
SELECT * FROM XWF063  -- Instancias de Control Documentario
SELECT * FROM XWFD01  -- Documentos Carpeta Digital
SELECT * FROM XWFD02  -- Versiones de Documentos
SELECT * FROM XWFD05  -- Documentos-Instancias
SELECT * FROM XWFD06  -- Documentos-Personas
SELECT * FROM XWFD07  -- Documentos-Cuentas
SELECT * FROM XWFD08  -- Documentos-Operación
SELECT * FROM XWFDE2  -- Extensión XWFD02
SELECT * FROM XWFD09  -- Relación documento Digital con doc. WF
```

### Programas

| Programa | Función |
|----------|---------|
| `hxwf060` | Crear Carpeta Digital |
| `hxwfd002` | Trabajar con Tareas |
| `hxwfd001` | Trabajar con Procesos (Id 2: Préstamos individuales) |
| `hxwfd006` | Trabajar con Roles (Analista de Riesgos) |
| `hxwfd007` | Trabajar con Rol Tareas |
| `hxwfd008` | Trabajar con Rol con Usuario (Reestructuraciones) |

---

## 20. Control Dual

```sql
-- Guía especial 38: Correlativo 1 = 0 (sin Control Dual)
SELECT * FROM fst198 WHERE tp1cod1=38 AND tp1corr1=457;
-- 0 = Inhabilitado (directo a producción)
-- 1 = Sin control dual pero genera log
-- 2 = Control dual activo
```

### Tablas de Control Dual

```sql
SELECT * FROM CTD000  -- Configuración Tablas
SELECT * FROM CTD001  -- Configuración Campos Tablas
SELECT * FROM CTD006  -- Tablas Relacionadas
SELECT * FROM CTD007  -- Agrupador Tablas
SELECT * FROM fst200 WHERE OpgCod = 3054  -- Aprueba Masivo
SELECT * FROM fst098 WHERE tpcod = 3422   -- Orden aparición en pantalla
```

### Programas Control Dual

| Programa | Función |
|----------|---------|
| `HMDA401` | Cotizaciones de Cierre |
| `HMDA401C` | Confirmación Cotizaciones de Cierre |
| `HMDA402` | Mantenimiento de Pizarras |
| `HMDA402C` | Confirmación Mantenimiento de Pizarras |
| `HCTD001` | Mantenimiento de Tablas Control Dual |

---

## 21. Formiik

```sql
SELECT * FROM homologacion
SELECT * FROM MasterMind.BITACORA_CATALOGOS WHERE CODIGOPETICION=9 ORDER BY secuencial DESC  -- Cargos

-- Documentos cargados
SELECT TOP 10 * FROM dbo.XWF700  -- Registro de Solicitud (XWFCar3: estados)
SELECT TOP 10 * FROM dbo.XWFD06  -- Documentos clientes - Cédula
SELECT TOP 10 * FROM dbo.XWFD01  -- Estado, tipo documento
SELECT TOP 10 * FROM dbo.XWFD02  -- Usuario, SHA, extensión

-- Datos exclusivos de Formiik
SELECT TOP 10 * FROM dbo.JCCM70
SELECT TOP 10 * FROM WFATTBVALUES  -- Valores de solicitud
SELECT TOP 10 * FROM WFATTSVALUES  -- Más datos de solicitud

-- Consulta Bus
SELECT * FROM MESSAGE_LOG
WHERE operation_Name LIKE ('CargueTercerosRelacionados')
ORDER BY id DESC;
```

### Programas Formiik

| Programa | Función |
|----------|---------|
| `hjccn403` | Mantenimiento motivos de negación rechazo |
| `hjccn551` | Mantenimiento Equivalencias (Catálogos de Formiik) |
| `HJCCN501` | Mantenimiento Equivalencia Sucursales Oficina |

---

## 22. Autorizaciones y Excepciones

```sql
SELECT * FROM FST039 WHERE EXCOD = 22   -- Si es Ligada (L) o Desligada (D)
SELECT * FROM FST098 WHERE TPCOD=203 AND TPCORR=22      -- Módulo
SELECT * FROM FST098 WHERE TPCOD=1495 AND TPCORR=22
SELECT * FROM FST198 WHERE TP1COD1=5557                 -- Transacciones
SELECT * FROM AUT000  WHERE JAutExCod=22
SELECT * FROM AUT0001 WHERE JAutExCod=22   -- Módulos excluidos
SELECT * FROM AUT0002 WHERE JAutExCod=22   -- Niveles de Autorización
SELECT * FROM AUT0003 WHERE JAutExCod=22   -- Montos o Rangos
SELECT * FROM AUT0004 WHERE JAutExCod=22   -- Perfiles de Autorización
```

### Árbol de Autorizaciones

| Tabla | Descripción |
|-------|-------------|
| `AUM000` | Códigos Árbol Autorizaciones |
| `AUM001` | Árbol / Nivel |
| `AUM003` | Grupos de Autorización |
| `AUM004` | Subniveles del Grupo |
| `AUM005` | Relación Mod/Trn con Árbol |
| `AUM006` | Autorización por Excepción |
| `AUT000` | Excepción por tasa o por monto |
| `AUT0002` | Excepción por Nivel |
| `AUT0003` | Excepción por Nivel/monto/tasa |
| `AUT0004` | Excepción por Nivel/Sucursal |

### Reglas de negocio

```sql
SELECT * FROM frng49 WHERE rng49cod=1187   -- Niveles Autorización Crédito
SELECT * FROM frng49 WHERE rng49cod=11400  -- Autorizaciones Créditos
SELECT * FROM FRNG49 WHERE rng49cod=12001  -- Arrendamiento Consumo
SELECT * FROM frng49 WHERE rng49cod=77101  -- Check ID
```

### Programa: `HRNG400` — Trabajar con Reglas de Negocio

---

## 23. Impuestos y GMF

```sql
SELECT * FROM FSI001 WHERE cicpo='UVT'
SELECT * FROM FSI002 WHERE cicpo='UVT' AND cifech='2025-01-01 00:00:00.000'
SELECT * FROM FAIt01 WHERE FAImpCod=9
SELECT * FROM FAIt02 WHERE FAImpCod=9
SELECT * FROM FAIt03 WHERE faimpcod=9
```

### Programas

| Programa | Función |
|----------|---------|
| `HFAIR01` | Relación Transacción/Impuesto |
| `HFAIT01` | Mantenimiento de Impuesto |
| `hfaid01` | Condición por Impuesto/Persona |
| `HFAIT01` | Parametrización de Impuestos |

### GMF — Nueva reforma tributaria (Ley 2277 de 2022)

```sql
SELECT * FROM fst200 WHERE opgcod=20368     -- Habilita Notificaciones
SELECT * FROM fSI001 WHERE CICpo='RUB_GMF'  -- Opciones Generales
SELECT * FROM fSI006 WHERE CICpo='RUB_GMF'  -- Rubro
SELECT * FROM fste03                         -- Alta Notificaciones
SELECT * FROM fste01                         -- Mantenimiento Eventos
SELECT * FROM fst998 WHERE ngtipo=940        -- Tipo de Numerador
SELECT * FROM fst198 WHERE tp1cod1=645       -- Guía principal GMF
```

---

## 24. Metas Comerciales

```sql
SELECT * FROM FST098 WHERE tpcod=81004   -- Las regiones
SELECT * FROM fbc205 WHERE bc205cod IN (811,812,813,814,815,816)
SELECT * FROM fbc206 WHERE bc205cod IN (811,812,813,814,815)
SELECT * FROM JCCY12 WHERE jccy12anio=2023 AND JCCY12DOfi LIKE '%Barbosa%'  -- Generales
SELECT * FROM JCCY13 WHERE jccy13anio=2024 AND JCCY13DOfi LIKE '%Barbosa%'  -- Detalladas
```

### Programas de Metas

| Programa | Función |
|----------|---------|
| `hjccy008` | Cargue de Metas |
| `hjccy009` | Asignación Metas |
| `hjccy010` | Metas comerciales asesor |
| `hjccy011` | Metas comerciales zona |
| `hjccy014` | Ajuste Metas oficina |
| `hjccy018` | Visualización Metas Comerciales |

### Jerarquía comercial

```sql
SELECT
    A.BC205Cod AS CODIGO_REGION, A.BC205Dsc AS NOMBRE_REGION,
    B.BC206Id1 AS CODIGO_ZONA,   B.BC206Chr1 AS NOMBRE_ZONA,
    C.OFICOD AS CODIGO_OFICINA,  D.SCNOM AS NOMBRE_SUCURSAL
FROM fbc205 AS A
INNER JOIN fBc206 AS B ON A.BC205EMP=B.BC205EMP AND A.BC205Cod=B.BC205Cod
INNER JOIN FST811 AS C ON A.BC205EMP=C.Pgcod AND B.BC206ID1=C.REGCOD
INNER JOIN FST001 AS D ON A.BC205EMP=c.PGCOD AND C.OFICOD=D.Sucurs
WHERE A.bc205cod IN (811,812,813,814,815,816)
ORDER BY CODIGO_REGION;
```

---

## 25. Corresponsal

```sql
SELECT TOP 10 * FROM dbo.DECO21  -- Corresponsalía (TMo: 1=Recaudo, 2=Reverso)
SELECT * FROM XCR060             -- Banco Corresponsales
```

### Proceso creación/actualización punto corresponsal

```
1. Solicitar a [contacto de negocio] la base para cruzar
2. Crear un punto o actualización de puntos
3. GLPI de Operaciones
4. Prueba revisión por front de BT → revisión sube a Producción
```

### Corresponsal — Comisiones

```
1. Creación nueva oficina
2. Script en FSD526:
   - 148: Comisión depósito $3.200 (asume banco)
   - 149: Comisión retiro $2.500 (asumido cliente — 100%)
3. Sube a pre-prod
```

---

## 26. Operaciones de Caja

### Caja saldo cero

```sql
SELECT Scsdo, * FROM FSD011
WHERE SCRUB=1105050001 AND SCSBOP=0 AND Scsdo <> 0;

SELECT Itafgt, * FROM fsd016
WHERE Itsuc=56 AND itmod=50 AND ittran=110 AND Itnrel=40
ORDER BY Itnrel, itord;

SELECT itcont, * FROM fsd015
WHERE Itsuc=56 AND itmod=50 AND ittran=110 AND Itnrel=40;
```

### Descruce de caja / Operaciones no contabilizadas

```sql
SELECT * FROM fsd015 a WITH (NOLOCK)
WHERE a.pgcod=1 AND a.itcont='E'
AND EXISTS (
    SELECT * FROM fsd016 b WITH (NOLOCK)
    WHERE b.pgcod=a.pgcod AND b.itsuc=a.itsuc
        AND b.itmod=a.itmod AND b.ittran=a.ittran
        AND b.itnrel=a.itnrel AND b.itafgt='E'
);

SELECT * FROM fsd015 WHERE ItconT <> 'S' AND Itsuc=93;  -- No contabilizadas
```

### Cierre de cajas

```sql
SELECT COUNT(mbccest) FROM MBC004 WHERE MBCCFch='2024-03-30' AND MBCCEst='A' AND MBCCCaj<>0  -- Abiertas
SELECT COUNT(mbccest) FROM MBC004 WHERE MBCCFch='2024-03-30' AND MBCCEst='C' AND MBCCCaj<>0  -- Cerradas
SELECT * FROM MBC004 WHERE MBCCFch='2024-03-30' AND MBCCEst='A' AND MBCCCaj<>0 ORDER BY MBCCSuc
```

---

## 27. Accesos Rápidos — Programas Bantotal

| Función | Programa |
|---------|---------|
| Consulta permisos Módulo/Transacción | `Hprf093` |
| Guías de Proceso | `htrt098` |
| Guías de Proceso Especial | `htrt198` |
| Opciones Generales de Proceso | `hfst200` |
| Reglas de Negocio | `hrng400` |
| Mantenimiento de Campos de Información | `HSI001` |
| Numeradores Genérico | `htrn999` |
| Plan de Códigos Contables | `hme00010` |
| Consola de Procesos | `HFRPRCCONSOLE` / `HCONSOL` |
| Cadena de Cierre | `hfst101` |
| Mantenimiento Paralelización | `hcap003` |
| Servicio Información Centinela | `HJCCI100` |
| Mantenimiento de Parámetros | `HSNGP001` |
| Validación de ambiente | `hchkenv001` |
| Reiniciar para tomar cambios | `HFRSERVICES` |
| SDTs | `HBTI025` |
| Programas Particulares por Cliente | `htrt900` |
| Módulo Normativo | `HBCGN100` |
| Mantenimiento Mod. Normativo | `HBCGN101` |
| Reportes Normativos (Archivos Planos) | `hrrcor05` |
| Carpetas Spool | `HFRSPOFOLDERS` |
| Spool | `HFRSPOOL` |
| Config. Repositorio de Archivos | `HFRCONFFILREP` |
| Permiso Carpetas del Spool | `HFRPROFOL` |
| Garantías | `HPPG001` |
| Mantenimiento Campos de Inf. | `hsi001` |
| Debug | `HTRT200` |
| Definición Transacciones | `HFST003` |
| Valores Módulo | `hfsv003` |
| Centros de Costos | `htrt051` |
| Trabajar con Elementos de Programa | `hsng038` |
| Tablas de Auditoría | `haud001w` |
| Anulación de Transacciones | `hw001rb` |
| Confirmación de Transacciones | `hw006b` |
| Ingreso de Operaciones | `hw000` |
| Operaciones de Caja | `hcaj23c` |
| Mantenimiento de Convenios | `HJCCN200` |
| Plantilla de Traslado de Cartera | `HJCCA601` |
| Trabajar con Niveles | `hsng415` |
| Evaluaciones Socioeconómicas | `Hsng254` |
| Parametrización Componentes | `HSNG459` / `HSNG266` |
| Alta Clientes Inst. Financieras | `HIF001APRO` |
| Administración de Corresponsales | `hfoc010a` |
| Mapeos Usuarios de Canal | `hbti010w` |
| Cargar Archivo | `hcat010` |
| Modelador Productos Vista | `hprd100` |
| Negociación de Pagos | `HPP9008` |
| Pizarra activación | `TTRT003` |
| Selección de Producto | `hsip501b` |
| Generar SDTs nativos para servicios | `HBTI019S` |
| Trabajar con Tablas (Normativo) | `HBC205T` |
| Minificadena | `HBCGN100` (50343 — Febrero) |

---

## 28. Tablas de Parámetros Globales

| Tabla | Descripción |
|-------|-------------|
| `FST300` | Códigos de Seguros |
| `FST050` | Códigos de Comisión |
| `FST029` | Clases de Tasa |
| `FST053` | Tipos de Pizarra |
| `FST013` | Países |
| `FST001` | Sucursales |
| `FST014` | Tipos de Documento |
| `FST024` | Tipos de Tasa |
| `FST026` | Estados de Operación |
| `FST028` | Tabla maestra de calendario |
| `FST034` | Descripción de transacciones |
| `FST038` | Códigos de relación |
| `FST111` | Módulos del sistema (Dscod=50: Cartera) |
| `FST130` | Códigos de Facultades |
| `FST138` | Códigos de Instrucción |
| `FST200` | Opciones Generales de Proceso |
| `FST004` | Descripción por Módulo y tipo operación |
| `FAIT01` | Impuestos |
| `FSX001` | Correos Electrónicos |
| `FSI001` | Campos de Información |
| `FSI002` | Valores de Campo |
| `FSM001` | Programas |
| `FSADBG` | Ver el Debug |

---

## 29. Queries Analíticos Complejos

### Consulta seguros (JCCA52)

```sql
SELECT DISTINCT
    A.JCCA52MOD AS MODULO, A.JCCA52SUC AS SUCURSAL,
    A.JCCA52CUC AS CUENTA,  A.JCCA52OPE AS OPERACION,
    B.PFNOM1 AS NOMBRE1,    B.PFNOM2 AS NOMBRE2,
    B.Pfape1 AS APELLIDO1,  B.PFAPE2 AS APELLIDO2,
    B.Pffnac AS FECHA_NACIMIENTO, B.Pfcant AS SEXO,
    A.JCCA52NDO AS NUMERO_DOCUMENTO, A.JCCA52TDO AS TIPO_DOCUMENTO,
    A.JCCA52SEG AS TIPO_SEGURO,      A.JCCA52ASE AS ASEGURADORA,
    A.JCCA52FIP AS FECHA_INICIO,     A.JCCA52FIP AS FECHA_FINAL,
    A.JCCA52PlC AS PLAZO,            A.JCCA52VPM AS VPM,
    A.JCCA52VAP AS VAP,              A.JCCA52VAR AS VAR,
    A.JCCA52EDP AS ESTADO,           A.JCCA52COM AS CODIGO,
    A.JCCA52CAS AS CODIGO_ASESOR
FROM JCCA52 AS A
LEFT JOIN FSD002 AS B ON A.JCCA52NDO = B.PFNDOC;
```

### Canje histórico

```sql
SELECT DISTINCT
    A.Hsucor AS SUCURSAL, A.HCMOD AS MODULO, A.HTRAN AS TRANSACCIÓN,
    C.Cle101Fch AS FECHA, A.HCTA AS CUENTA, A.HOPER AS OPERACION,
    A.Hnrel AS RELACION, A.Hcpzo AS PLAZO,
    C.CLE101BCO AS BANCO, C.CLE101ctal AS CTA_BANCO,
    A.Hccheq AS CHEQUE, A.HCIMP1 AS MONTO
FROM CLE101 AS C
LEFT JOIN fsH016 AS A
    ON C.CLE101SUC=A.Hsucor AND C.cle101cta=A.HCTA
    AND c.Cle101Chq=A.Hccheq AND C.Cle101Mod=A.Hmodul AND C.Cle101Imp=A.Hcimp1
LEFT JOIN fsh015 AS B
    ON A.PGCOD=B.PgCod AND a.hsucor=B.hsucor AND A.HCMOD=B.HCMOD
    AND A.Htran=B.Htran AND A.Hnrel=B.Hnrel AND A.HFCON=B.Hfcon
WHERE A.HCMOD=22 AND A.Htran IN (60,65,66)
    AND A.HTPOASR <> 'A' AND B.Hccorr <> 99
    AND A.Hrubro = '9170000021'
ORDER BY C.Cle101Fch;
```

### Pizarra de tasas (FSD026 + FSR026 + FSP026)

```sql
SELECT
    A.Comod AS Modulo, A.Cocod AS Pizarra, A.Cofech AS Fecha,
    A.Comto AS Monto, A.Cotasa AS Tasa,
    A.Comin AS Monto_Minimo, A.Comax AS Monto_Maximo, A.Coimp AS Importe,
    C.Comto AS Monto, C.Copzo AS Plazo, C.CotasaP AS Tasa,
    C.CominP AS Margen, C.ComaxP AS Maximo, C.CoimpP AS Importe
FROM FSD026 AS A
INNER JOIN FSR026 AS B ON A.Comod = B.Comod
INNER JOIN FSP026 AS C ON A.Comod = C.Comod
WHERE A.Comod=113 AND A.Cocod=11301
ORDER BY A.Cofech;
```

### Crédito mora y garantía real

```sql
SELECT *
FROM FRI101 AS A
LEFT JOIN SNG912 AS B ON A.RI101Cta=B.SNG912CTA AND A.RI101Ope=B.SNG912Op
WHERE SNG912DM > 1;
```

### Fondeadores

```sql
SELECT
    C70.Sngc11tdoc AS TipoId,
    C70.Sngc11ndoc AS Identificacion,
    ISNULL(D001.Penom,'') AS Nombre_Razon_social,
    ISNULL(R003.Pftdo1,0) AS TipoId_RL,
    ISNULL(R003.Pfndo1,0) AS Identificacion_RL,
    ISNULL(DR001.Penom,'') AS Nombre_RL,
    ISNULL(R003.Pfpai1,0) AS Pais_Sede_principal
FROM sngc70 C70
LEFT JOIN FSD001 D001  ON D001.pepais=c70.sngc11pais AND D001.petdoc=C70.Sngc11tdoc AND D001.Pendoc=C70.Sngc11ndoc
LEFT JOIN FSR003 R003  ON R003.pjpais=c70.sngc11pais AND R003.pjtdoc=C70.Sngc11tdoc AND R003.Pjndoc=C70.Sngc11ndoc AND vicod=1
LEFT JOIN FSD001 DR001 ON DR001.pepais=R003.pfpai1 AND DR001.petdoc=R003.pftdo1 AND DR001.Pendoc=R003.Pfndo1
WHERE (C70.sngc70Atr='HSNGCPF1_CMBAUX4' AND C70.sngc70Val='3')
   OR (C70.sngc70Atr='HSNGCPJ1_CHECK_1' AND C70.sngc70Val='S');
```

### Formas de desembolso (Órdenes de Pago)

```sql
-- Referencia: Reporte Regulatorio (FSH016.Hcord)
WHEN FSH016.Hcord='83' AND FSH016.Htran<>981 THEN '(Orden de Pago)'
WHEN FSH016.Hcord='81' AND FSH016.Htran=981  THEN '(Orden de Pago)'
WHEN FSH016.Hcord='75' AND FSH016.Htran=982  THEN '(Orden de Pago)'
WHEN FSH016.Hcord='82' AND FSH016.Htran=981  THEN '(Convenio)'
WHEN FSH016.Hcord='84' AND FSH016.Htran<>981 THEN '(Con cheque)'
WHEN FSH016.Hcord='83' AND FSH016.Htran=981  THEN '(Con cheque)'
WHEN FSH016.Hcord='84' AND FSH016.Htran=981  THEN '(Transferencia)'
WHEN FSH016.Hcord='85' AND FSH016.Htran<>981 THEN '(Transferencia)'
```

---

## 30. Debug y Diagnóstico

```sql
-- Activar Debug (Opción General 2850)
SELECT * FROM fst098 WHERE TPCOD=2006 AND TPCORR=999
SELECT * FROM FST200 WHERE Pgcod=1 AND OpgCod=2850  -- Estado debug
SELECT * FROM FSADBG WHERE SADbgPrg='PJCCY023'      -- Registros debug
SELECT * FROM FSADBG WHERE SADbgUsu=' '             -- Debug por usuario

-- Foto del día anterior
SELECT TOP 10 * FROM dbo.RRCO03  -- Foto del día anterior de la FSD010

-- Facturas electrónicas
SELECT TOP 10 * FROM dbo.JCCI02  -- Consultar facturas electrónicas
```

### Análisis de hilos (FRTASKS)

```sql
SELECT TOP 100
    DATEDIFF(MINUTE, CONVERT(DATETIME, FRTskTimSt), CONVERT(DATETIME, FRTskTimEn)) AS diff, *
FROM dbo.FRTASKS
WHERE FRPrcId=2200 ORDER BY diff DESC;

SELECT TOP 100 * FROM dbo.FRTASKS
WHERE FRTskDsc LIKE '%Cartera%' ORDER BY FRTskTimCr DESC;
```

### Diagnóstico: Listas negras

```sql
-- Si tiene registros vacíos, las consultas de listas negras fallan para todos
SELECT TOP 10 * FROM dbo.FSD201 WHERE LnApeA='' AND LnNomA='';
```

### Consola de procesos

```
HFRPRCCONSOLE → Ver procesos en ejecución
HTRT200       → Debug
hchkenv001    → Validación de ambiente
HFRSERVICES   → Reiniciar para tomar cambios
```

---

*Documento generado a partir de `Rapidas_2.txt` — Bantotal Core Bancario. Curado y redactado el 2026-07-20, ver nota de curación al inicio de este archivo.*
