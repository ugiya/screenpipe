# The v2.5.194 account gate — analysis for a personal local build

Written 2026-08-08 after upgrading this fork from v2.4.12 to v2.5.194. Findings are from
reading the v2.5.194 source; re-verify line numbers after any upstream sync.

## The symptom

A freshly built, correctly signed local build starts, but never records:

```
INFO screenpipe_app: Skipping server auto-start: screenpipe account access required
WARN screenpipe_app::sync: local retention paused until account plan verification completes
```

Neither the HTTP server nor capture is started. This is an upstream product gate added since
v2.4.12, not a build or signing fault.

## ⚠️ The non-obvious part: signing in on the FREE plan deletes your history

This is the finding that changes the decision. Three plan states exist
(`LocalPlanPolicy`, `src-tauri/src/store.rs:1884`), and they are **not** ranked the way you'd
expect for data safety:

| State | Recording | Local retention | Pipe cap |
|---|---|---|---|
| `Unknown` (signed out) | **denied** | never runs — nothing is deleted | 2 |
| `VerifiedFree` (signed in, free) | allowed, uncapped | **forced on: 7 days, mode `all`, cannot be disabled** | 2 |
| `VerifiedPaid` | allowed, uncapped | user-configured, opt-in, default 14 days mode `media` | unlimited |

On the free plan `sync.rs:457-487` ignores your `localRetentionEnabled` setting and forces
`enabled = true, days = 7, mode = "all"`. `RetentionMode::All`
(`crates/screenpipe-engine/src/retention.rs:505-524`) deletes frames, audio transcriptions, and
UI events. The clamp is re-applied to every `POST /retention/configure`
(`retention.rs:107-117`), so it cannot be overridden through the local API either.

**Signing in with a free account on a machine holding months of history would destroy
everything older than 7 days.** As of this writing that is ~22 GB in `db.sqlite` going back to
April. Back up first (see `.claude/skills/build-from-source/SKILL.md` §5) regardless of which
option you choose.

Recording quality itself is identical across free and paid — capture, FPS, audio, and OCR are
not plan-gated.

## What the gate actually checks

`recording_access_policy` (`src-tauri/src/recording.rs:124`) → `recording_access_allowed`
(`:146`). For a consumer release build, recording is allowed iff
`local_plan_policy() != Unknown`, which requires an account identity (`user.id` or
`user.clerk_id`) **plus** a server-verified plan blob. `cfg!(debug_assertions)` is the only
bypass; there is no environment variable escape hatch in release builds.

Five call sites: native auto-start (`main.rs:1661`), `start_capture` (`recording.rs:613`),
`spawn_screenpipe` (`:866`), `start_capture_internal` (`:1328`), and the health watchdog's
respawn check (`health.rs:971`), which will not restart a crashed engine while the gate is closed.

## Options

### A. Sign in, free plan
Recording works immediately, zero code changes, survives every future upgrade.
**Cost: forced 7-day destructive retention.** Only acceptable if you do not want a long archive.

### B. Sign in, paid plan
Recording plus configurable retention and no pipe cap. $25/mo.

### C. Patch the gate in this personal build
Keeps plan state at `Unknown`, so **retention never runs and nothing is auto-deleted** — the
safest state for an existing archive. Still subject to the 2-pipe cap; paid features stay
locked. The source license permits personal, non-commercial modification.

Two edits are required — a Rust-only patch does not work, because the webview stops the
recorder a moment after the backend starts it:

1. `src-tauri/src/recording.rs:149` — the `dev_bypass` argument, currently
   `cfg!(debug_assertions)`. It is the first early return in the policy, so it also neutralises
   the enterprise-app branch and covers all five call sites at once.
2. `components/app-entitlement-gate.tsx:374-399` — when `shouldGate` is true the primary window
   calls `commands.stopScreenpipe()` and renders a blocking shell.

Risk is low and bounded:

- **Tests:** the five gate tests (`recording.rs:1488` `mod recording_access_tests`) call the
  pure `recording_access_policy` with literal arguments, so they still pass. Do **not** patch
  `recording_access_policy` itself — that breaks them.
- **Rebase:** two small hunks to re-apply per upstream sync.
- **Side effects:** retention never auto-starts, so disk grows unbounded and needs manual
  management (~9 GB/month at current usage). The health watchdog will now respawn a crashed
  engine, which is the desired behaviour but removes a throttle on respawn storms.
- `app-entitlement-required` is emitted at `main.rs:1664` but has no listener anywhere in the
  repo — dead weight either way.

## Still open

An inventory of telemetry / analytics / tracking (PostHog, Sentry, usage pings, outbound
hosts) was in progress when this session paused and was not completed. `analytics.rs:502`
shows telemetry is already disabled in debug builds, which suggests `analytics.rs` is the place
to start.
