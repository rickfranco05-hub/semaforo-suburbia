# Estructura del Excel — Semáforo de Desempeño Comercial
> Archivo fuente: `Semaforo Actualizado enero 2026 - copia.xlsx`  
> Generado 2026-04-09. No duplica lo que ya está en CLAUDE.md (METRIC_MAP, flujo HTML, etc.).

---

## 1. Hojas del workbook

| Hoja | Rango | Filas datos | Propósito |
|---|---|---|---|
| `_com.sap.ip.bi.xl.hiddensheet` | — | — | Hoja oculta SAP BI, no tocar |
| `02_Div` | A1:DR53322 | ~53 K | Data warehouse por métrica × sección × mes. Fuente de todo. |
| `01_Consulta Secciones` | A1:P143033 | 143 031 | Pivot/query que el HTML lee directamente |
| `Secciones` | A1:AW111 | ~79 secciones | Dashboard semáforo nativo en Excel |

---

## 2. `02_Div` — Data warehouse

### 2.1 Layout de filas (fila = instancia de métrica × sección)

| Filas | Contenido |
|---|---|
| 1–4 | Encabezados y parámetros de control (mes activo, flags de mes futuro) |
| 5 | Fila de nombres de columnas (encabezados usables) |
| 6–N | Datos: una fila por métrica, una por sección. Patrón: todas las métricas primero para sección "#" (totales), luego para cada sección real. |

### 2.2 Columnas de identidad (fila 5 = header)

| Col | Header (fila 5) | Descripción |
|---|---|---|
| A | LLAVES | Clave primaria tipo 1: `seccion_cod + metrica` (ej. `"832Ventas Actual $"`) |
| B | 1 | Número índice |
| C | 2 | Número índice |
| D | SECCIÓN | Nombre de la sección (texto) |
| E | 4 | Nombre de la métrica |
| BB | SECCIÓN4 (`=BE5&BF5`) | Clave primaria tipo 2: `seccion_cod + metrica` — igual concepto que col A pero en bloque separado |
| BE | SECCIÓN | Parte izquierda de la clave BB (código sección) |
| BF | 4 | Parte derecha de la clave BB (nombre métrica) |

> Las claves A y BB son idénticas en concepto (`seccion + metrica`) pero sirven a diferentes grupos de columnas de resultado (ver §2.4).

### 2.3 Columnas de valores mensuales (bloque ENE–DIC)

| Col | Header (fila 5) | Valores |
|---|---|---|
| F–Q | `ENE FEB MAR ABR MAY JUN JUL AGO SEP OCT NOV DIC` | Valor raw de la métrica para cada mes del año **anterior** o según la lógica SAP |
| BG–BR | `ENE FEB MAR ABR MAY JUN JUL AGO SEP OCT NOV DIC` | Segundo bloque mensual (año en curso). Las fórmulas de resumen leen principalmente de aquí. |

### 2.4 Columnas de resultado (las que leen los SUMIF de `Secciones`)

| Col | Header (fila 5) | Fórmula | Qué contiene |
|---|---|---|---|
| T | Venta | `=SUM(F:Q)` (row-level) + `SUMIF($F$3:$Q$3,$T$2,F:Q)` | Valor del mes activo (single-month lookup por número de mes) |
| AN | (índice mes) | `=SUMIF($AO$5:$BA$5,$T$2,AO:BA)` | Inventario Actual del mes más reciente cerrado |
| BU | Vta Acum | `=IFERROR(SUMIF($BG$1:$BR$1,0,BG:BR),0)` | Ventas Acum YTD (suma meses con flag=0, es decir pasados) |
| BV | Promedio | `=IFERROR(AVERAGEIF($BG$2:$BS$2,0,BG:BS),0)` | Promedio de inventario para los meses pasados (denominador de Rotación Plan) |
| BX | Suma Inv | `=SUMIF($BG$3:$BS$3,$BW$3,BG:BS)` | Inventario Inicial del mes actual (snapshot point-in-time) |
| BY | Suma Inv | `=SUMIF($BG$3:$BS$3,$BX$3,BG:BS)` | Inventario Inicial del mes anterior (snapshot) |
| CL | Suma Meses | `=SUMIF($BZ$3:$CK$3,1,BZ:CK)` | Suma acumulada del año activo (meses con flag=1 en fila 3) |

### 2.5 Mecanismo de selección de mes activo

```
BV$2  = T2-1         → número del último mes cerrado (ej. 2 = FEB si mes activo = MAR)
T$2   = INDEX(…)     → mes activo actual (número, ej. 3 = MAR)
BG$1  = IF(BG3>BV$2+1,1,0)  → 1 si el mes es FUTURO (no aún cerrado)
BZ$3–CK$3  flags  → 1 si el mes pertenece al año en curso
```

Los flags en fila 1 (BG1:BR1) controlan qué meses entran en BU (Vta Acum YTD): solo los meses con flag=0 (ya cerrados).  
Los flags en fila 3 (BZ3:CK3) controlan qué deltas mensuales entran en CL (Suma Meses).

### 2.6 Bloque de métricas — catálogo completo en 02_Div

<details>
<summary>Ver todas las métricas en col E (clic para expandir)</summary>

```
Rotación Acum Mensual Actual / Año Ant / Plan Piedra / Proyectado
Utilidad Bruta Acum Actual $ / % | Año Ant $ / % | Plan Piedra $ / % | Plan Original $ / % | Proyectado $ / %
Utilidad Bruta Actual $ / % | Año Ant $ | Plan Piedra $ / % | Plan Original $ / %
Utilidad de lo Vendido Acum Actual $ / % | Año Ant $ / %
Ventas Actual $ | Acum Actual $ | Año Ant $ | Acum Año Ant $ | Plan Piedra $ | Acum Plan Piedra $
Ventas Plan Original $ | Acum Plan Original $ | Proyectado $ | Acum Proyectado $ | Plan Piedra Mes Futuro $
Inventario Inicial $ | Actual $ | Final Plan Piedra $ | Inicial Año Ant $ | Inicial Plan Piedra $
Inventario Promedio Acum Plan Piedra | Proyectado Control $ | Total
Inv Promedio Actual / Año Ant / Plan Piedra / Proyectado
Meses de Inventario Actual / Año Ant / Plan Piedra / Proyectado
ROI Acum Actual / Año Ant / Plan Piedra / Proyectado
Compra Total $ | Compra en Centro $ | Compras Totales Plan Piedra $ | Recepción en Centro $
Pedidos Pendientes / Recibidos (varios)
Total Markdowns Actual $ / Año Ant $ / Plan Piedra $ / Proyectado $ / Servicios $
Costo de MSI Actual $ / Año Ant $ / Plan Piedra $ / Proyectado $ / Servicios $
Filtraciones Actual $ / Año Ant $ / Plan Piedra $ / Proyectado $
OTB Total $ | In Transit Logístico $ | Mercancía Embarcada $
Ingresos / Utilidad / Gastos por Servicios (múltiples variantes)
Desviaciones % (vs Plan Piedra, vs Año Ant, vs Proyectado, varios)
```
</details>

---

## 3. `01_Consulta Secciones` — Fuente del HTML

### 3.1 Columnas (0-indexed)

| Índice | Contenido | Ejemplo |
|---|---|---|
| [0] | N (row number) | 1 |
| [1] | Mes (código) | ENE |
| [2] | Mes (duplicado) | ENE |
| [3] | Cód Dirección | D1 |
| [4] | **Dirección** (nombre) | SOFTLINE |
| [5] | **Cód División** | 14 |
| [6] | **División** (nombre) | HOMBRE |
| [7] | **Cód Sección** | 832 |
| [8] | **Sección** (nombre) | WEEKEND |
| [9] | **Nombre de métrica** | Inventario Inicial $ |
| [10] | **Valor numérico** | 411332.10686 |
| [11]–[12] | Vacíos / internos | — |
| [13] | Mes siguiente (texto) | FEB |
| [14] | Mes siguiente (número) | 2 |
| [15] | Mes siguiente (texto dup.) | FEB |

> **Filas de datos**: comienzan en fila 3 (índice 2 en 0-based). Filas 1–2 son encabezados (idénticos).  
> **Meses disponibles** en el archivo de ejemplo: ENE, FEB, MAR, ABR (4 meses = YTD abril 2026).  
> **Total filas** en el ejemplo: 143 031 (≈ 79 secciones × 4 meses × ~115 métricas/mes × variantes).

### 3.2 Métricas que el HTML consume (METRIC_MAP)

El HTML solo procesa 12 de las ~115 métricas disponibles en esta hoja:

| Métrica en col[9] | Campo JS interno | Tipo agregación |
|---|---|---|
| `Ventas Actual $` | `ventas_actual` | sum |
| `Ventas Plan Piedra $` | `ventas_plan` | sum |
| `Ventas Acum Actual $` | `vta_acum_actual` | sum |
| `Ventas Acum Plan Piedra $` | `vta_acum_plan` | sum |
| `Ventas Año Ant $` | `ventas_ano_ant` | sum |
| `Utilidad Bruta Acum Actual $` | `utilidad` | sum |
| `Utilidad Bruta Acum Plan Piedra $` | `utilidad_plan` | sum |
| `Utilidad Bruta Acum Año Ant $` | `utilidad_ano_ant` | sum |
| `Inventario Inicial $` | `inv_ini` | sum |
| `Inventario Inicial Plan Piedra $` | `inv_ini_plan` (sum) + `avg_inv_plan` (avg) |
| `Inventario Actual $` | `avg_inv_actual` | avg |
| `Inventario Inicial Año Ant $` | `avg_inv_aa` | avg |

### 3.3 Filas descartadas por el HTML (`parseExcel`)

| Condición | Motivo |
|---|---|
| `col[4] == '#'` o `col[6] == '#'` o `col[8] == '#'` | Filas de totales/separadores de SAP |
| `col[8] == 'COMPRAS ESPECIALES'` | Sección excluida por negocio |
| `col[9]` no está en METRIC_MAP | Métrica no relevante para el semáforo |

---

## 4. `Secciones` — Dashboard Excel nativo

### 4.1 Layout de la hoja

| Filas | Contenido |
|---|---|
| 1–6 | Parámetros, títulos, mes activo, pesos de calificación |
| 7 | Encabezados de columna (nombres visibles) |
| 8 | **Fila clave**: contiene los nombres de métrica usados como segunda parte de la llave SUMIF |
| 9–15 | Filas de subtotales / parámetros especiales (sección "#") |
| 16–18 | Filas de sección sin asignar / separadores |
| 19–111 | **Filas de datos**: una fila por sección real |

### 4.2 Columnas de la hoja `Secciones` (row 19 = ejemplo sección 832 WEEKEND)

| Col | Encabezado (fila 7) | Valor fila 8 / fórmula | Ejemplo (sec 832) |
|---|---|---|---|
| E | Sección | — | `832` (código sección — llave SUMIF) |
| F | — | — | `WEEKEND` (nombre) |
| G | Real Vta | `Ventas Actual $` | `=SUMIF('02_Div'!$A:$A,$E19&G$8,'02_Div'!$T:$T)` → 46 602 |
| H | Real Plan | `Ventas Plan Piedra $` | `=SUMIF('02_Div'!$A:$A,$E19&H$8,'02_Div'!$T:$T)` → 56 068 |
| I | Des Vs Plan | — | `=IFERROR((G19/H19-1)*100,"")` → −16.88% |
| J | Vta Real A.Ant | `Ventas Año Ant $` | `=SUMIF('02_Div'!$A:$A,$E19&J$8,'02_Div'!$T:$T)` → 53 705 |
| K | Des Vs A.Ant | — | `=IFERROR(((G19/J19)-1)*100,"")` → −13.23% |
| M | Mg Real | — | `=IFERROR((N19/G19)*100,"")` → −74.91% |
| N | Utilidad | `Utilidad Bruta Acum Actual $` | `=SUMIF('02_Div'!$BB:$BB,$E19&N$8,'02_Div'!$CL:$CL)` → −34 909 |
| O | Mg Plan % | — | `=IFERROR((P19/H19)*100,"")` → −80.71% |
| P | Utilidad Plan | `Utilidad Bruta Acum Plan Piedra $` | `=SUMIF('02_Div'!$BB:$BB,$E19&P$8,'02_Div'!$CL:$CL)` → −45 254 |
| Q | Des vs Plan (Mg) | — | `=IFERROR((M19-O19),"")` → +5.81 pp |
| R | Mg Año Ant | — | `=IFERROR((S19/J19)*100,"")` → −77.98% |
| S | Utilidad Año Ant | `Utilidad Bruta Acum Año Ant $` | `=SUMIF('02_Div'!$BB:$BB,$E19&S$8,'02_Div'!$CL:$CL)` → −41 878 |
| T | Des vs A.Ant (Mg) | — | `=IFERROR((M19-R19),"")` → +3.07 pp |
| V | Promedio Inventario | `Inventario Actual $` | `=SUMIF('02_Div'!$BB:$BB,$E19&V$8,'02_Div'!$AN:$AN)` → 355 360 |
| W | Vta Acum | `Ventas Actual $` | `=SUMIF('02_Div'!$BB:$BB,$E19&W$8,'02_Div'!$BU:$BU)` → 129 580 |
| X | Rotación Actual | — | `=IFERROR(W19/V19,"")` → 0.3646 |
| Y | Promedio Inv Plan | `Inventario Inicial Plan Piedra $` | `=SUMIF('02_Div'!$BB:$BB,$E19&Y$8,'02_Div'!$BV:$BV)` → 365 587 |
| Z | Vta Acum Plan | `Ventas Plan Piedra $` | `=SUMIF('02_Div'!$BB:$BB,$E19&Z$8,'02_Div'!$BU:$BU)` → 124 749 |
| AA | Rotación Plan | — | `=IFERROR(Z19/Y19,"")` → 0.3412 |
| AB | Des vs Plan (Rot) | — | `=IFERROR((X19-AA19),"")` → +0.0234 |
| AC | Inv Pro Año Ant | `Inventario Inicial Año Ant $` | `=SUMIF('02_Div'!$BB:$BB,$E19&AC$8,'02_Div'!$BV:$BV)` → 397 720 |
| AD | Vta Acum Año Ant | `Ventas Año Ant $` | `=SUMIF('02_Div'!$BB:$BB,$E19&AD$8,'02_Div'!$BU:$BU)` → 122 156 |
| AE | Rotación Año Ant | — | `=IFERROR(AD19/AC19,"")` → 0.3071 |
| AF | Des vs A.Ant (Rot) | — | `=IFERROR((X19-AE19),"")` → +0.0575 |
| AH | Peso Vta | — | `40` (fijo en celda) |
| AI | Peso Mg | — | `30` |
| AJ | Peso Rot | — | `25` |
| AK | Calif Vta | — | `=(G19/H19)*AH19` → 33.19 |
| AL | Calif Rot | — | `=(X19/AA19)*AJ19` → 26.72 |
| AM | Calif Mg | — | `=(M19/O19)*AI19` → 27.84 |
| AN | Calif General | — | `=IFERROR(SUM(AK19:AM19),0)` → 87.75 |
| AQ | Inv Ini Real | `Inventario Inicial $` | `=SUMIF('02_Div'!$BB:$BB,$E19&AQ$8,'02_Div'!$BX:$BX)` |
| AR | Inv Ini Plan | `Inventario Inicial Plan Piedra $` | `=SUMIF('02_Div'!$BB:$BB,$E19&AR$8,'02_Div'!$BX:$BX)` |

### 4.3 Los dos patrones SUMIF en `Secciones`

```
Tipo A  →  SUMIF('02_Div'!$A:$A,  $E_row & header_col$8, '02_Div'!$T:$T)
           Usa col A como llave, retorna col T (valor mes actual, single-month)
           Métricas: Ventas Actual $, Ventas Plan Piedra $, Ventas Año Ant $

Tipo BB →  SUMIF('02_Div'!$BB:$BB, $E_row & header_col$8, '02_Div'!$__:$__)
           Usa col BB como llave, retorna columna específica según métrica:
             $CL:$CL  → Utilidad Bruta Acum Actual/Plan/Año Ant (suma meses)
             $AN:$AN  → Inventario Actual $ (snapshot mes más reciente)
             $BU:$BU  → Ventas Acum YTD (Actual, Plan, Año Ant)
             $BV:$BV  → Promedio Inventario (denominador de Rotación)
             $BX:$BX  → Inventario Inicial $ snapshot (Real e Ini Plan)
             $BY:$BY  → Inventario Inicial Plan $ (versión alternativa)
```

---

## 5. Fórmulas de scoring en `Secciones`

### 5.1 Calificación individual (directa, sin redistribución)

```excel
Calif Vta  (AK) = (G / H) × Peso_Vta      = (Ventas Real / Ventas Plan) × 40
Calif Mg   (AM) = (M / O) × Peso_Mg       = (Mg Real % / Mg Plan %)    × 30
Calif Rot  (AL) = (X / AA) × Peso_Rot     = (Rot Actual / Rot Plan)    × 25
Calif Gen  (AN) = IFERROR(SUM(AK:AM), 0)
```

> **Nota clave**: el Excel usa directamente `(Mg Real % / Mg Plan %) × peso` (ratio de porcentajes), no la diferencia en pp. El HTML replica esta lógica.

### 5.2 Cálculo de Rotación — Excel original vs HTML

**Excel (`Secciones`):**
```excel
Rot Actual  = BU / AN    (Ventas Acum YTD / Inventario Actual snapshot)
Rot Plan    = BU / BV    (Ventas Acum Plan / AVERAGEIF inventarios plan)
Rot Año Ant = BU / BV    (Ventas Acum AA   / AVERAGEIF inventarios AA)
```

**HTML (`getAccumData`):**

```
Rot Actual  = vta_acum_YTD / ((Σ inv_ini_mes[1..N] + inv_actual_últimoMes) / (N+1))
Rot Plan    = vta_acum_plan / promedio(avg_inv_plan del período)    ← equiv. AVERAGEIF BV
Rot Año Ant = vta_acum_aa   / promedio(avg_inv_aa del período)      ← equiv. AVERAGEIF BV
```

Solo `rot_actual` usa la fórmula N+1. El punto N+1 es el Inventario Actual del último mes
(equivale al Inv Inicial del mes siguiente, o cierre de DIC para el último mes del año).

Ejemplos de denominador de `rot_actual`:
- ENE solo (N=1): `(Inv_Ini_ENE + Inv_Actual_ENE) / 2`
- ENE–MAR (N=3): `(Inv_Ini_ENE + Inv_Ini_FEB + Inv_Ini_MAR + Inv_Actual_MAR) / 4`
- Año completo (N=12): `(Inv_Ini_ENE…DIC + Inv_Actual_DIC) / 13`

### 5.3 Umbrales del semáforo

| Color | Calif General |
|---|---|
| 🟢 Verde | ≥ 100 |
| 🟡 Amarillo | ≥ 95 y < 100 |
| 🔴 Rojo | < 95 |
| ⚪ Sin Datos | = 0 o sin plan |

---

## 6. Flujo de datos completo (Excel nativo)

```
SAP BI
  └─► _com.sap.ip.bi.xl.hiddensheet  (raw dump, no modificar)
        │
        └─► 02_Div  (A1:DR53322)
              │  Cada fila = 1 métrica × 1 sección × valores por mes
              │  Columnas de resultado: T (mes actual), CL (suma meses),
              │  AN (inv actual), BU (Vta Acum YTD), BV (Prom Inv)…
              │
              ├─► Secciones  (dashboard Excel)
              │     SUMIF tipo A → col T   (mes actual)
              │     SUMIF tipo BB → col CL/AN/BU/BV/BX/BY
              │     Calcula scoring, semáforo, rotación, márgenes
              │
              └─► 01_Consulta Secciones  (pivot/query → fuente del HTML)
                    143 K filas, estructura flat: 1 fila por mes × sección × métrica
                    El HTML lee SOLO esta hoja
```

---

## 7. Diferencias clave entre Excel nativo y HTML

| Aspecto | Excel (`Secciones`) | HTML (`semaforo_v2.html`) |
|---|---|---|
| Fuente de datos | `02_Div` via SUMIF | `01_Consulta Secciones` via SheetJS |
| Granularidad | Un valor por sección (mes activo fijo) | Multi-mes, seleccionable en filtro |
| Mg Real denominador | `Ventas Actual $` (mes actual, col G) | `ventas_actual` sumado en el período |
| Mg Plan denominador | `Ventas Plan Piedra $` (mes actual, col H) | `ventas_plan` sumado |
| Rot Actual denominador | Inv Actual snapshot (col AN) | N+1 avg: `(Σ inv_ini + inv_actual_último) / (N+1)` |
| Rot Plan denominador | AVERAGEIF Inv Plan (col BV) | Promedio `avg_inv_plan` del período (equiv. AVERAGEIF) |
| Rot Año Ant denominador | AVERAGEIF Inv AA (col BV) | Promedio `avg_inv_aa` del período (equiv. AVERAGEIF) |
| Número de secciones | ~79 secciones fijas en hoja | Dinámico, según Excel cargado |
| Pesos scoring | En celda (AH, AI, AJ) por fila | Inputs readonly en UI (mismos valores) |
| Calif Mg fórmula | `(Mg_Real% / Mg_Plan%) × peso` | `(mg_real / mg_plan) × wMg` (idéntico) |
| Utilidad Plan (col P) | `SUMIF($A,$E&P$8,$T)` con P8 = `"Utilidad Bruta Plan Piedra $"` | `utilidad_plan` = `Utilidad Bruta Plan Piedra $` (mensual) o `Utilidad Bruta Acum Plan Piedra $` (toggle en UI) |

---

## 8. Mecanismo Acum vs no-Acum en 02_Div — crítico

### 8.1 El bloque BZ:CK y la columna CL

Cada fila de `02_Div` tiene valores mensuales en dos bloques:

| Bloque | Cols | Header fila 5 | Propósito |
|---|---|---|---|
| BG:BR | idx 58–69 | ENE…DIC | Valores que SAP entrega para cada mes |
| BZ:CK | idx 77–88 | Enero…Diciembre | Deltas calculados (ver abajo) |

Las fórmulas en BZ:CK son siempre deltas:
```
BZ = BG              (Enero: primer valor directo)
CA = BH - BG         (Febrero: BH minus BG)
CB = BI - BH         (Marzo:   BI minus BH)
… y así sucesivamente
```

La columna **CL** (`Suma Meses`) = `SUMIF(BZ3:CK3, 1, BZ:CK)` — suma solo las celdas de BZ:CK donde el flag de fila 3 = 1.  
**El flag es 1 únicamente para el mes activo** (en marzo → solo CB = flag 1).  
Por tanto: **CL = valor del mes activo únicamente** (no es YTD, es solo el mes actual).

### 8.2 Por qué CL funciona SOLO para métricas Acum

| Tipo de métrica en BG:BR | Ejemplo: Marzo (BI) | Delta CB = BI − BH | CL = CB |
|---|---|---|---|
| **Acum** (SAP entrega YTD acumulado) | `BI` = Ene+Feb+Mar | `(Ene+Feb+Mar) − (Ene+Feb)` = **Mar mensual** ✓ | Valor mensual correcto |
| **No-Acum** (SAP entrega solo ese mes) | `BI` = Mar mensual | `Mar mensual − Feb mensual` = **delta incorrecto** ✗ | Diferencia entre meses, no el valor de Marzo |

> **Conclusión**: el SUMIF `→ $CL:$CL` está diseñado exclusivamente para métricas Acum (donde SAP entrega YTD). Usarlo con métricas no-Acum produce un valor incorrecto (diferencia entre meses consecutivos).

### 8.3 Columna correcta para métricas no-Acum

Para métricas no-Acum usa el **tipo A de SUMIF** (col A como llave, col T como resultado):
```excel
=SUMIF('02_Div'!$A:$A, $E19 & "NombreMetrica", '02_Div'!$T:$T)
```

Col T = `SUMIF($F$3:$Q$3, $T$2, F:Q)` — busca el valor del mes activo en el bloque F:Q (datos originales SAP).  
Este patrón es el mismo que usan Ventas Actual $, Ventas Plan Piedra $, Ventas Año Ant $ (cols G, H, J en Secciones).

### 8.4 Caso concreto: Utilidad Plan

| Métrica SAP | SAP entrega | SUMIF correcto | Columna resultado |
|---|---|---|---|
| `Utilidad Bruta Acum Plan Piedra $` | YTD acumulado mensual | `SUMIF($BB, key, $CL)` | CL (delta = mensual) |
| `Utilidad Bruta Plan Piedra $` | Valor mensual directo | `SUMIF($A, key, $T)` | T (valor directo del mes) |

Si los datos SAP de la versión Acum están mal, **cambiar solo P8 no basta** — hay que cambiar también el patrón de fórmula (de `$BB→$CL` a `$A→$T`).

Verificado: `SUM(col T)` para `Utilidad Bruta Plan Piedra $` en todos los registros = ~603,101 (correcto); `SUM(col CL)` para la misma métrica = −473,383 (incorrecto por el delta).

### 8.5 Toggle en el HTML

El HTML (`semaforo_v2.html`) tiene un botón **"Util Plan: Mensual / Acum"** en la barra de filtros para elegir la fuente de `utilidad_plan` en tiempo real:
- **Mensual** (default): usa `Utilidad Bruta Plan Piedra $` del Excel
- **Acum**: usa `Utilidad Bruta Acum Plan Piedra $` del Excel

El toggle recalcula `mg_plan`, `des_vs_plan_mg`, scoring y semáforo para todos los registros sin necesidad de recargar el archivo.
