---
type: session-summary
project: "screenpipe"
topic: "screenpipe vision allowlist fix + Rust/macOS learning"
session_type: "development"
created: 2026-04-22
---

# screenpipe vision allowlist fix + Rust/macOS learning

**Date:** 2026-04-20 → 2026-04-22 (2 engaged clusters)
**Duration:** ~3h 50min engaged (wall-clock: 2026-04-20 09:30 → 2026-04-22 12:41)
**Type:** personal

## Objective

Why was the screenpipe tray stuck on "Stopped" after monitor changes, and how do I ship a fix PR while learning enough Rust + macOS signing/TCC to maintain this subsystem?

## Summary

Traced, fixed, and landed upstream a real production bug in `screenpipe` — the tray stuck on "Stopped — click to record" after a monitor-configuration change. Root cause was a three-link structural chain: `VisionManager::start()` (`crates/screenpipe-engine/src/vision_manager/manager.rs:138`) silently returned `Ok(())` when the persisted allowlist matched zero connected monitors; `CaptureSession::start` (`apps/screenpipe-app-tauri/src-tauri/src/capture_session.rs:49`) fired that call inside a detached `tokio::spawn`, so the outer `Ok(Self)` returned before the spawn was polled once; and `recording.rs::start_capture:240` then stored `Some(dead_session)` in `RecordingState.capture`, making every subsequent tray click short-circuit on `is_some()`. The fix (D1 from the recovered ultraplan) was surgical: two Rust files changed, ~100 lines of diff, one new unit test. `VisionManager::start` now returns `Err(…)` on empty `recording_tasks` with status rolled back to `Stopped` for retry-ability, and `vision_manager.start().await` was pulled out of the `tokio::spawn` so the `?` at `recording.rs:252` can propagate the Err and keep `state.capture = None`.

Committed three times on branch `fix/vision-allowlist-reconcile-on-wake` (plan + diagnostic, fix + test, Rust primer), forked `screenpipe/screenpipe` → `uri-gil/screenpipe`, pushed, opened **[PR #3017](https://github.com/screenpipe/screenpipe/pull/3017)** — merged by **Louis Beaumont** (`louis030195`, the product's founder) at 2026-04-20 13:19 UTC, ~30 min after opening, with no review comments. Merge commit `077578a203fa3098700b6c317a89f056cb899920`. Unit test `start_with_no_allowed_monitors_returns_err` passes 4/4 alongside the pre-existing suite, `cargo check` on the tauri crate is clean, and `cargo build --release` on the engine completes in 4m 27s.

Heavy learning track layered on top of the fix per Jeremy Howard's SIWC ("scratch your own itch") pedagogy. Wrote `docs/rust-for-screenpipe.md` — a 358-line Python-programmer's Rust primer anchored to every concept in the diff: ownership / `&` vs `&mut`, `Option` / `Result` / `?`, `Arc<T>` and interior mutability (`Mutex`, `RwLock`, `DashMap`), traits (`Clone`, `Send`/`Sync`, `Future`, `Drop`), async and `tokio::spawn`, macros (`anyhow!`, `info!`, `format!`), and a lifetimes primer. Every glossary row points at a specific file:line in the PR. Also used the session for live diagnostic work against the running screenpipe instance: mapped the 4 physical monitors (1× DELL UP3216Q 5K main, 1× MacBook Retina built-in, 2× DELL U27xx QHD) against `list-monitors` / `/health` / per-monitor `frames` DB counts, surfacing three observability gaps (G1 HiDPI pixel-loss silently 4×-downsampling captures; G2 scale-slider changes reproduce the same stuck state; G3 `/health.monitors` lists enumerated displays, not actually-recording ones) — all three now documented in the plan file as follow-ups, and `Plans/diagnostic-check-what-is-recording.sh` is a workaround for G3 until a per-monitor `/health` endpoint lands.

While analyzing a `screenpipe-app` crash report Uri shared, identified a **separate** bug unrelated to the vision fix — `EXC_BAD_ACCESS` in a `clipboard-capture` thread during `objc_autoreleasePoolPop`. Classic Objective-C over-release / dangling-pointer pattern. Filed upstream as **[Issue #3023](https://github.com/screenpipe/screenpipe/issues/3023)** with exception details, the triggering stack, evidence of bundle ID + version + thread identification, and suspected candidate causes (`NSPasteboardItem` retain/release imbalance, per-thread autorelease pool missing on worker threads, Rust FFI freeing an object it doesn't own). Explicit PR-body note that it is NOT related to PR #3017 so the two investigations don't get tangled.

Session closed with a full Tauri app rebuild (5m 02s, leveraging the warm cargo cache from the engine-only build) producing `apps/screenpipe-app-tauri/src-tauri/target/release/bundle/macos/screenpipe - Development.app` and a matching `.dmg`. Discovered during install-prep that the build fell back to **adhoc signing** because `tauri.conf.json` is missing `bundle.macOS.signingIdentity` — meaning the new binary's Designated Requirement doesn't match the stored `csreq` for the old app, and TCC would re-prompt for screen-recording permission. The local cert `Uri Gil Code Sign` is in the keychain though, so re-signing with `codesign --force --deep --sign "Uri Gil Code Sign"` before install will preserve the TCC grant. That rabbit hole was worth its own learning artifact: `docs/macos-signing-and-tcc.md` — a ~550-line practitioner's manual covering `.app` bundle anatomy, `Info.plist` keys, code signing modes, Designated Requirements, entitlements, hardened runtime, notarization + stapling, Gatekeeper, the quarantine xattr, LaunchServices, TCC's SQLite tables and `(service, client, csreq)` keying, Tauri specifics, a full diagnostic-command cookbook, and a section applying every concept to our concrete screenpipe install scenario.

## Key Identifiers

- **Branch:** `fix/vision-allowlist-reconcile-on-wake`
- **PR:** [screenpipe/screenpipe#3017](https://github.com/screenpipe/screenpipe/pull/3017) — **MERGED** 2026-04-20T13:19:31Z
- **Merge commit:** `077578a203fa3098700b6c317a89f056cb899920`
- **Commits on branch:**
  - `319575d24` — docs(plan): add ultraplan + live diagnostic for the stuck-vision bug
  - `876d7baf2` — fix(vision): surface allowlist-mismatch as Err so tray can retry
  - `23fd4801b` — docs(rust): add Python-programmer's Rust primer anchored to this PR
- **Upstream issue filed:** [screenpipe/screenpipe#3023](https://github.com/screenpipe/screenpipe/issues/3023) — clipboard-capture SIGSEGV — **OPEN**
- **Fork:** [uri-gil/screenpipe](https://github.com/uri-gil/screenpipe)
- **Remotes:** `origin=screenpipe/screenpipe`, `fork=uri-gil/screenpipe`
- **Signing cert:** `Uri Gil Code Sign` (SHA1 `E6A37BA8FE7781D4FBDB8887B0BE8D714E781CF9`)
- **Bundle ID:** `screenpi.pe.dev`

## People & Contacts

- **Louis Beaumont** (`louis030195`) — screenpipe founder, merged PR #3017 within ~30 min of opening with no review comments. No direct contact during the session; merge happened while other work was in flight.

## Deliverables

### Obsidian note
- [Session note](file:///Users/uri/obsidian/work_notes/2026/04_April/week_4_19_25/22_Wednesday/screenpipe-vision-allowlist-fix-and-rust-macos-learning.md)

### Task folder (in `screenpipe` repo)
- [Session summary — this file](file:///Users/uri/projects/ai/evaulate/screenpipe/screenpipe-vision-allowlist-fix-and-rust-macos-learning.md)
- [Ultraplan — fix + SIWC learning layer + observability gaps](file:///Users/uri/projects/ai/evaulate/screenpipe/Plans/sparkling-churning-gosling-ultraplan.md)
- [Diagnostic: what is actually recording?](file:///Users/uri/projects/ai/evaulate/screenpipe/Plans/diagnostic-check-what-is-recording.sh)
- [Rust-for-screenpipe — Python-programmer's primer](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/rust-for-screenpipe.md)
- [macOS signing + TCC manual](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/macos-signing-and-tcc.md)

### Code changed (merged upstream)
- [`manager.rs` — empty-tasks guard + status rollback + unit test](file:///Users/uri/projects/ai/evaulate/screenpipe/crates/screenpipe-engine/src/vision_manager/manager.rs)
- [`capture_session.rs` — `.start().await` pulled out of detached `tokio::spawn`](file:///Users/uri/projects/ai/evaulate/screenpipe/apps/screenpipe-app-tauri/src-tauri/src/capture_session.rs)

### Built binaries (local, not yet installed)
- [Built `.app` bundle (adhoc-signed)](file:///Users/uri/projects/ai/evaulate/screenpipe/apps/screenpipe-app-tauri/src-tauri/target/release/bundle/macos/screenpipe%20-%20Development.app)
- [Built `.dmg` installer](file:///Users/uri/projects/ai/evaulate/screenpipe/apps/screenpipe-app-tauri/src-tauri/target/release/bundle/dmg/screenpipe%20-%20Development_2.4.12_aarch64.dmg)
- [Engine-only release binary](file:///Users/uri/projects/ai/evaulate/screenpipe/target/release/screenpipe)

## Key Decisions

- **D1 from the plan** (make `VisionManager::start()` fail-fast on empty `recording_tasks`) over D2 (add `vision_alive` field to `CaptureSession`) and D3 (health-driven watchdog). Smallest blast radius; most correct; leverages existing `?` in `recording.rs:252` without new state.
- **Await `vision_manager.start()` inline** instead of `.await`ing a `JoinHandle` from `tokio::spawn`. Both would work, but inline is structurally clearer: the spawn is left owning only the genuinely fire-and-forget parts (monitor watcher + shutdown handler).
- **Roll status back to `Stopped` on the Err path** so a retry isn't blocked by `VisionManager::start`'s idempotency guard at line 140–143. Without this, the first `.start()` call failing would make all subsequent tray-click retries no-op.
- **Include `docs/rust-for-screenpipe.md` in the PR** with an explicit note to the reviewer that it can be stripped if unwelcome upstream — preserving personal learning material without forcing it on the project.
- **Filed clipboard-capture crash as a separate issue** (#3023), not tacked onto #3017, so the two investigations don't get tangled in review.
- **Rename-and-replace install (option A)** over side-by-side (B) or Finder drag (C) — clean rollback, avoids LaunchServices confusion from duplicate bundle IDs. Not yet executed.
- **Re-sign post-build with `Uri Gil Code Sign`** (`codesign --force --deep --sign ...`) instead of rebuilding. Faster and preserves TCC grants.

## Technical Context

- **Architecture anchor:** the tray-click path is React (`deeplink-handler.tsx`) → Tauri IPC → `recording.rs::start_capture` → `CaptureSession::start` → `Arc<VisionManager>` → per-monitor `tokio::spawn`'d recording tasks → frames to SQLite. Fix lives at the boundary between `CaptureSession::start` and `VisionManager::start`.
- **Interior mutability is load-bearing:** `VisionManager::start(&self)` (not `&mut self`) because the fields it mutates (`status: Arc<RwLock<_>>`, `recording_tasks: Arc<DashMap<_,_>>`) lock themselves. This is what lets `Arc<VisionManager>` be shared freely across `tokio::spawn` without fighting the borrow checker.
- **Explicit `drop(status)` mid-function** at `manager.rs:147` is *not* an optimization — it's correctness. `start_monitor` inside the loop re-acquires `self.status.write().await`; holding the outer guard past that would deadlock.
- **Stable-ID format `"Display N_WIDTHxHEIGHT_X,Y"` encodes logical (HiDPI-scaled) resolution, not native pixels** — so moving your laptop from home to office *and* adjusting the "Larger Text / More Space" scale slider both invalidate the allowlist identically. G2 observability gap.
- **Per-monitor recording state is DB-only**, not exposed via `/health`. `/health.monitors` lists what ScreenCaptureKit enumerates, which is the superset of what's being captured. G3 observability gap. `Plans/diagnostic-check-what-is-recording.sh` is the workaround.
- **The Tauri adhoc-signing fallback** happens when `tauri.conf.json → bundle.macOS.signingIdentity` is missing. This is a silent fallback — the build succeeds — but TCC will treat the resulting binary as a different app than the previously-installed one with the same bundle ID.

## Research Findings

- **Jeremy Howard's SIWC (Scratch Your Own Itch + Whole-game first) mapped cleanly onto fixing a real bug in unfamiliar code.** Reading `recording.rs::start_capture` → `CaptureSession::start` → `VisionManager::start` → `is_monitor_allowed` in exactly that order before editing anything gave full context by the time the first Edit landed.
- **Rust's `?` operator and the detached-`tokio::spawn` bug class are intimately related.** Any time you `tokio::spawn { result_returning_fn().await }`, the `Result` is swallowed unless you explicitly `JoinHandle::await` it. Pull the call out, propagate with `?`, and the type system enforces correctness. This single pattern is probably 30% of async Rust bugs in production.
- **TCC keys on `(service, client, csreq)`**, where `csreq` is a serialized Designated Requirement. Rebuilding an app changes only the binary hash; re-signing with the same cert preserves the DR → preserves TCC grants. Rebuilding with adhoc (or a different cert) changes the DR → TCC treats as fresh app → re-prompts for all permissions.
- **macOS boot-to-syscall path for a privileged app is a 6-step pipeline:** quarantine xattr → Gatekeeper → LaunchServices → kernel code-sign verify + entitlement check → TCC lookup + csreq match → actual syscall. Any "permission doesn't work" diagnostic should traceroute which step denied, not ask "is TCC broken."
- **`NSScreenCaptureUsageDescription` misconfiguration is a crash, not a prompt-miss.** If the Info.plist string is missing, the first call to a TCC-protected API kills the process via a privacy violation before TCC even renders a prompt.
- **Stable IDs based on resolution are inherently brittle** across any user-facing display setting change. A truly stable ID would use IOKit serial or display UUID; that's out of scope for this PR and filed as G2 follow-up.

## Current Status & Next Steps

- **Resolved:**
  - PR #3017 merged upstream (`077578a2`), tests green, release build clean.
  - Full Tauri `.app` rebuilt locally and signature state diagnosed.
  - `rust-for-screenpipe.md` and `macos-signing-and-tcc.md` written and committed to the branch.
  - Issue #3023 filed for the unrelated clipboard-capture SIGSEGV.

- **Pending (end-to-end fix verification — the critical gap):**
  - Fix is proven at the **unit test level** (`start_with_no_allowed_monitors_returns_err` — 4/4 green) and at the **structural reasoning level** (detached-spawn → Ok-too-early → short-circuit chain is broken).
  - Fix is **NOT yet proven end-to-end against the running app** — nobody has clicked the tray on the rebuilt binary against a deliberately-bad `monitorIds` and watched the error state appear.
  - Three possible outcomes when that verification runs: (1) full success; (2) state machine correct but UI doesn't surface err nicely → follow-up PR for a toast; (3) unexpected regression → revert `077578a2`, diagnose, resubmit.

- **Next actions:**
  1. **Re-sign the built Tauri app** with `Uri Gil Code Sign` to preserve TCC grants:
     ```
     codesign --force --deep --sign "Uri Gil Code Sign" \
       "/Users/uri/projects/ai/evaulate/screenpipe/apps/screenpipe-app-tauri/src-tauri/target/release/bundle/macos/screenpipe - Development.app"
     ```
  2. **Rename-and-replace install** (option A from the install plan):
     ```
     mv /Applications/screenpipe.app /Applications/screenpipe.app.v2.4.12-prefix.bak
     cp -R "apps/.../bundle/macos/screenpipe - Development.app" /Applications/screenpipe.app
     ```
  3. **End-to-end repro + fix verification:** set `monitorIds` to `["Display 999_9999x9999_0,0"]` via the app UI or by editing `~/.screenpipe/store.bin`. Quit and relaunch. Click the tray. Expect:
     - Log line `VisionManager failed to start: no monitors matched the allowed list`
     - `state.capture` stays None → second tray click retries cleanly (no short-circuit)
     - Ideally the UI surfaces the error visibly; if not, that's a follow-up PR, not a regression.
  4. **Fix follow-up on the config:** add `"signingIdentity": "Uri Gil Code Sign"` to `tauri.conf.json → bundle.macOS` as a separate commit so all future local builds sign themselves without a post-build step. Push to fork or keep local.
  5. **File G1/G2/G3 as separate upstream issues** when bandwidth allows — each is a small tracked improvement, not urgent.
  6. **Revisit clipboard-capture Issue #3023** if a repro surfaces — currently nobody has a deterministic way to trigger it.

- **Blockers:** None. Everything on the next-actions list is a local or personal-GitHub action; no waiting on upstream review.

## Related Topics

- PR #3017 — the merged fix
- Issue #3023 — clipboard-capture SIGSEGV (separate bug)
- Rust async — `tokio::spawn`, `Arc`, `Mutex`, `RwLock`, `?`
- Tauri — `.app` bundling, `tauri.conf.json`, adhoc-signing fallback
- macOS TCC — `csreq`, Designated Requirement, `(service, client)` key
- Code signing — `codesign`, `spctl`, keychain-based dev certs
- Jeremy Howard SIWC — scratch your own itch, whole-game first
- VisionManager / CaptureSession / RecordingState — the state-machine triangle
