---
---
# Perfil Municipal de Puerto Rico
### Rafael E Vega Rodriguez
COMP4010

```js
import * as Plot from "npm:@observablehq/plot"
import * as topojson from "npm:topojson-client"
```

```js
const data = await FileAttachment("data/pr_78_municipios_anual_completo.csv").csv({typed: true})
const topo = await FileAttachment("data/pr_municipios_topo.json").json()
```

```js
const varLabels = {
  poblacion: "Poblacion",
  pct_poverty: "% Pobreza",
  median_hh_income: "Ingreso Mediano",
  pct_unemployment: "% Desempleo",
  pct_bachelors_plus: "% Bachillerato",
  median_age: "Edad Mediana",
  densidad_pob_km2: "Densidad por km2"
}
```

---

## Controles globales

```js
const año_sel = view(Inputs.range([1970, 2025], {value: 2020, step: 1, label: "Año"}))
```

```js
const variable_sel = view(Inputs.select(
  Object.keys(varLabels),
  {value: "poblacion", label: "Variable", format: d => varLabels[d]}
))
```

---

## Resumen

```js
const snap = data.filter(d => d.año === año_sel)

const rico    = [...snap].sort((a, b) => b.median_hh_income - a.median_hh_income)[0]
const pobre   = [...snap].sort((a, b) => b.pct_poverty - a.pct_poverty)[0]
const poblado = [...snap].sort((a, b) => b.poblacion - a.poblacion)[0]

const decline = data
  .filter(d => d.año === 2000)
  .map(d => {
    const pop = snap.find(x => x.municipio === d.municipio)
    return {...d, cambio: ((pop?.poblacion - d.poblacion) / d.poblacion * 100)}
  })
  .sort((a, b) => a.cambio - b.cambio)[0]

display(html`
  <div style="display:grid; grid-template-columns: repeat(4,1fr); gap:1rem; margin:1rem 0">
    <div style="background:#1a1a2e; padding:1.2rem; border-radius:8px; border-left:4px solid #4caf50">
      <div style="color:#aaa; font-size:0.8rem">Wealthiest</div>
      <div style="font-size:1.4rem; font-weight:bold">${rico.municipio}</div>
      <div style="color:#4caf50">$${rico.median_hh_income.toLocaleString()}</div>
    </div>
    <div style="background:#1a1a2e; padding:1.2rem; border-radius:8px; border-left:4px solid #f44336">
      <div style="color:#aaa; font-size:0.8rem">Highest Poverty</div>
      <div style="font-size:1.4rem; font-weight:bold">${pobre.municipio}</div>
      <div style="color:#f44336">${pobre.pct_poverty}% poverty</div>
    </div>
    <div style="background:#1a1a2e; padding:1.2rem; border-radius:8px; border-left:4px solid #2196f3">
      <div style="color:#aaa; font-size:0.8rem">Most Populated</div>
      <div style="font-size:1.4rem; font-weight:bold">${poblado.municipio}</div>
      <div style="color:#2196f3">${poblado.poblacion.toLocaleString()} residents</div>
    </div>
    <div style="background:#1a1a2e; padding:1.2rem; border-radius:8px; border-left:4px solid #ff9800">
      <div style="color:#aaa; font-size:0.8rem">Biggest Decline</div>
      <div style="font-size:1.4rem; font-weight:bold">${decline.municipio}</div>
      <div style="color:#ff9800">${decline.cambio.toFixed(1)}% since 2000</div>
    </div>
  </div>
`)
```

---

## Tabla Comparativa
Compara los 78 municipios en indicadores demograficos y socioeconomicos. Presiona en cualquier columna para ordenar.

```js
const snapshot = data
  .filter(d => d.año === año_sel)
  .sort((a, b) => b[variable_sel] - a[variable_sel])

display(Inputs.table(snapshot, {
  columns: ["municipio", "poblacion", "pct_poverty", "median_hh_income", "pct_unemployment", "pct_bachelors_plus", "median_age"],
  header: {
    municipio: "Municipio",
    poblacion: "Poblacion",
    pct_poverty: "% Pobreza",
    median_hh_income: "Ingreso Mediano",
    pct_unemployment: "% Desempleo",
    pct_bachelors_plus: "% Bachillerato",
    median_age: "Edad Mediana"
  }
}))
```

---

## Ranking de 78 Municipios
Los 78 municipios ordenados por la variable seleccionada. Verde indica mejor desempeno, rojo indica peor.

```js
const color78 = d3.scaleSequential(
  [d3.min(snapshot, d => d[variable_sel]), d3.max(snapshot, d => d[variable_sel])],
  d3.interpolateRdYlGn
)

display(Plot.plot({
  width: 800,
  height: 1400,
  marginLeft: 130,
  x: { grid: true, label: varLabels[variable_sel] },
  marks: [
    Plot.barX(snapshot, {
      x: variable_sel,
      y: "municipio",
      sort: { y: "-x" },
      fill: d => color78(d[variable_sel]),
      tip: true
    }),
    Plot.ruleX([0])
  ]
}))
```

---

## Top 20 — Bubble Chart
Los 20 municipios con mayor valor de la variable seleccionada.

```js
const top20 = snapshot.slice(0, 20)

const colorTop = d3.scaleSequential(
  [d3.min(top20, d => d[variable_sel]), d3.max(top20, d => d[variable_sel])],
  d3.interpolateRdYlGn
)

display(Plot.plot({
  width: 900,
  height: 600,
  marginLeft: 130,
  marginRight: 150,
  r: { range: [8, 30] },
  x: { grid: true, type: "log", label: varLabels[variable_sel] },
  marks: [
    Plot.dot(top20, {
      x: variable_sel,
      y: "municipio",
      sort: { y: "-x" },
      r: variable_sel,
      fill: d => colorTop(d[variable_sel]),
      tip: true
    }),
    Plot.text(top20, {
      x: variable_sel,
      y: "municipio",
      sort: { y: "-x" },
      text: "municipio",
      textAnchor: "start",
      dx: 35
    })
  ]
}))
```

---

## Lineas de Tiempo
Evolucion de cualquier variable desde 1950 hasta 2025. Los puntos marcan anos censales reales.

```js
const municipios_sel = view(Inputs.select(
  [...new Set(data.map(d => d.municipio))].sort(),
  {label: "Municipios", multiple: true, value: ["San Juan", "Bayamón", "Ponce", "Caguas", "Mayagüez"]}
))
```

```js
const filtered = data.filter(d => municipios_sel.includes(d.municipio))
const censales = [1970, 1980, 1990, 2000, 2010, 2020]

display(Plot.plot({
  width: 800,
  height: 400,
  x: { label: "Año" },
  y: { label: varLabels[variable_sel], grid: true },
  color: { legend: true },
  marks: [
    Plot.lineY(filtered, {
      x: "año",
      y: variable_sel,
      stroke: "municipio",
      strokeWidth: 2,
      tip: true
    }),
    Plot.dot(filtered.filter(d => censales.includes(d.año)), {
      x: "año",
      y: variable_sel,
      fill: "municipio",
      r: 4
    })
  ]
}))
```

---

## Mapa Coropletico
Distribucion geografica de la variable seleccionada en Puerto Rico.

## Mapa Coropletico
Distribucion geografica de la variable seleccionada en Puerto Rico.

```js
const L = await import("https://esm.sh/leaflet@1.9.4")
```

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
```

```js
const features = topojson.feature(topo, topo.objects.municipios)

const mapSnap = new Map(
  data.filter(d => d.año === año_sel).map(d => [d.municipio, d[variable_sel]])
)

const values = [...mapSnap.values()].filter(v => v != null)
const lo = d3.min(values)
const hi = d3.max(values)

const getColor = val => {
  if (val == null) return "#ccc"
  const t = (val - lo) / (hi - lo)
  return d3.interpolateYlOrRd(t)
}

const div = display(html`<div style="height:500px; width:100%"></div>`)

const map = L.map(div).setView([18.2, -66.5], 9)

L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
  attribution: "© OpenStreetMap"
}).addTo(map)

L.geoJSON(features, {
  style: f => ({
    fillColor: getColor(mapSnap.get(f.properties.municipio)),
    fillOpacity: 0.75,
    color: "white",
    weight: 1
  }),
  onEachFeature: (f, layer) => {
    const val = mapSnap.get(f.properties.municipio)
    layer.bindTooltip(`<b>${f.properties.municipio}</b><br>${varLabels[variable_sel]}: ${val ?? "N/A"}`)
  }
}).addTo(map)
```

---

## Clustering de Municipios
Municipios agrupados por similitud socioeconomica usando k-means. Cada color representa un grupo distinto.

```js
const k = view(Inputs.range([2, 6], {value: 4, step: 1, label: "Numero de grupos"}))
```

```js
const clusterVars = ["pct_poverty", "median_hh_income", "pct_unemployment", "pct_bachelors_plus", "median_age"]
const clusterSnap = data.filter(d => d.año === año_sel && clusterVars.every(v => d[v] != null))

const scaleVar = v => {
  const lo = d3.min(clusterSnap, d => d[v])
  const hi = d3.max(clusterSnap, d => d[v])
  return d => (d[v] - lo) / (hi - lo)
}

const scaled = clusterSnap.map(d => clusterVars.map(v => scaleVar(v)(d)))

let centers = d3.range(k).map(i => scaled[Math.floor(i * scaled.length / k)])
let clusterLabels = scaled.map(() => 0)

for (let i = 0; i < 30; i++) {
  clusterLabels = scaled.map(p => d3.leastIndex(centers, c => d3.sum(p.map((v, j) => (v - c[j]) ** 2))))
  centers = d3.range(k).map(ki => clusterVars.map((_, j) => d3.mean(scaled.filter((_, i) => clusterLabels[i] === ki), p => p[j])))
}

const etiquetas = ["Alta Pobreza", "Ingreso Medio-Bajo", "Ingreso Medio-Alto", "Alto Ingreso"]

const porIngreso = d3.range(k)
  .map(ki => ({ ki, avg: d3.mean(clusterSnap.filter((_, i) => clusterLabels[i] === ki), d => d.median_hh_income) }))
  .sort((a, b) => a.avg - b.avg)

const labelMap = Object.fromEntries(porIngreso.map((g, i) => [g.ki, etiquetas[Math.min(i, etiquetas.length - 1)]]))

const clusters = clusterSnap.map((d, i) => ({...d, cluster: labelMap[clusterLabels[i]]}))

display(Plot.plot({
  width: 800,
  height: 500,
  color: { legend: true, label: "Grupo" },
  x: { label: "Ingreso Mediano" },
  y: { label: "% Pobreza" },
  marks: [
    Plot.dot(clusters, {
      x: "median_hh_income",
      y: "pct_poverty",
      fill: "cluster",
      r: 8,
      fillOpacity: 0.8,
      tip: true,
      title: d => `${d.municipio} — ${d.cluster}`
    }),
    Plot.text(clusters, {
      x: "median_hh_income",
      y: "pct_poverty",
      text: "municipio",
      fontSize: 9,
      dy: -10
    })
  ]
}))
```

---

## Correlaciones entre Variables
Que tan fuerte es la relacion entre cada par de variables. Azul significa que se mueven juntas, rojo que se mueven en direcciones opuestas.

```js
const corrVars = ["pct_poverty", "median_hh_income", "pct_unemployment", "pct_bachelors_plus", "median_age"]
const corrSnap = data.filter(d => d.año === año_sel && corrVars.every(v => d[v] != null))

const corr = (a, b) => {
  const xa = corrSnap.map(d => d[a])
  const xb = corrSnap.map(d => d[b])
  const ma = d3.mean(xa)
  const mb = d3.mean(xb)
  const num = d3.sum(xa.map((v, i) => (v - ma) * (xb[i] - mb)))
  const den = Math.sqrt(d3.sum(xa.map(v => (v - ma) ** 2)) * d3.sum(xb.map(v => (v - mb) ** 2)))
  return num / den
}

const pairs = corrVars.flatMap(a => corrVars.map(b => ({
  a: varLabels[a],
  b: varLabels[b],
  r: corr(a, b)
})))

display(Plot.plot({
  width: 550,
  height: 550,
  marginBottom: 80,
  marginLeft: 120,
  color: { scheme: "RdBu", domain: [-1, 1], legend: true },
  marks: [
    Plot.cell(pairs, { x: "a", y: "b", fill: "r", tip: true }),
    Plot.text(pairs, { x: "a", y: "b", text: d => d.r.toFixed(2), fontSize: 11 })
  ]
}))
```
## Conclusiones

```js
display(html`
  <div style="display:grid; grid-template-columns: repeat(2,1fr); gap:1rem; margin:1rem 0">
    <div style="background:#1a1a2e; padding:1.5rem; border-radius:8px; border-left:4px solid #4caf50">
      <div style="font-size:1rem; font-weight:bold; margin-bottom:0.5rem">Desigualdad economica marcada</div>
      <div style="color:#aaa; font-size:0.9rem">Existe un gap significativo entre municipios como Guaynabo y Dorado versus Las Marias y Maricao. El ingreso mediano puede variar hasta 4 veces entre municipios.</div>
    </div>
    <div style="background:#1a1a2e; padding:1.5rem; border-radius:8px; border-left:4px solid #f44336">
      <div style="font-size:1rem; font-weight:bold; margin-bottom:0.5rem">Despoblacion generalizada</div>
      <div style="color:#aaa; font-size:0.9rem">Puerto Rico ha perdido una poblacion significativa desde el 2000. Municipios del Sur como Guanica y Utuado muestran los mayores decrementos.</div>
    </div>
    <div style="background:#1a1a2e; padding:1.5rem; border-radius:8px; border-left:4px solid #2196f3">
      <div style="font-size:1rem; font-weight:bold; margin-bottom:0.5rem">Educacion e ingreso van juntos</div>
      <div style="color:#aaa; font-size:0.9rem">Los municipios con mayor porcentaje de bachillerato tienen consistentemente mayor ingreso mediano. La correlacion entre estas dos variables es de las mas fuertes del dataset.</div>
    </div>
    <div style="background:#1a1a2e; padding:1.5rem; border-radius:8px; border-left:4px solid #ff9800">
      <div style="font-size:1rem; font-weight:bold; margin-bottom:0.5rem">Cuatro perfiles municipales</div>
      <div style="color:#aaa; font-size:0.9rem">El clustering revela cuatro grupos claros: municipios de alto ingreso, ingreso medio-alto, ingreso medio-bajo y alta pobreza. Cada grupo tiene caracteristicas socioeconomicas consistentes.</div>
    </div>
  </div>
`)
```






*Todo fue hecho con D3 de observable frameworks*

---

*Rafael E Vega Rodriguez · COMP4010 · Universidad de Puerto Rico*