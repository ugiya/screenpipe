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

## Telemetry inventory

Separate from the gate, and independently actionable.

**Four PostHog senders** share one project key pointing at `us.i.posthog.com`: the webview
(~305 capture sites across 71 files), the Tauri host (`src-tauri/src/analytics.rs`, including an
`app_still_running` heartbeat every 6 h), the engine (`crates/screenpipe-engine/src/analytics.rs`,
including `vision_pipeline_health` and `audio_pipeline_health` **every 60 s**), and the resource
monitor. Identifying data sent includes email, name, user/clerk id, GitHub username,
subscription plan, machine hostname, GPU names, audio device names, and a persistent
`~/.screenpipe/machine_id` UUID. `src-tauri/src/analytics.rs:86` also does an IP-matched UTM
attribution lookup against `screenpipe.com/api/attribution` on first boot.

**Sentry** runs in both the app (`main.rs:550`) and the engine, gated on the same analytics
setting. `send_default_pii` is false and paths are stripped.

**Worth knowing:** `components/desktop-remote-control.tsx:69` lets PostHog **feature flags mutate
your settings and restart the recorder** — inbound remote control, not just outbound reporting.
It is disabled when analytics are opted out.

**Good news for this build:** the auto-updater is already inert. `updates.rs:118-122`
short-circuits when neither the `official-build` nor `enterprise-build` cargo feature is set, and
the base `tauri.conf.json:48` ships `"endpoints": []` anyway.

**Fastest complete off-switch:** set `SCREENPIPE_DISABLE_TELEMETRY=1` in the app's environment
**and** turn off Settings → Privacy → Telemetry → Analytics. Together those cover every Rust
sender and both Sentry clients. One residual remains: `app/providers.tsx:91` calls
`posthog.init()` unconditionally, so the webview still makes a flags/config request on each
launch before the opt-out applies. Killing that needs a source edit — either skip `init` or pass
`opt_out_capturing_by_default: true` and `advanced_disable_decide: true`.

Disabling all of the above is safe: the local `/health` sampler and `last-panic.log` are written
locally and do not depend on telemetry. Do **not** block the auth, AI-inference, or model-download
hosts — those are real functionality.
