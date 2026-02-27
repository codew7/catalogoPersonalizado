# CLAUDE.md — catalogoPersonalizado

## Project Overview

A **static, no-build product catalog web application** hosted on GitHub Pages at `catalogo.dev.ar`. It pulls product data in real-time from a Google Sheet and renders a paginated, filterable, searchable card grid. The catalog can be customized per-client via URL parameters (custom name, per-category price markups).

---

## Repository Structure

```
/
├── index.html          # Full application: HTML structure + all CSS (inline)
├── indexm.js           # Main application logic (minified)
├── configm.js          # Configuration (obfuscated): Google Sheets API key + spreadsheet ID
├── CNAME               # GitHub Pages custom domain → catalogo.dev.ar
├── no-disponible.png   # Fallback image shown when a product image fails to load
└── faviconnegro.png    # Favicon (referenced in index.html, may not be tracked)
```

**There is no package.json, no node_modules, no build step.** All dependencies are loaded via CDN at runtime.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Markup | HTML5 (lang="es") |
| Styling | CSS3 with custom properties, inline in `index.html` |
| Logic | Vanilla JavaScript ES6+ (minified in `indexm.js`) |
| Icons | Font Awesome 6.4.2 (CDN) |
| Data source | Google Sheets API v4 (REST, no OAuth) |
| Real-time | Firebase SDK 10.12.0 compat (CDN) |
| Analytics | Google Analytics 4 (tag G-L3QLH6MHDX) |
| Hosting | GitHub Pages |

---

## Data Source: Google Sheets

Configuration lives in `configm.js` (obfuscated):

```js
const G_S_CONFIG = {
  API_KEY:        "AIzaSyCLRUk_7BRg4TxtcrFYzsV7opqMlqeXD3s",
  SPREADSHEET_ID: "10QcnEi_INRU6Fn0UbdluXQC5KrU3JptMx5Y2ODF8OPg",
  RANGO:          "Lista!A2:Z"
};
```

The API call in `indexm.js`:
```
GET https://sheets.googleapis.com/v4/spreadsheets/{SPREADSHEET_ID}/values/{RANGO}?key={API_KEY}
```

### Column Schema (0-indexed, columns A–Z)

| Index | Column | Description |
|---|---|---|
| 0 | A | Category (used for filtering and category dropdown) |
| 1 | B | Image URL(s) — comma-separated for multi-image products |
| 2 | C | Product code |
| 3 | D | Product name / title |
| 4 | E | Availability — `"no disponible"` hides the product |
| 6 | G | Price (string, may contain commas; parsed as float) |
| 7 | H | Category duplicate (used for price markup lookup) |
| 10 | K | Stock quantity (integer) |
| 12 | M | Last updated date (format: `DD/MM/YYYY`) — used for sort order |

### Product Filtering Logic (applied on load)

A product is **excluded** from `productos[]` if any of:
- `row[4].toLowerCase() === "no disponible"`
- `row[10]` is empty, NaN, or `<= 0`
- URL param `c` is present AND `row[7]` (category) is NOT in the decoded category map

Products are **sorted descending** by `row[12]` (most recently updated first). Date parsing handles `DD/MM/YYYY` format; invalid/empty dates are treated as `1900-01-01`.

---

## URL Parameters

The catalog is customizable per-client by appending query parameters:

| Param | Description |
|---|---|
| `n` | Custom catalog name. Replaces the header text and document title. Default: `"Catálogo"` |
| `c` | Base64-encoded category-to-markup map. See encoding spec below. |

### `c` Parameter Encoding Spec

Binary format, then Base64-encoded:
```
[1 byte: length of category name] [N bytes: category name] [1 byte: markup % as integer]
```
Repeated for each category. Example to add 20% to "Ropa" and 15% to "Calzado":
```
\x04Ropa\x14\x07Calzado\x0F
```
Then base64-encode the whole string.

Usage example:
```
https://catalogo.dev.ar/?n=MiTienda&c=BFJvcGEU
```

---

## Key Application Logic (`indexm.js`)

### Functions

| Function | Purpose |
|---|---|
| `getUrlParameter(name)` | Reads a URL query param via `URLSearchParams` |
| `parseCategoryPercentages()` | Decodes the `c` URL param into a `Map<string, number>` |
| `getPorcentajeParaCategoria(cat)` | Returns the markup % for a category (0 if not in map) |
| `mostrarPagina(page, products?)` | Renders the given page of `products` (defaults to all `productos`) |
| `cargarGrid(items)` | Builds and inserts product card DOM elements |
| `refreshStockInfo()` | Re-evaluates stock badges on already-rendered cards |
| `actualizarPaginacion(page, products)` | Renders adaptive pagination buttons |
| `getMaxVisiblePaginationButtons()` | Calculates how many pagination buttons fit by container width |
| `isValidImageUrl(url)` | Returns true if url ends with `.jpeg/.jpg/.gif/.png` |
| `abrirLightbox(images[])` | Opens the lightbox modal with the first image |
| `actualizarBotonesLightbox()` | Shows/hides prev/next arrows based on image count |
| `closeLightbox(event)` | Closes lightbox if clicking backdrop or close button |

### Stock Badge Logic

| Stock value | Badge |
|---|---|
| NaN or `<= 5` | `✗ Sin stock` (orange, `#ff9800`) |
| `6 – 15` | `⚠ Pocas unidades` (yellow, `#ffeb3b`) |
| `> 15` | `✓ Disponible` (green, `#4CAF50`) |

### Price Markup Calculation (when `c` param is set)

```js
adjustedPrice = Math.round(rawPrice * (1 + markup / 100));
adjustedPrice = Math.round(adjustedPrice / 10) * 10; // Round to nearest 10
```

### Search & Filter

Both operate together via an inner `o()` function:
- Category select (`#categorias`) filters by `row[0]`
- Text input (`#buscar`) filters by `row[3]` (name) or `row[2]` (code), case-insensitive
- "Limpiar Filtro" button (`#todos`) resets both and calls `mostrarPagina(1)`
- Results always call `mostrarPagina(1, filteredProducts)`

### Pagination

- `ITEMS_PER_PAGE = 30` (constant)
- `currentPage`, `productos` are module-level variables
- On window resize, pagination is re-rendered to fit available width

---

## CSS Architecture (all inline in `index.html`)

### CSS Custom Properties

```css
:root {
  --color-primary:    #44484c;
  --color-secondary:  #7a7e82;
  --color-accent:     #b0b3b8;
  --color-success:    #a3a6a9;
  --color-warning:    #d3d3d3;
  --color-background: #f5f5f5;
  --color-card:       #f0f0f0;
  --color-text:       #44484c;
  --color-text-light: #a0a0a0;
  --color-border:     #cccccc;
}
```

Brand accent color (not in vars): `#b50b9e` / `#ce0db4` (purple-magenta, used in header, spinner, active pagination).

### Responsive Breakpoints

| Breakpoint | Grid columns |
|---|---|
| `<= 600px` | 2 columns |
| `601px – 1024px` | 3 columns |
| `>= 1025px` | 5 columns |

Mobile/desktop header text is toggled by `.header-content-mobile` / `.header-content-desktop` using `display: none`.

### Key Animations

| Name | Used on |
|---|---|
| `mayorista-gradient` | Header text background gradient |
| `header-gradient` | Header background sweep |
| `header-move` | Header vertical float |
| `pulse` | Header title text glow |
| `spin` | Loading spinner |
| `fadeIn` | General fade-in utility |

---

## HTML Structure

```
<head>
  CSS variables + animations + component styles (all inline)
  CDN: Font Awesome, Firebase app+database, configm.js, indexm.js (defer)
  Google Analytics inline script
</head>
<body>
  <header>           — Animated gradient header with catalog name
  #loadingOverlay    — Full-screen spinner shown during data fetch
  .controls          — Category select, text search input, clear button
  #error             — Red error message paragraph (hidden by default)
  .grid#catalogo     — Product cards injected here by JS
  .pagination#pagination — Pagination buttons injected here by JS
  #lightbox          — Fixed-position image modal with prev/next nav
  <footer>           — Copyright line with dynamic catalog name + version
</body>
```

Key IDs referenced by JS:
- `#categorias` — category `<select>`
- `#buscar` — search `<input>`
- `#todos` — clear filter `<button>`
- `#catalogo` — product grid container
- `#pagination` — pagination container
- `#loadingOverlay` — loading screen
- `#error` — error display
- `#lightbox`, `#lightbox-img`, `#lightbox-value`, `#prev-btn`, `#next-btn`
- `#footer-catalog-name` — injected with `customName`

---

## Development Workflow

### No build step

Edit files directly. There is no compilation, transpilation, or bundling. To preview locally, open `index.html` in a browser **via a local server** (not `file://`) because the Google Sheets API fetch will fail due to CORS on `file://` origins.

```bash
# Quick local server options:
python3 -m http.server 8080
# or
npx serve .
```

### Editing minified files

`indexm.js` and `configm.js` are **minified/obfuscated** — there are no original source files in this repository. When making changes:

1. If the change is small, edit `indexm.js` directly (the logic is readable despite minification).
2. For larger changes, consider writing the change in readable JS, minifying it externally, and splicing it in.
3. **Do not un-minify and re-save** without re-minifying — keeping the files small matters for GitHub Pages performance.

### Deployment

Push to the `master` branch. GitHub Pages automatically serves from the repository root. The CNAME file routes `catalogo.dev.ar` to the GitHub Pages URL.

```bash
git add .
git commit -m "description of change"
git push origin master
```

---

## Key Conventions

- **Language**: The application is in Spanish (UI text, variable names like `categorias`, `buscar`, `productos`).
- **File naming**: Minified/processed files use the `m` suffix (`indexm.js`, `configm.js`).
- **No classes**: The JS uses functional patterns — arrays, `Set`, `Map`, `fetch`, `Promise`, DOM manipulation.
- **No frameworks**: Plain vanilla JS and CSS only. Do not introduce React, Vue, jQuery, etc.
- **No npm**: Do not add a `package.json`. All deps are CDN-loaded.
- **Inline CSS**: All styles live inside `<style>` tags in `index.html`. Do not create external `.css` files.
- **Error fallback**: Images use `onerror` to fall back to `no-disponible.png`.
- **Lazy loading**: All product images use `loading="lazy"`.
- **Prices**: Formatted with `toLocaleString("es-AR")` for Argentine locale (period as thousands separator).

---

## Important Constraints

- The Google Sheets API key in `configm.js` is intentionally public (read-only, restricted to the spreadsheet). Do not treat it as a secret.
- The `c` URL parameter encoding is a custom binary-over-base64 format — do not change it without updating all clients that generate catalog links.
- The `primeraCarga` flag ensures the first page load scrolls to top; subsequent filter/page changes scroll to the controls row. Preserve this UX.
- `refreshStockInfo()` is called 100ms after the initial render to re-check stock badges using the live `productos` array — this is intentional for any async data updates.
