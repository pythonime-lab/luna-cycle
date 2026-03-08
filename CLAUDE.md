# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Your Cycle Keeper** is a privacy-first period tracking PWA with client-side AES-256-GCM encryption. Zero server communication—all data stays on-device using IndexedDB. Built with vanilla JavaScript ES6 modules (no frameworks, no build tools, no dependencies).

**Live URL:** https://yourcyclekeeper.web.app

## Development Commands

```bash
# Start local server (required for Service Worker + Web Crypto API)
python -m http.server 8000
# or
npx http-server

# Access at http://localhost:8000 (NOT file://)

# Deploy to Firebase
firebase deploy
```

### Pre-Deploy Checklist

1. Bump `CACHE_VERSION` in `service-worker.js` (e.g., `v20260307`)
2. Update `?v=` query param in `index.html` for CSS/JS cache busting
3. Test offline: DevTools → Network → Offline → Reload

## Architecture

### Core State Management

Central `state` object in `js/script.js` holds all app data:

```javascript
state = {
  lastPeriodStart: "YYYY-MM-DD",    // ISO date string
  cycleLength: 28,                   // days
  periodDuration: 5,                 // days
  logs: {                            // { "YYYY-MM-DD": { flow, pain, mood, note, periodStart, periodEnd } }
    "YYYY-MM-DD": {
      flow: 1-3,                     // 1=light, 2=medium, 3=heavy
      pain: 1-10,                    // 0.5 increments
      mood: 0-100,                   // 0=low, 100=happy
      note: "...",                   // max 500 chars
      periodStart: boolean,
      periodEnd: boolean
    }
  },
  cycleHistory: []                   // past cycle records
};
```

**State is passed by reference** to modules via `setState()` calls—never duplicate or pass as function arguments.

### Storage Layer: IndexedDB → AES-GCM → State

1. **IndexedDB** (`js/indexeddb-storage.js`): Persistent key-value store
2. **Encryption** (`js/crypto.js`): PBKDF2 (250k iterations) + AES-256-GCM with 12-byte IV
3. **Session** (`js/session.js`): PIN held in memory only, auto-lock after 5 min idle

### Module Structure

| Module | Purpose |
|--------|---------|
| `js/script.js` | Main app logic, state management, UI rendering, chart drawing |
| `js/crypto.js` | `deriveKey`, `encryptData`, `decryptData`, `hashPin` |
| `js/indexeddb-storage.js` | `getFromDB`, `setInDB`, `deleteFromDB`, `clearDB` |
| `js/cycles.js` | Cycle predictions via Calendar Rhythm Method |
| `js/periodMarking.js` | Period start/end logic, auto-cleanup of consecutive markers |
| `js/validators.js` | Input normalization (flow 1-3, pain 1-10, mood 0-100) |
| `js/dateUtils.js` | ISO date utilities (`toISO`, `fromISO`, `addDays`, `diffDays`) |
| `js/session.js` | Timeout warnings, countdown timers, lock triggers |
| `js/navigation.js` | Keyboard accessibility and focus management |

### Cycle Prediction Algorithm

Calendar Rhythm Method + Standard Days Method:
- **Fertile window:** Days 8 through `(cycleLength - 11)`
- **Ovulation:** Day `(cycleLength - 14)` from period start
- **getDayType()** returns: `"period"`, `"fertile"`, `"ovulation"`, or `"normal"`

## Critical Conventions

### Date Handling

- **Always use ISO format:** `"YYYY-MM-DD"` strings
- **Convert with:** `toISO(Date)` and `fromISO(string)` from `js/dateUtils.js`
- **Never use:** `.toISOString()` (includes timezone)

### Security Rules

- **No network calls:** CSP blocks `connect-src: none`—app is 100% offline after first load
- **PIN never stored:** Derived to key on-demand using PBKDF2
- **Fast PIN validation:** HMAC hash check before attempting decryption
- **Schema versioning:** Encrypted envelope wraps state as `{ v: SCHEMA_VERSION, payload: state }`

### UI Screens

Toggle `.hidden` class to switch between:
- `#onboarding` - Initial PIN setup
- `#lock-screen` - Session timeout lock
- `#app-screen` - Main interface

Tabs: `"calendar"`, `"insights"`, `"history"`, `"settings"`

## Common Patterns

### Adding New Symptoms/Fields

1. Update `state.logs[date]` structure in `js/script.js`
2. Add validator in `js/validators.js`
3. Bump `SCHEMA_VERSION` in `js/crypto.js` if breaking change
4. Update `cleanupEmptyLogs()` to check new field

### Modifying Cycle Calculations

Edit `js/cycles.js`:
- `getCycleInfo()` - Current cycle phase/day
- `calculatePredictions()` - Future period/ovulation dates
- `getDayType()` - Period vs. fertile vs. ovulation classification

### Chart Rendering (Insights Tab)

Canvas-based with DPR scaling for retina displays:
- Month view: Daily bars (days 1-31)
- Year view: Monthly aggregates
- `activeChartFilter`: `"all"`, `"period"`, `"ovulation"`, `"flow"`, `"pain"`, `"mood"`

## Forbidden Patterns

- **Do not** use `localStorage` (replaced with IndexedDB)
- **Do not** call `fetch()` or `XMLHttpRequest` (CSP blocks all network)
- **Do not** store PIN in state or IndexedDB
- **Do not** use `Date.toISOString()` (use `toISO()` instead)
- **Do not** pass `state` as function parameter (use module-level `setState()` reference)

## Testing Checklist

- Offline mode: Disable network in DevTools, reload app
- Session timeout: Wait 5 min idle, verify auto-lock with countdown
- Wrong PIN: Should fail fast with HMAC check (no decryption attempt)
- Period marking: Toggle start/end, verify cleanup of consecutive markers
