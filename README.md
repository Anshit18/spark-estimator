# Spark Estimator

A mobile-first PWA for real estate investors to build repair cost estimates during property walkthroughs. Works offline after the first load and exports a styled Excel spreadsheet plus photos as a ZIP.

---

## Running locally

A local HTTP server is required — the service worker will not register on the `file://` protocol.

```bash
cd ~/Downloads/spark-estimator
python3 -m http.server 8080
```

Then open `http://localhost:8080` in Chrome (Android) or Safari (iOS). To test as a PWA, use Chrome DevTools > Application > Service Workers to inspect cache state, or add the page to your home screen on a real device.

---

## Tech stack

| Piece | Reason |
|---|---|
| Single HTML file | Zero build step. Works as a standalone deliverable; contestants, reviewers, and real agents can open it from a file share or email without npm install. |
| Tailwind CSS (Play CDN) | Utility classes keep all styling co-located with markup in a single file. The Play CDN version is one script tag; no build pipeline needed. |
| xlsx-js-style | Generates a styled XLSX with orange headers, bold subtotals, and number formatting. Lighter than SheetJS Pro and supports per-cell styles. |
| JSZip | Bundles the XLSX and compressed photo JPEGs into a single ZIP download without any server involvement. |
| Service worker | Cache-first strategy with precaching of all CDN assets. After the first online load, the full app runs offline including photo capture and ZIP export. |
| localStorage | Simple key-value persistence that works in both browser and installed PWA contexts with no server. Photos are stored as compressed base64 JPEG (~150–200 KB each). Hard ceiling is ~5 MB across all projects combined. |
| Native BarcodeDetector API | Zero-dependency barcode scanning. Attempts serial number extraction on every captured photo using the browser's built-in API (Chrome 83+, Safari 17+). Silently skips if the API is unavailable. |

---

## Architecture

### Data model

**CATALOG** — 108 repair line items (IDs like `ig-01`, `kt-06`, `ba-10`) with a name, default cost, and unit. These are verbatim from the Pricing List CSV.

**GROUPS** — 19 named groups (Flooring, HVAC, Cabinets, etc.), each pointing to an ordered list of catalog IDs.

**ROOM_TYPES** — 7 room types mapping to a subset of groups:
- `interior`, `systems`, `exterior` — singletons, always present once per project
- `kitchen`, `bathroom`, `bedroom`, `living` — non-singletons, can be added multiple times with auto-numbered labels

**Project** — the top-level storage unit:
```
{
  id, name, address, createdAt, updatedAt,
  rooms: [Room, ...],
  priceOverrides: { catalogId: cost },   // project-level price overrides
  photos: [{ id, dataUrl, name, serialNumber, scanning, timestamp }],
  notes: ''
}
```

**Room** — an instance of a room type, with its own copy of every group's state:
```
{
  id, type, label,
  groups: {
    [groupKey]: {
      noAction: bool,
      items: { [catalogId]: { qty, checked, costOverride } },
      customItems: [{ id, name, cost, unit, qty, checked }],
      removedItems: [catalogId, ...]
    }
  }
}
```

**Price resolution chain** (most specific wins):
```
item.costOverride → project.priceOverrides[catalogId] → globalPrices[catalogId] → CATALOG[catalogId].cost
```

**localStorage keys:**
- `spark_projects` — array of project stubs `{ id, name, address, updatedAt }`
- `spark_proj_{id}` — full project data
- `spark_active` — ID of the last-opened project
- `spark_gprices` — global price overrides (applied across all projects)

### Rendering

The app is a single-pass innerHTML renderer (`render()`) that rebuilds the active view from scratch. Frequent updates (checking an item, changing a qty) use targeted DOM patches (`patchLineTotal`, `patchGroupSub`, `patchRoomSub`, `patchFooter`) to avoid re-rendering the full page on every keystroke. All event handling is delegated to `document.body` via `dataset` attributes.

### Export pipeline

1. `buildWorkbook(proj)` walks every room → group → item and builds a 2D array of rows. Per-cell styles (orange headers, bold totals, currency formatting) are applied via a parallel `styleMap` object, then merged onto the worksheet before handing it to `xlsx-js-style`.
2. `exportProject()` serializes the workbook to an `ArrayBuffer`, adds each photo as a file inside a `photos/` folder using the base64 data URL, then calls `JSZip.generateAsync()` to produce a blob that gets downloaded via a temporary `<a>` tag.

### Creative features (Phase 8)

**Cost Triage** — ranks all checked line items by line total descending and computes a Pareto cutoff: the smallest set of items whose combined cost is ≥ 80% of the project total. Displayed as a ranked list with proportional bar charts. Intended to tell an investor at a glance which repairs are worth negotiating on.

**Project Comparison** — side-by-side view of any two saved projects. Shows total cost, percent reviewed, a cheaper/more-expensive summary pill, and a section-by-section cost breakdown (Interior, Kitchen, Systems, etc.) rendered as paired horizontal bars.

---

## Known limitations

**5 MB localStorage ceiling** — Photos are compressed to ~150–200 KB each before storage, giving headroom for roughly 20–25 photos before the storage guard kicks in and blocks further additions. The workaround is to export and clear photos between sessions. A proper fix would be IndexedDB.

**Portrait-only in PWA mode** — `manifest.json` sets `"orientation": "portrait-primary"`. The app works in landscape in a browser tab but is not optimized for it; the design is built around vertical scrolling.

**Price override tap target** — The inline price override button (the small cost/unit text below each item name) is ~20 px tall. It is intentionally small to keep item rows compact; it is a secondary action not used during normal walkthroughs.

**Camera cancel on iOS < 16** — On some older iOS versions, canceling out of the camera and immediately tapping the label again may not reopen the camera picker. Tapping a second time resolves it. Modern iOS handles this correctly.

**No conflict resolution** — If the same project is opened in two browser tabs simultaneously, the later save overwrites the earlier one. Not a real-world problem for a single-user walkthrough tool but worth noting.

**Offline requires one prior online load** — The service worker precaches all assets on first install, but the very first page load must be online. After that, the app is fully offline-capable including photo capture and ZIP export.
