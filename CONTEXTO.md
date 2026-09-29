# CONTEXTO.md — FCI Acindar Pymes

Web de seguimiento en vivo del FCI "Acindar Pymes". App de una sola página
(`index.html`, HTML+CSS+JS vanilla, sin build step, sin dependencias en el
repo) que lee/escribe directo contra Supabase y contra APIs públicas de
precios de mercado.

## Repo y deploy

- Repo: `github.com/fbroggi-coder/Acindar-Pymes`, rama `main`.
- Carpeta local: `C:\Users\fbroggi\Documents\GitHub\Acindar-Pymes`.
- Un solo archivo relevante: `index.html` (~2670 líneas). No hay `package.json`,
  `vercel.json` ni carpeta de build — es HTML servido tal cual (chequear en
  GitHub cómo está publicado: Pages o Vercel apuntando a la raíz del repo).
- No existía ningún archivo de contexto/README antes de este documento.

## Base de datos (Supabase)

Proyecto Supabase propio de este fondo (**distinto** del proyecto Supabase
usado por "Web de informes" / iebfondos-informes, que es `xycjffxjbnstpaceribu`):

- `SB_URL`: `https://xyoxonwufbrorwsoyyce.supabase.co/rest/v1`
- Auth: solo `anon key` embebida en el HTML (sin login de usuario en esta app;
  a diferencia del proyecto "Web de informes" que sí tiene login).
- Tablas:
  - `funds_ap` — config del fondo: `id` (`ACINDAR_PYMES`), `nombre`, `moneda`,
    `color`, `cuotapartes`, `nav_cierre`, `pn_cierre`, `fee_anual`, `pdf_date`,
    `ajustes` (array de ajustes contables).
  - `holdings_ap` — cartera actual: `fund_id`, `ticker`, `cantidad`, `cierre`.
    Se borra e inserta entera cada vez que se carga un PDF nuevo.
  - `activos_ap` — maestro de instrumentos/pricing: `ticker`, `nombre`,
    `tipo_activo`, `categoria`, `moneda`, `fuente_precio`, `live_symbol`,
    `tasa`, `lamina`, `base`, `precio_inicial`. Define cómo se pricea en vivo
    cada activo (ver más abajo).
  - `cortes_cupon_ap` — cortes de cupón puntuales: `fund_id`, `fecha`, `monto`,
    `moneda`, `descripcion`. Impactan el PN solo el día que se cargan.
  - `fx_history` — histórico de MEP/CCL (se usa para tomar el tipo de cambio
    del día hábil anterior, `cargarFxAyer()`).
- Edge function `yahoo-prices` (en el mismo proyecto Supabase) — proxy a Yahoo
  Finance para activos con `fuente_precio = 'yahoo'`.

## Flujo de trabajo del usuario

1. **Carga de PDF** (`handlePDFUpload` → `parsePDF`): el usuario sube el
   "Informe de Gestión" del fondo (PDF con la cartera). Se parsea con pdf.js
   (cargado on-demand desde cdnjs) extrayendo posiciones por coordenadas de
   texto, se arma una preview (`showPreviewModal`).
2. **Confirmación** (`confirmarCargaPDF`): borra `holdings_ap` del fondo e
   inserta la cartera nueva; actualiza `funds_ap` (NAV, cuotapartes, fecha,
   PN, ajustes contables). Si aparecen tickers no presentes en `activos_ap`,
   abre un mini-CMS (`showActivosCMS` / `guardarNuevosActivos`) para que el
   usuario defina cómo se priceará ese activo nuevo (con `autoSuggest()`
   proponiendo fuente/moneda/lámina según el ticker, p.ej. `AL30`/`GD` →
   bonos soberanos hard-dollar).
3. **Valuación en vivo** (`actualizarPrecios` / `fetchLivePrecios`, cada 10
   minutos, solo lun–vie 09:00–17:20 hora Argentina): trae precios de
   `data912.com` (bonos, letras, ON, CEDEARs, opciones y acciones argentinas;
   acciones y ADRs de EEUU; MEP/CCL) y de la edge function `yahoo-prices` para
   lo que use esa fuente. `calcPrecio()` combina esto con el tipo de activo
   (`kind`: `market`, `usMarket`, `mmUsd`/`mmArs` devengamiento por tasa,
   `yahoo`, `hold`) para valuar cada holding en la moneda del fondo.
4. **Cortes de cupón** (`abrirCorteCupon`/`guardarCorteCupon`): permite sumar
   manualmente un monto puntual (cupón cobrado) al PN del día, convertido a
   la moneda del fondo con el MEP del momento.
5. **Vista "Activos"** (`renderActivosPage`/`cargarActivosTabla`): tabla CRUD
   sobre `activos_ap` para editar cómo se priceá cada instrumento
   (`abrirEditarActivo`/`guardarEditarActivo`/`eliminarActivo`).
6. **Exportación**: `exportarExcel()` (usa SheetJS/XLSX cargado on-demand) y
   `exportarPDF()` generan la composición de cartera con precios live, NAV y
   PN para descargar.

## Detalles a tener en cuenta

- El PDF expresa las cuotapartes "en miles" → hay un `NAV_FACTOR = 1000`
  hardcodeado.
- La ventana de mercado (`dentroHorarioMercado`) es lun–vie 09:00–17:20
  hora Argentina; fuera de esa ventana no se refrescan precios en vivo.
- **Pendiente de limpieza**: `renderTopBar()` todavía muestra el logo/nombre
  hardcodeado "FCI Nación Seguros" (línea ~912) — parece un remanente de
  copiar la app desde otro fondo (Nación Seguros) sin renombrar. Convendría
  cambiarlo por "Acindar Pymes" o por `FUND?.nombre`.
- No hay tests ni CI. Los commits históricos son casi todos "Update
  index.html" vía GitHub Desktop (edición directa del archivo compilado, sin
  fuente separada como sí existe para "Web de informes" con sus `.dc.html`).

## Relación con otros proyectos de Federico

Esta app es hermana de "Web de informes" (`iebfondos-informes`, memoria
`/areas/web-informes-ieb.md`): mismo patrón general (single-file HTML +
Supabase + APIs de mercado) pero proyecto Supabase, tablas y alcance
funcional distintos y sin login. No compartir credenciales ni tablas entre
ambos proyectos.
