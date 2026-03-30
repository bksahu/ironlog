# IronLog — Progressive Web App Workout Tracker

> A fully offline-capable, cross-device gym session logger built as a single-file PWA. Powered by Supabase for authentication and cloud storage, Chart.js for analytics, and hosted for free on GitHub Pages.

---

## Table of Contents

1. [Overview](#overview)
2. [Feature List](#feature-list)
3. [Architecture](#architecture)
4. [File Structure](#file-structure)
5. [Technology Stack](#technology-stack)
6. [How It Works — End to End](#how-it-works--end-to-end)
7. [Database Schema](#database-schema)
8. [Authentication Flow](#authentication-flow)
9. [Data Flow & Sync](#data-flow--sync)
10. [Charts & Metrics](#charts--metrics)
11. [Exercise Library](#exercise-library)
12. [PWA Behaviour](#pwa-behaviour)
13. [Deployment Guide — GitHub Pages](#deployment-guide--github-pages)
14. [Supabase Setup Guide](#supabase-setup-guide)
15. [Google OAuth Setup Guide](#google-oauth-setup-guide)
16. [Installing on Mobile](#installing-on-mobile)
17. [Usage Guide](#usage-guide)
18. [Security Model](#security-model)
19. [Offline Behaviour](#offline-behaviour)
20. [Known Limitations](#known-limitations)
21. [Design System](#design-system)

---

## Overview

IronLog is a personal gym workout tracker designed to be used daily across multiple devices — phone at the gym, laptop at home. It is built as a **Progressive Web App (PWA)**, meaning it installs directly to your home screen on both Android and iOS without going through an app store, works offline after the first load, and syncs all data to the cloud via Supabase so your sessions are always available regardless of which device you open it on.

The entire application is a **single HTML file** (`index.html`) — there is no build step, no Node.js server, no backend to maintain. The only external dependencies are loaded from CDNs at runtime: the Supabase JavaScript SDK for auth and database, and Chart.js for data visualizations. Everything else — UI, state management, routing, data logic — is hand-written vanilla JavaScript.

---

## Feature List

### Session Logging
- Start a workout session with a single tap
- Set a custom date for the session (defaults to today)
- Optionally label the session (e.g. "Push Day", "Leg Day", "Upper A")
- Add any number of exercises per session
- For each exercise, log any number of sets with:
  - **Weight** (kg, decimal supported e.g. 82.5)
  - **Reps** (integer)
  - **RPE** (Rate of Perceived Exertion, 1–10 scale, decimal supported)
  - **Done checkbox** — mark a set complete as you go
- Delete individual sets or entire exercises
- Add free-text notes to the session
- Finish or discard the session at any time

### History
- Full paginated history of all past sessions, newest first
- Each session shows date, total sets, exercise count, and label
- Tap any session to expand and see full exercise/set breakdown
- Set details shown as pills: `80kg × 8r`
- Session notes displayed at the bottom of the expanded view
- Search bar filters sessions by exercise name or session label in real time
- Delete any session (removes from Supabase too)

### Charts & Metrics
Four interactive charts rendered with Chart.js:
- **Volume Over Time** — bar chart of total kg lifted per session (weight × reps summed across all sets). Time range toggles: 4W / 3M / 6M / ALL
- **Session Frequency** — bar chart of sessions per ISO week. Range toggles: 8W / 6M / ALL
- **PR Progression** — line chart showing the max weight logged for a selected exercise over time. Exercise dropdown auto-populates from your history
- **Muscle Group Frequency** — horizontal progress bars showing how many sets per muscle group you have logged in the last 30 days, sorted by volume

### Stats
- Total sessions logged (all time)
- Total sets logged (all time)
- Sessions this month
- Week streak (consecutive ISO weeks with at least one session)
- Personal Records table — max weight ever logged per exercise, sorted by heaviest first, top 12 shown

### Authentication
- Google OAuth via Supabase Auth
- One-tap sign-in with your Google account
- Sessions persist across devices — sign in on phone, laptop, tablet, all data is there
- Supabase credentials are hardcoded in the app (anon key, safe to be public)
- No email/password, no separate account creation

### Settings
- Account info panel showing your Google name and email with avatar
- Sync status indicator (syncing / synced / error) with last sync time
- Manual "Sync Now" button
- Export all sessions as a JSON file (for manual backup)
- Import sessions from a previously exported JSON file (merges, no duplicates)
- Clear local cache (does not affect Supabase data)
- Custom exercise library management — add your own exercises, remove custom ones

### PWA / Install
- Installable to home screen on Android (Chrome) and iOS (Safari)
- Runs fullscreen with no browser UI once installed
- Offline capable after first load — service worker caches the app shell
- Custom favicon and app icon (dumbbell, yellow-on-black)
- Splash screen uses theme colour

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     USER'S DEVICE                       │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Browser / PWA Shell                 │   │
│  │                                                  │   │
│  │  ┌────────────────────────────────────────────┐  │   │
│  │  │              index.html                    │  │   │
│  │  │                                            │  │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌─────────┐   │  │   │
│  │  │  │   UI     │  │  State   │  │ Charts  │   │  │   │
│  │  │  │  Layer   │  │ Manager  │  │Chart.js │   │  │   │
│  │  │  └──────────┘  └──────────┘  └─────────┘   │  │   │
│  │  │                                            │  │   │
│  │  │  ┌──────────────────────────────────────┐  │  │   │
│  │  │  │         Supabase JS SDK              │  │  │   │
│  │  │  │   auth.signInWithOAuth()             │  │  │   │
│  │  │  │   from('sessions').select()          │  │  │   │
│  │  │  │   from('sessions').upsert()          │  │  │   │
│  │  │  └──────────────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────────────┘  │   │
│  │                                                  │   │
│  │  ┌────────────────────────────────────────────┐  │   │
│  │  │           Service Worker (sw.js)           │  │   │
│  │  │   Caches app shell for offline use         │  │   │
│  │  └────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                          │
                          │ HTTPS
                          │
┌─────────────────────────────────────────────────────────┐
│                      SUPABASE                           │
│                                                         │
│  ┌──────────────────┐   ┌──────────────────────────┐    │
│  │   Auth Service   │   │    PostgreSQL Database   │    │
│  │                  │   │                          │    │
│  │  Google OAuth    │   │  sessions table          │    │
│  │  JWT tokens      │   │  Row Level Security      │    │
│  │  User profiles   │   │  (user sees own data)    │    │
│  └──────────────────┘   └──────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                          │
                          │ OAuth redirect
                          │
┌─────────────────────────────────────────────────────────┐
│                    GOOGLE CLOUD                         │
│              OAuth 2.0 Identity Provider                │
│         (issues tokens, verifies identity)              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   GITHUB PAGES                          │
│         Static file hosting (index.html etc.)           │
│         Free, served over HTTPS, custom domain ok       │
└─────────────────────────────────────────────────────────┘
```

### Key architectural decisions

**Single file app** — The entire application logic, styles, and HTML live in `index.html`. This makes deployment trivial (just upload one file), eliminates build tooling, and keeps the project maintainable by a single person. The trade-off is that the file is large (~1500 lines), but this is acceptable for a personal tool.

**No backend** — There is no custom server. Supabase handles authentication and the database. GitHub Pages handles static file serving. This means zero ongoing infrastructure cost and zero maintenance burden.

**Supabase as BaaS** — Supabase provides a hosted PostgreSQL database with a REST API auto-generated from the schema, plus built-in auth with social providers. The JavaScript SDK abstracts all HTTP calls. Row Level Security at the database level ensures data isolation between users.

**CDN-loaded dependencies** — Chart.js and the Supabase SDK are loaded from CDNs at runtime with a fallback chain (jsdelivr → unpkg) to handle CDN outages. If both fail, a friendly error screen with a retry button is shown.

**State management** — There is no Redux or reactive framework. State is a handful of module-level variables (`sessions`, `currentUser`, `currentSession`, etc.). UI is re-rendered by calling render functions explicitly after state changes. This is intentional simplicity for a small personal app.

---

## File Structure

```
iron-log/                   ← GitHub repo root
│
├── index.html              ← Entire application (HTML + CSS + JS)
├── manifest.json           ← PWA web app manifest
├── sw.js                   ← Service worker for offline caching
├── icon-192.png            ← App icon (192×192, for Android home screen)
├── icon-512.png            ← App icon (512×512, for splash screens)
└── README.md               ← This file
```

### index.html internals

```
index.html
│
├── <head>
│   ├── Meta tags (charset, viewport, PWA, theme-color)
│   ├── Google Fonts (Bebas Neue, DM Mono, Space Grotesk)
│   ├── Chart.js (from cdnjs)
│   └── Supabase JS SDK (jsdelivr with unpkg fallback)
│
├── <style>
│   ├── CSS custom properties (design tokens)
│   ├── Screen layouts (auth, config, loading, app)
│   ├── Navigation styles
│   ├── Form elements
│   ├── Exercise block and sets table
│   ├── History items
│   ├── Chart cards
│   ├── Stats cards
│   ├── Modal overlay
│   └── Toast notification
│
├── <body>
│   ├── #loading-screen       ← Spinner shown while SDK loads
│   ├── #auth-screen          ← Google sign-in button
│   ├── #config-screen        ← (Legacy, no longer shown)
│   ├── #app                  ← Main application shell
│   │   ├── <header>          ← Logo, sync indicator, + LOG button, avatar
│   │   ├── <nav>             ← Today / History / Charts / Stats / Settings tabs
│   │   └── <main>
│   │       ├── #view-log     ← Active session logging UI
│   │       ├── #view-history ← Session history list
│   │       ├── #view-charts  ← Chart.js visualizations
│   │       ├── #view-stats   ← Stat cards + PR table
│   │       └── #view-settings← Account, sync, data, library
│   ├── #confirm-modal        ← Reusable confirmation dialog
│   └── #toast                ← Notification snackbar
│
└── <script>
    ├── Constants (SUPABASE_URL, SUPABASE_KEY, SETUP_SQL)
    ├── Default exercise library (130+ exercises)
    ├── Muscle group mapping (for frequency chart)
    ├── State variables
    ├── init() — entry point
    ├── showScreen() — screen switcher
    ├── Auth functions (signInWithGoogle, signOut, updateUserUI)
    ├── Sync functions (syncAll, pushSession, removeSessionRemote)
    ├── Navigation (switchView)
    ├── Session management (startSession, finishSession, discardSession)
    ├── Exercise block DOM builders (addExercise, addSet, removeSet)
    ├── History renderer (renderHistory, renderRecentMini)
    ├── Stats renderer (renderStats, calcStreak)
    ├── Chart renderers (renderVolumeChart, renderFreqChart, renderPRChart, renderMuscleChart)
    ├── Exercise library (renderLibraryList, addToLibrary, removeFromLibrary)
    ├── Import/Export (exportData, handleImport)
    ├── Helpers (formatDate, fmtDateShort, fmtTime, isoWeek, showConfirm, toast)
    ├── waitForSupabase() — SDK load gate
    └── Service worker registration
```

---

## Technology Stack

| Layer | Technology | Why |
|---|---|---|
| Hosting | GitHub Pages | Free, HTTPS, no config |
| Auth | Supabase Auth + Google OAuth | Free tier, social login, JWT |
| Database | Supabase (PostgreSQL) | Free tier, RLS, REST API |
| SDK | Supabase JS v2 | Official client, CDN available |
| Charts | Chart.js v4 | Lightweight, CDN available |
| Fonts | Google Fonts | Bebas Neue, DM Mono, Space Grotesk |
| PWA | Service Worker + Web App Manifest | Offline support, installable |
| Styling | Vanilla CSS with custom properties | No framework needed |
| Logic | Vanilla JavaScript (ES6+) | No build step |

---

## How It Works — End to End

### First visit on a new device

1. Browser requests `index.html` from GitHub Pages over HTTPS
2. The Supabase SDK fallback loader runs — tries jsdelivr, falls back to unpkg if needed
3. `waitForSupabase()` polls until `window.supabase.createClient` is available
4. `init()` runs — creates a Supabase client with the hardcoded URL and anon key
5. `supabase.auth.getSession()` checks for an existing session (stored in the browser's localStorage by the SDK)
6. No session found → `showScreen('auth')` → user sees the Google sign-in screen
7. User taps "Continue with Google" → `supabase.auth.signInWithOAuth()` redirects to Google
8. Google authenticates the user and redirects back to the app URL with an OAuth code
9. Supabase SDK intercepts the redirect, exchanges the code for a JWT session
10. `onAuthStateChange` fires with `SIGNED_IN` event
11. `loadApp()` runs — loads exercise library from localStorage, shows the app shell
12. `syncAll()` fetches all sessions from Supabase for this user
13. `renderAll()` populates history, stats, recent sessions
14. Service worker registers in the background, caching the app shell

### Subsequent visits (same device)

1. Service worker intercepts the request and serves `index.html` from cache instantly
2. Supabase SDK finds the JWT in localStorage — session is still valid
3. `onAuthStateChange` fires immediately with `SIGNED_IN`
4. App loads and syncs in the background — typically under 1 second

### Logging a session

1. Tap **+ LOG** or **START SESSION**
2. A `currentSession` object is created in memory with a unique ID and today's date
3. One empty exercise block is added to the DOM automatically
4. User types exercise name — browser shows autocomplete suggestions from the datalist (populated from `exerciseLibrary`)
5. Three empty set rows are pre-added; user fills weight / reps / RPE
6. Tap **+ SET** to add more rows, **+ ADD EXERCISE** for another exercise
7. Mark sets done with the circle button (turns yellow)
8. Add optional label and notes
9. Tap **✓ FINISH** → `collectExercises()` scrapes all input values from the DOM into an array
10. Session object is pushed to the front of the `sessions` array in memory
11. `pushSession()` calls `supabase.from('sessions').upsert()` — writes to Supabase
12. UI resets, history re-renders with the new session at the top

### Viewing history on another device

1. Open the app on the second device
2. Sign in with the same Google account
3. `syncAll()` fetches all sessions with `supabase.from('sessions').select('*').eq('user_id', currentUser.id)`
4. Sessions render immediately — identical to the first device

---

## Database Schema

```sql
create table if not exists sessions (
  id        text primary key,          -- "session_1234567890" (timestamp-based)
  user_id   uuid references auth.users not null, -- Supabase auth user ID
  date      text not null,             -- "YYYY-MM-DD" ISO date string
  label     text,                      -- Optional session label e.g. "Push Day"
  notes     text,                      -- Optional free-text notes
  exercises jsonb not null default '[]', -- Array of exercise objects (see below)
  saved_at  bigint                     -- Unix timestamp ms when saved
);

-- Row Level Security
alter table sessions enable row level security;

create policy "own_sessions" on sessions
  for all using (auth.uid() = user_id);
```

### exercises JSONB structure

```json
[
  {
    "name": "Bench Press",
    "sets": [
      {
        "weight": "100",
        "reps": "5",
        "rpe": "8",
        "done": true
      },
      {
        "weight": "100",
        "reps": "5",
        "rpe": "8.5",
        "done": true
      },
      {
        "weight": "97.5",
        "reps": "5",
        "rpe": "9",
        "done": true
      }
    ]
  },
  {
    "name": "Incline Dumbbell Press",
    "sets": [
      { "weight": "36", "reps": "10", "rpe": "7", "done": true },
      { "weight": "36", "reps": "9",  "rpe": "8", "done": true }
    ]
  }
]
```

All numeric values (weight, reps, RPE) are stored as strings because they come directly from HTML input fields. They are cast to `parseFloat()` at query time when used in calculations.

---

## Authentication Flow

```
User taps "Continue with Google"
          │
          ▼
supabase.auth.signInWithOAuth({ provider: 'google' })
          │
          ▼
Browser redirects to accounts.google.com
          │
          ▼
User selects Google account / already logged in
          │
          ▼
Google redirects to: https://dwwpjwfbmbsoqfslolgc.supabase.co/auth/v1/callback
          │
          ▼
Supabase exchanges OAuth code for access token + refresh token
          │
          ▼
Supabase redirects back to: https://yourusername.github.io/iron-log
          │
          ▼
Supabase JS SDK reads the URL hash, stores JWT in localStorage
          │
          ▼
onAuthStateChange fires: event = 'SIGNED_IN'
          │
          ▼
loadApp() → syncAll() → app is ready
```

The JWT is automatically refreshed by the Supabase SDK before it expires. The refresh token is stored in localStorage so the user stays logged in indefinitely across browser sessions without re-authenticating.

---

## Data Flow & Sync

IronLog uses a **cloud-primary** sync model — Supabase is the source of truth.

```
┌──────────┐   upsert()    ┌────────────┐   select()   ┌──────────┐
│ Device A │ ────────────► │  Supabase  │ ◄─────────── │ Device B │
│ (phone)  │               │ PostgreSQL │              │ (laptop) │
└──────────┘               └────────────┘              └──────────┘
```

- On **finish session** → `pushSession()` upserts the session to Supabase immediately
- On **app open** → `syncAll()` fetches all sessions from Supabase and overwrites local state
- On **delete session** → `removeSessionRemote()` deletes from Supabase, removed from local array
- On **manual sync** (Settings → Sync Now) → `syncAll()` re-fetches everything
- There is **no offline queue** — if you finish a session while offline, it will be saved in memory for the current browser session but won't persist if you close the tab before connectivity is restored. A future improvement would be to queue failed upserts in IndexedDB.

The sync status indicator in the header shows:
- **⟳** (yellow, pulsing) — sync in progress
- **No indicator** — synced successfully
- **!** (red) — last sync failed

---

## Charts & Metrics

### Volume Over Time

**Formula:** For each session, sum all `weight × reps` across every set of every exercise.

```
session_volume = Σ (exercise.sets → weight × reps)
```

Plotted as a bar chart with one bar per session. Sessions with no weight data (cardio only, bodyweight) show as 0. Time range filter clips sessions older than the selected window.

### Session Frequency

Sessions are grouped by **ISO week** (Monday to Sunday). The bar height is the count of sessions in that week. Useful for spotting weeks where you missed training.

ISO week calculation:
```javascript
function isoWeek(dateStr) {
  // Returns "YYYY-Wnn" e.g. "2025-W14"
}
```

### PR Progression

For a selected exercise (chosen from dropdown), plots the **maximum weight** recorded across all sets in each session that included that exercise, in chronological order. This shows your strength progression on that lift over time. Only exercises where at least one set has a weight > 0 appear in the dropdown.

### Muscle Group Frequency

Each exercise in the library is mapped to one or more muscle groups via `muscleMap`. For every session in the last 30 days, the number of sets per exercise is accumulated into the muscle group bucket. The result is displayed as horizontal progress bars sorted by volume (most to least). This helps identify muscle groups you are undertraining.

**Muscle group mappings:**

| Group | Example exercises |
|---|---|
| Chest | Bench Press, Incline Bench, Cable Fly, Pec Deck |
| Back | Deadlift, Barbell Row, Pull-Up, Lat Pulldown |
| Shoulders | Overhead Press, Lateral Raise, Rear Delt Fly |
| Biceps | Barbell Curl, Hammer Curl, Preacher Curl |
| Triceps | Skull Crusher, Tricep Pushdown, Dip |
| Quads | Squat, Leg Press, Leg Extension, Lunge |
| Hamstrings | Leg Curl, Nordic Curl, Good Morning |
| Glutes | Hip Thrust, Glute Bridge, Cable Kickback |
| Calves | Standing Calf Raise, Seated Calf Raise |
| Core | Plank, Ab Wheel Rollout, Hanging Leg Raise |

---

## Exercise Library

The default library contains **130+ exercises** across all major muscle groups and movement patterns. These are pre-loaded into a `<datalist>` element linked to each exercise name input, providing native browser autocomplete as the user types.

Categories included:
- Chest (17 exercises)
- Back (23 exercises)
- Shoulders (12 exercises)
- Biceps (11 exercises)
- Triceps (10 exercises)
- Quads (12 exercises)
- Hamstrings & Glutes (10 exercises)
- Calves (3 exercises)
- Core (11 exercises)
- Full Body / Olympic (15 exercises)
- Cardio machines (10 exercises)

Custom exercises can be added via Settings → Exercise Library. They are persisted in `localStorage` and survive page refreshes. Custom exercises can be deleted; default exercises cannot. The library is per-device (not synced to Supabase) since it is a preference, not workout data.

---

## PWA Behaviour

### Web App Manifest (`manifest.json`)

```json
{
  "name": "IronLog",
  "short_name": "IronLog",
  "display": "standalone",
  "background_color": "#0a0a0a",
  "theme_color": "#0a0a0a",
  "start_url": ".",
  "icons": [...]
}
```

- `display: standalone` — hides browser chrome when launched from home screen
- `theme_color` — colours the status bar on Android
- `start_url: "."` — opens at the root, works with GitHub Pages subdirectory URLs

### Service Worker (`sw.js`)

Uses a **cache-first for assets, network-first for API calls** strategy:

```
Fetch request
    │
    ├─ Is it a Supabase API call? ──► Network first (fresh data)
    │
    └─ Everything else ──► Cache first
                              │
                              ├─ Cache hit? ──► Return cached response
                              │
                              └─ Cache miss? ──► Fetch from network
                                                  └─► Store in cache
                                                  └─► Return response
```

The service worker caches: `index.html`, `manifest.json`, and itself. On activation, old cache versions are deleted. On install, `skipWaiting()` is called so the new service worker activates immediately without waiting for all tabs to close.

---

## Deployment Guide — GitHub Pages

### Step 1 — Create a GitHub account
If you don't have one, sign up at [github.com](https://github.com).

### Step 2 — Create a new repository
1. Click the **+** icon → **New repository**
2. Name it `iron-log` (or anything you like)
3. Set visibility to **Public** (required for free GitHub Pages)
4. Do NOT initialise with README — leave it empty
5. Click **Create repository**

### Step 3 — Upload the files
1. On the empty repo page, click **uploading an existing file**
2. Drag and drop all 5 files: `index.html`, `manifest.json`, `sw.js`, `icon-192.png`, `icon-512.png`
3. Scroll down, write a commit message like "Initial upload"
4. Click **Commit changes**

### Step 4 — Enable GitHub Pages
1. Go to your repo → **Settings** tab
2. Scroll down to **Pages** in the left sidebar
3. Under **Source**, select **Deploy from a branch**
4. Branch: `main`, Folder: `/ (root)`
5. Click **Save**

### Step 5 — Wait and access
- GitHub takes 1–3 minutes to build and deploy
- Go to the **Actions** tab to watch the deployment progress
- Once complete, your app is live at:
  `https://yourusername.github.io/iron-log`

### Updating the app
Just upload a new `index.html` to the repo (drag and drop → commit). GitHub Pages will redeploy automatically within 1–2 minutes. Hard refresh (`Ctrl+Shift+R`) on the browser to bypass cache.

---

## Supabase Setup Guide

### Step 1 — Create a Supabase account
Go to [supabase.com](https://supabase.com) → Sign up (free) → Create a new organisation and project. Choose a region close to you. Wait ~1 minute for the project to initialise.

### Step 2 — Get your credentials
Go to **Project Settings → API**:
- Copy the **Project URL** (looks like `https://xxxxxxxxxxxx.supabase.co`)
- Copy the **anon / public** key (the long `eyJ…` string)

These are already hardcoded into `index.html` for this project.

### Step 3 — Create the database table
Go to **SQL Editor** in the Supabase dashboard and run:

```sql
create table if not exists sessions (
  id        text primary key,
  user_id   uuid references auth.users not null,
  date      text not null,
  label     text,
  notes     text,
  exercises jsonb not null default '[]',
  saved_at  bigint
);

alter table sessions enable row level security;

create policy "own_sessions" on sessions
  for all using (auth.uid() = user_id);
```

This only needs to be done **once**.

### Step 4 — Configure allowed URLs
Go to **Authentication → URL Configuration**:
- **Site URL**: `https://yourusername.github.io/iron-log`
- **Redirect URLs**: add `https://yourusername.github.io/iron-log`

---

## Google OAuth Setup Guide

### Step 1 — Google Cloud Console
1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (or use an existing one)
3. Go to **APIs & Services → OAuth consent screen**
4. Choose **External** → Create
5. Fill in App name ("IronLog"), support email, developer email
6. Skip scopes, skip test users, publish the app

### Step 2 — Create OAuth credentials
1. Go to **APIs & Services → Credentials**
2. Click **+ Create Credentials → OAuth Client ID**
3. Application type: **Web application**
4. Under **Authorised redirect URIs**, add:
   ```
   https://dwwpjwfbmbsoqfslolgc.supabase.co/auth/v1/callback
   ```
5. Click **Create**
6. Copy the **Client ID** and **Client Secret**

### Step 3 — Add to Supabase
1. Go to Supabase → **Authentication → Providers → Google**
2. Toggle **Enable**
3. Paste your Client ID and Client Secret
4. Click **Save**

### Done
Google sign-in will now work. The redirect goes: App → Google → Supabase callback → App.

---

## Installing on Mobile

### Android (Chrome)
1. Open the app URL in **Chrome**
2. Chrome will show a banner: **"Add IronLog to Home screen"** — tap it
3. Or: tap ⋮ menu → **Add to Home screen**
4. Tap **Add** in the confirmation dialog
5. IronLog appears in your app drawer and home screen

### iOS (Safari)
1. Open the app URL in **Safari** (must be Safari — Chrome/Firefox on iOS cannot install PWAs)
2. Tap the **Share button** (box with upward arrow) at the bottom of the screen
3. Scroll down and tap **Add to Home Screen**
4. Tap **Add** in the top right
5. IronLog appears on your home screen

Once installed, the app opens fullscreen with no browser address bar, exactly like a native app.

---

## Usage Guide

### Starting a session
1. Tap **+ LOG** in the header or **START SESSION** on the Today tab
2. The date defaults to today — tap it to change if logging a past session
3. Optionally enter a session label (e.g. "Push A", "Deadlift day")
4. The first exercise block is added automatically

### Logging exercises
1. Type the exercise name in the input — autocomplete suggestions appear from the library
2. Fill in sets: enter weight (kg), reps, and optionally RPE (1–10)
3. Tap the circle button on each set to mark it done (turns yellow with a checkmark)
4. Tap **+ SET** to add more set rows
5. Tap **+ ADD EXERCISE** to add another exercise block
6. Tap ✕ on a set row or exercise block to remove it

### Finishing a session
1. Tap **✓ FINISH** when done
2. The session is saved to Supabase immediately
3. You return to the Today tab showing recent sessions

### Viewing history
1. Tap the **History** tab
2. Sessions are listed newest first with date, set count, and exercise count
3. Tap any session to expand and see the full breakdown
4. Use the search bar to filter by exercise name or session label
5. Tap the ✕ button on a session to delete it permanently

### Viewing charts
1. Tap the **Charts** tab
2. Use the time range chips (4W / 3M / 6M / ALL) to change the window
3. For PR Progression, use the dropdown to select an exercise

### Viewing stats
1. Tap the **Stats** tab
2. Stat cards show at a glance: sessions, sets, this month, week streak
3. Scroll down for your Personal Records table

### Adding custom exercises
1. Go to **Settings** tab → Exercise Library
2. Type the exercise name in the input field
3. Press Enter or tap **ADD**
4. The exercise now appears in autocomplete on all future exercise inputs

### Exporting data
1. Go to **Settings → Data → ⬇ EXPORT JSON**
2. A JSON file downloads with all your sessions
3. Store it as a backup — it can be re-imported later

### Importing data
1. Go to **Settings → Data → ⬆ IMPORT JSON**
2. Select a previously exported JSON file
3. Sessions are merged in — duplicates (same ID) are skipped

---

## Security Model

### Anon key exposure
The Supabase anon key is embedded in `index.html` and therefore visible to anyone who views the page source. **This is intentional and safe** because:
- The anon key only grants access to operations that Row Level Security permits
- RLS policy: `auth.uid() = user_id` — every database operation is filtered to the authenticated user's own data
- An attacker with the anon key can only read/write data for accounts they are authenticated as
- There is no way to read another user's sessions using the anon key

### Authentication
- Authentication is handled entirely by Supabase and Google — no passwords are stored anywhere in the app
- JWTs are stored in the browser's localStorage by the Supabase SDK (standard practice for web apps)
- JWTs expire and are automatically refreshed by the SDK

### Data isolation
- Every session row has a `user_id` column
- The RLS policy enforces this at the PostgreSQL level — even if a bug existed in the app code, a user could never read or write another user's data
- Supabase's own infrastructure handles encryption at rest and in transit (HTTPS)

---

## Offline Behaviour

After the first load, IronLog works offline with the following behaviour:

| Action | Offline behaviour |
|---|---|
| Open the app | Works — served from service worker cache |
| Start a session | Works — state is in memory |
| Log exercises | Works — DOM only, no network needed |
| Finish a session | Session is added to memory, Supabase upsert fails silently |
| View history | Shows cached sessions from last sync |
| View charts/stats | Calculated from cached sessions |
| Sign in | Fails — OAuth requires network |
| Manual sync | Fails — shows error toast |

**Important:** If you finish a session while offline, the data is held in the JavaScript `sessions` array in memory. If you close the browser tab before regaining connectivity, that session is lost because there is no IndexedDB queue. To avoid data loss while offline, keep the tab open until you have signal, then tap **Sync Now** in Settings.

---

## Known Limitations

- **No offline session persistence** — sessions finished offline are lost if the tab is closed before syncing. Future fix: queue failed upserts in IndexedDB.
- **No workout templates** — you can't save a workout structure to repeat. You can reference history as a guide.
- **No rest timer** — there is no built-in rest timer between sets.
- **Weight in kg only** — there is no lb toggle. All weights are assumed to be kg.
- **No sets copy/paste** — you can't duplicate last week's session automatically.
- **Exercise library not synced** — custom exercises added via Settings are stored in localStorage per device, not in Supabase. Adding a custom exercise on your phone won't appear on your laptop.
- **Single user per install** — the app is designed for one person. There is no sharing or coaching feature.
- **Chrome only for Android PWA install** — other Android browsers (Firefox, Samsung Internet) may not show the install prompt or may not support PWA install.

---

## Design System

IronLog uses a custom design system with CSS custom properties.

### Colour palette

| Token | Value | Usage |
|---|---|---|
| `--bg` | `#0a0a0a` | Page background |
| `--surface` | `#111111` | Card backgrounds |
| `--surface2` | `#1a1a1a` | Input backgrounds, nested surfaces |
| `--border` | `#2a2a2a` | All borders and dividers |
| `--accent` | `#e8ff47` | Primary accent — yellow-green |
| `--accent2` | `#ff4757` | Danger / error / delete |
| `--text` | `#f0f0f0` | Primary text |
| `--text-muted` | `#666666` | Secondary text, labels |
| `--text-dim` | `#444444` | Placeholder text, set numbers |

### Typography

| Font | Source | Usage |
|---|---|---|
| Bebas Neue | Google Fonts | Display headings, dates, stat values |
| DM Mono | Google Fonts | Labels, metadata, monospaced values, navigation |
| Space Grotesk | Google Fonts | Body text, inputs, paragraphs |

### Design principles
- **Dark first** — the app is designed for use in a gym environment, often in low light. A near-black background reduces eye strain and battery usage on OLED screens.
- **Industrial aesthetic** — uppercase monospaced labels, tight letter-spacing, sharp corners. Feels like a logbook, not a social app.
- **Yellow accent** — `#e8ff47` is high-contrast against black, easy to read in bright gym lighting, and feels energetic without being aggressive.
- **Minimum chrome** — no gradients, no shadows, no decorative elements. Every pixel serves a function.
- **Mobile-first layout** — max-width 700px centered, padded for thumb reach, large tap targets on interactive elements.
