# PCE v2.1 Copilot Instructions

## Project Overview

**PCE (Spatial Price Card Engine)** is a single-file HTML/CSS/JavaScript application for extracting room dimensions from floor plan images using a webhook-based OCR/AI pipeline. It provides a neumorphic UI for manual room data entry, unit conversion (Ft↔m), and forwarding room data to external analysis services.

**Key characteristics:**
- Single-page application (`index.html` only)
- No build process, build tools, or dependencies
- Client-side state management with vanilla JavaScript
- Webhook-based integration with n8n automation platform
- Neumorphic design system with CSS custom properties

---

## Architecture & Data Flow

### State Management

The core application state is managed by a single `state` object:

```javascript
var state = { 
  unit: 'ft',           // Current unit ('ft' or 'm')
  rooms: [],            // Array of room objects: {room, length, width, unit}
  sortAsc: true,        // Sort direction for alphabetical ordering
  sourceName: null,     // Uploaded filename
  sourceMeta: null,     // File metadata {size, type}
  sourceFile: null,     // File object reference
  analysis: {}          // Room analysis data {doorCount, artifacts}
};
```

**Key data structures:**
- **Room object:** `{room: string, length: number|null, width: number|null, unit: 'ft'|'m'}`
- **Analysis object:** `{doorCount: number, artifacts: {[name]: {H, W, D}}}`

### Critical Workflow: Upload → Parse → Display → Send

1. **Upload:** User drops/selects an image → `uploadToWebhook()` POSTs to UPLOAD_WEBHOOK endpoint
2. **Parse Response:** Webhook returns room dimensions (various JSON shapes handled by `deriveRooms()`)
3. **Display:** `renderRooms()` builds room cards with editable dimensions
4. **Proceed:** `sendProceed()` POSTs finalized data + analysis to PROCEED_WEBHOOK for backend processing

### Webhook Integration

- **UPLOAD_WEBHOOK:** Default `https://n8n.kamalpriyank.com/webhook-test/upload-floor-plan`  
  Accepts multipart image → returns JSON array of rooms
  
- **PROCEED_WEBHOOK:** `https://n8n.kamalpriyank.com/webhook-test/Proceed-With-Spatial-Analysis`  
  Accepts multipart form data (rooms JSON, unit, file, metadata) → returns analysis data (door counts, artifacts)

**Configuration:** Override webhooks via:
- `window.SPATIAL_UPLOAD_WEBHOOK` (global before script)
- `<body data-upload-webhook="...">` attribute
- Hardcoded defaults (not exposed in UI)

### Dimension Normalization & Unit Conversion

**Key functions:**
- `parseFeetInches(value)` → Accepts "12'6\"", "12 feet", "6\"", "12", decimal numbers → returns decimal feet
- `round1(n)` → Rounds to 1 decimal place; treats 0 as null (important for "linear" measurements)
- `ftToM(ft)` → Multiply by 0.3048
- `mToFt(m)` → Multiply by 3.280839895013123

**Important:** Zero length/width values are normalized to `null` to distinguish between "no data" and "zero dimensions". This happens in `loadRoomsFromJSON()` and persists through UI updates.

### Payload Shape Flexibility

`deriveRooms(payload)` accepts multiple JSON formats:
- Direct array: `[{room, length, width}, ...]`
- Wrapper objects: `{rooms: [...]}, {data: [...]}, {result: [...]}, {output: [...]}`
- Unit field auto-detection: `unit: 'ft'|'m'|'feet'|'meters'` at payload or item level

---

## UI Rendering & Event Binding

### Rendering Functions

- **`renderRooms(list)`** → Clears container, builds room cards with inputs/badges/analysis sections
- **`updateTotals()`** → Recalculates total area, updates aria-live region
- **`validateProceed()`** → Enables/disables Proceed button (requires ≥1 room with name + any dimension)
- **`renderArtifactsSection(card, roomName)`** → Renders expandable door count + artifact (furniture) dimension tracker

### Contenteditable Room Titles

Room names use `contentEditable="true"` with custom focus styling (3px outline, inset shadow). Input triggers `state.rooms[].room = trimmed text` and validates Proceed button.

### Input Normalization

Width/Length inputs (type="number") trigger `input` events:
1. Get raw value, clamp to non-negative with `clampNonNeg()`
2. Convert zero to null via `round1()`
3. Update state + re-render badges (Area/Linear status)
4. Call `updateTotals()` and `validateProceed()`

---

## Design System (Neumorphism)

### CSS Custom Properties

Located in `:root` selector:
- **Colors:** `--terra` (terracotta), `--sand` (light yellow), `--text`, `--muted`
- **Surfaces:** `--card`, `--card-hi`, `--card-lo` (for subtle gradients)
- **Shadows:** `--shadow-ambient` (8px spread), `--shadow-key` (2px sharp)
- **Border:** `--border-soft` (rgba(0,0,0,.08))
- **Texture:** Base64-encoded sand JPEG overlaid with multiply blend mode

### Reusable Components

- `.neo-card` → Main elevated card (negative/positive shadows)
- `.neo-inset` → Recessed surface (inset shadows)
- `.icon-btn` → Square icon button (40×40, terracotta border)
- `.mini-btn` → Inline action button (38px height)
- `.room-card` → Complex grid layout for room edits (3-column grid with named areas)
- `.unit-switch` → iOS-style toggle (Ft/m) with track + thumb animation

### Responsive Grid

Room cards use `grid-template-columns: repeat(auto-fit, minmax(420px, 1fr))` for adaptive layout.

---

## Event Handling & State Updates

### Key Event Listeners

- **Add Room button** → `state.rooms.push({room:'', length:null, width:null, unit:state.unit})`
- **Sort button** → Toggle `state.sortAsc`, sort array, re-render
- **Unit toggle/select** → Call `setUnit('ft'|'m')` (converts all dimensions in place)
- **Room input blur** → Format value back to `fmt1()` (e.g., 12.3)
- **Dropzone drag/click** → `handleFiles()` → upload flow
- **Reset button** → `resetApp()` (clears state, re-renders, shows welcome)

### Async Patterns

`uploadToWebhook()` and `sendProceed()` use `fetch()` with:
- 30-second timeout via `AbortController`
- CORS fallback (if primary fails, retry with `mode: 'no-cors'`)
- Content-Type detection (JSON parsed if available)
- Toast notifications for success/error

---

## Testing (Built-in)

A minimal test suite runs on load (console-only, no assertion library):

```javascript
(function runTests() {
  // Tests cover:
  // - parseFeetInches("12'6\"") → 12.5
  // - Zero length normalization
  // - Unit conversion precision (ft ↔ m)
  // - deriveRooms() shape handling
  // - round1() null handling
})();
```

**Pattern:** `t(condition, message)` increments pass/fail counters. Failures logged as warnings.

---

## Common Patterns & Conventions

### Null as "No Data"

- Dimensions default to `null`, not 0
- `round1()` explicitly converts 0 → null
- UI shows "—" badge when length OR width is null (badge: "Linear"); both → "Area"
- Area badge hidden until both dimensions exist

### Unit Awareness

- Room objects always carry `.unit` field (should match `state.unit`)
- `setUnit()` converts all room dimensions in place using precise multipliers
- Unit conversion is lossless (stores 1 decimal place)

### Accessibility (ARIA)

- Live regions: `#totalArea` (aria-live="polite"), `#loading`, `#toast`
- Buttons/inputs have `aria-label` attributes
- Contenteditable elements use `role="textbox"`
- Focus outlines: 3px solid terracotta with 2px offset

### Error Handling

- Webhook failures fall back to `mode: 'no-cors'` (request succeeds, response hidden)
- Invalid JSON responses handled gracefully (toast: "Response received but not recognized")
- Network timeouts: 30 seconds → abort signal

---

## File Location Reference

**Single file structure:**
- `index.html` contains all HTML, CSS (in `<style>`), and JavaScript (in `<script>`)
- Configuration hardcoded in `<script>` (DEFAULT_UPLOAD_WEBHOOK, PROCEED_WEBHOOK)
- No external assets except Base64 sand texture

---

## Development Notes

1. **No dependencies:** Vanilla JS, no libraries. Keep it that way for portability.
2. **Testing:** Add new tests to the `runTests()` IIFE before deployment.
3. **HTTPS requirement:** Webhook URLs must use HTTPS if served via HTTPS (browser security).
4. **Dimension precision:** Always use 1 decimal place (`round1()`); displays with `fmt1()` for UI.
5. **Room naming:** Names are trimmed on input; empty names allowed (shows as "Room N" in badges).
6. **State persistence:** No localStorage/IndexedDB; session only. Data lost on page reload.

---

## Quick Reference: Common Tasks

| Task | Key Function | State Update |
|------|--------------|--------------|
| Add room | `state.rooms.push({...})` + `renderRooms()` | ✓ |
| Update room name | Contenteditable triggers `input` → `state.rooms[i].room = value` | ✓ |
| Change unit | `setUnit('ft'|'m')` (converts all dimensions) | ✓ |
| Send to server | `sendProceed()` (multipart form) | — |
| Load webhook response | `applyServerPayload()` → `deriveRooms()` → `loadRoomsFromJSON()` | ✓ |
| Load analysis | `applyAnalysisPayload()` → `loadArtifactsFromJSON()` | ✓ |

