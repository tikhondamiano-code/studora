# Studora MVP Technical Specification

Nov 25, 2026 · @TIkhon

## 1. Overview and scope

Studora v1 is an installable Progressive Web App (PWA) that runs entirely on the student's device. It stores all data in IndexedDB, needs no account or server, and works offline after the first load.

The MVP exists to test one hypothesis: **if a student gets a realistic, spread-out plan that recovers automatically when he misses a day, he will keep studying until his exams instead of abandoning the plan.**

### In scope (the five MVP user stories)

| # | Story | Screens |
| --- | --- | --- |
| 1 | Enter finals: class, exam date, topics, confidence 1–5 | Add your finals |
| 2 | Mark weekly availability and a daily session cap | When you're free |
| 3 | Get a day-by-day spaced plan with catch-up days | Your if-then plan, Your plan |
| 4 | Know exactly what to do today; run a timed session; self-rate | Today, Study session, Session done |
| 5 | Missed sessions move automatically, without guilt | Missed day recovery |

Supporting features: calendar export (.ics) for reminders, JSON backup and restore, and an on-device pilot stats screen.

### Out of scope for v1

- Accounts, cloud sync, any backend or API
- School system (LMS) sync and study-guide photo scanning
- AI features and auto-generated quiz questions
- Push notifications (needs a server; the .ics export replaces it)
- Focus mode, sharing with parents or friends, dashboards, weekly review, exam mode, post-exam reflection

### Target user

A 15-year-old high school student who wants 90%+ on every final, uses a phone as his main device, and sometimes a school Chromebook or home PC.

## 2. Architecture principles

The device is the single source of truth. The web host only delivers static files; after install, the app makes no network requests.

1. **Edge-canonical.** IndexedDB on the device holds the only authoritative copy of the student's data. There is no server copy to reconcile with.
2. **Client-side only.** All logic, including the plan engine and recovery, runs in the browser. Hosting is static files over HTTPS.
3. **Offline-first.** Every feature works with no connection. The service worker precaches the whole app shell and all assets, including fonts.
4. **Deterministic core.** The plan engine is a pure function: the same inputs always give the same plan. This makes it testable and makes recovery predictable.
5. **Local events, never sent.** Pilot measurements are written to an on-device event log. They leave the device only when the student exports them.
6. **Few, small dependencies.** Use the platform first (crypto.randomUUID, Intl, Blob downloads, Web Share). Add an open-source library only where it removes real complexity.
7. **Mobile-first, responsive to PC.** Design for a 360–430 px phone first; scale up to tablet and desktop layouts with CSS only.
8. **Data portability.** Because there is no cloud copy, backup and restore are core features, not extras.

## 3. Technology stack

The stack is TypeScript, Preact, Dexie and Vite, with eight small runtime packages in total. All are MIT- or Apache-licensed open source.

### Runtime

| Concern | Choice | Why |
| --- | --- | --- |
| Language | TypeScript (strict) | Type-safe data model shared by engine, storage and UI |
| UI | Preact + preact/hooks | React-style components at about 4 KB; large talent pool |
| State | @preact/signals | Fine-grained reactive state without a store framework |
| Routing | wouter-preact | Tiny hash/path router; hash mode avoids server rewrite rules |
| IndexedDB | Dexie.js | Schema versions, migrations, compound indexes, liveQuery for reactive reads |
| Dates | date-fns | Tree-shakable local-date maths (add days, week starts, formatting) |
| Validation | zod | Validates backup files on import and guards schema boundaries |
| Fonts | @fontsource/inter, @fontsource/space-grotesk | Self-hosted, so offline and no third-party requests |

No libraries are used for: IDs (crypto.randomUUID), the .ics file (small in-house generator, about 80 lines), CSV export, timers, or styling.

### Build and PWA

| Concern | Choice |
| --- | --- |
| Bundler / dev server | Vite |
| Service worker and manifest | vite-plugin-pwa (Workbox generateSW) |
| Styling | Plain CSS with custom properties (design tokens); CSS modules per component |

### Quality tooling

| Concern | Choice |
| --- | --- |
| Unit tests | Vitest |
| Property-based tests (plan engine) | fast-check |
| IndexedDB in tests | fake-indexeddb |
| Component tests | @testing-library/preact |
| End-to-end, offline, mobile viewports | Playwright |
| Accessibility checks | @axe-core/playwright |
| PWA and performance audits | Lighthouse CI |
| Lint / format | ESLint + Prettier |

## 4. Application architecture

The app has four layers, and dependencies only point downward: UI calls services, services call the engine and the repositories, and the engine depends on nothing.

&#91;embedded content: Studora architecture · 4 layers on the device, static host outside\]

The host delivers files once and for updates; after that, data moves only from the UI down to IndexedDB and out through files the student chooses to save.

| Layer | Responsibility | Rules |
| --- | --- | --- |
| UI (`src/ui`) | Screens, components, routing, responsive layout | No direct IndexedDB access; reads via liveQuery hooks, writes via services |
| Services (`src/services`) | Use cases: commit plan, start/complete session, run recovery, export .ics, backup/restore, log events | Wrap each use case in one Dexie transaction; log a pilot event for each |
| Engine (`src/engine`) | Plan generation, packing, recovery repair, risk status | Pure TypeScript functions; no I/O, no Date.now(); `today` is passed in |
| Data (`src/data`) | Dexie database, schema versions, migrations, repositories | The only code that imports Dexie |

Platform adapters (`src/platform`) isolate browser APIs: clock, file download, Web Share, storage persistence and the service-worker update prompt. Tests swap them for fakes.

### Folder structure

```
studora/
  public/                 icons, maskable icon, robots.txt
  src/
    main.tsx              boot: open DB, run day-rollover check, mount app
    app.tsx               router + layout shell
    ui/
      screens/            Welcome, AddFinals, Availability, IfThen,
                          PlanPreview, Today, Session, SessionDone,
                          Recovery, PlanWeek, Data
      components/         Button, Chip, Card, ConfidencePicker,
                          SlotGrid, TimerRing, TabBar, SideRail
      styles/             tokens.css, base.css, layout.css
    services/             planService, sessionService, recoveryService,
                          exportService, backupService, eventLog
    engine/               buildPlan.ts, pack.ts, repair.ts, risk.ts,
                          types.ts, config.ts
    data/                 db.ts (Dexie schema), migrations.ts, repos/
    platform/             clock.ts, files.ts, share.ts, storage.ts, sw.ts
  tests/
    engine/               unit + property-based tests
    e2e/                  Playwright specs
  vite.config.ts          includes vite-plugin-pwa config
```

### Boot sequence

1. Open the Dexie database and run any pending migrations.
2. Request persistent storage (`navigator.storage.persist()`) once onboarding is complete.
3. Run the day-rollover check: any session dated before today and still `scheduled` becomes `missed`, then recovery runs (Section 7).
4. Route to Welcome if onboarding is incomplete, the Recovery screen if recovery moved anything, otherwise Today.
5. Re-run step 3 on `visibilitychange` (app brought back to the front) and at local midnight while open.

## 5. Data model and IndexedDB schema

One Dexie database, `studora`, holds eight object stores. Dates are local calendar dates as `YYYY-MM-DD` strings; times of day are minutes after midnight. This avoids time-zone drift when a date has no time.

### Stores

| Store | Dexie index string | Purpose |
| --- | --- | --- |
| `meta` | `key` | Settings and app state (key/value) |
| `courses` | `id, examDate` | One row per final |
| `topics` | `id, courseId, [courseId+order]` | Topics within a course |
| `slots` | `id, weekday` | Weekly availability template |
| `ifThenPlans` | `id, active` | WOOP goal and if-then plans |
| `workItems` | `id, topicId, courseId, sessionId, status` | Individual study tasks (learn, review) |
| `sessions` | `id, date, status, [date+status]` | Time-boxed study sessions holding work items |
| `events` | `++seq, type, at` | Append-only pilot event log |

### Record shapes (TypeScript)

```ts
type ISODate = string;              // 'YYYY-MM-DD', local
type Minutes = number;              // minutes after midnight

interface Course {
  id: string; name: string; color: CourseColor;
  examDate: ISODate; examStart?: Minutes;
  confidence: 1 | 2 | 3 | 4 | 5;    // at setup
  targetPercent: number;            // default 90
  createdAt: number; updatedAt: number;
}

interface Topic {
  id: string; courseId: string; name: string; order: number;
  confidence?: 1 | 2 | 3 | 4 | 5;   // latest self-rating, starts as course's
}

interface Slot {                    // weekly template
  id: string; weekday: 0|1|2|3|4|5|6; // 0 = Sunday
  start: Minutes; end: Minutes; label?: string;
}

interface IfThenPlan {
  id: string; wish: string; outcome?: string; obstacle: string;
  ifText: string; thenText: string; active: boolean; createdAt: number;
}

type ItemKind = 'learn' | 'review1' | 'review2' | 'review3' | 'final';

interface WorkItem {
  id: string; courseId: string; topicId: string | null; // null for 'final'
  kind: ItemKind; minutes: number;
  earliest: ISODate;                // spacing constraint
  deadline: ISODate;                // last allowed day
  sessionId: string | null;         // null = unplaced
  status: 'planned' | 'done' | 'dropped';
  selfScore?: number;               // 0-100, set on completion
}

interface Session {
  id: string; date: ISODate; slotId: string; start: Minutes;
  minutes: number;                  // default 25
  courseId: string;                 // sessions are single-course
  status: 'scheduled' | 'in_progress' | 'done' | 'missed' | 'skipped';
  originalDate: ISODate; movedCount: number;
  startedAt?: number; completedAt?: number; actualMinutes?: number;
  selfScore?: number; confidenceAfter?: 1|2|3|4|5;
}

interface StudoraEvent {
  seq?: number; id: string; type: EventType; at: number;
  data?: Record<string, unknown>;
}
```

### Meta keys

| Key | Type | Default |
| --- | --- | --- |
| `schemaVersion` | number | 1 |
| `onboardingDone` | boolean | false |
| `sessionMinutes` | number | 25 |
| `maxSessionsPerDay` | 1 \| 2 \| 3 | 2 |
| `planCommittedAt` | number \| null | null |
| `lastRolloverDate` | ISODate | today at install |
| `lastBackupAt` | number \| null | null |
| `participantCode` | string \| null | null (e.g. "P07", set for pilot) |
| `persistGranted` | boolean | false |

### Migrations

- Each schema change bumps the Dexie version and ships an `upgrade()` function in `migrations.ts`.
- Migrations must be forward-only and tested against a saved v1 fixture database.
- Backups record `schemaVersion`; restore runs the same migrations on older backups before writing.

### Storage safety

- Call `navigator.storage.persist()` after onboarding and record the result.
- Show a backup reminder on the Data screen when `lastBackupAt` is more than 7 days old.
- Expected size is under 1 MB for a full season, far below browser quotas.

## 6. Plan engine

`buildPlan()` turns courses, topics and availability into dated, single-course study sessions. It works backward from each exam, spaces reviews at +1, +3 and +7 days after first learning, and holds back catch-up days for recovery.

### Interface

```ts
function buildPlan(input: PlanInput): PlanResult;

interface PlanInput {
  today: ISODate; nowMinutes: Minutes;      // injected clock
  courses: Course[]; topics: Topic[]; slots: Slot[];
  settings: { sessionMinutes: number; maxSessionsPerDay: number };
  locked: { sessions: Session[]; items: WorkItem[] }; // done work, never moved
  config?: Partial<EngineConfig>;
}

interface PlanResult {
  sessions: Session[]; items: WorkItem[];
  dropped: WorkItem[];                       // trimmed to fit
  risk: Record<string /*courseId*/, 'on_track' | 'at_risk'>;
  bufferDates: ISODate[];
  stats: { avgMinutesPerDay: number; totalSessions: number };
}
```

### Default configuration (`engine/config.ts`)

| Setting | Default | Notes |
| --- | --- | --- |
| Learn item | 20 min | First pass on a topic. Pilot starts at 20 and tunes down from real session data |
| Review 1 / 2 / 3 | 15 / 10 / 5 min | Quiz-yourself reviews |
| Final mixed review | 20 min per course | Placed 1–3 days before the exam |
| Review offsets | +1, +3, +7 days after learn | Compressed when the exam is closer |
| Skip learn when confidence is | 4 or 5 | Topic starts at review 1 |
| Last day for new learning | Exam − 2 days |  |
| Catch-up days | 1 per 6 days of window | Never the day before an exam |
| Session length | 25 min | From `meta.sessionMinutes` |
| Sessions per day | 2 | From `meta.maxSessionsPerDay` |

### Algorithm

1. **Window.** Start today, or tomorrow if `nowMinutes` is past today's last slot. Each course's window ends the day before its exam.
2. **Capacity.** Expand the weekly slot template into dated slots. Each slot holds `floor(slotLength / sessionMinutes)` sessions, and each day holds at most `maxSessionsPerDay`. A course gets no sessions on or after its exam date.
3. **Catch-up days.** Split the window into 6-day blocks. In each block, the last day with capacity becomes a catch-up day, held empty for recovery.
4. **Generate work items.** Per topic: `learn` (unless confidence ≥ 4), then `review1`, `review2`, `review3`. Per course: one `final` item. Set each item's `deadline` (learn: exam − 2; reviews and final: exam − 1).
5. **Order topics.** Sort by exam date (soonest first), then confidence (lowest first), then the topic's own order.
6. **Place learn items.** In that order, put each learn item in the earliest session with room for its course on or after today. If none has room, open a new session in the earliest free slot.
7. **Set review dates.** For a learn placed on day *d*, review earliest dates are *d*+1, *d*+3 and *d*+7. If *d*+7 is past the deadline, scale all three offsets by (deadline − *d*) / 7, with a minimum of 1 day each.
8. **Place reviews and finals.** Sort the remaining items by earliest date, then deadline. Place each in the first fitting session between its earliest date and its deadline.
9. **Interleave (soft rule).** When choosing between equal days, prefer a day where that course has no session yet. This spreads subjects across days.
10. **Trim if overloaded.** If items stay unplaced, drop them in this order until the rest fit: `review3` of confidence-4+ topics, `review3` of the others, then `review2` of confidence-4+ topics. Learn, review 1 and final items are never dropped.
11. **Risk status.** A course is `at_risk` if any of its learn, review 1, review 2 or final items is unplaced or dropped. Otherwise it is `on_track`.
12. **Name tasks.** A session's title comes from its first item: "Learn: *topic*", "Quiz yourself: *topic*" or "Mixed review: *course*". The other items are listed underneath.

### Invariants (enforced by property-based tests)

- No session is scheduled on or after its course's exam date.
- No day has more sessions than `maxSessionsPerDay`; no session holds more minutes than `sessionMinutes`.
- Every review is dated after its own learn item; review *n* is after review *n*−1.
- Locked (done) sessions and items are returned unchanged.
- Catch-up days hold no sessions after a fresh build.
- Same input always gives the same output (no randomness, no clock reads).
- Runtime under 100 ms for 8 courses, 80 topics and 30 days on a mid-range phone.

### "Make it lighter" (Plan preview screen)

It rebuilds with the review 3 items of confidence-4+ topics removed up front, showing the new minutes per day. The student sees the change before committing.

## 7. Missed-session recovery

Recovery moves missed work with as little change to the rest of the plan as possible: catch-up days and free capacity first, a full rebuild only when they run out.

### Trigger

The day-rollover check runs on boot, on `visibilitychange` and at local midnight. Any session with `date < today` and status `scheduled` or `in_progress` becomes `missed`, and a `session_missed` event is logged. The student can also tap "Can't today" on a session, which marks it missed immediately.

**Skip without a miss.** Every session on Today has a Skip option with two reasons: "Already know this" or "Did it another way". A skipped session is marked `skipped`, its items count as done, and recovery does not run. "Already know this" also sets the topic's confidence to 5. Skips are logged as `session_skipped` and are left out of the Sessions done stat, so they never look like misses. The skip count is reported separately so the pilot can spot overuse.

### `repairPlan()` steps

1. Collect the missed sessions' work items. Return them to unplaced (`sessionId = null`).
2. If a missed item is a `learn`, shift that topic's unplaced reviews so each keeps its spacing after the learn's new date.
3. Place each item, deadline first, into: (a) the earliest catch-up day before its deadline, then (b) free room in an existing session of the same course, then (c) a new session in any free slot.
4. If anything still cannot be placed, run `buildPlan()` with all done work locked. This may trim reviews (Section 6, step 10).
5. Recompute risk status for each course.
6. Return a diff: each moved item's old and new date, plus any course whose status changed.

### What the student sees

| Situation | Recovery screen shows |
| --- | --- |
| All moved, still on track | "Yesterday didn't happen. That's normal." + the moves + "Still on track" |
| Moved, but a course is now at risk | The moves + an honest heads-up naming the course + 2 options: add a study slot, or raise the daily cap |
| Missed more than 3 days in a row | A gentler welcome back + a fresh plan from today |

The screen also asks an optional "What got in the way?" (too tired, practice ran late, phone, forgot, didn't know how to start). Each answer maps to a suggested if-then plan the student can accept or edit. No streak counters, red alerts or guilt language anywhere.

### Events logged

`session_missed`, `plan_recovered` (with number of moves and whether a rebuild was needed), `obstacle_reported`, `ifthen_updated`.

## 8. Screens, routing and responsive layout

The app has 11 routes matching the approved mockups plus a full Plan view. Hash routing (`#/today`) is used so the static host needs no rewrite rules.

### Routes

| Route | Screen | Notes |
| --- | --- | --- |
| `#/welcome` | Welcome | Shown until onboarding is complete |
| `#/setup/finals` | Add your finals | Step 1 of 4; at least 1 course with 1 topic to continue |
| `#/setup/availability` | When you're free | Step 2; at least 3 slots per week to continue |
| `#/setup/if-then` | Your if-then plan | Step 3; if and then fields required |
| `#/setup/plan` | Your plan | Step 4; Commit, Make it lighter |
| `#/today` | Today | Default home after onboarding |
| `#/session/:id` | Study session | Timer, 3 retrieval steps |
| `#/session/:id/done` | Session done | Self-score slider, confidence 1–5 |
| `#/recovery` | Missed day recovery | Shown when recovery moved anything |
| `#/plan` | Full plan | Week-by-week list; edit courses, topics, slots |
| `#/data` | Calendar, backup, pilot stats |  |

Setup screens save each step to IndexedDB as the student goes, so closing the app mid-setup loses nothing.

### Session timer

- Store `startedAt` in the session record, and compute time left from the clock on every tick. The timer stays correct if the app is backgrounded or reloaded.
- Use the Screen Wake Lock API while the timer runs, where supported.
- At zero: a short vibration (`navigator.vibrate`, where supported) and an on-screen prompt. No sound by default.

### Responsive breakpoints (mobile-first CSS)

| Width | Layout | Navigation |
| --- | --- | --- |
| < 600 px (phone, default styles) | Single column, full width, 20 px side padding | Bottom tab bar: Today, Plan, Data |
| 600–1023 px (tablet, small laptop) | Single column, max 560 px, centred | Bottom tab bar |
| ≥ 1024 px (desktop) | Side rail + main area; Today shows today's sessions and the week plan side by side | Left side rail replaces the tab bar |

Layout rules:

- Use CSS grid and flex with `min()` / `clamp()`; no fixed pixel page widths.
- Touch targets at least 44 × 44 px at every size; also support keyboard and mouse on desktop.
- Respect safe-area insets (`env(safe-area-inset-*)`) for phones with notches when installed.
- Respect `prefers-reduced-motion` (no timer ring animation) and `prefers-color-scheme` (light and dark themes ship in v1; see Dark theme below).

### Design tokens (`styles/tokens.css`)

| Token | Value |
| --- | --- |
| `--bg` | #F6F5F1 |
| `--surface` | #FFFFFF |
| `--ink` / `--ink-muted` | #16181D / #4F5462 |
| `--accent` | #2F4BD8 |
| `--success` | #0F7B6C |
| Course colors | Orange #FCEBD7 / #6B3A06, blue #E8ECFC / #1E2A6B, teal #E2F2EE / #0B4F45, grey #F0EEE8 / #3E414B (fill / text) |
| Fonts | Space Grotesk (headings), Inter (body) |
| Radius | 8 / 12 / 16 px |

### Dark theme

Dark mode ships in v1 because most studying happens in the evening. It follows the phone's setting by default; a Light / Dark / System choice on the Data screen overrides it and is saved as `meta.theme`.

| Token | Light | Dark |
| --- | --- | --- |
| `--bg` | #F6F5F1 | #121318 |
| `--surface` | #FFFFFF | #1C1E25 |
| `--ink` / `--ink-muted` | #16181D / #4F5462 | #ECEDF1 / #A8ADBA |
| `--accent` | #2F4BD8 | #8EA2FF (button text #121318) |
| `--success` | #0F7B6C | #4FC3AE |
| Course orange (fill / text) | #FCEBD7 / #6B3A06 | #3A2A1A / #F5C28F |
| Course blue | #E8ECFC / #1E2A6B | #1F2645 / #B9C3F5 |
| Course teal | #E2F2EE / #0B4F45 | #16332E / #8ED9C8 |
| Course grey | #F0EEE8 / #3E414B | #2A2C33 / #D4D6DC |

- Both themes use the same token names, switched with a `data-theme` attribute on `<html>`. Components never hard-code colours.
- Two `<meta name="theme-color">` tags (one per `prefers-color-scheme`) colour the installed app's title bar to match.
- Every text and chip pair must pass 4.5:1 contrast in both themes; axe runs on both in CI.

## 9. PWA packaging

The final package is a static PWA: a precached app shell, a web app manifest and a Workbox service worker, generated by vite-plugin-pwa. After the first visit, the app loads and runs with no network.

### Web app manifest

| Field | Value |
| --- | --- |
| `name` / `short_name` | Studora |
| `start_url` | `/#/today` |
| `scope` | `/` |
| `display` | `standalone` |
| `orientation` | `any` (phone portrait, desktop landscape both work) |
| `background_color` / `theme_color` | #F6F5F1 / #2F4BD8 |
| `icons` | 192 px and 512 px PNG, plus a 512 px maskable icon; 180 px apple-touch-icon in the HTML head |
| `description` | Your finals study plan, built for you and fixed when you miss a day. |

### Service worker

- Strategy: `generateSW`, precaching every built asset (JS, CSS, HTML, fonts, icons). No runtime caching rules are needed because the app calls no APIs.
- Navigation fallback: `index.html`, so any route opens offline.
- Updates: `registerType: 'prompt'`. When a new version is ready, show a small "Update available, reload" banner. Never reload during an active session.
- The service worker never touches IndexedDB, so app updates cannot change student data. Schema changes run only through Dexie migrations on the next boot.

### Install experience

- Chrome, Edge and Android: capture `beforeinstallprompt` and show an "Install Studora" button on the Today screen after the plan is committed.
- iPhone and iPad (Safari): show a one-time card with Add to Home Screen steps, since Safari has no install prompt.
- After install, call `navigator.storage.persist()` again if it was previously denied.

### PWA acceptance checks

- [ ] Lighthouse PWA installability checks pass
- [ ] App installs on Android Chrome, iOS Safari, and desktop Chrome/Edge
- [ ] With the network off, every route loads and a full session can be completed
- [ ] A new deploy shows the update banner and does not lose any data after reload

## 10. Calendar export and backup

The .ics export gives the student reminders without a server, and the JSON backup is the only way to keep data across devices or a wiped browser.

### Calendar export (.ics, RFC 5545)

- One `VEVENT` per scheduled session from today onward. Done and missed sessions are left out.
- `UID` = `<session UUID>`, stable across exports so calendars that support updates can replace events instead of duplicating them. No domain is needed, because the standard only requires a globally unique ID.
- `SUMMARY` = session title ("Biology: Quiz yourself on cell transport"); `DESCRIPTION` = the session's items plus the if-then plan text.
- `DTSTART` / `DTEND` in UTC, converted from the local date and minutes. `SEQUENCE` increases each time a session moves.
- One `VALARM` per event, 10 minutes before (`TRIGGER:-PT10M`).
- Delivery: on phones, `navigator.share({ files: [icsFile] })` when supported, which hands the file to the calendar app. Otherwise, a Blob download named `studora-plan-YYYY-MM-DD.ics`.
- After recovery or edits, the Data screen shows "Your plan changed. Add it to your calendar again."

### Backup and restore

```json
{
  "app": "studora",
  "schemaVersion": 1,
  "exportedAt": "2026-12-01T22:15:00Z",
  "data": {
    "meta": [], "courses": [], "topics": [], "slots": [],
    "ifThenPlans": [], "workItems": [], "sessions": [], "events": []
  }
}
```

- **Save backup:** Read all stores in one transaction, write the JSON above, and share or download it as `studora-backup-YYYY-MM-DD.json`. Update `meta.lastBackupAt`.
- **Restore:** Pick a file, parse it, and validate it with a zod schema. Run migrations if `schemaVersion` is older. Show a summary ("5 classes, 64 sessions, last saved Dec 1") and require confirmation. Then clear and refill all stores in one transaction.
- Reject files that are not Studora backups, are newer than the app's schema, or fail validation, with a plain-language message. A failed restore must leave existing data untouched.
- Backups are not encrypted. The Data screen says the file contains the student's study plan and should be kept private.

## 11. Pilot event log and stats

Every pilot measure is computed on the device from the `events` store. Nothing is sent anywhere; the student exports stats only when the research team asks.

### Event types

| Event | Logged when | Data |
| --- | --- | --- |
| `app_opened` | Boot or return to foreground (max 1 per 30 min) | — |
| `onboarding_started` / `onboarding_completed` | Welcome button / plan committed | Duration in seconds on completion |
| `plan_committed` | Commit button | Courses, topics, sessions, avg minutes per day |
| `plan_lightened` | Make it lighter | Minutes per day before and after |
| `session_started` / `session_completed` | Start / I'm done | Session id, actual minutes, self-score, confidence |
| `session_missed` | Rollover or "Can't today" | Session id, reason (rollover or manual) |
| `plan_recovered` | After repair | Moves, rebuild yes/no, risk changes |
| `obstacle_reported` / `ifthen_updated` | Recovery screen | Obstacle key |
| `ics_exported` / `backup_saved` / `backup_restored` | Data screen | — |

Events hold IDs and numbers only, never course names, topic names or free text.

### Stats on the Data screen

| Stat | Formula | Pilot target |
| --- | --- | --- |
| Setup time | `onboarding_completed.duration` | ≤ 10 min |
| Sessions done | completed ÷ (completed + missed), sessions dated up to yesterday | ≥ 60% |
| Back within 48 hrs | misses followed by a `session_completed` within 48 h ÷ all misses | ≥ 70% |
| Started before 1st final | days between `plan_committed` and the earliest exam | ≥ 10 days |
| Active days | distinct dates with ≥ 1 completed session | Tracked, no target |

### Export

The student enters a participant code (for example "P07") once. "Export stats" shares a CSV of the event log plus a one-row summary, tagged with that code. The file contains no names.

## 12. Non-functional requirements

The app must load fast on a budget Android phone, meet WCAG 2.2 AA, and send no data off the device.

### Performance

| Measure | Budget |
| --- | --- |
| Initial JS (gzipped) | ≤ 120 KB |
| Largest Contentful Paint, first visit, mid-range Android on 4G | ≤ 2.5 s |
| Repeat launch from home screen (offline) | ≤ 1 s to the Today screen |
| `buildPlan()` / `repairPlan()` | ≤ 100 ms for 8 courses, 80 topics, 30 days |
| Lighthouse performance score (mobile) | ≥ 90 |

### Accessibility (WCAG 2.2 AA)

- Text contrast at least 4.5:1; touch targets at least 44 × 44 px.
- Real `<button>`, `<a>`, `<input>` and `<label>` elements; full keyboard use on desktop with visible focus.
- Timer and recovery messages announced through an `aria-live` region.
- Colour is never the only signal: course chips also carry the course name; status carries an icon and words.
- Works with text zoom up to 200% without horizontal scrolling.

### Privacy and data protection

- No accounts, analytics, ads, trackers or third-party scripts; no network calls after install.
- Data never leaves the device except through files the student chooses to save or share.
- Collect no full names, emails or birthdates. Pilot stats use a participant code only.
- For the pilot, follow the consent process in Pilot consent (US) below.

### Pilot consent (US)

A 15-year-old cannot consent to a study on his own, so each pilot student needs **two signatures on one form: a parent's permission and the student's own agreement (assent)**. This is not legal advice; confirm with your state's rules, and with the school if you recruit through one.

| Law or rule | Does it apply to the Studora pilot? | What to do |
| --- | --- | --- |
| COPPA (FTC) | No. It covers online services directed to children under 13. Pilot students are 14–17, and the app collects nothing online. | Keep the app free of online data collection; recruit high schoolers only |
| FERPA | Only if a school runs or shares records for the pilot. It applies to schools that receive U.S. Department of Education funds and their education records. | Recruit through families, not school records. If a school partners, get its written approval and follow its process |
| PPRA | Only for surveys run by ED-funded schools or programs that ask about 8 protected areas (e.g. mental health, family income). | Keep pilot questions to study habits; do not ask about mental health, family income, religion or similar topics |
| Human-subjects research rules (Common Rule, 45 CFR 46 Subpart D) | Legally required if the pilot is federally funded or run through a university or institution with an ethics board (IRB). Otherwise it is the standard to follow. | Parent permission + student assent; one parent's signature is normally enough for minimal-risk studies |
| State laws | Vary by state | Check your state's student-privacy and minors' research rules before recruiting |

**Simple process that meets these rules:**

1. Give each family a one-page, plain-language info sheet: what Studora is, what data is collected (the exported stats file only), who sees it, and that they can stop at any time.
2. Collect one form with two signatures: the parent or guardian (permission) and the student (assent). E-signatures are fine.
3. Assign each student a participant code (P01, P02…). Store signed forms separately from stats files.
4. If a family withdraws, delete that student's stats files and confirm by email.

Sources: [FTC: COPPA FAQs](https://www.ftc.gov/business-guidance/resources/complying-coppa-frequently-asked-questions) · [HHS: Research with children FAQs](https://www.hhs.gov/ohrp/regulations-and-policy/guidance/faq/children-research/index.html) · [U.S. Dept. of Education: Online educational services (FERPA and PPRA)](https://studentprivacy.ed.gov/resources/protecting-student-privacy-while-using-online-educational-services-requirements-and-best) · [U.S. Dept. of Education: PPRA guidance](https://studentprivacy.ed.gov/sites/default/files/resource_document/file/20-0379.PPRA_508.pdf)

### Security

- Strict Content Security Policy: `default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; connect-src 'self'; object-src 'none'; base-uri 'none'; frame-ancestors 'none'`.
- HTTPS only, with HSTS; no `eval` or inline scripts.
- Render all user text as text (Preact escapes by default); never inject HTML.
- Validate every imported file with zod before writing.

### Browser support

| Platform | Minimum |
| --- | --- |
| iOS / iPadOS Safari | 16.4 |
| Android Chrome | Last 2 major versions |
| Desktop Chrome, Edge | Last 2 major versions |
| Desktop Firefox, Safari | Last 2 major versions (install support varies; app works in the browser) |
| ChromeOS (school Chromebooks) | Last 2 major versions, if school policy allows installing PWAs |

## 13. Testing strategy

The plan engine carries the most risk, so it gets the deepest tests; everything else is covered by component tests and a small set of end-to-end flows on real mobile viewports.

| Level | Tool | What it covers | Gate |
| --- | --- | --- | --- |
| Engine unit | Vitest | Each algorithm step; offset compression; trimming order; risk status | 95% line coverage on `src/engine` |
| Engine properties | fast-check | All invariants in Section 6 over random courses, topics and slots | 1,000 runs per invariant in CI |
| Data | Vitest + fake-indexeddb | Repositories, transactions, migrations from a v1 fixture, backup round-trip | All pass |
| Components | @testing-library/preact | Setup validation, confidence picker, slot grid, timer maths | All pass |
| End-to-end | Playwright (Pixel 7, iPhone 14, 1440 px desktop) | Full setup → commit → session → done; missed day → recovery (fake clock); offline run; backup and restore | All pass on every PR |
| Accessibility | @axe-core/playwright | Every route | No serious or critical issues |
| PWA / performance | Lighthouse CI | Installability, budgets in Section 12 | Budgets met |

Manual checks before the pilot: install and complete a session on at least one real iPhone, one Android phone and one Chromebook; import the .ics into Apple Calendar and Google Calendar; restore a backup on a second device.

Time-based behaviour (rollover, timers, recovery) is always tested through the injected clock adapter, never the real system clock.

## 14. Build, hosting and release

`vite build` outputs a folder of static files that any HTTPS static host can serve; no server code is deployed.

- **Hosting:** Any static host with HTTPS and custom headers, such as Cloudflare Pages, Netlify or GitHub Pages.
- **Headers:** `index.html` and `sw.js` with `Cache-Control: no-cache`; hashed assets with `Cache-Control: public, max-age=31536000, immutable`; the CSP from Section 12.
- **CI (GitHub Actions) on every pull request:** lint, type-check, unit and property tests, build, Playwright, axe and Lighthouse CI. A preview deploy per pull request.
- **Release:** Merge to `main` deploys to production. Versions follow semantic versioning and are shown on the Data screen.
- **Rollback:** Redeploy the previous build. Because data lives only on devices and migrations are forward-only, a rollback must never ship an older schema version than a released one.

### Domain

There is no domain yet, and none is needed to build. Development and the first test users can use the host's free HTTPS subdomain (for example `studora.pages.dev` on Cloudflare Pages).

**Buy and switch to the final domain before real pilot users install.** Browsers tie IndexedDB to the exact web address (origin). If the address changes later, installed students start with an empty app and must move their data with Save backup and Restore. Nothing else in the spec depends on the domain: the CSP uses `'self'`, and calendar UIDs are plain UUIDs.

## 15. Iterative delivery plan

Six one-week sprints take Studora from an empty repo to a pilot-ready PWA. Each sprint ends with something installable and demoable to a test student.

### Sprint 0 — Walking skeleton

- [ ] Vite + Preact + TypeScript repo with ESLint, Prettier, Vitest and CI
- [ ] Dexie database with the v1 schema and a migration test fixture
- [ ] Hash router with placeholder screens and the responsive shell (tab bar / side rail)
- [ ] vite-plugin-pwa manifest and service worker; installs and opens offline
- [ ] Design tokens for light and dark themes, theme switch, and self-hosted fonts

### Sprint 1 — Setup flow (stories 1–2, if-then)

- [ ] Add your finals: add, edit and delete courses and topics; confidence picker
- [ ] When you're free: slot grid and daily cap, saved per step
- [ ] Your if-then plan: obstacle chips, if and then fields
- [ ] Onboarding timing events logged

### Sprint 2 — Plan engine (story 3)

- [ ] `buildPlan()` with all 12 steps, unit and property tests passing
- [ ] Plan preview screen with real output; Commit and Make it lighter
- [ ] Full Plan screen (week list)
- [ ] Engine meets the 100 ms budget on a mid-range phone

### Sprint 3 — Daily loop (story 4)

- [ ] Today screen from live data, with if-then card and countdown
- [ ] Study session: clock-based timer, wake lock, 3 steps
- [ ] Session done: self-score, confidence; topic confidence updated
- [ ] Install prompt (Chromium) and iOS Add to Home Screen card

### Sprint 4 — Recovery (story 5) and export

- [ ] Day-rollover check on boot, foreground and midnight
- [ ] `repairPlan()` with diff; Recovery screen for all three situations
- [ ] Obstacle question and suggested if-then plans
- [ ] .ics export with Web Share fallback to download
- [ ] Backup and restore with zod validation

### Sprint 5 — Pilot hardening

- [ ] Pilot stats screen, participant code and CSV export
- [ ] Accessibility, Lighthouse and performance budgets met
- [ ] Real-device test matrix (Section 13) completed
- [ ] Short in-app help: what retrieval practice is and why plans spread out
- [ ] Pilot release tagged and deployed

### Definition of done (every story)

Code reviewed; tests written and passing in CI; works offline; works at phone and desktop widths; no axe serious issues; events logged where Section 11 lists them.

## 16. Risks and open questions

### Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Browser clears site data (storage eviction), especially Safari when not installed | Student loses the plan | Ask for persistent storage; push home-screen install; backup reminders after 7 days |
| No push reminders without a server | Students forget sessions | .ics export with alarms; if-then plan tied to a daily cue; re-export prompt after changes |
| Calendar apps handle re-imported .ics differently (some duplicate events) | Cluttered calendar | Stable UIDs and SEQUENCE; test Apple and Google Calendar; document the best path per app |
| Plan is overloaded for a student with many topics and little time | Plan feels impossible | Trimming rules, Make it lighter, honest at-risk status with clear options |
| Phone lost, replaced or reset | All data gone | Backup and restore; backup reminder on the Data screen |
| Time-zone or daylight-saving changes during exams | Sessions shift by an hour | Store local dates and minutes; convert to UTC only at .ics export |
| School Chromebooks block PWA installs | Can't install at school | App works fully in the browser tab; phone remains the main device |

### Open questions

| Question | Decision (Nov 25, 2026) | Where it lands |
| --- | --- | --- |
| Hosting domain | None yet. Build on the host's free subdomain; buy the final domain before pilot users install | Section 14, Domain |
| Item lengths | Start at 20 min and tune down: learn 20, reviews 15 / 10 / 5 | Section 6 |
| Skipping a session | Allowed, and never counts as a miss | Section 7 |
| Pilot consent | One form: parent permission + student assent | Section 12, Pilot consent (US) |
| Dark mode | In v1 | Section 8, Dark theme |

Still open:

- [ ] Which domain to buy, and by when?
- [ ] Who stores the signed consent forms and the exported stats files?
