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

## v2.7.66 port (2026-09-24)

Synced from v2.5.194 to release tag `app-v2.7.66` (1,300+ upstream commits). The v2.5.194 hunks
were not replayed: the entitlement gate had been rewritten, and a clean textual apply would have
missed new gates. The port re-implemented the same intent, then went through five independent
review rounds, each of which found another layer. Commits: `834c7f745`, `fdf9550ba`, `8bf28dfbd`,
`32f2fe070`.

### What changed upstream

- **Obsolete:** the free-plan forced 7-day `mode=all` retention clamp was removed upstream in
  `79321df68` (app, engine and `POST /retention/configure`). Those old hunks were dropped.
- **New gates the old patch never covered:** an explicit `LoggedOut` recording blocker and a
  trial-activation paywall ahead of the plan policy; startup split into a server gate and a capture
  gate; a paywall onboarding step that calls `stop_capture()` directly; mandatory onboarding login.
- **New 24-hour history walls for free/unattributed users**, enforced independently in: the native
  history policy (`recording.rs` `refresh_history_access_policy`), the timeline hook
  (`lib/hooks/use-timeline-cache.tsx`), the Activity view (`components/activity-ledger.tsx`), and
  native Activity read/generation (`src-tauri/src/activity_history.rs`). Engine routes enforce
  whatever policy is injected, so they follow the native policy.
- **New senders:** automatic remote support-log polling (`remote_support_logs.rs`), pipe-store
  install-count POSTs, Workflows trajectory uploads, and the `screenpipe-mcp@latest` process
  (separate npm package with its own Sentry client) launched or configured from six places.
- **Fresh installs seeded `localRetentionEnabled=true`.**
- **Bundle identity** changed to `screenpi.pe` in `acd070441`; the fork pins `screenpi.pe.dev`.

### The fork decisions now in force

| Area | Fork behaviour | Seam |
|---|---|---|
| Recording gate | consumer builds always allowed; enterprise unchanged | wrapper around the unchanged pure policy in `recording.rs` |
| UI wall / onboarding | never walls or stops capture signed-out; login skippable; consumer trial paywall inert | `app-entitlement-gate.tsx`, `login-gate.tsx`, `onboarding-auth-restore.ts`, `commands.rs` transition executor, `store.rs` |
| History | unrestricted for consumers everywhere listed above | shared `history_access` policy |
| Telemetry | PostHog, Sentry, attribution, resource monitor, support logs, install counts, trajectory uploads all hard-off; every MCP launch/config forces `SCREENPIPE_DISABLE_TELEMETRY=1` and `SCREENPIPE_MCP_SENTRY_DISABLED=1` | per sender |
| Retention | never auto-starts at launch (owner decision 2026-09-24); omitted `enabled` keeps state; UI shows runtime state; cutoff changes send only `retention_days` | early return in `sync.rs` `auto_start_retention` |
| Paid features | **left as upstream** (pipe cap, hosted AI quota, team, cloud) | — |

### Sync checklist for next time

1. Grep every account-plan predicate (`is_free_or_unattributed_user`, `isFreeOrUnattributedUser`,
   `set_last_24_hours`, `history_access`) and classify each call: local-history restriction (open
   it), paid feature (leave), or unrelated. Five review rounds were spent finding these one by one.
2. Grep every `screenpipe-mcp` launcher and config writer (Rust and TS) for the two env opt-outs.
3. Grep for new outbound `fetch(` / `reqwest` / `/api/` senders.
4. Confirm `tauri.conf.json` is still `screenpi.pe.dev` before building.
5. Build a release tag, not the `main` tip.

### Upstream bug found and fixed in the fork

The storage conversion (optional, prompted at first launch) failed deterministically with
`unterminated string` on databases whose first frame's text begins with NUL bytes. Fixed in
`32f2fe070` (`crates/screenpipe-db/src/storage/lifecycle.rs`); worth filing upstream.

### Known upstream test failures (not caused by the fork)

`store::tests::new_store_marks_remote_control_preferences_as_inherited`,
`health::tests::overlay_failure_details_are_brief_specific_and_privacy_safe`,
`skills::tests::validate_repo_rejects_junk`; `disk_usage` and `updates::await_safe_restart` tests
are flaky under parallel execution and pass serially.
