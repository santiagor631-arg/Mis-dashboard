# CLAUDE.md

Este archivo proporciona orientación a Claude Code (claude.ai/code) cuando trabaja con el código en este repositorio.

## Descripción del proyecto

**Mis-dashboard** es una plataforma de monitoreo logístico estática (sin compilación) para Loginter. Consiste en dos archivos HTML autocontenidos que obtienen datos en vivo desde Google Sheets y renderizan dashboards interactivos completamente en el navegador.

- `dashboard_pedidos (4).html` — Dashboard de Pedidos/Backlog
- `dashboard_proximos_arribos (13).html` — Dashboard de Próximos Arribos

No hay gestor de paquetes, paso de compilación, bundler ni framework de pruebas. Para "ejecutar" el proyecto, abrí el archivo HTML directamente en un navegador o serví con cualquier servidor de archivos estáticos (ej. `python3 -m http.server`).

## Arquitectura

Cada dashboard es un único archivo HTML monolítico (~530–720 líneas) que contiene CSS inline, JavaScript inline e importación de Chart.js desde CDN. Ambos archivos siguen la misma arquitectura:

### Flujo de datos

```
Google Sheets (URL pública CSV)
  → loadData()          fetch + parseo
  → parseCSV()          máquina de estados carácter a carácter (maneja campos entre comillas)
  → buildColMap()       mapea nombres lógicos de columnas a índices por coincidencia de palabras clave
  → populateFilters()   construye opciones de dropdowns multi-selección
  → renderTable()
       ├── getFiltered()    aplica filtros activos + búsqueda de texto
       ├── applySort()      ordena por columna (detecta automáticamente tipo fecha/número/texto)
       ├── DOM update       reemplazo masivo de innerHTML para filas de tabla
       ├── updateKPIs()     recalcula métricas agregadas desde filas filtradas
       └── updateCharts()   destruye y recrea instancias de Chart.js
```

### Mapeo de columnas

Las columnas se identifican por coincidencia de palabras clave contra los nombres de encabezado (`findCol()`), no por posición fija. Esto hace que los dashboards sean resistentes al reordenamiento de columnas en la hoja fuente. Los nombres de columna están en español y pueden tener caracteres acentuados.

### Estado

Todo el estado se mantiene en variables a nivel de módulo: `allRows` (datos crudos), `headers`, `colMap`, `sortCol`/`sortDir` y un objeto `charts`/`chartInstances`. No hay framework — los cambios de estado disparan un re-render completo de `renderTable()`.

### Filtros multi-selección

Construidos a medida (sin librería). Funciones clave: `toggleDrop(msId)`, `getMS(msId)`, `updateBtn(msId)`, `buildDrop(dropId)`. Los dropdowns son elementos `<div>`, no `<select>`.

## Convenciones clave

### Formateo de números y fechas
- Los números usan `toLocaleString('es-AR')` — separador de miles es `.`, decimal es `,`
- Las fechas se almacenan como `DD/MM/YYYY`; `parseDateAR(s)` convierte a `YYYY-MM-DD` para comparaciones de orden
- `num(v)` maneja tanto `.` como `,` como separador decimal al parsear datos entrantes

### Badges de estado
- `badgeEstado(v)` / `badgeStatus(v)` — mapean strings de estado en español a pills con color
- `badgeASN(v)` — badge específico para presencia/ausencia de ASN
- Código de colores: verde = Finalizado/Con ASN, amarillo = Pendiente, rojo = Cancelado/Sin ASN

### Gráficos
- Todos los gráficos son Chart.js 4.4.0, cargados desde CDN
- Antes de crear un gráfico, siempre llamar `.destroy()` en la instancia existente para evitar errores de reutilización de canvas
- Doble eje Y (izquierdo = unidades, derecho = líneas/porcentaje) es un patrón recurrente

### Validación de datos
- Las filas se validan verificando que las primeras 4 columnas contengan números antes de incluirlas
- Los números de orden SAP deben ser strings numéricos de 8+ dígitos
- Propagación de "celdas combinadas": las celdas en blanco en ciertas columnas heredan el último valor no vacío (patrón `lastValue`)

## Fuentes de datos en Google Sheets

- **Pedidos:** `https://docs.google.com/spreadsheets/d/e/2PACX-1vSLQWkhqF_hC4v6MjBKXBuWV8XaMtTL5F3Zn2l6efVjMf7EdMNJxxCsmuU9raeCug/pub?gid=941366860&single=true&output=csv`
- **Próximos Arribos:** `https://docs.google.com/spreadsheets/d/e/2PACX-1vQ-ElVsKzXGfIFq4yC4RshpJhQ7IGb7JlBpmjshI4kK4T0H6Gg1LyiOeLt6Raf_OE3HYum8LPOmOPYd/pub?gid=1842608063&single=true&output=csv`

Las hojas deben estar publicadas como CSV (Archivo → Compartir → Publicar en la web). Si los datos dejan de cargar, la causa más probable es que se haya revocado la publicación de la hoja.

## UI y estilos

- Fuentes cargadas desde Google Fonts: **DM Sans** (cuerpo), **DM Mono** (números/códigos), **Nunito** (encabezados)
- La paleta de colores usa variables CSS definidas al inicio de cada bloque `<style>`
- Breakpoints responsivos en `1000px` y `900px` — la tabla colapsa a scroll horizontal por debajo de estos anchos
- La grilla de KPIs usa CSS Grid; los gráficos usan un layout de fila con flex-wrap
