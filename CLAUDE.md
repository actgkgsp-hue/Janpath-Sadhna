# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Sadhana Tracker (Janpath)** is a Progressive Web App (PWA) for tracking daily spiritual practices (sadhana) for Krishna Parayan devotees at two coordinator levels: Coordinators and Overall Coordinators.

Firebase project: `janpath-sadhna`

## Running the App

No build system. This is a static vanilla JS/HTML app. To develop:

- Open `index.html` in a browser, or serve with any static file server:
  ```
  npx serve .
  python -m http.server 8080
  ```
- No compilation, bundling, or test runner.

## Architecture

### Files

- **index.html** — Full single-page app UI. All CSS is embedded. Contains all modals, forms, and layout for every role.
- **app.js** — Core application logic (~3976 lines). All Firebase interactions, scoring, UI rendering, and event handlers live here.
- **signup.html** — Separate registration page. Creates Firebase Auth user + Firestore profile.
- **sw.js** — Service Worker: pre-caches static assets, network-first fetch strategy, push notification handling, background sync (`sadhana-reminder` tag). Never caches Firestore API calls.
- **manifest.json** — PWA manifest.

### app.js Section Map

The file is divided into numbered sections (not modules):

| # | Section | Purpose |
|---|---|---|
| 1 | Firebase Setup | Config and initialization |
| 2 | Role Helpers | `isSuperAdmin()`, `isCategoryAdmin()`, `visibleCategories()` |
| — | Back-Button / History | Modal close ordering via browser back, `_closeTopmostOverlay()` |
| 3 | Helpers | `t2m()`, `fmt12()`, `localDateStr()`, `getNRData()`, `getPreJoinData()`, `calculateScores()` |
| 4 | Excel Download | `downloadUserExcel()`, `downloadMasterReport()`, `downloadCategoryExcel()` |
| 5 | Auth | Login, logout, password reset/change |
| 6 | Init Dashboard | `initDashboard()` — entry point post-login |
| 7 | Navigation | Panel switching, tab management |
| 7b | Home Panel | Ring chart, streak, activity bars for regular users |
| 7c | SA Home Panel | Overview dashboard for super admin |
| 8 | Super Admin Tabs | Tab switching, level filters, UAC sheet |
| 9 | Reports Table | `loadReports()`, `fairDenominator()` |
| 10 | Progress Charts | Chart.js rendering with week/month/all views |
| 11 | Sadhana Form | Daily entry submission |
| 12 | Admin Panel | User management, role assignment, inactivity tracking |
| 13 | Edit Sadhana | Admin modal to edit past entries, recalculates scores |
| 14 | Reject / Revoke | Reject/revoke entry workflow |
| 14 | Date Select & Profile | Date picker and profile form |
| 15 | Password Modal | Change password UI |
| 16 | Misc Bindings | `friendlyAuthError()`, utility event bindings |
| 17 | Forgot Password | Password reset flow |
| 18 | PWA — Service Worker | SW registration |
| 19 | Notifications | FCM push notifications, VAPID key, token management |
| 20 | User Sidebar | Slide-out sidebar |
| — | Best & Weak Performers | Performance rankings, ring charts, category breakdowns, WCR |
| 21 | Profile Picture | Avatar upload (data URL stored in Firestore) |
| 22 | Leaderboard | Cross-user ranking with date range filters |
| 23 | Tasks | Admin task assignment and tracking system |
| 24 | Activity Analysis | Per-user activity deep-dive modal |

### Role System

Three roles stored in Firestore `users/{userId}.role`:
- `superAdmin` — sees all users across all levels and categories
- `admin` (Category Admin) — sees only users in their `adminCategory`
- `user` — sees only their own data

### Level System

Two levels stored in Firestore `users/{userId}.level`:
- `Coordinators`
- `Overall Coordinators`

Short labels used in UI: `Coord` and `OC`.

### Firestore Data Model

```
users/{userId}
  name, email, level, chantingCategory, exactRounds, role, adminCategory, notificationToken
  sadhana/{dateStr (YYYY-MM-DD)}
    sleepTime, wakeupTime, chantingTime (HH:MM strings)
    readingMinutes, hearingMinutes, serviceMinutes, notesMinutes, daySleepMinutes (numbers)
    totalScore, dayPercent, scores (computed by calculateScores())
```

### Scoring Algorithm (`calculateScores()`)

Max score: **160 points**. Uniform scoring for both levels — no notes field.

| Component | Max Points | Notes |
|---|---|---|
| Sleep time | 25 | Earlier is better (≤22:30 = 25, scales down in 5-min steps) |
| Wake-up time | 25 | Earlier is better (≤5:05 = 25, scales down) |
| Chanting | 25 | Earlier start time = higher score |
| Reading | 25 | Threshold: 30 min |
| Hearing | 25 | Threshold: 30 min |
| Service | 25 | Threshold: 30 min |
| Day sleep | 10 | ≤60 minutes = 10, else -5 |

Not Reported (NR) days get a penalty of -30 (`nrPenalty()`).

### External Libraries (CDN, no npm)

- **Firebase 8.10.1** — Auth, Firestore, Messaging (loaded via gstatic CDN)
- **Chart.js 4.4.0** — progress charts
- **XLSX 0.18.5** — Excel export

### PWA / Service Worker

Cache version is hardcoded in sw.js (`CACHE_NAME = 'sadhana-tracker-v1'`). The app is deployed at path `/Coordinators-Sadhana-Tracker/` — asset paths in sw.js use this prefix. When adding new static assets, update `CACHE_NAME` and the pre-cache list in the `install` handler.

## Key Conventions

- UI events use `window.*` functions (e.g., `window.downloadUserExcel`) assigned in app.js and called from `onclick` attributes in index.html.
- Date keys in Firestore use `YYYY-MM-DD` format via `localDateStr()`.
- "Not Reported" days use `getNRData()` which creates a placeholder with penalty scores. Pre-join days use `getPreJoinData()` with zero scores and dashes.
- `fairDenominator()` excludes Not Reported days when calculating score percentages.
- Firebase config is embedded directly in `app.js` and `signup.html` (no `.env` file).
- `t2m(timeStr, isSleep)` converts `HH:MM` strings to minutes for scoring; sleep times past midnight (00:00–03:00) are treated as 24:00–27:00.
- `activeListener` holds the current Firestore real-time subscription and is torn down/replaced on navigation to avoid stale listeners.
- XSS prevention: all user-supplied strings inserted via `innerHTML` must go through `esc()`.
- Modals use a `.hidden` CSS class to toggle visibility. `_closeTopmostOverlay()` defines the close priority order — new modals must be added there.
- `APP_START` is the earliest date the reports table will show.
