# Your Cycle Keeper — AI Agent Instructions

## Project Overview

Privacy-first period tracking PWA with client-side AES-256-GCM encryption. **Zero server communication.** All data stays on-device using IndexedDB. Built with vanilla JavaScript ES6 modules—no frameworks or build tools.

## Architecture & Data Flow

### Core State Management

Central `state` object in [js/script.js](../js/script.js) holds all app data:

```javascript
state = {
  lastPeriodStart: "YYYY-MM-DD", // ISO date string
  cycleLength: 28, // days
  periodDuration: 5, // days
  logs: {}, // { "YYYY-MM-DD": { flow, pain, mood, note, periodStart, periodEnd } }
  cycleHistory: [], // past cycle records
};
```

State is passed by reference to modular functions via `setState()` calls—**never duplicate or recreate state**.

### Storage Layer: IndexedDB → AES-GCM → State

1. **IndexedDB** ([js/indexeddb-storage.js](../js/indexeddb-storage.js)): Persistent key-value store. Use `getFromDB(key)` / `setInDB(key, value)` only.
2. **Encryption** ([js/crypto.js](../js/crypto.js)): PBKDF2 (250k iterations) + AES-256-GCM with 12-byte IV. PIN never stored—derived to key on-demand.
3. **Session**: PIN held in `sessionPin` (memory only). Auto-lock after 5 min idle ([js/session.js](../js/session.js)).

**Never** use localStorage—replaced with IndexedDB for persistence.

### Modular Architecture (ES6 Modules)

- **[js/dateUtils.js](../js/dateUtils.js)**: ISO date strings (`toISO`, `fromISO`, `addDays`, `diffDays`)—always use YYYY-MM-DD format
- **[js/crypto.js](../js/crypto.js)**: `deriveKey`, `encryptData`, `decryptData`, `hashPin` using Web Crypto API
- **[js/cycles.js](../js/cycles.js)**: Cycle predictions via **Calendar Rhythm Method** (fertile window: days 8 to `cycleLength - 11`; ovulation: `cycleLength - 14`)
- **[js/periodMarking.js](../js/periodMarking.js)**: Period start/end logic, auto-cleanup of consecutive markers
- **[js/validators.js](../js/validators.js)**: Normalize user input (flow: 1-3, pain: 1-10 in 0.5 steps, mood: 0-100)
- **[js/session.js](../js/session.js)**: Timeout warnings, countdown timers, lock triggers

Each module calls `setState(state)` to receive shared state reference—**do not pass state as function arguments**.

## Critical Conventions

### Date Handling

- **ISO-only**: Internal dates are always `"YYYY-MM-DD"` strings
- Convert with `toISO(Date)` and `fromISO(string)`
- Never use `.toISOString()` (includes time zone)—use `toISO()` for consistency

### Log Structure

Daily logs keyed by ISO date in `state.logs`:

```javascript
state.logs["2026-03-07"] = {
  flow: 2, // 1=light, 2=medium, 3=heavy (optional)
  pain: 5.5, // 1-10 in 0.5 increments (optional)
  mood: 75, // 0-100 (optional)
  note: "...", // string (optional)
  periodStart: true, // marker (optional)
  periodEnd: true, // marker (optional)
};
```

Empty logs auto-deleted by `cleanupEmptyLogs()`.

### Security Rules

1. **No external network calls** (CSP blocks `connect-src: none`)—Service Worker cache-first for app shell only
2. **PIN validation**: Fast-fail with HMAC hash (`PINHASH_KEY`) before attempting decryption
3. **Schema versioning**: Encrypted envelope includes `{ v: SCHEMA_VERSION, payload: state }`
4. **Salt management**: Random 16-byte salt stored plaintext in IndexedDB (`SALT_KEY`)—**not secret**

### UI State Management

- `currentTab`: "calendar" | "insights" | "history" | "settings"
- `selectedDate`: ISO string for detail panel
- `viewMonth`: Date object for calendar navigation
- PIN screens: `#setup-screen`, `#unlock-screen`, `#app-screen`—toggle `.hidden` class

## Development Workflows

### Local Testing

```bash
# Requires HTTPS or localhost for Service Worker + Web Crypto API
python -m http.server 8000
# or
npx http-server
```

Access at `http://localhost:8000` (not file://)

### Deploying Updates

1. Bump `CACHE_VERSION` in [js/service-worker.js](../js/service-worker.js) (e.g., `v3.2.0`)
2. Update `?v=` query param in [index.html](../index.html) for CSS/JS busting
3. Deploy with `firebase deploy` ([firebase.json](../firebase.json) configured)

Service Worker auto-purges old caches on activation via `self.skipWaiting()`.

## Common Patterns

### Adding New Symptoms/Fields

1. Update `state.logs[date]` structure in [js/script.js](../js/script.js)
2. Add validator in [js/validators.js](../js/validators.js) (normalize + get value)
3. Bump `SCHEMA_VERSION` in [js/crypto.js](../js/crypto.js) if breaking change
4. Update `cleanupEmptyLogs()` to check new field

### Modifying Cycle Calculations

Edit [js/cycles.js](../js/cycles.js):

- `getCycleInfo()`: Current cycle phase/day
- `calculatePredictions()`: Future period/ovulation dates
- `getDayType()`: Period vs. fertile vs. ovulation classification

**Never** access `state` directly—always use module exports.

### Debugging Encryption Issues

- Check `PINHASH_KEY` in IndexedDB for fast PIN verification
- Decrypt manually: `decryptData(encrypted, pin, salt)` returns `{ v: 1, payload: state }`
- Salt mismatch = cannot decrypt (user must reset app)

## Chart Rendering (Insights Tab)

### Canvas-Based Symptom Tracking

Vanilla Canvas API renders multi-symptom charts in [js/script.js](../js/script.js):

```javascript
renderPainChart() → getPainDataMonth()/getPainDataYear()
```

**Key Implementation Details:**

- **DPR Scaling**: Canvas uses `window.devicePixelRatio` for retina displays—always scale context after setting pixel dimensions
- **View Modes**: Month view (daily bars) vs. Year view (monthly aggregates)—controlled by `#pain-view-month` select
- **Bar Segments**: Each day shows up to 3 symptoms side-by-side (flow, pain, mood)—fixed 1/3 width per symptom
- **Active Filter**: `activeChartFilter` toggles visibility ("all" | "period" | "ovulation" | "flow" | "pain" | "mood")

**Data Aggregation Pattern:**

```javascript
// Month: loop days 1-31, check state.logs[dateStr]
// Year: loop months 0-11, average non-null values
// Background markers: period (pink) and ovulation (gold) bands
```

**Chart Export**: `downloadChart()` creates new canvas with header, copies original, converts to PNG blob

## Firebase Hosting Configuration

### Deployment Strategy ([firebase.json](../firebase.json))

```bash
firebase deploy  # Deploys root directory as public folder
```

**Cache Headers by Asset Type:**

- **JS/CSS**: `max-age=2592000` (30 days, immutable)—safe because `?v=` query string busts cache
- **Images**: `max-age=31536000` (1 year, immutable)—icon hashes never change
- **index.html**: `no-cache`—always fetch fresh to load new versions
- **Service Worker**: `no-cache`—critical for update detection

**Security Headers (HTML only):**

```json
"X-Frame-Options": "DENY"           // Prevent clickjacking
"X-Content-Type-Options": "nosniff" // Block MIME sniffing
"Permissions-Policy": "camera=(), microphone=(), geolocation=()"  // Disable sensors
```

**SPA Routing**: All 404s rewrite to `/index.html` for client-side routing (not used currently but configured)

### Pre-Deploy Checklist:

1. Bump `CACHE_VERSION` in [js/service-worker.js](../js/service-worker.js)
2. Update `?v=3.x.x` in [index.html](../index.html) `<link>` and `<script>` tags
3. Test offline: Network tab → Offline checkbox → Reload

## PWA Offline Behavior

### Service Worker Strategy ([js/service-worker.js](../js/service-worker.js))

**Install**: Immediately activate new worker with `self.skipWaiting()` + delete all old caches
**Fetch Handling**:

- **HTML**: Network-first (try fetch → fallback to cache on fail)—ensures updates propagate
- **CSS/JS/Images**: Cache-first (return cache → fetch + cache if miss)—fast offline loads

**Critical Pattern**: HTML uses network-first to force app updates, but caches response for offline fallback.

### Manifest Configuration ([manifest.json](../manifest.json))

```json
"display": "standalone"          // Hides browser chrome (fullscreen PWA)
"orientation": "portrait"        // Locks to portrait on mobile
"background_color": "#120818"    // Splash screen background
"theme_color": "#5b1a8a"         // OS UI color (Android taskbar)
"categories": ["health", "lifestyle"]  // App store discovery
```

**Icon Sizes**: Must include 192×192 and 512×512 with `"purpose": "any maskable"` for Android adaptive icons.

**Install Prompt Trigger**: Browser auto-prompts when:

1. Service Worker registered
2. Manifest linked with valid icons
3. Site served over HTTPS
4. User engages with site (e.g., 30s interaction)

### Offline-First Requirements

- **No `fetch()` calls**: CSP blocks `connect-src: none`—app is 100% offline after first load
- **IndexedDB persistence**: Survives browser restarts (unlike `sessionStorage`)
- **Web Crypto API**: Requires HTTPS or localhost (fails on `file://` protocol)

## Testing Checklist

- [ ] Offline mode: Disable network, reload app
- [ ] Session timeout: Wait 5 min idle, verify auto-lock
- [ ] Wrong PIN: Should fail fast without decryption attempt
- [ ] Period marking: Toggle start/end, verify cleanup of consecutive markers
- [ ] Export/import: Backup should be encrypted blob (`.ycb` file)

## Forbidden Patterns

- ❌ **Do not** use `localStorage` (replaced with IndexedDB)
- ❌ **Do not** call `fetch()` or `XMLHttpRequest` (CSP blocks all network)
- ❌ **Do not** store PIN in state or IndexedDB
- ❌ **Do not** use `Date.toISOString()` (returns UTC with time; use `toISO()` instead)
- ❌ **Do not** pass `state` as function parameter (use module-level `setState()` reference)
