# Reporte de Columnas — Página de Detalle (`semaforo_v2.html`)

Documento generado el 2026-03-27. Describe todas las columnas visibles en la tabla de la Página 2 (Detalle), sus campos internos en JS, cómo se calculan y su comportamiento de color.

---

## Grupos de columnas

| Grupo | Color encabezado | Cantidad de columnas |
|---|---|---|
| Dimensiones | Oscuro (`#2D0A1A`) | 3 |
| Venta | Rosa (`#D93690`) | 5 |
| Margen (GM) | Morado (`#67308C`) | 8 |
| Rotación | Teal (`#0E7490`) | 5 |
| Calificación | Lila (`#A78FBF`) — **oculta por default** | 4 |
| Semáforo | Rosa acento (`#E5007E`) | 1 |

---

## Dimensiones (3 columnas)

| # | Encabezado tabla | Campo JS | Fuente | Descripción |
|---|---|---|---|---|
| 1 | Dir. | `direccion` | Excel col [4] | Nombre de la dirección comercial |
| 2 | División | `division` / `div_cod` | Excel col [6] / [5] | Nombre y código de la división |
| 3 | Sección | `seccion` / `sec_cod` | Excel col [8] / [7] | Nombre y código de la sección |

> Las columnas División y Sección muestran el código como subíndice debajo del nombre.

---

## Venta (5 columnas)

| # | Encabezado tabla | Campo JS | Fórmula | Tipo | Color |
|---|---|---|---|---|---|
| 4 | Real Vta | `vta` | `SUM(ventas_actual)` sobre el período | Suma de meses | Sin color condicional |
| 5 | Real Plan | `vtaPlan` | `SUM(ventas_plan)` sobre el período | Suma de meses | Sin color (gris) |
| 6 | Des Vs Plan | `des_vs_plan_vta` | `(vta / vtaPlan − 1) × 100` | % | Verde ≥ 0%, Amarillo [-10%, 0%), Rojo < -10% |
| 7 | Vta Real A.Ant | `vtaAA` | `SUM(ventas_ano_ant)` sobre el período | Suma de meses | Sin color (gris) |
| 8 | Des Vs A.Ant | `des_vs_aa_vta` | `(vta / vtaAA − 1) × 100` | % | Verde > 0%, Rojo < 0% |

---

## Margen — GM (8 columnas)

| # | Encabezado tabla | Campo JS | Fórmula | Tipo | Color |
|---|---|---|---|---|---|
| 9 | Mg Real % | `mg_real` | `utilidad / vta` | Ratio (se muestra como %) | Sin color condicional |
| 10 | Utilidad | `utilidad` | `SUM(utilidad_bruta_actual)` sobre el período | Suma de meses | Sin color (gris) |
| 11 | Mg Plan % | `mg_plan` | `utilidad_plan / vtaPlan` | Ratio (se muestra como %) | Sin color (gris) |
| 12 | Util Plan | `utilidad_plan` | `SUM(utilidad_bruta_plan)` sobre el período | Suma de meses | Sin color (gris) |
| 13 | Des vs Plan | `des_vs_plan_mg` | `mg_real − mg_plan` (diferencia en pp) | Puntos porcentuales | Verde ≥ 0 pp, Amarillo [-3 pp, 0), Rojo < -3 pp |
| 14 | Mg Año Ant | `mg_ano_ant` | `utilidad_ano_ant / vtaAA` | Ratio (se muestra como %) | Sin color (gris) |
| 15 | Util Año Ant | `utilidad_ano_ant` | `SUM(utilidad_bruta_ano_ant)` sobre el período | Suma de meses | Sin color (gris) |
| 16 | Des vs A.Ant | `des_vs_aa_mg` | `mg_real − mg_ano_ant` (diferencia en pp) | Puntos porcentuales | Verde ≥ 0 pp, Rojo < 0 pp |

> **Nota importante sobre `des_vs_aa_mg`:** El campo JS almacena la diferencia como ratio/decimal (ej. `0.05 = 5 pp`), pero se muestra en pantalla multiplicado × 100 con sufijo " pp". En el CSV se exporta también × 100.

---

## Rotación (5 columnas)

La rotación usa la **fórmula N+1 de promedio de inventario**: para N meses seleccionados, el denominador es el promedio de N+1 puntos de inventario (N inventarios iniciales + el inventario actual/cierre del último mes del período).

**Ejemplo**: ENE–MAR (N=3) → `(Inv_Ini_ENE + Inv_Ini_FEB + Inv_Ini_MAR + Inv_Actual_MAR) / 4`

Esta fórmula es la práctica contable estándar: el inventario actual del último mes equivale al inventario inicial del mes siguiente (o al cierre de diciembre).

| # | Encabezado tabla | Campo JS | Fórmula | Tipo | Color |
|---|---|---|---|---|---|
| 17 | Rot Actual | `rot_actual` | `vta_acum_YTD / ((Σ Inv_Ini_mensual + Inv_Actual_últimoMes) / (N+1))` | Veces | Sin color condicional |
| 18 | Rot Plan | `rot_plan` | `vta_acum_plan_YTD / ((Σ Inv_Ini_Plan_mensual + Inv_Ini_Plan_últimoMes) / (N+1))` | Veces | Sin color (gris) |
| 19 | Des vs Plan | `des_vs_plan_rot` | `rot_actual − rot_plan` (delta) | Delta veces | Verde ≥ 0, Rojo < 0 |
| 20 | Rot Año Ant | `rot_ano_ant` | `ventas_ano_ant_suma / ((Σ Inv_Ini_AA_mensual + Inv_Ini_AA_últimoMes) / (N+1))` | Veces | Sin color (gris) |
| 21 | Des vs A.Ant | `des_vs_aa_rot` | `rot_actual − rot_ano_ant` (delta) | Delta veces | Verde ≥ 0, Rojo < 0 |

> **Acumuladores internos**: `_inv_ini_s/_inv_ini_n` (Inv Ini $ para rot_actual), `_ap_s/_ap_n` (plan), `_aa_s/_aa_n` (año ant). El punto N+1 usa `_ai_latest`, `_ap_latest`, `_aa_latest` (snapshot del último mes).

> **Nota:** En las filas de total (División, Dirección, Gran Total), `Rot Año Ant` muestra "—".

---

## Calificación — oculta por default (4 columnas)

Se activan/ocultan haciendo clic en el botón "+" del encabezado "Semáforo". Los pesos son configurables en pantalla (defaults: Vta=45, Mg=35, Rot=20). Si no hay plan de rotación o wRot=0, su peso se redistribuye proporcionalmente entre Vta y Mg.

| # | Encabezado tabla | Campo JS | Fórmula |
|---|---|---|---|
| 22 | Calif Vta | `calif_vta` | `(vta / vtaPlan) × wVta` — o ajustado si wRot=0 |
| 23 | Calif Mg | `calif_mg` | `(mg_real / mg_plan) × wMg` — o ajustado si wRot=0 |
| 24 | Calif Rot | `calif_rot` | `(rot_actual / rot_plan) × wRot` — o 0 si sin plan |
| 25 | Calif Gen | `calif_general` | `calif_vta + calif_mg + calif_rot` |

> Si no hay `vtaPlan` o `mg_plan`, `calif_general = 0` → nivel `sin_datos`.
> En filas de total, estas columnas muestran "—" (no se agregan).

---

## Semáforo (1 columna)

| # | Encabezado tabla | Campo JS | Lógica |
|---|---|---|---|
| 26 | Semáforo | `nivel` | Verde: `calif_general ≥ 100` · Amarillo: `≥ 95` · Rojo: `< 95` · Sin Datos: `= 0` o null |

---

## Filas especiales en la tabla

Además de las filas de sección, la tabla incluye:

| Tipo de fila | Cuándo aparece | Color de fondo |
|---|---|---|
| **Total División** | Al cambiar de división | Morado (`#6C3B8C`) |
| **Total Dirección** | Solo cuando hay más de una dirección visible | Morado oscuro (`#67308C`) |
| **Gran Total** | Siempre, al final | Rosa (`#D93690`) |

Los totales calculan rotación como promedio ponderado por ventas entre las secciones del grupo. Las columnas de Calificación y Semáforo muestran "—" en filas de total.

---

## Columnas adicionales en el Tooltip (hover)

El tooltip al hacer hover sobre una fila muestra datos extras que **no están en la tabla principal** pero sí en el CSV:

| Campo JS | Etiqueta tooltip | Descripción |
|---|---|---|
| `vta_acum_actual` | Vta Acum Actual | Ventas acumuladas YTD del SAP (snapshot último mes) |
| `vta_acum_plan` | Vta Acum Plan | Ventas acumuladas plan YTD (snapshot último mes) |
| `inv_ini_real` | Inv Inicial Real | Inventario Inicial $ del mes más reciente |
| `inv_ini_plan_val` | Inv Ini Plan | Inventario Inicial Plan Piedra $ del mes más reciente |
| `inv_pro_aa` | Inv Pro Año Ant | Promedio de `Inventario Inicial Año Ant $` del período |
| `prom_inv_act` | Prom Inv Año Act | Promedio de `Inventario Actual $` del período |
| `des_vs_plan_inv_ini` | Des vs Plan Inv Ini | `(inv_ini_real / inv_ini_plan_val − 1) × 100` |
| `record_count` | Record Count | Número de filas del Excel que componen este grupo |

---

## Columnas del Export CSV (38 columnas)

El CSV incluye todo lo anterior más campos que no aparecen en la tabla visual:

| # | Encabezado CSV | Campo JS | Notas |
|---|---|---|---|
| 1 | Mes | `mes` | Mes más reciente del período |
| 2 | Dirección | `direccion` | |
| 3 | División | `division` | |
| 4 | Cód División | `div_cod` | |
| 5 | Sección | `seccion` | |
| 6 | Cód Sección | `sec_cod` | |
| 7 | Record Count | `record_count` | Filas fuente del Excel |
| 8 | Vta Acum Actual | `vta_acum_actual` | Solo en CSV/tooltip, no en tabla |
| 9 | Vta Acum Plan | `vta_acum_plan` \| `vtaPlan` | Solo en CSV/tooltip |
| 10 | Vta Real A.Ant | `vtaAA` | |
| 11 | Real Vta (Mensual) | `ventas_actual` | SUM mensual del período |
| 12 | Real Plan (Mensual) | `ventas_plan` | SUM mensual del período |
| 13 | Des vs Plan Vta (%) | `des_vs_plan_vta` | 4 decimales |
| 14 | Des vs A.Ant (%) | `des_vs_aa_vta` | 4 decimales |
| 15 | Utilidad | `utilidad` | |
| 16 | Utilidad Plan | `utilidad_plan` | |
| 17 | Utilidad Año Ant | `utilidad_ano_ant` | |
| 18 | Mg Real (%) | `mg_real` | Exportado × 100, 4 decimales |
| 19 | Mg Plan (%) | `mg_plan` | Exportado × 100, 4 decimales |
| 20 | Mg Año Ant (%) | `mg_ano_ant` | Exportado × 100, 4 decimales |
| 21 | Des vs Plan GM (pp) | `des_vs_plan_mg` | Exportado × 100, 4 decimales |
| 22 | Des vs A.Ant Mg | `des_vs_aa_mg` | 4 decimales (ya en pp) |
| 23 | Rot Actual | `rot_actual` | 5 decimales |
| 24 | Rot Plan | `rot_plan` | 5 decimales |
| 25 | Rot Año Ant | `rot_ano_ant` | 5 decimales |
| 26 | Des vs Plan Rot | `des_vs_plan_rot` | 5 decimales |
| 27 | Des vs A.Ant Rot | `des_vs_aa_rot` | 5 decimales |
| 28 | Inv Ini Real | `inv_ini_real` | Snapshot último mes |
| 29 | Inv Ini Plan | `inv_ini_plan_val` | Snapshot último mes |
| 30 | Inv Pro Año Ant | `inv_pro_aa` | Promedio del período |
| 31 | Prom Inv Año Act | `prom_inv_act` | Promedio del período |
| 32 | Prom Inv Plan | `prom_inv_plan` | Promedio del período (= `avg_inv_plan`) |
| 33 | Des vs Plan Inv Ini (%) | `des_vs_plan_inv_ini` | 4 decimales |
| 34 | Calif Vta | `calif_vta` | 4 decimales |
| 35 | Calif Mg | `calif_mg` | 4 decimales |
| 36 | Calif Rot | `calif_rot` | 4 decimales |
| 37 | Calif General | `calif_general` | 4 decimales |
| 38 | Color Semaforo | `nivel` | Texto: "🟢 Verde", "🟡 Amarillo", "🔴 Rojo", "⚫ Sin Datos" |

---

## Reglas de agregación multi-mes (`getAccumData`)

Cuando se seleccionan múltiples meses (o ninguno = todos), los datos se agregan así:

| Tipo de campo | Método | Ejemplos |
|---|---|---|
| Ventas y Utilidad | **SUM** del período | `vta`, `vtaPlan`, `vtaAA`, `utilidad`, `utilidad_plan`, `utilidad_ano_ant` |
| Inventarios | **Promedio simple** entre meses del período | `avg_inv_actual`, `avg_inv_plan`, `avg_inv_aa` |
| Inventario Inicial y Acum YTD | **Snapshot del mes más reciente** | `inv_ini_real`, `inv_ini_plan_val`, `vta_acum_actual`, `vta_acum_plan` |
| Todos los derivados | Se recalculan desde los agregados | márgenes, rotaciones, desviaciones, calificaciones |
