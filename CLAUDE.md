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
| Columna Excel (row[9]) | Campo interno JS |
|---|---|
| Ventas Actual $ | `ventas_actual` (sum) |
| Ventas Plan Piedra $ | `ventas_plan` (sum) |
| Ventas Acum Actual $ | `vta_acum_actual` (sum) |
| Ventas Acum Plan Piedra $ | `vta_acum_plan` (sum) |
| Ventas Año Ant $ | `ventas_ano_ant` (sum) |
| Utilidad Bruta Acum Actual $ | `utilidad` (sum) |
| Utilidad Bruta Acum Plan Piedra $ | `utilidad_plan` (sum) |
| Utilidad Bruta Acum Año Ant $ | `utilidad_ano_ant` (sum) |
| Inventario Inicial $ | `inv_ini` (sum) |
| Inventario Inicial Plan Piedra $ | `inv_ini_plan` (sum) + `avg_inv_plan` (avg) |
| Inventario Actual $ | `avg_inv_actual` (avg) |
| Inventario Inicial Año Ant $ | `avg_inv_aa` (avg) |

### MES_ORDER
`['ENE','FEB','MAR','ABR','MAY','JUN','JUL','AGO','SEP','OCT','NOV','DIC']`

### NIVEL — umbrales del semáforo
| Nivel | Condición |
|---|---|
| Verde | calif_general ≥ 100 |
| Amarillo | calif_general ≥ 95 |
| Rojo | calif_general < 95 |
| Sin Datos | calif_general == 0 o null |

> ⚠️ El README dice 90/75/60 pero el código usa 100/95. El código manda.

---

## Flujo de datos

```
Excel (.xlsx)
  → SheetJS → raw rows[]
  → parseExcel()  → allData[]       (1 objeto por mes × dirección × división × sección)
  → onFilter()    → filteredData[]  (subconjunto según filtros activos)
  → getAccumData()→ rows agregados  (1 objeto por dirección × división × sección, suma/promedio multi-mes)
  → renderAll()   → DOM
```

**Importante**: `allData` y `filteredData` son per-mes. `getAccumData()` agrega y es lo que usan los charts y la tabla.

---

## Campos derivados (calculados en parseExcel / getAccumData)

| Campo | Fórmula |
|---|---|
| `vta` | = `ventas_actual` |
| `vtaPlan` | = `ventas_plan` |
| `mg_real` | = `utilidad / ventas_actual` (ratio, e.g. 0.35 = 35%) |
| `mg_plan` | = `utilidad_plan / ventas_plan` |
| `rot_actual` | = `ventas_actual / avg_inv_actual` |
| `rot_plan` | = `ventas_plan / avg_inv_plan` |
| `des_vs_plan_vta` | = `(vta/vtaPlan - 1) × 100` |
| `des_vs_plan_mg` | = `mg_real - mg_plan` (diferencia en pp) |
| `des_vs_aa_mg` | = `mg_real / mg_ano_ant` (ratio, se muestra como %) |
| `des_vs_plan_rot` | = `rot_actual - rot_plan` (delta) |

---

## Scoring (pesos dinámicos)

```js
computeScoring(vta, vtaPlan, mg_real, mg_plan, rot_actual, rot_plan)
```
- Pesos: `wVta=45`, `wMg=35`, `wRot=20` (fijos en pantalla, readonly)
- Si no hay plan de rotación o wRot=0: el peso de rotación se redistribuye proporcionalmente entre vta y mg
- `calif_general = calif_vta + calif_mg + calif_rot`
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
- Sort multi-columna (click encabezado; 3er click resetea al orden default)
- Paginación (25 filas por página)
- Tooltip al hover: desglose completo de la sección (ventas, margen, rotación, inventario, calificación)
- Toggle para ocultar/mostrar columnas de calificación individual
- Export CSV (38 columnas)

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
- Campos de snapshot (inv inicial, mes display para tooltip) → **mes más reciente** del grupo

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
