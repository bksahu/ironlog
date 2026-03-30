# IronLog — Claude Session Context Dump
# Generated: 2025
# Purpose: Paste this file into a new Claude conversation to restore full project context instantly.

---

## PROJECT SUMMARY

IronLog is a **personal gym workout tracker PWA** hosted on GitHub Pages, using Supabase for auth + database, and Chart.js for analytics. The entire app is a single `index.html` file with no build step.

---

## LIVE DETAILS

- **GitHub Pages URL**: https://[yourusername].github.io/iron-log  (user to fill in)
- **Supabase Project URL**: https://dwwpjwfbmbsoqfslolgc.supabase.co
- **Supabase Anon Key**: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImR3d3Bqd2ZibWJzb3Fmc2xvbGdjIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzQ4MTQ2NzMsImV4cCI6MjA5MDM5MDY3M30.oGFq4NAyEQpbzk6t0L7pJHsbxlsyMQh7ReGxVkm3rfU
- **Auth provider**: Google OAuth via Supabase
- **Hosting**: GitHub Pages (free, static)

---

## FILES IN THE REPO

```
index.html       ← Entire app (HTML + CSS + JS, ~1250 lines)
manifest.json    ← PWA manifest (name: IronLog, theme: #0a0a0a)
sw.js            ← Service worker (cache-first for assets)
icon-192.png     ← PWA icon 192x192 (dumbbell, yellow on black)
icon-512.png     ← PWA icon 512x512
README.md        ← Full architecture docs
CLAUDE.md        ← This file
```

---

## TECH STACK

| Layer       | Technology              |
|-------------|-------------------------|
| Hosting     | GitHub Pages            |
| Auth        | Supabase + Google OAuth |
| Database    | Supabase (PostgreSQL)   |
| Charts      | Chart.js v4 (CDN)       |
| SDK         | Supabase JS v2 (CDN, jsdelivr → unpkg fallback) |
| Fonts       | Google Fonts (Bebas Neue, DM Mono, Space Grotesk) |
| PWA         | Service Worker + Web App Manifest |
| Framework   | None — vanilla JS, vanilla CSS |

---

## DATABASE SCHEMA

```sql
create table if not exists sessions (
  id        text primary key,
  user_id   uuid references auth.users not null,
  date      text not null,             -- "YYYY-MM-DD"
  label     text,                      -- e.g. "Push Day"
  notes     text,
  exercises jsonb not null default '[]',
  saved_at  bigint                     -- Unix ms
);
alter table sessions enable row level security;
create policy "own_sessions" on sessions
  for all using (auth.uid() = user_id);
```

### exercises JSONB shape:
```json
[
  {
    "name": "Bench Press",
    "sets": [
      { "weight": "100", "reps": "5", "rpe": "8", "done": true }
    ]
  }
]
```

---

## AUTH FLOW

1. User taps "Continue with Google"
2. `supabase.auth.signInWithOAuth({ provider: 'google', redirectTo: window.location.href })`
3. Redirects to Google → user authenticates
4. Google redirects to Supabase callback URL
5. Supabase exchanges code for JWT, redirects back to app URL
6. Supabase SDK reads token from URL hash, stores JWT in localStorage
7. `onAuthStateChange` fires `SIGNED_IN` → `boot(session)` → `loadApp()`

**Key fix applied (logout/login loop bug):**
- Used `INITIAL_SESSION` event as primary boot trigger (not `getSession()`)
- Added `booted` flag to prevent double-boot race condition between `INITIAL_SESSION` and `SIGNED_IN`
- `SIGNED_IN` resets `booted = false` before calling `boot()` to allow re-login after logout
- Supabase client created with `detectSessionInUrl: true, persistSession: true, autoRefreshToken: true`
- URL hash is cleaned after OAuth redirect with `history.replaceState`

---

## KEY JAVASCRIPT FUNCTIONS

| Function | Purpose |
|---|---|
| `init()` | Entry point — creates Supabase client, sets up auth listener |
| `waitForSupabase(cb)` | Polls until Supabase SDK CDN has loaded, then calls cb |
| `boot(session)` | Single gated entry to app — sets currentUser, calls loadApp() |
| `loadApp()` | Shows app screen, loads library from localStorage, calls syncAll() |
| `showScreen(name)` | Switches between 'loading', 'auth', 'app' screens |
| `syncAll(silent)` | Fetches all sessions from Supabase for current user |
| `pushSession(s)` | Upserts a session to Supabase |
| `removeSessionRemote(id)` | Deletes a session from Supabase |
| `startSession()` | Creates currentSession object, shows session UI |
| `finishSession()` | Collects DOM data, saves to state + Supabase |
| `collectExercises()` | Scrapes exercise blocks and set inputs from DOM |
| `addExercise(name)` | Creates exercise block DOM element with datalist autocomplete |
| `addSet(exId)` | Adds a set row to an exercise block |
| `renderHistory()` | Renders session history list with search filter |
| `renderStats()` | Renders stat cards and PR table |
| `renderCharts()` | Calls all 4 chart renderers |
| `renderVolumeChart()` | Bar chart: total volume per session |
| `renderFreqChart()` | Bar chart: sessions per ISO week |
| `renderPRChart()` | Line chart: max weight over time for selected exercise |
| `renderMuscleChart()` | Horizontal bars: sets per muscle group (last 30 days) |
| `renderAll()` | Calls history + stats + recentMini renderers |
| `switchView(v)` | Tab navigation — activates view and nav tab |
| `isoWeek(dateStr)` | Returns "YYYY-Wnn" ISO week string for a date |

---

## STATE VARIABLES

```javascript
var supabase        = null;       // Supabase client instance
var currentUser     = null;       // Supabase auth user object
var sessions        = [];         // All sessions for current user (from Supabase)
var exerciseLibrary = [...];      // 130+ exercises, stored in localStorage
var currentSession  = null;       // In-progress session object (not yet saved)
var chartVolume     = null;       // Chart.js instance refs (destroyed before re-render)
var chartFreq       = null;
var chartPR         = null;
var volumePeriod    = '4w';       // Active filter for volume chart
var freqPeriod      = '8w';       // Active filter for freq chart
```

---

## UI STRUCTURE

```
#loading-screen    — spinner while Supabase SDK loads
#auth-screen       — Google sign-in button
#app               — main app shell (display:flex when active)
  header           — logo, sync indicator, +LOG button, avatar
  nav              — Today / History / Charts / Stats / Settings tabs
  main
    #view-log      — active session UI or recent sessions
    #view-history  — searchable session history list
    #view-charts   — 4 Chart.js visualizations
    #view-stats    — stat cards + PR table
    #view-settings — account info, sync, data export, exercise library
#confirm-modal     — reusable confirm dialog
#toast             — snackbar notification
```

---

## DESIGN SYSTEM

```css
--bg:          #0a0a0a   /* page background */
--surface:     #111111   /* cards */
--surface2:    #1a1a1a   /* inputs, nested surfaces */
--border:      #2a2a2a   /* all borders */
--accent:      #e8ff47   /* yellow-green primary accent */
--accent2:     #ff4757   /* red, danger/delete */
--text:        #f0f0f0   /* primary text */
--text-muted:  #666666   /* secondary text */
--text-dim:    #444444   /* placeholders */

Fonts:
  Bebas Neue   — display headings, dates, numbers
  DM Mono      — labels, metadata, nav tabs, monospaced values
  Space Grotesk — body text, inputs
```

---

## KNOWN BUGS FIXED

1. **Apostrophe in exercise name broke JS** — `Farmer's Walk` renamed to `Farmers Walk`
2. **Duplicate `waitForSupabase` function** — caused by layered edits; fixed by full script rewrite
3. **Corrupted `saveConfig` function** — old SDK loader code was jammed inside it; fixed in rewrite
4. **Login loop after logout** — `getSession()` fired before OAuth URL hash was processed, returned null, redirected to auth screen. Fixed with `INITIAL_SESSION` event + `booted` gate + `detectSessionInUrl: true`

---

## KNOWN LIMITATIONS (NOT YET FIXED)

- Offline session persistence — sessions finished offline are lost if tab is closed before reconnection (no IndexedDB queue)
- Exercise library not synced to Supabase — stored in localStorage per device only
- No workout templates / copy-last-session feature
- No rest timer
- Weight unit is kg only (no lb toggle)
- No sets auto-fill from previous session

---

## HOW TO USE THIS FILE

Paste the contents of this file at the start of a new Claude conversation with a message like:

> "Here is the context for my IronLog project. [paste file]. I want to [describe what you need]."

Claude will have full context of the architecture, all decisions made, bugs fixed, and current state of the codebase without needing to re-explain anything.
