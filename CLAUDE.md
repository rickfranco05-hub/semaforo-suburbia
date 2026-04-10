# CLAUDE.md — Semáforo de Desempeño Comercial

## Archivo activo
**`semaforo_v2.html`** es la versión en desarrollo activa (tema claro/rosa, Suburbia).
- `semaforo_desempeno.html` — versión anterior (tema oscuro), no se toca
- `semaforo_desempeno_OFFLINE.html` — generado por `_build_offline.js`, no editar a mano

## Deploy — GitHub Pages
El sitio público es **https://rickfranco05-hub.github.io/semaforo-suburbia/** y sirve `index.html`.

> ⚠️ **REGLA OBLIGATORIA**: Cada vez que se haga `git push`, se debe copiar `semaforo_v2.html` a `index.html` antes del commit. Sin esto, los cambios no se ven en el link público.

Secuencia correcta de push:
```bash
cp semaforo_v2.html index.html
git add semaforo_v2.html index.html
git commit -m "mensaje"
git push origin main
```

## Arquitectura
HTML + CSS + JS vanilla en un único archivo. Sin frameworks. Dependencias vía CDN:
- **SheetJS 0.18.5** — lee el `.xlsx` en el navegador (sin servidor)
- **Chart.js 4.4.0** — gráficas
- **DM Sans + JetBrains Mono** — Google Fonts

Todo el procesamiento ocurre en el navegador. Ningún dato sale al exterior.
El logo de Suburbia está **incrustado en Base64** directamente en el HTML (no requiere el `.jpg` externo).

---

## Estructura del Excel esperada

- **Hoja requerida**: `01_Consulta Secciones` (nombre exacto)
- **Datos desde fila 3** (índice 2 en 0-based), las primeras 2 filas son encabezados
- **Columnas por posición** (0-indexed):

| Índice | Campo |
|--------|-------|
| [1]    | Mes (`ENE`, `FEB`, … `DIC`) |
| [4]    | Dirección |
| [5]    | Cód División |
| [6]    | División |
| [7]    | Cód Sección |
| [8]    | Sección |
| [9]    | Nombre de métrica (ver METRIC_MAP) |
| [10]   | Valor numérico |

- Filas con dirección/división/sección == `'#'` se descartan
- Filas con sección `'COMPRAS ESPECIALES'` se descartan
- Solo se procesan métricas que estén en `METRIC_MAP`

---

## Constantes clave

### METRIC_MAP — métricas reconocidas del Excel

> Auditado 2026-04-10 contra la fila 8 de la hoja `Secciones` del Excel. Los nombres de métricas en col[9] de `01_Consulta Secciones` son los mismos que usa SAP.

| Columna Excel (row[9]) | Campo interno JS | Tipo | Equivalente Excel Secciones |
|---|---|---|---|
| Ventas Actual $ | `ventas_actual` | sum | Col G (Real Vta) — SUMIF $A→$T |
| Ventas Plan Piedra $ | `ventas_plan` | sum | Col H (Real Plan) — SUMIF $A→$T |
| Ventas Acum Actual $ | `vta_acum_actual` | sum | Col W (Vta Acum, Rot Actual) — SUMIF $BB→$BU |
| Ventas Acum Plan Piedra $ | `vta_acum_plan` | sum | Col Z (Vta Acum Plan, Rot Plan) — SUMIF $BB→$BU |
| Ventas Acum Año Ant $ | `vta_acum_ano_ant` | sum | Col AD (Vta Acum AA, Rot AA) — SUMIF $BB→$BU |
| Ventas Año Ant $ | `ventas_ano_ant` | sum | Col J (Vta Real A.Ant) — SUMIF $A→$T |
| Utilidad Bruta Actual $ | `utilidad` | sum | Col N usa `Acum Actual $`+CL delta; HTML usa Mensual directo = mismo resultado |
| Utilidad Bruta Acum Plan Piedra $ | `utilidad_plan_acum` | sum | Col P — SUMIF $BB→$CL (primario, con delta) |
| Utilidad Bruta Plan Piedra $ | `utilidad_plan_mensual` | sum | Fallback para `utilidad_plan` cuando SAP no tiene Acum |
| Utilidad Bruta Año Ant $ | `utilidad_ano_ant` | sum | Col S usa `Acum Año Ant $`+CL delta; HTML usa Mensual directo = mismo resultado |
| Inventario Inicial $ | `inv_ini` | sum | Col AQ — SUMIF $BB→$BX |
| Inventario Inicial Plan Piedra $ | `inv_ini_plan` (sum) + `avg_inv_plan` (avg) | sum+avg | Col AR (ini) + Col Y (Rot Plan denom) — $BX y $BV |
| Inventario Actual $ | `avg_inv_actual` | avg | Col V (Prom Inventario, Rot Actual denom) — SUMIF $BB→$AN |
| Inventario Inicial Año Ant $ | `avg_inv_aa` | avg | Col AC (Inv Pro Año Ant, Rot AA denom) — SUMIF $BB→$BV |

### Util Plan Toggle (⚙ Ajustes → "Util Plan: Acum/Mensual")

El campo `utilidad_plan` se determina por el toggle en el icono de ajustes (⚙):

- **Modo Acum (default)**: usa `utilidad_plan_acum` (`Utilidad Bruta Acum Plan Piedra $`). Como es YTD acumulado de SAP, se aplica mecanismo de **delta** para extraer el valor mensual: `mes[n] = acum[n] - acum[n-1]`. Esto replica la columna CL del Excel.
- **Modo Mensual (fallback)**: usa `utilidad_plan_mensual` (`Utilidad Bruta Plan Piedra $`) directo.

Al cambiar modo, recalcula en cadena: `utilidad_plan` → `mg_plan` → `des_vs_plan_mg` → `calif_mg` → `calif_general` → `nivel`.

### MES_ORDER
`['ENE','FEB','MAR','ABR','MAY','JUN','JUL','AGO','SEP','OCT','NOV','DIC']`

### NIVEL — umbrales del semáforo
| Nivel | Condición |
|---|---|
| Verde | calif_general ≥ 100 |
| Amarillo | calif_general ≥ 95 |
| Rojo | calif_general < 95 |
| Sin Datos | calif_general == 0 o null |

---

## Flujo de datos

```
Excel (.xlsx)
  → SheetJS → raw rows[]
  → parseExcel()       → allData[]       (1 obj por mes × dir × div × sec)
  → applyUtilPlanMode()→ allData[]       (asigna utilidad_plan según modo, recalcula scoring)
  → initDashboard()    → tooltipIndex, filtros, render
  → onFilter()         → filteredData[]  (subconjunto según filtros activos)
  → getAccumData()     → rows agregados  (1 obj por dir × div × sec, suma/promedio multi-mes)
  → renderAll()        → DOM
```

**Importante**: `allData` y `filteredData` son per-mes. `getAccumData()` agrega y es lo que usan los charts y la tabla.

---

## Campos derivados (calculados en parseExcel / getAccumData)

### Ventas
| Campo | Fórmula | Excel equiv |
|---|---|---|
| `vta` | = `ventas_actual` | Col G |
| `vtaPlan` | = `ventas_plan` | Col H |
| `vtaAA` | = `ventas_ano_ant` | Col J |
| `des_vs_plan_vta` | = `(vta/vtaPlan - 1) × 100` | Col I: `=(G/H-1)*100` |
| `des_vs_aa_vta` | = `(vta/vtaAA - 1) × 100` | Col K: `=((G/J)-1)*100` |

### Márgenes (almacenados como ratio, e.g. 0.35 = 35%)
| Campo | Fórmula | Excel equiv |
|---|---|---|
| `mg_real` | = `utilidad / vta` | Col M: `=(N/G)*100` |
| `mg_plan` | = `utilidad_plan / vtaPlan` | Col O: `=(P/H)*100` |
| `mg_ano_ant` | = `utilidad_ano_ant / vtaAA` | Col R: `=(S/J)*100` |
| `des_vs_plan_mg` | = `mg_real - mg_plan` (diferencia en pp) | Col Q: `=M-O` |
| `des_vs_aa_mg` | = `mg_real - mg_ano_ant` (diferencia en pp) | Col T: `=M-R` |

### Rotación (numerador = Ventas Acum YTD snapshot, no mensual)
| Campo | Fórmula | Excel equiv |
|---|---|---|
| `rot_actual` | = `vta_acum_actual / avg_inv_actual` | Col X: `=W/V` (W=BU, V=AN) |
| `rot_plan` | = `vta_acum_plan / avg_inv_plan` | Col AA: `=Z/Y` (Z=BU, Y=BV) |
| `rot_ano_ant` | = `vta_acum_ano_ant / avg_inv_aa` | Col AE: `=AD/AC` (AD=BU, AC=BV) |
| `des_vs_plan_rot` | = `rot_actual - rot_plan` | Col AB: `=X-AA` |
| `des_vs_aa_rot` | = `rot_actual - rot_ano_ant` | Col AF: `=X-AE` |

> **Asimetría en denominadores** (igual que el Excel): Rot Actual usa inventario snapshot (AN). Rot Plan y Rot Año Ant usan promedio/AVERAGEIF (BV).

> **Guard MIN_INV**: Si `|denominador| ≤ 1`, rotación = null (evita div/0 por artefactos SAP como inv=-0.00001). Equivale al `IFERROR` del Excel.

> **Totales de rotación**: Se calculan como `Σ(numeradores) / Σ(denominadores)` (no promedio ponderado de ratios). Cada fila almacena `_rot_num_*` y `_rot_den_*` para que `computeTotals` sume correctamente. Esto replica el Excel donde el total = SUMIF total / SUMIF total.

---

## Scoring (pesos dinámicos)

```js
computeScoring(vta, vtaPlan, mg_real, mg_plan, rot_actual, rot_plan)
```
- Pesos: `wVta=40`, `wMg=30`, `wRot=25` (fijos en pantalla, readonly)
- `calif_vta = (vta / vtaPlan) × wVta` — Excel AK: `=(G/H)*AH`
- `calif_mg  = (mg_real / mg_plan) × wMg` — Excel AM: `=(M/O)*AI`
- `calif_rot = (rot_actual / rot_plan) × wRot` — Excel AL: `=(X/AA)*AJ`
- `calif_general = calif_vta + calif_mg + calif_rot` — Excel AN: `=SUM(AK:AM)`
- Si no hay plan de rotación o wRot=0: el peso de rotación se redistribuye proporcionalmente entre vta y mg
- Sin vtaPlan o mg_plan → `calif_general = 0` → `nivel = 'sin_datos'`

---

## Colores en tabla

| Métrica | Verde | Warn | Rojo |
|---|---|---|---|
| Des vs Plan Vta | ≥ 0% | [-10%, 0%) | < -10% |
| Des vs Plan Mg | ≥ 0pp | [-3pp, 0) | < -3pp |
| Des vs Plan Rot | ≥ 0 | < 0 | — |
| Otros (Des vs AA) | > 0 | — | < 0 |

---

## Vistas del dashboard

### Página 1 — Dashboard (`btn-p1`)
- **5 scorecards**: Ventas Acum Real, Utilidad Bruta Acum, Margen Real %, Rotación Promedio, Calificación Promedio
- **Gráficas**:
  - `ch-donut` — distribución del semáforo (verde/amarillo/rojo/sin datos)
  - `ch-vta-div` — ventas reales por división (barras)
  - `ch-dist-bar` — barras de distribución de secciones por nivel
  - `ch-vta-inv` — ventas vs inventario (top secciones)
  - `ch-inv-plan-real` — inventario inicial plan vs real por división

### Página 2 — Detalle (`btn-p2`)
- Tabla con todas las secciones, subtotales por división y dirección, gran total
- **Freeze panes**: las 2 filas de thead (grupos + columnas) y la fila de Gran Total quedan fijas al scrollear (`.table-wrap` es el scroll container con `max-height`, sticky via CSS+JS `updateStickyOffsets`)
- Sort multi-columna (click encabezado; 3er click resetea al orden default)
- Tooltip al hover: desglose completo de la sección
- Toggle para ocultar/mostrar columnas de calificación individual
- Export CSV

### Página 3 — Insights (`btn-p3`)
- **Brecha al Plan** — Top 25 secciones por mayor desviación negativa vs plan
- **Tendencia Mensual** — Evolución de KPI por mes (requiere 2+ meses)
- **Pareto 80/20** — Análisis de concentración de ventas

### Barra superior
- Navegación: Dashboard / Detalle / Insights
- **⚙ Ajustes** (dropdown): contiene el toggle "Util Plan: Acum/Mensual"
- Exportar CSV / Otro archivo

---

## Filtros

| Filtro | Tipo | Comportamiento |
|---|---|---|
| Dirección | `<select>` | Single, filtra divisiones disponibles |
| División | `<select>` | Single, dependiente de Dirección |
| Mes | Slicer (botones pill) | **Multi-selección** — sin selección = todos los meses |
| Sección | `<input>` texto | Búsqueda parcial case-insensitive |

`clearFilters()` limpia todo y llama `onFilter()`.

---

## Agregación multi-mes (`getAccumData`)

Agrupa `filteredData` por `direccion × division × seccion`:
- Ventas, Utilidad → **suma** acumulada
- Inventarios → **promedio** entre meses del período
- Ventas Acum (rotación) → **snapshot del mes más reciente** (`_vta_acum_latest`, `_vta_acum_plan_latest`, `_vta_acum_aa_latest`)
- Inventario Actual (rot actual denom) → **snapshot del mes más reciente** (`_ai_latest`)
- Inv Plan/AA (rot plan/aa denom) → **promedio** (equiv. AVERAGEIF del Excel)

---

## Build / scripts

```bash
npm install              # solo primera vez (instala xlsx para Node)
node _build_offline.js   # genera semaforo_desempeno_OFFLINE.html con CDN embebido
node build_semaforo.js   # embebe logo desde logo_b64.txt (para semaforo_desempeno.html)
```

El logo en `semaforo_v2.html` ya está incrustado en Base64 directamente — no necesita build script.

---

## Notas técnicas

- `esc()` — helper XSS para todas las interpolaciones de strings en el DOM
- `tooltipIndex` — mapa `"mes||dir||div||sec"` → objeto row; se puebla en `initDashboard()`; el tooltip de la tabla lo usa para mostrar datos del mes original (no del acumulado)
- `charts{}` — objeto global con referencias a instancias Chart.js; siempre se destruyen antes de redibujar (`killChart()`)
- `DEFAULT_SORT` — `[direccion asc, div_cod asc, sec_cod asc]`
- Filas vacías (`calif_general === 0 && vta === 0`) se excluyen en `onFilter()`
- El archivo HTML pesa ~1-2 MB por el Base64 del logo — es normal
- **`border-collapse: separate`** en `#main-table` — necesario para que los bordes de celdas sticky se rendericen correctamente al scrollear
- **`updateStickyOffsets()`** — mide alturas del thead con `getBoundingClientRect()` y asigna `top` dinámico a la fila de columnas y al Gran Total. Se llama con `requestAnimationFrame` en `renderTable()` y al entrar a Page 2 (porque en `display:none` las alturas son 0)
