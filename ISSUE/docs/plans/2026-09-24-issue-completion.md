# ISSUE Family Safety Platform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn ISSUE into a reliable, transparent parent/child safety app with its own authenticated backend, reference-inspired blue/white UI, tested Android controls, and account/password work last.

**Architecture:** The Fastify/SQLite backend remains the source of truth for parent policy and enrolled-device events. Android keeps a visible child supervision service, validates and caches versioned policies, and applies only OS-supported controls. Parent dashboard, child detail, reports, and control pages use the approved blue/white design; every feature communicates permission, device-owner, support, and child-acceptance requirements honestly.

**Tech Stack:** Kotlin Android app (compileSdk 35, minSdk 26, targetSdk 35, Java 17); Fastify 5, Node.js ESM, SQLite via better-sqlite3, Zod, JWT; Android Gradle build and Node test runner.

**Spec:** `docs/superpowers/specs/2026-09-24-issue-background-and-core-features-design.md`

## Global Constraints

- Preserve launcher icon, splash artwork, and wallpaper; redesign in-app screens only.
- Keep child mode read-only for parent policy and clearly disclose active data categories.
- Parent API actions require authenticated parent JWT and child ownership; device routes require an enrolled, revocable device token.
- Keep supervision visible with a foreground-service notification; location requires parent policy and Android permission.
- Device Owner, Android roles, permissions, and child acceptance must be shown as explicit readiness states; never claim unsupported success.
- No keylogger, covert camera/microphone/screen access, secret message scraping, or hidden call/SMS content collection.
- Password and recovery implementation is the final phase.
- Create a dated, restorable backup before each user-applied batch; run checks before replacing source and save a GitHub checkpoint after each passing batch.
- Build and test on Android 13 / OPPO CPH2325 where available; use a separate resettable phone for Device Owner enforcement verification.

## Review Focus

- A stale or revoked child token must never accept newer policies or commands; test rejection after revocation.
- An invalid policy update must preserve the previous valid cached policy; test malformed values and lower versions.
- Offline sync must retain restrictions and show stale sync age; test network loss and recovery.
- Missing Usage Access, location, notification permission, Device Owner, or an Android role must produce a precise readiness state; test each missing-capability state.
- A child declines or stops an assistance session; test that no camera, microphone, or screen capture begins or continues.

---

## Batch 0 — Source baseline, recoverability, and test harness

### Task 1: Establish a reproducible source and backup checkpoint

**Files:**
- Inspect: Android project under `ISSUE-android/`; backend under `ISSUE/backend/`.
- Create: dated source archive under `/sdcard/Download/ISSUE_Backups/` (Termux workflow).
- Create: `backend/test/smoke.test.js`.

**Interfaces:**
- Produces: verified Android/backend source snapshot and baseline build/test commands for later batches.

- [ ] Record `git status`, Android Gradle wrapper/version, `node -v`, `npm -v`, and `npm test`/backend start health before changes.
- [ ] Create and list a dated backup archive; verify it contains `MainActivity.kt`, `ControlCenterActivity.kt`, manifest, backend `src/`, and package lockfiles.
- [ ] Add a Node built-in test that imports the backend health server factory and asserts `GET /api/health` returns HTTP 200 and `{ok:true}`; if no server factory exists, extract only the route-registration factory from `src/server.js` and keep CLI start behavior unchanged.
- [ ] Run the test and Android debug build; save exact outputs as the baseline checkpoint.
- [ ] Save baseline commit/checkpoint to GitHub before feature batches.

## Batch 1 — Backend security and background core

### Task 2: Lock down API construction and health checks

**Files:**
- Modify: `ISSUE/backend/src/server.js`.
- Modify: `ISSUE/backend/src/config.js`.
- Test: `ISSUE/backend/test/server.test.js`.

**Interfaces:**
- Produces: `buildServer({ config, database })` for tests and `GET /api/health`, `GET /api/db/health` for operation checks.

- [ ] Write failing tests for health routes, missing JWT secret, and allowed parent origin.
- [ ] Extract an injectable Fastify server factory while keeping `npm start` listening on configured host/port.
- [ ] Reject production startup with empty/default JWT secrets; avoid logging credentials or event payloads.
- [ ] Configure CORS to explicit configured origins rather than reflecting arbitrary origins with credentials.
- [ ] Run backend tests and launch `npm start`; verify `/api/health` and `/api/db/health`.
- [ ] Commit the passing backend-core checkpoint.

### Task 3: Enforce enrollment, ownership, versioning, and command idempotency

**Files:**
- Modify: `ISSUE/backend/src/db.js`.
- Modify: `ISSUE/backend/src/device-session.js`.
- Modify: `ISSUE/backend/src/routes/parent-auth.js`, `family.js`, `family-admin.js`, `policy-sync.js`, and `device-commands.js`.
- Test: `ISSUE/backend/test/auth.test.js`, `policy-sync.test.js`, `device-commands.test.js`.

**Interfaces:**
- Produces: parent guard that validates JWT plus child ownership; device guard that validates active device token; monotonic `policyVersion`; idempotent command acknowledgement.

- [ ] Add failing route tests for cross-parent child access, invalid/revoked device token, stale policy versions, duplicate acknowledgement, and malformed policies.
- [ ] Add/verify schema migrations for token revocation, policy version, audit record, and command status without dropping existing data.
- [ ] Apply authorization guards to every parent/device route before reading or mutating protected records.
- [ ] Validate policy writes with Zod, persist and increment version transactionally, append an audit event, and queue exactly one sync command.
- [ ] Make acknowledgement idempotent and return the stored result for duplicate acknowledgements.
- [ ] Run backend tests against a temporary SQLite database; commit the checkpoint.

### Task 4: Make Android policy sync resilient and observable

**Files:**
- Modify: `app/src/main/java/com/issue/app/DeviceSyncManager.kt`.
- Modify: `app/src/main/java/com/issue/app/IssueRuleEngine.kt`.
- Modify: `app/src/main/java/com/issue/app/IssueSupervisionService.kt`.
- Modify: `app/src/main/java/com/issue/app/IssueBootReceiver.kt` and `AndroidManifest.xml` only for validated lifecycle requirements.
- Test: `app/src/test/java/com/issue/app/PolicySyncTest.kt`, `RuleEngineTest.kt`.

**Interfaces:**
- Produces: validated versioned policy cache, deterministic rule evaluation, bounded retry/backoff, visible service status, and idempotent command outcomes.

- [ ] Add failing tests for valid/invalid/lower-version policy, offline policy retention, retry cap, schedule boundary, and protected package exclusion.
- [ ] Persist the last valid policy atomically; reject malformed or lower-version payloads without replacing it.
- [ ] Evaluate cached rules while offline and expose last successful sync time and pending/failed command state.
- [ ] Ensure child supervision starts only for active child role plus valid enrollment, stays visible, and stops on session clear/revocation.
- [ ] Add bounded exponential retry with a maximum delay; never block the main thread or enforcement loop on network work.
- [ ] Run unit tests and `./gradlew :app:assembleDebug`; commit the passing checkpoint.

## Batch 2 — Approved dashboard and child experience

### Task 5: Build shared blue/white UI components and readiness labels

**Files:**
- Modify: `app/src/main/java/com/issue/app/IssueUi.kt`.
- Create: `app/src/main/java/com/issue/app/ui/IssueTheme.kt` and `ReadinessState.kt` only if matching project structure supports packages.
- Test: `app/src/test/java/com/issue/app/ReadinessStateTest.kt`.

**Interfaces:**
- Produces: reusable card/button/status components and readiness states `ACTIVE`, `NEEDS_PERMISSION`, `DEVICE_OWNER_REQUIRED`, `CHILD_ACCEPTANCE_REQUIRED`, `UNSUPPORTED`, `STALE`.

- [ ] Test mapping from Android/backend capability input to each readiness label, including unknown state.
- [ ] Implement approved blue/white surfaces, primary blue action, accessible contrast, spacing, and status accents.
- [ ] Add reusable readiness card that includes the missing permission/action in plain language.
- [ ] Confirm `issue_icon_source.jpg`, splash resources, and wallpaper assets are unchanged using file hashes before/after.
- [ ] Run Android build and UI unit tests; commit the visual foundation checkpoint.

### Task 6: Redesign parent dashboard, child detail, and child read-only home

**Files:**
- Modify: `app/src/main/java/com/issue/app/MainActivity.kt`.
- Modify: `app/src/main/java/com/issue/app/IssueUi.kt`.
- Test: `app/src/test/java/com/issue/app/DashboardModelTest.kt`.

**Interfaces:**
- Produces: parent dashboard with child cards, online/sync/alerts summary, quick controls, feed; child detail with day/week usage and status; read-only child home with SOS.

- [ ] Add failing model tests for no children, one child offline, stale status, alert count, and empty activity feed.
- [ ] Replace plain parent home with screenshot-inspired blue/white child cards, quick controls, recent alerts, and clear routes to controls/reports.
- [ ] Add child detail screen for day/week usage, app list, active limits/schedule, battery/location/sync readiness, and recent events.
- [ ] Remove parent-setting/mode-switch/admin entry points from child role while keeping visible supervision explanation and SOS.
- [ ] Verify parent/child login and navigation on device; confirm the icon, splash, wallpaper stay unchanged; commit checkpoint.

### Task 7: Redesign five control categories without breaking their policy contracts

**Files:**
- Modify: `app/src/main/java/com/issue/app/ControlCenterActivity.kt`.
- Modify: `app/src/main/java/com/issue/app/MainActivity.kt` only at the Controls navigation boundary.
- Test: `app/src/test/java/com/issue/app/ControlNavigationTest.kt`.

**Interfaces:**
- Produces: blue/white card pages for Screen Time, App Controls, Bedtime/School Mode, Location/Safe Zone, and SOS/Alerts; edits return typed policy intents to the sync layer.

- [ ] Test all five routes and a safe back path.
- [ ] Restyle category cards and fields to match approved references; keep categories and feature scope intact.
- [ ] Add field validation and per-control readiness state; disable or explain unsupported actions instead of displaying success.
- [ ] Preserve existing data keys until migration tests prove a safe rename.
- [ ] Run Android build and manually open/save each category; commit checkpoint.

## Batch 3 — Core controls and policy enforcement

### Task 8: Screen time, per-app limits, bedtime, and routines

**Files:**
- Modify: `ControlCenterActivity.kt`, `IssueRuleEngine.kt`, `IssueSupervisionService.kt`, backend `routes/policy-sync.js`, and `db.js`.
- Create: focused policy/time helpers under `app/src/main/java/com/issue/app/` and corresponding Android/backend tests.

**Interfaces:**
- Produces: validated policy fields for daily limit, package limits, bedtime windows, weekly school schedule, and current usage summary.

- [ ] Write failing tests for timezone/day rollover, overnight bedtime, schedule-day selection, per-app limit, and unset-limit behavior.
- [ ] Define one versioned policy shape and validate the same bounds at backend and Android boundaries.
- [ ] Store edits through authenticated parent endpoints and render pending/synced/applied status.
- [ ] Apply local cached rules; use Device Owner only for actions the Android platform actually permits.
- [ ] Verify offline behavior and run backend tests plus Android tests/build; commit checkpoint.

### Task 9: App approval/blocking, web filtering, purchase restrictions, remote lock

**Files:**
- Modify: `ControlCenterActivity.kt`, `DeviceOwnerManager.kt`, `DeviceSyncManager.kt`, `IssueRuleEngine.kt`.
- Modify: backend `routes/device-commands.js`, `routes/policy-sync.js`, and `db.js`.
- Test: Android capability/command tests and backend command/policy tests.

**Interfaces:**
- Produces: package allow/block/approval policy, supported domain/DNS policy, remote lock command, and truthful purchase-restriction readiness.

- [ ] Test command authorization, duplicate lock request, unsupported package control, and DNS setting validation.
- [ ] Implement package enforcement only when Device Owner capability is active; protect ISSUE and required system packages.
- [ ] Present purchase restrictions as supported only when the managed OS/store control is verified; otherwise give setup guidance and `unsupported` state.
- [ ] Queue lock, return command ID, display queued/applied/failed state, and lock only after an authenticated enrolled-device command.
- [ ] Verify APK build and backend tests; on a provisioned test phone only, verify enforcement; commit checkpoint.

## Batch 4 — Reports, location, status, SOS, and alerts

### Task 10: Usage report, battery/status, SOS, and parent activity feed

**Files:**
- Modify: `IssueSupervisionService.kt`, `DeviceSyncManager.kt`, `MainActivity.kt`, `ControlCenterActivity.kt`.
- Modify: backend `db.js`, `routes/family.js`, and `routes/policy-sync.js`.
- Test: Android usage/status tests and backend report/ownership tests.

**Interfaces:**
- Produces: aggregate day/week/app usage, battery/online/sync status, SOS event, and child event feed with ownership-checked reads.

- [ ] Test empty reports, timezone day boundary, unavailable Usage Access, duplicate SOS submission, and unauthorized child report access.
- [ ] Upload only permitted aggregates/status; persist SOS idempotently and with optional permission-approved location.
- [ ] Render day/week chart, most-used apps, device state, and recent activity with empty/loading/stale/error states.
- [ ] Verify opt-out/missing permission is visible; run backend tests and Android build; commit checkpoint.

### Task 11: Location history, geofences, speeding, and Wi-Fi controls

**Files:**
- Modify: `IssueLocationService.kt`, `IssueNotificationHelper.kt`, `DeviceSyncManager.kt`, `ControlCenterActivity.kt`.
- Modify: backend `db.js`, `routes/family.js`, `routes/policy-sync.js`.
- Test: Android geofence/speed tests and backend retention/history tests.

**Interfaces:**
- Produces: policy-enabled location samples/events with timestamps, configured safe zones, speed threshold events, and location retention/deletion behavior.

- [ ] Test disabled-policy no-collection, denied permission, duplicate transition, inaccurate/out-of-order location, speed boundary, and retention expiry.
- [ ] Collect only while policy and permission are active; keep the foreground location disclosure visible; reduce sample frequency when stationary.
- [ ] Implement parent-authorized geofence and speed settings; persist only necessary event data and enforce retention/deletion.
- [ ] Gate Wi-Fi network restrictions on actual Device Owner support and present precise status.
- [ ] Run backend tests, Android tests/build; verify physically only on provisioned child phone; commit checkpoint.

### Task 12: Safe web filtering, contact roles, and transparent content alerts

**Files:**
- Modify: `ControlCenterActivity.kt`, Android manifest and focused policy helpers.
- Modify: backend `db.js`, `routes/policy-sync.js`, `routes/family.js`.
- Test: capability tests, data-minimization tests, and backend authorization tests.

**Interfaces:**
- Produces: disclosed DNS/domain policy, role-gated contact allow/block settings, and narrowly scoped user-configured alerts from supported ISSUE-visible sources.

- [ ] Test that unsupported browser/private-app sources are reported unsupported and do not create monitoring records.
- [ ] Implement DNS filtering only through a visible VPN or managed Private DNS mode; document coverage limits and SafeSearch caveats.
- [ ] Enable call/SMS/contact configuration only if the child device explicitly assigns the required supported Android role; collect no message bodies by default.
- [ ] Implement keyword alert rules only for supported, disclosed sources and send category/time/minimum necessary excerpt; never read keystrokes or silently scrape notifications.
- [ ] Review disclosure, retention, and child-visible state; run tests/build; commit checkpoint.

## Batch 5 — Child-accepted support sessions and media-safety assessment

### Task 13: Assess and implement optional local media safety and accepted support flow

**Files:**
- Create/modify: a dedicated `AssistanceSessionActivity.kt` and session controller only after OS API and consent-flow tests pass.
- Modify: `AndroidManifest.xml` only for permissions required by the explicit accepted flow.
- Modify: backend command/session routes only if the session protocol can enforce short expiry and explicit child acceptance.
- Test: consent/session lifecycle tests and Android API tests.

**Interfaces:**
- Produces: expiring `requested → accepted → active → stopped/expired/declined` session state; no camera/audio/screen stream starts before child acceptance. Optional gallery classification remains local and user-visible unless the user explicitly enables disclosed sharing.

- [ ] Test decline, timeout, permission revocation, parent disconnect, child stop, and session expiry all terminate capture/transport.
- [ ] Confirm current Android camera/mic/screen-capture APIs expose system indicators and require immediate child acceptance; if not, leave the feature unavailable and document why.
- [ ] Build a visible child consent sheet with purpose, duration, data path, accept/decline, and persistent stop action before requesting OS capture permission.
- [ ] Assess on-device image classification with selected-media-only access, local processing, deletion control, and no silent upload; omit if these guarantees cannot be met.
- [ ] Verify no background capture after app/session stop; run tests/build and consent-device tests; commit checkpoint.

## Batch 6 — End-to-end regression and release hygiene

### Task 14: Verify full parent-to-child loop and recovery behavior

**Files:**
- Test: backend route suites, Android unit suites, and API smoke scripts under `ISSUE/backend/test/` and `app/src/test/`.
- Modify: `ISSUE/STATUS.md` and operational README only after verified results.

**Interfaces:**
- Produces: reproducible release checklist with separate `tested`, `simulated`, and `requires child phone` status.

- [ ] Test parent login → child selection → policy write → device sync → rule evaluation → command acknowledgement → updated dashboard.
- [ ] Test backend restart, phone offline/online transition, revoked token, malformed response, and app process restart.
- [ ] Run clean `npm ci`, backend test suite, Android debug build, startup/login regression, and available-device feature checks.
- [ ] Remove temporary crash logger only after repeated clean startup/login; verify no crash-report writer remains.
- [ ] Create final source backup, save passing GitHub checkpoint, and record unverified physical-device features accurately.

## Batch 7 — Password and recovery, last

### Task 15: Finalize parent password and recovery flows

**Files:**
- Modify: `MainActivity.kt` and focused authentication UI helpers.
- Modify: backend `routes/parent-auth.js`, `config.js`, and `db.js`.
- Test: backend auth tests and Android login/recovery tests.

**Interfaces:**
- Produces: rate-limited password update and recovery flow with hashed secrets, one-time expiring recovery credentials, audit events, and session revocation.

- [ ] Write failing tests for wrong password, lockout/rate limit, expired/used recovery credential, and session revocation after credential change.
- [ ] Implement server-side password hashing/verification using existing approved auth library; never log or store plaintext secrets.
- [ ] Add UI validation and non-enumerating recovery responses; keep recovery values one-time and revocable.
- [ ] Run all auth tests, full build, and startup/login regression; save final backup and GitHub checkpoint.

