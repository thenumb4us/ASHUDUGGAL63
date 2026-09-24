# ISSUE Family Safety Platform — Product and Technical Design

Date: 2026-09-24

## Goal and approved direction

Build ISSUE as its own transparent family-safety system: parent-managed policies, an enrolled child role that displays its supervision state, and a backend that securely synchronizes rules and device status. The new parent dashboard and feature pages follow the supplied references: clean blue/white surfaces, blue primary actions, colored status accents, child cards, quick controls, usage charts, and a readable activity feed. Preserve the existing launcher icon, splash artwork, and wallpaper. Replace the current plain controls presentation as part of this redesign; do not copy another product's logos or exact screen assets.

Success means the parent can configure policies, see honest per-feature readiness, and receive child-device results; the child sees what is active and can use SOS; backend/device sync works with outages; and every enforcement claim is proven by tests or clearly marked as needing a provisioned child phone. Password/recovery work stays last.

## Feature scope and honest Android limits

All requested feature areas are tracked below. “Supported with consent” means a visible feature disclosure, required Android permission/role, and an obvious active state. No capability may be advertised as working when the OS, permission, enrollment, or device-owner status does not support it.

| Area | Planned behavior | Boundary / readiness |
| --- | --- | --- |
| Screen time, per-app limits, app usage reports | Day/week totals, app breakdown, daily cap, warnings, and policy sync | Usage Access required; hard enforcement depends on Device Owner or a supported accessibility-based flow clearly disclosed to the child |
| Downtime, bedtime, school mode, schedules/routines | Recurring schedules, manual school mode, offline rule evaluation | Local cached policy; Device Owner required for reliable app suspension |
| App blocking and approval | Parent allow/block list and approval queue for newly installed apps | Device Owner required for package suspension; unsupported actions stay visibly pending/unavailable |
| Web filtering and SafeSearch | Family DNS policy and domain allow/block rules | Use a clearly disclosed VPN or Private DNS method; filtering is not represented as universal in-app content inspection |
| In-app purchase protection | Explain and configure available Play Store/device restrictions, with parent guidance | No claim of universal purchase blocking; OS/store controls and child-device setup determine support |
| Live location, history, geofences, speeding/driving alerts | Policy-enabled location, visible tracking state, configurable safe zones, event history, speed threshold | Location permission and foreground notification required; retention is limited/configurable; background location and device testing required |
| Battery, network, online, and device status | Last-seen, battery, sync age, permission/protection readiness | Upload only enrolled-device operational status |
| Remote lock and uninstall protection | Authenticated queued command, acknowledgement, owner/protection state | Device Admin can lock; stronger controls and uninstall protection require Device Owner |
| SOS / panic | Child-visible one-tap event with location when available and parent alert | Child action is explicit; no covert activation |
| Call/SMS and contact controls | Contact allow/block management only through supported default dialer/SMS roles; call/SMS event reporting only where Android grants the role and the child/parent setup clearly discloses it | No hidden message-body collection; Android role, regional law, and distribution rules may make this unavailable |
| Social/content/keyword alerts | Parent-configured risk keywords applied only to content ISSUE itself receives through an explicitly enabled, visible, supported source; report category/time and minimum necessary excerpt | No keystroke capture, private-app scraping, or silent notification harvesting; unsupported apps show unavailable |
| Gallery safety | Optional on-device, user-visible classification of selected media or an explicitly enabled media library, with parent-configured categories and deletion controls | No secret upload or remote browsing of a child's gallery; process locally where feasible and disclose data handling |
| Remote camera, one-way audio, screen mirroring | Replace covert remote access with a child-initiated or child-accepted, time-limited assistance session; Android camera/mic indicators and screen-capture consent remain visible, and either participant can stop | No silent camera, ambient listening, or hidden screen capture. Denied/unavailable unless the child accepts each OS-mediated session |
| Incognito/private browsing | Report that private browsing cannot be reliably identified by a normal app; apply DNS-level family filtering where configured | Do not claim history capture or bypass browser privacy boundaries |

Features explicitly excluded: keylogging, covert camera/microphone/screen access, secret private message/social-media scraping, and hidden collection of call/SMS contents. These are not included under alternate names. The safe alternatives above preserve child awareness and OS consent.

## UI and navigation

- Parent home: greeting, child cards with online/supervision state, quick actions, active limits, recent alerts/feed, and clear navigation to reports and controls.
- Child detail: day/week usage graph, total screen time, most-used apps, remaining limit, active schedules, location/status, and recent policy events.
- Controls use the same five existing functional groups—Screen Time, App Controls, Bedtime/School Mode, Location/Safe Zone, SOS/Alerts—but redesigned in the approved blue/white card style. Supporting areas (reports, content alerts, contact options, and assistance sessions) appear as subpages where supported.
- Every control has a readiness label such as `Active`, `Needs permission`, `Device Owner required`, `Child acceptance required`, `Not supported on this device`, or `Last synced …`. Never show a false-success state.
- Child mode is read-only for parent policies, displays active supervision and data-sharing status, and retains SOS and relevant permission explanations. It offers no mode switch, parent controls, or hidden administrative actions.
- Preserve launcher icon, splash image/artwork, and wallpaper. UI redesign applies to in-app screens only.

## Architecture

### Backend and security

Fastify is the source of truth for parent-managed policy and enrolled-device events. SQLite stores parent identities, children/devices and active links, hashed/revocable device credentials, versioned policies, commands and acknowledgements, operational status, aggregate usage, SOS/location/geofence events, and audit records. Avoid storing sensitive content when a category/event is sufficient.

Parent routes require a valid parent JWT and ownership check for the selected child. Device routes require a valid, revocable device token and active enrollment. Validate payloads and policy versions, rate-limit authentication/sensitive writes, use parameterized database access, and audit administrative changes. Secrets are never logged or committed. Retention/deletion rules apply to location, usage, and event records.

### Android child service and policy engine

Start a visible foreground supervision service only after valid child enrollment and required user-visible setup. Evaluate the last valid policy locally, synchronize on a bounded schedule with retry/backoff, persist policy atomically, upload minimal status and permitted aggregates, and acknowledge each command idempotently. Preserve the last-known policy offline. Boot resume is allowed only for an enrolled child role with a valid token. Clearing enrollment stops supervision and revokes local credentials.

The rule engine resolves schedule, daily screen cap, app limits, app allow/block policy, and network policy into an explicit state. Device Owner-only actions (package suspension, uninstall protection and managed restrictions) report exact readiness and cannot silently degrade to success. Prevent ISSUE and required system packages from being blocked.

### API/data flow

1. Parent authenticates and selects an owned child.
2. Parent edits a policy or requests an allowed assistance session.
3. Backend validates authorization and payload, persists the version/audit event, and queues a command if needed.
4. Enrolled child syncs over authenticated transport, validates and atomically caches policy, then applies supported rules.
5. Child acknowledges command outcome and sends only permitted status, aggregate usage, SOS, and policy-enabled location/geofence events.
6. Parent reads the latest state, history, and readiness from authenticated endpoints.

## Reliability, privacy, and errors

- Invalid parent sessions, wrong child ownership, revoked device credentials, invalid policy versions, malformed payloads, and replayed commands must not change device policy.
- Network loss keeps the last valid policy active and retries with bounded backoff; UI shows stale sync time.
- Missing permission, role, Device Owner provisioning, or child acceptance is reported as a specific state, not an exception or success.
- Background failures are logged without sensitive payloads and cannot crash launcher/login flows.
- Collection is purpose-limited, visible, revocable where OS permits, and retained only as specified. Child can see what categories are enabled. Location collection requires an active parent policy and Android permission; audio/video/screen sessions require immediate child acceptance.
- No private content is uploaded by default. If a supported alert source is enabled, disclose the exact source, data category, and retention before enabling it.

## Verification strategy

### Automated

- Backend tests: authentication, ownership, revocation, schema validation, policy versioning, audit events, command idempotency, retention, and history access.
- Android tests: schedule/time boundaries, usage aggregation, policy parsing/version rollback, rule-state calculation, protected-package allowlist, offline cache, retry/backoff, and capability/readiness states.
- Static checks: Android manifest permissions/components and prohibition of undeclared sensitive collection; API contract and migration checks.
- Build, launch/splash/login regression, backend health/database health, policy-write/read, and simulated device sync/ack smoke tests.

### Available single phone

Verify parent login/dashboard, pages and input validation, persistence, backend sync, Usage Access/location prompts where available, visible-service lifecycle, offline cache, SOS submission, and honest unsupported states. Simulate remote device commands rather than claim physical enforcement.

### Separate child phone required

Factory-reset/provision a dedicated test phone to verify Device Owner setup, app suspension, uninstall protection, network restrictions, remote lock, reboot recovery, live location/geofence events, and accepted assistance-session UX. Camera/audio/screen APIs require device-side consent tests and must remain disabled absent a clear child-accepted flow. Mark these `Not verified on a child device` until tested.

## Delivery sequence

1. **Baseline and security contract** — inventory current Android/backend branches and migrations; lock API/data schemas, auth boundaries, audit/retention, and explicit capability states; restore reproducible build/test baseline.
2. **Background core** — service lifecycle, enrollment/session hardening, policy cache/versioning, retry/offline behavior, status/usage/battery sync, location lifecycle, command acknowledgement, and tests.
3. **Approved dashboard redesign** — implement the blue/white parent dashboard, child detail, navigation, five control groups, child read-only state, and truthful readiness labels while preserving icon/splash/wallpaper.
4. **Core controls** — screen/app limits, app approval/blocking, schedules, network filtering, supported purchase restrictions, remote lock, and uninstall-protection readiness; integrate backend policy sync.
5. **Safety reports and alerts** — usage reports, location history/geofences/driving alerts, device status, SOS, content alert sources with disclosure, and supported contact-role controls.
6. **Assisted support and media safety** — assess local media classification and child-accepted time-limited camera/mic/screen support against OS APIs, disclosure, consent, and safety checks; omit any method that cannot preserve these guarantees.
7. **End-to-end verification and cleanup** — backend and Android tests, build/install regression, child-device tests when available, remove temporary crash logger, review retention/security, and save GitHub checkpoints.
8. **Account finalization** — password and recovery only after prior work is stable and tested.

Each implementation batch must be small enough to build and test, create a targeted backup, verify before replacement, and checkpoint to GitHub. Fix a failed batch before beginning the next. No batch may change launcher icon, splash artwork, or wallpaper.

## Definition of done

Background core is done when automated tests pass, Android builds and opens through splash/login, simulated enrolled devices synchronize authenticated policy and idempotent command results, cached rules survive offline periods, and service/readiness states are visible.

Feature code is done when each supported control has validated parent policy, persists/syncs, displays honest Android capability requirements, and has unit/API coverage. UI is done when the approved dashboard and reference-inspired feature pages work responsively and preserve the icon/splash/wallpaper. Device Owner and consent-gated features are not called verified until tested on the appropriate physical child-device flow. Password/recovery remains the final phase.
