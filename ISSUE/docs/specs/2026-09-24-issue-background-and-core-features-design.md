# ISSUE Background Core and Safe Parental Controls Design

Date: 2026-09-24

## Purpose

Complete ISSUE as a transparent parental-control application with one Android app that supports separate parent and child roles. The parent configures policies; the enrolled child device receives and enforces them. The current UI structure, icon, splash artwork, and wallpapers remain unchanged.

Implementation order is fixed:

1. Background core and backend reliability.
2. Safe parental-control features inside the five existing control categories.
3. End-to-end tests and fixes.
4. Password, recovery, and remaining account work last.

## Constraints

- Do not redesign `MainActivity`, `ControlCenterActivity`, icons, splash artwork, or wallpapers.
- Keep the existing five top-level control categories.
- Child mode is read-only and must not expose parent settings, mode switching, or administrative controls.
- Parent actions require an authenticated parent session.
- Device actions require an enrolled device token and active parent-child link.
- Background supervision must remain visible through an Android foreground-service notification.
- Location collection runs only when enabled by policy and granted by Android permission.
- Features that require Device Owner report `Provisioning required` when unavailable; they must never report false success.
- Current single-phone development can verify builds, backend APIs, local policy persistence, and non-Device-Owner paths. Device Owner enforcement remains marked unverified until a separate resettable child phone is available.
- Password and recovery changes are excluded until the final phase.
- Hidden keylogging, covert camera or microphone access, secret screen capture, and covert message monitoring are excluded.

## Current Baseline

The current project already contains foundations for:

- Parent authentication and parent dashboard.
- Child enrollment/device sessions.
- Versioned policy sync and offline policy storage.
- Command queue and acknowledgements.
- Usage-stat collection.
- Foreground supervision service.
- Boot receiver.
- Location foreground service.
- Device Admin and Device Owner operations.
- Screen-time, bedtime, school-mode, app suspension, uninstall protection, private DNS, Wi-Fi restriction, remote lock, SOS, status, and event code.

The baseline builds, launches, displays the custom splash, reaches parent login, connects to the Kali-hosted backend, and opens the existing five-category Controls screen. Foundations are not yet considered complete until their API contracts, lifecycle behavior, and failure paths are verified.

## Architecture

### Backend Core

The Fastify backend is the source of truth for parent-managed policy and device commands. SQLite stores:

- parents, children, devices, and active parent-child links;
- hashed device credentials and revocation state;
- one versioned policy per child;
- pending/applied/failed device commands;
- device status, usage summaries, SOS events, locations, geofence events, and audit events.

Every parent route verifies a parent JWT and ownership of the requested child. Every device route verifies the enrolled device token. Policy writes increment the policy version and queue a `SYNC_POLICY` command. Remote lock queues a `LOCK_DEVICE` command. Device acknowledgements are idempotent.

### Android Background Core

The child role starts a visible foreground supervision service only after valid enrollment. The service:

- evaluates cached rules every minute;
- syncs policy and commands every five minutes when online;
- keeps the last valid policy active while offline;
- uses bounded retry/backoff after network errors;
- uploads status and usage summaries without blocking enforcement;
- starts or stops location tracking according to policy and permission;
- acknowledges each command once;
- stops when the device session is cleared or the app is no longer in child role.

Boot handling only resumes supervision when the stored role is child and a valid device token exists. Package replacement does not start a foreground service from the background.

### Enforcement

The rule engine calculates one state: `NORMAL`, `SCREEN_TIME_LOCK`, `BEDTIME_LOCK`, or `SCHOOL_MODE`. It combines the global state with explicit blocked packages, per-app limits, purchase blocking, and approved-app rules.

Device Owner capabilities perform package suspension, uninstall protection, private-DNS policy, and Wi-Fi configuration restrictions. Device Admin may perform immediate lock but does not pretend to provide Device Owner capabilities. ISSUE and required system packages are always excluded from suspension.

### Existing Control Categories

The top-level UI remains unchanged. Existing screens gain backend-backed behavior:

1. **Screen Time**: daily limit, remaining time, per-app limits, usage report, and warning status.
2. **App Controls**: blocked/allowed/approved packages, block-new-apps, purchase blocking, remote lock, uninstall-protection status, safe DNS, and Wi-Fi restriction.
3. **Bedtime / School Mode**: bedtime, manual school mode, weekly school schedule, and offline routine status.
4. **Location / Safe Zone**: current/last location, history, geofences, and speeding threshold/events.
5. **SOS / Alerts**: SOS history, battery/device/online/sync status, command results, and policy alerts.

Controls that cannot operate on the current device show the exact missing permission or provisioning requirement.

### Child Experience

Child mode shows supervision state, usage/remaining time, active schedule, last policy-sync time, location-permission state, and a visible SOS action. It exposes no policy editors, parent tokens, mode switching, removal controls, or Device Owner toggles.

## Data Flow

1. Parent authenticates and selects an owned child.
2. Parent updates a control inside an existing category.
3. Backend validates and persists the policy, increments its version, records an audit event, and queues sync.
4. Enrolled child service authenticates, downloads the policy and pending commands, validates them, and saves the policy atomically.
5. Rule engine evaluates cached policy and applies available Android controls.
6. Child posts command acknowledgement, status, usage, battery, and allowed event data.
7. Parent dashboard reads the latest stored device state and history.

## Failure Handling

- Invalid or expired parent sessions return authentication errors without changing policy.
- Revoked or invalid device tokens stop sync and supervision until re-enrollment.
- Malformed policy values are rejected server-side and ignored client-side without replacing the last valid policy.
- Network failure retains the last valid policy and schedules a bounded retry.
- Missing Usage Access, location permission, notification permission, Device Admin, or Device Owner is surfaced explicitly.
- A failed command is acknowledged as failed with a safe reason; it is not silently marked applied.
- Background exceptions are recorded without crashing the launcher activity.

## Testing Strategy

### Automated and Local

- Backend route authentication, ownership, validation, policy versioning, command idempotency, and history tests.
- Android unit tests for time windows, schedules, policy parsing, state calculation, allowlist protection, and retry timing.
- Static manifest/component checks.
- Android debug build and startup test.
- Backend health and database-health checks.
- Parent policy write/read cycle and simulated device sync/ack cycle.

### Current Phone

- Parent login and dashboard.
- All five category screens and validation.
- Backend persistence and displayed status.
- Usage Access, notification, location, Device Admin, offline-cache, reboot, and visible-service behavior where supported.

### Deferred Physical Verification

A separate factory-reset child phone is required to verify Device Owner provisioning, package suspension, uninstall blocking, private-DNS enforcement, Wi-Fi restrictions, and true parent-to-child remote commands. These capabilities remain labeled `Ready — child device testing required` until verified.

## Delivery Batches

1. **Background Reliability**: schemas, authentication contracts, service lifecycle, offline cache, retry/backoff, boot resume, status/usage/battery, location lifecycle, command acknowledgement, and automated tests.
2. **Core Controls Wiring**: connect the five existing categories to backend policies and current-device capability status without changing the top-level UI.
3. **Reports and Alerts**: location history, geofence/speed events, usage/device status, SOS history, command results, and parent-visible alerts.
4. **Verification and Cleanup**: full builds, backend tests, API smoke tests, startup/login regression, remove temporary crash logger, backups, and GitHub checkpoint.
5. **Account Finalization**: password and recovery only after the prior batches pass.

Each batch includes an apply script, automatic targeted backup, validation before replacement, and an independently buildable checkpoint. A failed batch is fixed before the next batch begins.

## Definition of Done

Background core is complete when backend tests pass, Android builds, startup/login remain stable, a simulated device can sync policy and acknowledge commands, cached rules survive offline operation, lifecycle/status paths are visible, and no UI/icon/wallpaper regression occurs.

Core feature code is complete when all five existing categories persist validated backend policy, expose capability status, and drive the corresponding child policy fields. Device Owner-dependent items are not called fully verified until tested on a separate provisioned child phone.

