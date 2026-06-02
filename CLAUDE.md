# Printmaker IAPP

Print shop management PWA for a shop in Puerto Rico.
Single-file app — all HTML, CSS, and JavaScript live in `index.html`.

## Project structure

```
printflow-pwa/
├── index.html      ← entire app (HTML + CSS + JS, ~1,600 lines)
├── manifest.json   ← PWA manifest
├── sw.js           ← service worker (cache key: printmaker-iapp-v3)
└── icons/
    ├── icon-192.png
    └── icon-512.png
```

## Critical rules — read before editing

- **Never split index.html into multiple files.** It must stay a single file for the PWA and offline cache to work correctly.
- **Tax rate is 11.5%** (Puerto Rico). The constant `TAX = 0.115` is defined near the top of the script block. Do not change it.
- **No backend, no build step.** Everything runs in the browser. Data persists in localStorage only.
- **Do not add npm, webpack, or any build tooling.** Deploy by dragging the folder to Netlify Drop.
- **External libraries are loaded via CDN** (SheetJS, jsPDF, Tabler Icons, DM Sans font). Do not replace or remove them.
- When updating the service worker cache, increment the cache key version (e.g. `v3` → `v4`) in `sw.js`.

## localStorage keys

All data lives in these keys — never rename them or existing user data will be lost:

| Key | Contents |
|-----|----------|
| `pm_orders` | Work orders array |
| `pm_mats` | Materials array |
| `pm_pricing` | Pricing rules array |
| `pm_clients` | Clients array |
| `pm_jobtypes` | Job types array |
| `pm_printpricing` | Print pricing (per-sheet rates) array |
| `pm_sizes` | Sizes (divisors) array |
| `pm_quotes` | Quotes array |

## Data schemas

### Order
```js
{
  id,           // uid()
  client,       // string (name)
  clientId,     // optional — linked client id
  contact,      // string (phone / contact info)
  jobtype,      // string (from jobtypes list)
  status,       // 'Pending' | 'In Progress' | 'Completed' | 'Cancelled'
  duedate,      // YYYY-MM-DD string
  notes,        // string
  lineItems,    // [{ id, desc, qty, price }]
  subtotal,     // number (pre-tax)
  total,        // number (subtotal * 1.115)
  created       // Date.now() timestamp
}
```

### Quote
```js
{
  id,             // uid()
  clientId,       // linked client id
  clientName,     // string (stored for display if client deleted)
  clientPhone,    // string
  clientEmail,    // string
  clientAddress,  // string
  jobs,           // array of job snapshots (see below)
  total,          // number (grand total with tax)
  status,         // 'Draft' | 'Sent' | 'Accepted' | 'Declined'
  created,        // Date.now()
  updated         // Date.now()
}
```

### Quote job snapshot (inside quote.jobs[])
```js
{
  jobtype,   // string
  qty,       // number
  matId,     // material id
  szId,      // size id
  ppId,      // print pricing id
  ruleId,    // pricing rule id
  notes      // string
}
```

### Material
```js
{ id, name, category, unit, size, cost, stock, reorder }
```

### Pricing rule
```js
{ id, name, jobtype, base, perunit, minqty, markup, notes }
```

### Print pricing
```js
{ id, name, rate }   // rate = $ per sheet
```

### Size
```js
{ id, name, div, desc }   // div = pieces per sheet (divisor)
```

### Client
```js
{ id, name, phone, email, address, notes }
```

### Job type
```js
{ id, name, desc }
```

## Pricing formula

```
costPerPiece = (material.cost + printPricing.rate) / size.div
lineCost     = costPerPiece * qty
ruleBase     = pricingRule.base + (pricingRule.perunit * qty)
preTax       = (lineCost + ruleBase) * (1 + pricingRule.markup / 100)
grandTotal   = sum(preTax for all jobs) * 1.115
```

## Navigation & views

Views are toggled with `showView(viewName)`. Valid view names:

`orders` · `new-order` · `quote` · `clients` · `jobtypes` · `materials` · `pricing` · `printpricing` · `sizes`

The quote view uses `display:flex` (not the default block) — `showView()` handles this already.

## Key global state variables

```js
orders, materials, pricing, clients, jobtypes, printpricing, sizes, quotes
// edit IDs (null when creating new)
editOId, editMId, editPId, editPPId, editSzId, editJTId, editCLId
// order form
lineItems[]
// quote builder
qJobs[], qSelectedClient, qJobIdCounter, activeQuoteId
// navigation
fromOrderCancel   // set to 'quote' when order was started from a quote transfer
```

## Quote → Order transfer flow

1. User fills out quote with client + jobs → `calcQ()` builds the output panel
2. User clicks "Create order" → `showTransferBox()` renders a preview
3. User confirms → `confirmTransfer()`:
   - Builds `lineItems[]` from each quote job (one line item per job)
   - Pre-fills the order form (client, contact, job type, notes with quote number)
   - Sets `fromOrderCancel = 'quote'` so Cancel returns to quote builder
   - Marks the quote status as `'Accepted'`
   - Calls `showView('new-order')`
4. After saving the order, `saveOrder()` checks `fromOrderCancel` and navigates back to quote

## Important functions reference

| Function | What it does |
|----------|-------------|
| `init()` | Loads all data from localStorage, populates dropdowns, renders all views |
| `showView(v)` | Switches the active view, triggers relevant render |
| `saveOrder()` | Validates and saves work order, respects `fromOrderCancel` |
| `saveQuote()` | Saves active quote state (client + jobs snapshot + total + status) |
| `loadQuote(id)` | Restores a saved quote into the active builder (client + jobs with 30ms timeout for DOM) |
| `renderActiveQuoteForm()` | Rebuilds the entire quote form HTML in `#quote-content` |
| `calcQ()` | Recalculates all job prices and updates the output panel |
| `confirmTransfer()` | Transfers quote to a new work order pre-filled form |
| `newQuoteForClient(id)` | Opens quote builder pre-selected for a specific client |
| `openQuoteFromClient(qid)` | Navigates to quote builder and loads a specific saved quote |
| `populateAllJTDropdowns()` | Refreshes job type options in order form and pricing rule form |
| `renderQuoteList()` | Updates the left-side saved quotes panel |
| `dbGet(key)` / `dbSet(key, data)` | localStorage read/write with JSON parse/stringify |
| `uid()` | Generates a unique ID |
| `qNum(id)` | Returns formatted quote number like `#Q-0042` |

## CSS architecture

- CSS custom properties (variables) defined in `:root` with dark mode overrides in `@media(prefers-color-scheme:dark)`
- Key variables: `--teal`, `--teal-dark`, `--teal-light`, `--teal-mid`, `--bg`, `--surface`, `--surface2`, `--border`, `--border2`, `--text`, `--text2`, `--text3`, `--radius`, `--radius-sm`, `--shadow`, `--shadow-md`
- Mobile breakpoint: `max-width: 700px` — sidebar becomes a slide-in drawer
- Quote view layout: `display:flex` with `.quote-side` (230px) + `.quote-main` (flex:1)

## External libraries (CDN)

```html
<!-- Fonts & icons -->
https://fonts.googleapis.com/css2?family=DM+Sans...
https://cdn.jsdelivr.net/npm/@tabler/icons-webfont@2.44.0/tabler-icons.min.css

<!-- Loaded via <script> tags at bottom of body -->
https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js
https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js
```

## Deployment

Drag the `printflow-pwa/` folder to **Netlify Drop** (netlify.com/drop).
No build step. No config. The service worker handles offline caching automatically.
User data is stored in the browser's localStorage — it survives app updates.
