# ESCCOM National Spatial Addressing Portal — Project Summary

Continuity doc for picking this up in a new chat. Paste/upload this file plus
the current `national-addressing-portal.html` at the start of a new session.

## What this is

A single-file HTML/JS web app (Leaflet + satellite imagery) that lets someone
draw a plot boundary on a map and get back:
- A **Master Plus Code** (Google Open Location Code, 10-char, centroid of the shape)
- Every other Plus Code grid cell the shape covers
- A resolved **"Full Qualified Address"** combining nearest road + locality
- Export of all registered plots to JSON or Excel

Built for **ESCCOM (Eswatini Communications Commission)**.

## Current state of the app (feature by feature)

**Drawing / plots**
- Draw tools: point, line, polygon, rectangle, circle (Leaflet.Draw)
- Multiple shapes accumulate on the map — each becomes its own "plot" card in
  the sidebar, doesn't get wiped when you draw another
- Each plot: auto-incrementing Plot ID (e.g. `PLT-2026-001` → `002`), Master
  Plus Code (large/bold/dominant), other Plus Codes covered (capped at 8
  shown by default, **"Show all" toggle** per-plot reveals up to 60),
  boundary point count, center coordinates
- Delete a plot via its card's "×" or the map's trash tool (kept in sync)
- "Locate on map ↗" button per card flies to that plot

**Gate Pin (11-char Plus Code) — DONE.** Added to solve: a 10-char Master
Plus Code's grid cell is ~14m x 14m, so two neighboring premises (or two
entrances on one large plot) can share the same master cell — ambiguous for
delivery. Fix:
- After drawing an **area or line** shape (polygon/rectangle/circle/line),
  the plot is created immediately but flagged `gateStatus: 'required'` and
  the app **forces** gate capture: an amber banner appears ("Click on the
  map to drop the Gate Pin…"), the marker draw tool auto-activates, and the
  next marker placed is *not* a new plot — it's routed via
  `finalizeGatePin()` into that plot's `gatePlusCode`, encoded at **11
  characters** (~3m x 3m) via `OpenLocationCode.encode(lat, lng, 11)`.
- Gate marker is visually distinct: amber/orange divIcon with a 🚪 emoji
  (vs. the default blue Leaflet pin for the Master Centroid marker), added
  to `markersGroup` (not `drawnItems`) — consistent with how the master pin
  and other-code dots are handled, so the trash tool never touches it
  directly.
- A **point or circle-marker** plot is already a single exact location, so
  it auto-fills its own gate code (same lat/lng, just re-encoded at 11
  chars) — no extra click forced for those.
- If gate placement is interrupted (user switches tools, deletes the plot
  mid-placement, etc.), the plot card shows a red "⚠ Gate Pin required"
  state with a **"Place Gate Pin"** button that re-triggers `requestGatePin()`.
- **Enforced at save**: `saveAllPlots()` blocks and lists any plot(s) still
  missing a gate pin rather than exporting incomplete data; the Save button
  label itself shows "N Gate Pins Pending" until resolved.
- Plot card UI: amber box showing the Gate Plus Code once set, with a note
  that it — not the Master Code — is the one to use for delivery.
- Export (JSON + Excel): `gate_plus_code` and `gate_coordinates` added
  alongside the existing master/other-code fields.

**Map / imagery**
- Base layers: **Satellite (Esri World Imagery, default)** and Street Map
  (OSM), switchable via top-right layer control
- Overlay: **Labels** (Esri place names/roads reference layer)
- Decision made: **NOT using Google Maps** — considered it (better zoom/3D)
  but it requires a billed Google Cloud API key the user doesn't have yet.
  Explicitly chose to stay on free Esri satellite for now. (There was a
  full working Google Maps rewrite done earlier in the chat — JS Drawing
  Manager, Geocoder, etc — but it was abandoned/reverted per user request
  to go back to the Leaflet+Esri version. That Google Maps code is NOT in
  the current file, just mentioned here in case it's ever wanted again.)

**Custom overlay data (ESCCOM-supplied, both embedded inline in the HTML as
`<script type="application/json">` blocks so the file works standalone with
no server/CORS issues)**
1. **ESCCOM Road Network** (`eswatini-roads.geojson`, also saved standalone
   in outputs) — 2,862 road segments. Originally a shapefile in the
   **"Lo31" projection** (Cape Datum, Transverse Mercator, central meridian
   31°E — historic South African/Swazi survey grid, closest EPSG equivalent
   is 22291 but with non-flipped axes) — reprojected to WGS84 in QGIS.
   Simplified via Ramer-Douglas-Peucker (~3m tolerance) from 397K to 79K
   vertices, 17MB → ~2MB. Only ~1,100 of 2,862 segments (38%) have a name
   or route code (`NAME` or fallback `ROAD_NO` — e.g. `MR3`, `DR12`); the
   rest are unnamed but still real, correctly-positioned geometry.
   Rendered as a togglable layer: named roads = bold brand blue, unnamed =
   thin muted gray, with hover tooltips.
2. **Tinkhundla Boundaries** (`tinkhundla.geojson`) — 61 constituency
   polygons, already in WGS84 (no reprojection needed). Simplified 5.58MB
   → 0.86MB. Each has `name`, `code`, and `region` (derived from the
   category code prefix: H=Hhohho, M=Manzini, S=Shiselweni, L=Lubombo).
   Rendered as outline-only polygons (near-zero fill opacity), color-coded
   by region, togglable layer, hover tooltip shows name + region.

**Address resolution ("Full Qualified Address")**
This is built in layers, each added in a separate step:
1. **Nearest-road lookup against ESCCOM's own road data** (not OSM) —
   point-to-line-segment distance search across all 2,862 segments (not
   just named ones, per explicit user request), within 300m. Formats as
   `"On {road}"` if within 8m, else `"~{N}m from {road}"`. Falls back to
   `"Unnamed {Type} Road"` (e.g. "Unnamed Paved Road") when the nearest
   segment has no name, rather than silently skipping it.
2. **Nominatim (OSM) reverse geocoding** for suburb/town context — the
   OSM-reported road name is deliberately excluded from this part now
   (since ESCCOM's own road data is the better source and showing two
   possibly-conflicting road names would be confusing).
3. **Tinkhundla (constituency) lookup — DONE.** `findInkhundla(lat, lng)`
   does a point-in-polygon check against `tinkhundlaData.features` (outer
   ring only — none of the 61 features have holes) and returns
   `{name, code, region}`. Wired into `resolveAddressFromCentroid()`,
   replacing Nominatim's `a.state` field with `"{name} Inkhundla,
   {region} Region"` since the inkhundla data is the more authoritative
   local source. Verified against the real Mbabane CBD test point from
   earlier in the chat — correctly resolved to "Mbabane East, Hhohho."

**Branding**
- ESCCOM logo badge cropped from the full logo file (transparent PNG,
  `esccom-badge.png` in outputs) — sits directly in the header, no
  box/border/padding
- Brand colors sampled directly from the logo and applied throughout:
  - Blue `#0C5092` (primary accent — buttons, Plus Code text, plot badges,
    drawn shape outlines, focus rings)
  - Red `#E92228` (circle-marker draw tool, Lubombo region on tinkhundla layer)
  - Black (header background, to match the logo's own black badge backdrop)
- Left the green "Save"/"Plots Registered" badge colors alone deliberately
  (that's a UX success/active-state convention, not brand-color territory)

**Export**
- Format dropdown: JSON or Excel (`.xlsx` via SheetJS, loaded from CDN,
  fully client-side, no server round-trip)
- Excel export flattens nested fields into clean columns
- **Short-format Plus Codes — tried, then explicitly REVERTED.** Built once
  using `OpenLocationCode.shorten(code, lat, lng)` against the code's own
  point — this produced broken/garbage output (e.g. `+PQ9` with nothing
  before the `+`) because shortening against the exact same point makes the
  algorithm strip the maximum digits. Fixed once by shortening against a
  fixed Eswatini-center reference point instead, but the user then asked to
  drop the whole feature rather than resolve the remaining edge case
  (border plots wouldn't shorten consistently). **Current state: fully
  removed** — no short-code fields in the export, no `shortenPlusCode`
  function, no `PLUS_CODE_SHORTEN_REF` constant. Only full Master/Gate
  codes are exported. Don't re-add this without being asked.
- Google Maps base-layer switch was discussed and **declined** — still
  needs a billed Google Cloud API key the user doesn't have set up yet.
  Gave step-by-step setup instructions (project → billing → enable Maps
  JavaScript API + Geocoding API → create + restrict key) for when they're
  ready; offered to build it with a placeholder key slot in the meantime
  but user chose to skip for now and stay on Leaflet + Esri.

**Address wording fix — DONE.** "Full Qualified Address" used to render
like `MANZINI SOUTH Inkhundla, Manzini Region`. User asked to drop the
"Inkhundla" and "Region" suffix words — now reads `MANZINI SOUTH, Manzini`.
Changed in the `inkhundlaPart` string template inside
`resolveAddressFromCentroid()`.

**Line/polyline Plus Code coverage — DONE (two related fixes).**
1. Originally `computeCoveringPlusCodesForLine()` only encoded a Plus Code
   at each vertex the user actually clicked, so a long straight segment
   with no intermediate clicks got zero interior dots — visibly sparser
   than polygons. Fixed by sampling along every segment at Plus-Code-cell
   resolution (same step size the area grid-scan uses, derived from
   `OpenLocationCode.decode()` on the master code), so a line now shows
   dots continuously along its whole length, not just at clicked vertices.
2. Separately, the user pointed out that when someone traces a closed
   shape (building/compound outline) with the **line** tool instead of the
   polygon tool, it should fill the interior with the same dense grid a
   polygon gets — not just dots along the outline. Added `isClosedLoop()`:
   checks whether a polyline's last point comes back within 15m of its
   first point; if so, `computeCoveringPlusCodesForArea()` is used instead
   of the line-sampling function. Genuinely open lines (roads, boundaries
   that don't close) still get the path-sampling behavior from fix #1.

**Improvement suggestions — discussed, none built yet.** User asked "what
improvements would you suggest" — gave a prioritized list, no action taken
on any of them:
- Biggest gap: no backend (data lives in browser memory only), no
  validation/approval workflow, no duplicate-plot detection
- Data quality: road naming still only ~38% covered; multiple gate pins
  per plot (compound with 2+ entrances currently needs 2 separate shapes);
  edit-in-place instead of delete-and-redraw
- Field usability: offline/low-connectivity caching; mobile layout
  (currently desktop sidebar-plus-map, would need a bottom-sheet redesign)
- Smaller wins: address/road search box; printable single-plot certificate
  with QR code; undo for accidental deletes

## Known limitations / honest caveats already discussed with the user
- Everything lives in browser memory — no backend/database. Closing the tab
  loses unsaved data unless exported.
- Nominatim and Esri's free imagery/tiles are rate-limited public services,
  not meant for production-scale national traffic.
- No validation/approval workflow — anyone can draw a wrong shape and get a
  confident-looking address for it.
- No legal/policy standing yet — a Plus Code isn't an official address until
  Eswatini/ESCCOM formally recognizes it as one.
- Satellite imagery resolution is good in Mbabane/Manzini, weaker in rural
  areas — exactly where informal addressing is most needed.
- Road naming coverage is genuinely incomplete in the source data (not a
  code problem) — 62% of road segments have no name on file.
- User asked directly whether this is "a great innovation" — honest answer
  given: not novel technology (Plus Codes + centroid GIS is standard), but a
  legitimate and useful *application* of proven tech to a real local gap,
  comparable to Ghana's GhanaPostGPS / Cape Verde's national geocode rollout.
  Viable as a pilot/demo now; needs backend + data validation + policy
  backing before being a system of record.

**Matsapha known-address data — DONE.** Embedded ESCCOM-supplied Matsapha
residential address points (1,856 points: house number + street + township +
town + postal code, e.g. "19 Umgaco Street, Matsapha Township, Matsapha
M202") the same way roads/tinkhundla are embedded — inline as
`<script id="matsapha-data" type="application/json">`, parsed into
`matsaphaData` (compact tuple format per point: `[lng, lat, hsNumber,
street, township, town, postcode]` to keep the file small — ~145KB added,
no reprojection needed, already WGS84).
- New `findNearestMatsaphaAddress(lat, lng, maxDistanceMeters = 60)` —
  nearest-point search (not nearest-segment; these are point addresses).
- Wired into `resolveAddressFromCentroid()`: if a plot centroid is within
  60m of a known Matsapha address point, that address's exact house
  number + street + township is used as `roadPart` instead of the
  nearest-road-segment guess — a real surveyed address is more precise
  than "~40m from Umgaco Street".
- New togglable map layer **"Matsapha Addresses"** — 1,856 small
  brand-blue `circleMarker` dots (radius 3, matches ESCCOM blue `#0C5092`)
  with a hover tooltip per point, added to the same `L.control.layers`
  panel as roads/tinkhundla. Off elsewhere in the country — this is
  Matsapha-town-specific data — but harmless to have loaded nationally
  since the nearest-point search has a 60m cap.
- Not yet done: no dedupe/conflict handling if a user later draws a plot
  that's *also* meant to become part of this dataset (i.e. no write-back —
  this is read-only reference data for now).

## Files that should exist in outputs (regenerate if missing)
- `national-addressing-portal.html` — the main app (self-contained, all
  overlay data embedded inline)
- `eswatini-roads.geojson` — standalone copy of the slimmed/simplified road data
- `tinkhundla.geojson` — standalone copy of the slimmed/simplified constituency data
- `matsapha.json` — standalone copy of the compact Matsapha address-point data
- `esccom-badge.png` — cropped transparent logo badge used in the header

## IN PROGRESS — QR code feature (this is where the session stopped)
User asked (following the improvement-suggestions discussion) to add a QR
code per plot so scanning it opens a read-only view of that plot's shape,
Plus Codes, and Full Qualified Address.

**Important discovery: the file already contains a half-built, non-functional
QR scaffold that predates this specific request being acted on** — found
while investigating, not something built in this session before this point:
- `<script>` tag loading `qrcode@1.5.3` from CDN (in `<head>`) — present
- A `#qrModal` div — fully built out: title, plot-ID subtitle, close button,
  a `<canvas id="qrCanvas">` for the QR image, an explanatory line
  ("Scanning this opens a read-only view of this plot's shape, Master and
  Gate Plus Codes, and Full Qualified Address — no editing tools."), a
  read-only `#qrModalUrl` text input with a Copy button, and a "Download QR
  (PNG)" button
- A "QR Code ⬚" button on each plot card that calls `showQrModal('${plot.id}')`
- **What's missing / broken:** none of the JS functions referenced by that
  HTML actually exist anywhere in the file — `showQrModal()`,
  `hideQrModal()`, `copyQrUrl()`, `downloadQr()` are all undefined. The
  `qrcode` library is loaded but never called. Clicking "QR Code" on a plot
  card right now would throw a JS error and do nothing.

**Plan discussed for what "done" looks like** (not yet implemented):
- QR payload = a URL back to this same portal page with the plot's data
  compactly encoded in the URL itself (Plot ID, Master + Gate Plus Codes,
  Full Qualified Address, shape geometry) — no server needed
- Scanning opens a **read-only viewer mode**: draws the exact shape,
  drops master + gate pins, shows the codes/address in a clean panel — no
  drawing tools, no sidebar
- Honest caveat already raised with the user: since this is currently just
  a downloaded HTML file (not hosted anywhere), the QR will encode a
  `file://` link and won't open on another phone until ESCCOM hosts this
  at a real URL — that's fine, nothing needs to be redone once it is hosted

**Next step when resuming:** implement `showQrModal`, `hideQrModal`,
`copyQrUrl`, `downloadQr`, and the viewer-mode URL-parameter parsing (e.g.
`?plot=<base64>`) described above. The QR generation itself should use the
already-loaded `qrcode` library (likely `new QRCode(...)` or similar per
that library's API — check its docs, don't assume the exact call signature)
targeting the `#qrCanvas` element already in the modal markup.

## Other possible future directions (raised, not acted on)
Production backend/database, paid geocoding/imagery for scale, a
validation/approval workflow before a plot becomes an official record,
extending named-road coverage, letting a plot register *multiple* gate pins
(e.g. a large compound with more than one entrance) rather than exactly
one, offline caching, mobile layout redesign, search box, printable
certificate, undo for deletes, duplicate-plot detection.
