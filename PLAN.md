# Alpha — Persistent Calendar-Aware Reminders (Feature 1)

## Context

Building a personal, single-user iOS productivity app ("Alpha"). This first feature is a
**short-term reminder system designed for ADHD**: normal reminders are too easy to dismiss and
forget, so nudges must be *persistent* yet *non-intrusive*.

Core user flow (the "book a doctor's appointment" case):
1. Create a task with a completion window — **next day** or **next week**.
2. Nudging starts the **next day** (never same-day), no earlier than a configurable
   "start of day" hour.
3. The app reads the user's **Google Calendar** (already synced into the iOS Calendar app),
   finds a **free gap**, and sends a gentle **push notification** at that time.
4. It must **NOT** notify while the user is in a meeting / has something scheduled.
5. If ignored, it **re-nudges at the next free gap** (persistence without being an alarm).
6. The user can **mute** a task for N hours / days.

### Key architecture decisions (confirmed with user)
- **Native Swift / SwiftUI** (user prefers native despite JS background; better background +
  calendar/notification reliability for a persistence-critical app).
- **No server, no backend, no cost.** Uses **on-device local notifications** — not remote push.
  (A server/Express app cannot send iOS notifications anyway; it must go through APNs, which
  needs a paid account. Local notifications need neither.)
- **Free Apple ID provisioning** to start. Caveats baked into the design (see Constraints).
- Calendar read via **EventKit** against the iOS Calendar store (where Google Calendar syncs) —
  no Google OAuth, no network.

### The core iOS constraint that shapes everything
iOS does not let an app run code continuously to "watch" for a gap opening up. Instead the app
must **pre-schedule** local notifications ahead of time. So the design is:
**whenever the app gets a chance to run** (foreground launch, Background App Refresh wake, or a
calendar-changed event), it re-analyzes the near-term calendar, computes free gaps, and
schedules/reschedules local notifications into those gaps — canceling any that are now stale.

## Constraints & caveats (free provisioning)
- **7-day expiry**: app must be re-installed from Xcode weekly. Acceptable for personal use.
- **Background App Refresh is opportunistic**, not guaranteed on a schedule (iOS decides). We do
  not rely on it alone — every app open also re-analyzes and reschedules.
- **64 pending local notifications max per app** → schedule only a rolling window (next ~24–48h
  of gaps across active tasks), not everything.
- **Time-Sensitive interruption level** needs a special entitlement not available on free
  accounts → use the standard `.active` level (a normal, non-alarm banner). This actually matches
  the "gentle nudge, not an alarm" requirement.

## Prerequisites (must do before any code runs)
1. **Install full Xcode.app** from the Mac App Store. Current machine has only Command Line Tools
   (`xcode-select -p` → `/Library/Developer/CommandLineTools`; no `/Applications/Xcode.app`).
   Then `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`.
2. Ensure the **Google account is added under iOS Settings → Calendar** on the physical device so
   events sync into EventKit. (In the Simulator, either add a Google account or seed test events.)
3. Sign in Xcode with a **free Apple ID** (Settings → Accounts) for device provisioning.

## Architecture

Layered so the scheduling brain is pure and unit-testable, with iOS specifics at the edges.

```
Alpha/
  AlphaApp.swift              // @main; SwiftData ModelContainer; registers BG task + notif delegate
  Models/
    ReminderTask.swift        // SwiftData @Model
    AppSettings.swift         // day window, min gap, buffers, selected calendars
  Services/
    EventKitService.swift     // auth + fetch busy intervals from iOS Calendar (EventKit)
    GapAnalyzer.swift         // PURE: (busy intervals, window, config) -> ranked free slots
    NudgeScheduler.swift      // reconciles tasks x gaps -> UNNotificationRequests (respects cap)
    NotificationCenterDelegate.swift  // handles Done / Mute / Snooze actions
    BackgroundRefreshManager.swift    // BGAppRefreshTask registration + handler
  Views/
    TaskListView.swift        // list of active tasks, status, next-nudge time
    AddTaskView.swift         // title, window (Tomorrow / This Week), notes
    TaskDetailView.swift      // complete, mute-for picker, delete
    SettingsView.swift        // day start/end hours, min gap, buffer, calendar selection
```

### Data model (`SwiftData`)
`ReminderTask`:
- `id: UUID`, `title: String`, `notes: String?`
- `createdAt: Date`
- `dueDate: Date` — end of window (createdAt + 1 day or + 7 days, snapped to day-end)
- `status: enum { active, completed, expired }`
- `muteUntil: Date?` — no nudges before this instant
- `lastNudgedAt: Date?`, `nudgeCount: Int` — for spacing / diagnostics

`AppSettings` (single instance): `dayStartHour` (default 9), `dayEndHour` (default 21),
`minGapMinutes` (default 30), `meetingBufferMinutes` (default 10),
`selectedCalendarIDs: [String]`, `maxNudgesPerDay` (default 4).

### `EventKitService`
- `requestAccess()` → `EKEventStore.requestFullAccessToEvents()` (iOS 17+).
  Info.plist: `NSCalendarsFullAccessUsageDescription`.
- `busyIntervals(on day:) -> [DateInterval]`: build a day predicate, fetch events across selected
  calendars, keep only `event.availability == .busy` (and `.tentative` optionally), **skip all-day
  events** so they don't block the whole day, return merged intervals.
- Observe `.EKEventStoreChanged` to trigger reschedule when the calendar changes.

### `GapAnalyzer` (pure, no iOS deps → fully unit-testable)
- Input: `busy: [DateInterval]`, `window: DateInterval` (that day's start/end hours), `config`.
- Merge overlapping busy intervals; pad each by `meetingBufferMinutes`; subtract from window to get
  free slots; drop slots shorter than `minGapMinutes`.
- **Score & rank** slots (the "intelligently pick" part): prefer mid-morning, avoid immediately
  before a meeting boundary, prefer earlier in the window. Return ranked candidate nudge `Date`s
  (e.g., a point inside each qualifying gap).

### `NudgeScheduler`
- For each `active`, non-muted task, over the horizon (from `max(now, nextDayStart)` up to
  `min(dueDate, now+48h)`): ask `GapAnalyzer` for ranked slots per day and pick the next few
  candidate nudge times.
- Build `UNNotificationRequest`s with `UNCalendarNotificationTrigger`, category
  `TASK_NUDGE` (actions below), interruption level `.active`, gentle/no sound.
- **Reconcile**: stable identifiers per (task, slot); cancel pending requests that are now stale
  (task completed/muted, or gap no longer free), keep valid ones, add new. Enforce the 64 cap by
  taking the soonest-first across tasks.
- **Re-nudge on next gap** = the next scheduled candidate simply fires later; completing/muting a
  task cancels its pending requests.

### `NotificationCenterDelegate` (interactive actions — persistence without an alarm)
`UNNotificationCategory` `TASK_NUDGE` with actions:
- **Done** → mark `completed`, cancel that task's pending notifications.
- **Mute 2h** (and a longer "Mute for…" opened via app) → set `muteUntil`, reschedule.
- **Snooze to next gap** → skip current, keep the next candidate.
Tapping the banner opens `TaskDetailView`. `willPresent` shows a quiet banner if in-app.

### `BackgroundRefreshManager`
- Register `BGAppRefreshTaskRequest` (id e.g. `com.alpha.refresh`) in Info.plist
  `BGTaskSchedulerPermittedIdentifiers`; enable **Background Modes → Background fetch**.
- On wake: re-run analyze + `NudgeScheduler.reconcile()`, then reschedule the next refresh.
- Also run reconcile on `.EKEventStoreChanged` and on every foreground `.active` transition.

## Build order (phased so each phase is runnable)
1. **Project + model + UI shell**: Xcode SwiftUI app, SwiftData `ReminderTask`, `AddTaskView`
   (Tomorrow / This Week), `TaskListView`, complete/delete. No calendar yet.
2. **Calendar + gap analysis**: `EventKitService` (permission + busy fetch), `GapAnalyzer` with
   unit tests. Show each task's computed "next nudge time" in the UI to validate visually.
3. **Local scheduling**: `NudgeScheduler` + notification permission; schedule into gaps on app
   open; verify a real notification fires (use short triggers while testing).
4. **Interactive actions + mute**: `NotificationCenterDelegate` (Done / Mute / Snooze),
   `TaskDetailView` mute-for-N picker.
5. **Background + reconciliation**: `BackgroundRefreshManager`, `.EKEventStoreChanged` observer,
   stale-notification reconciliation and 64-cap handling.
6. **Settings polish**: day-window hours, min gap, buffer, calendar selection.

## Verification
- **Unit tests (`GapAnalyzer`)**: feed fabricated `busy` intervals + window/config, assert ranked
  free slots — e.g., back-to-back meetings leave no gap; a lunch gap is found; buffer shrinks
  slots correctly; sub-`minGap` slots dropped. Run with `xcodebuild test` / ⌘U.
- **EventKit (Simulator)**: seed a few `EKEvent`s via a debug helper (or add a Google account in
  Simulator Settings), then confirm `busyIntervals(on:)` and the UI's "next nudge time" match.
- **Notification firing (Simulator/device)**: temporarily schedule with a short
  `UNTimeIntervalNotificationTrigger` (e.g., 15s) to confirm banner + Done/Mute/Snooze actions
  work and update task state.
- **Busy-suppression**: create a task, put a live "meeting" over the chosen gap, reschedule, and
  confirm no notification lands inside the meeting and the nudge moves to the next free gap.
- **Mute**: mute a task 2h, confirm pending notifications are canceled and nothing fires until
  `muteUntil`.
- **End-to-end on device**: create a "Tomorrow" task, confirm nothing fires today, and a nudge
  appears in the next day's first qualifying free gap after `dayStartHour`.

## Out of scope (future)
Remote push / cross-device sync (would need paid account + APNs + server), other productivity
features, widgets/Live Activities, Focus-mode awareness.
