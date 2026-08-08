---
type: session-summary
project: "screenpipe"
topic: "Screenpipe Fork Degate & Alternatives"
session_type: "development"
created: 2026-08-08
time_periods:
  - start_time: "00:30"
    end_time: "02:07"
    duration_minutes: 96
  - start_time: "11:28"
    end_time: "20:02"
    duration_minutes: 514
duration_minutes: 610
agent: "claude"
agent_version: "2.1.225"
harness: "native"
model: "claude-opus-5"
effort: "high"
---

# Screenpipe Fork Degate & Alternatives

**Date:** 2026-08-08 (Saturday)
**Time periods:** 00:30–02:07, 11:28–20:02 (~610 active min)
**Type:** personal

## Objective

How do I update my screenpipe fork past the new account gate and telemetry without losing four
months of history — and is there a better tool to move to?

## Summary

The fork at `~/projects/ai/evaulate/screenpipe` had not been updated since 2026-04-19 (app
v2.4.12) and was **3,819 commits** behind upstream. Syncing to v2.5.194 surfaced two problems.
First, the build failed: `ort-sys` downloads a prebuilt ONNX Runtime from `cdn.pyke.io`, which
returns **HTTP 403 to this network** — a Cloudflare block reproduced from curl, from the home-lab
VM, and from real Chrome. Solved by fetching the exact tarball through a one-job temporary
workflow on the fork's own GitHub Actions (US runner), verifying its SHA-256 against the hash
`ort-sys` itself expects, and seeding it into the content-addressed cache at
`~/Library/Caches/ort.pyke.io/`. The payload is a **raw LZMA2 stream**, which neither `tar` nor
`xz` will open; it needed a Python `FORMAT_RAW` decode.

Second, and more seriously, the app launched but refused to record: upstream had added an
**account gate** (`LocalPlanPolicy` is `Unknown | VerifiedFree | VerifiedPaid`; recording is
denied while `Unknown`), introduced 2026-07-20 in PR #5191. Investigating it produced the finding
that governed everything after: **signing in on the free tier is the destructive option.**
`sync.rs` overrides the user's retention setting and forces `enabled=true, days=7, mode="all"`,
which deletes frames, OCR, transcripts and UI events. Signed out, retention refuses to run at all.
The state that blocks recording is the state that protects the archive.

The fix was patched across six commits on a dedicated branch, and needed **two independent
external reviews to get right**. Codex (gpt-5.6-sol) corrected an over-broad first patch and
proved the claim "no tests break" false — measured at 10 of 39 frontend tests, narrowed to 6.
ChatGPT Pro then found three real defects: PostHog was re-enabled three lines after being
disabled, the standalone engine's Sentry was untouched, and — critically — **disabling
`auto_start_retention` was not the safety boundary**, because `enforce_free_plan_retention` is
armed from cached plan state on every background account refresh. Four defence-in-depth guards
were added, the innermost in the engine where no caller can route around it. A later Fable 5
review converged independently on the same defect.

The recording still did not start after the first install, and the cause was a wrong assumption
of mine: I had left one gate condition intact reasoning it "needs `Boolean(user)`, which never
happens when signed out." This machine holds a **cached user object** with no usable token, so the
webview called `stopScreenpipe()` 40 microseconds after the engine reached `ready`. Patched, and
verified working: frame timestamps advancing 30 seconds apart, `frame_status: ok`, and **zero
outbound sockets** at steady state.

The second half was research into whether to leave. Round one was thin; round two ran a
**19-agent workflow** (74 candidates, 10 deep-dived, 1.9M tokens) plus two Codex sol-max passes
and NotebookLM Deep Research (91 sources imported). It reframed the problem: **the archive is
already portable** — a self-contained FTS5-indexed SQLite file that `sqlite3` queries in
milliseconds with no vendor code, verified live at 10,328 full-text hits on the frozen backup. And
**the priorities were misordered**: data minimisation, security boundaries and measured retrieval
utility matter more than licence and telemetry, because every OSS candidate stores the archive in
plaintext readable by any same-user process.

## Work Log

1. Confirmed the tree: fork of `screenpipe/screenpipe`, branch `fix/vision-allowlist-reconcile-on-wake`, 3,819 commits behind. Committed and pushed local state first so nothing lived only on disk.
2. Fixed the stale fork remote (`uri-gil` → `ugiya`), fast-forwarded `main`, branched `update/v2.5.194-uri`.
3. Re-applied local keepers by grep, not by path — the chat 5xx-retry tweak had moved from `standalone-chat.tsx` to `components/chat/standalone/hooks/use-pi-foreground-events.ts`.
4. Build failed on `ort-sys` HTTP 403. Confirmed the block from curl, the home-lab VM and Chrome — Cloudflare, not screenpipe.
5. Read the expected URL + SHA-256 from `ort-sys/build/download/dist.txt`; fetched via a temp GitHub Actions workflow on the fork; verified the hash locally; decoded the raw LZMA2 with Python and seeded `~/Library/Caches/ort.pyke.io/dfbin/aarch64-apple-darwin/<sha>/`. Deleted the temp branch.
6. Built with `bun tauri build --no-sign` (not `tauri:build`, which hardcodes an upstream developer's signing identity). 48 min cold.
7. Signed in three stages: Mach-O sidecars, then `mlx.metallib` (which lives in `Contents/MacOS/` and blocks bundle signing if unsigned), then the bundle — all with `Uri Gil Code Sign`, `--timestamp=none`.
8. **Gated on DR equality before installing.** New and old Designated Requirements byte-identical → TCC grants preserved. No permission prompt across three installs.
9. Backed up with APFS clone (`cp -c`): 22 GB of databases instantly, zero disk consumed. Later extended to the 11 GB `data/` media tree after two reviewers flagged that a DB-only backup cannot restore deleted media.
10. Installed rename-and-replace (never side-by-side — two bundles sharing `screenpi.pe.dev` make LaunchServices nondeterministic).
11. Launched: `Skipping server auto-start: screenpipe account access required`. Traced the gate through `recording.rs`, `store.rs`, `sync.rs`, `main.rs` and `app-entitlement-gate.tsx`.
12. Ran two Opus agents (gate map + telemetry inventory); found the forced 7-day free-plan retention and four PostHog senders plus Sentry.
13. Patched the gate and telemetry; committed. Ran an independent Codex review against a **pristine worktree at upstream HEAD** so it wasn't anchored on my conclusions.
14. Codex corrected the patch scope; measured frontend test damage at 10/39, narrowed to 6/39 by patching only the signed-out case.
15. Sent the diff to ChatGPT Pro via Claude-in-Chrome (clipboard paste; first attempt merged a ghost suggestion and ran on the wrong model — redone cleanly).
16. Acted on its three findings: forced PostHog opt-out, disabled the standalone engine's Sentry via a single `config.analytics_enabled` authority, and unconditionally refused automatic retention.
17. Closed the **server-side** retention trap both reviewers converged on: `retention.rs` (innermost), `store.rs`, `commands.rs`, and `free-plan-retention.ts`.
18. Never constructed the PostHog client at all, after the analytics audit found `autocapture: true` ships up to 1024 chars of clicked-element text with only three surfaces carrying `ph-no-capture`.
19. Split the branches on the user's correct objection: `update/v2.5.194-uri` = upstream sync + docs only; `fork/local-degate` = the six behaviour commits.
20. Rebuilt, re-signed, reinstalled — recording still stopped. Log showed `boot phase → ready` then `stop_screenpipe` 40 µs later. Root cause: a **cached user object** made `Boolean(user)` true, firing the gate I'd left intact.
21. Patched it; verified by frame timestamps advancing (13:38:59 → 13:39:29) and `lsof -a -p <pid> -i` showing only a local listener.
22. Wrote the `/build-from-source` skill capturing the whole procedure, including the five failure modes actually hit.
23. Round-two research: 19-agent workflow, two Codex sol-max passes, NotebookLM Deep Research (91 sources). Built the notebook to 108 sources.
24. Verified the portability claim myself: `sqlite3 "file:...?mode=ro&immutable=1" "select count(*) from frames_fts where frames_fts match 'anthropic'"` → **10,328** with no screenpipe running.

## Key Identifiers

- **Branches:** `update/v2.5.194-uri` (sync + docs), `fork/local-degate` (six behaviour commits, HEAD `2fab59a10`)
- **Fork remote:** `github.com/ugiya/screenpipe` (moved from `uri-gil`)
- **Upstream gate PR:** [screenpipe#5191](https://github.com/screenpipe/screenpipe/pull/5191), merged 2026-07-20, 46 files, +5,642/−841
- **Bundle id / DR:** `screenpi.pe.dev`, cert leaf `H"e6a37ba8fe7781d4fbdb8887b0be8d714e781cf9"` ("Uri Gil Code Sign")
- **NotebookLM notebook:** `58464f72-7054-4367-b947-8d5d1493a804` — 108 sources
- **Workflow run:** `wf_d1ef59e5-5d4` — 19 agents, 74 candidates, 1.9M tokens
- **Archive measured:** 141,089 frames · 3,349,826 a11y elements · 3,648 audio transcriptions · 22 GB DB + 11 GB media · 2026-04-11 → 2026-08-08
- **Backup:** `~/.screenpipe-backup-v2.4.12-20260808/` (databases + `data/`)
- **Rollback apps:** `/Applications/screenpipe-2.4.12.bak.app`, `/Applications/screenpipe-2.5.194-pregate.bak.app`

## Deliverables

### Obsidian Note
- [Session note in Obsidian](file:///Users/uri/obsidian/work_notes/2026/08_August/week_2_02_08/08_Saturday/screenpipe-fork-degate-and-alternatives.md)

### Task Folder
- [macOS signing & TCC manual](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/macos-signing-and-tcc.md)
- [Account gate + telemetry analysis](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/local-build-account-gate-analysis.md)
- [Final alternatives report (19-agent sweep)](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/final-alternatives-report.md)
- [Rug-pull checklist](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/rugpull-checklist.md)
- [Alternatives inventory (round 1)](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/alternatives-inventory.md)
- [Threat model & cost of 24/7 capture](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/threat-and-cost.md)
- [Adversarial corrections — read before the two above](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/corrections-adversarial.md)
- [build-from-source skill](file:///Users/uri/projects/ai/evaulate/screenpipe/.claude/skills/build-from-source/SKILL.md)

### External
- Upstream gate PR: https://github.com/screenpipe/screenpipe/pull/5191
- Fork branches: https://github.com/ugiya/screenpipe/tree/fork/local-degate
- Artifact (illustrated explainer): https://claude.ai/code/artifact/c49c6503-bfd1-4b05-8939-9a9349530896

## Key Decisions

- **Patch rather than sign in.** Recording is a free-tier capability, so bypassing the gate skips a sign-in, not a payment; `LICENSE.md` §2 grants modification for personal non-commercial use. Paid features stay locked because plan policy is untouched. Signing in on free would have destroyed everything older than 7 days.
- **Patch the `has_verified_local_plan` argument, not `dev_bypass`.** `dev_bypass` is the first early return and would also neutralise the enterprise branches and flip `app_entitled_or_dev()`. The argument-level patch keeps the pure-function unit tests green.
- **Refuse retention in the engine, not just the app.** Both reviewers independently showed the app-level guard is bypassable via cached plan state on background refresh. The engine boundary is the only one no caller can route around.
- **Never construct PostHog** rather than opting out after `init()`. Autocapture ships clicked-element text and session replay is remotely switchable; both die only if the client never exists.
- **Split the branches.** Version sync and behaviour patches are separate concerns; mixing them would force taking the patches to get the update.
- **Keep 6 of 39 frontend tests failing, and convert 3 to fork invariants.** The 6 assert exactly the behaviour the fork inverts — if they ever pass, the patch has stopped working.

## Technical Context

- **Tauri is two programs in one process.** Rust owns the engine; the webview owns the UI; they enforce the same policy from *separate state*. A Rust-only patch is silently undone by the webview. This cost one full build cycle to learn.
- **TCC binds grants to the Designated Requirement** — `(bundle id, signing cert)`, not path or version. Signing with the same cert preserves every permission; adhoc signing loses them all. Entitlements and the hardened-runtime flag are **not** part of the DR, so adding them is safe.
- **The gate lives only in the Tauri app.** `recording_access_allowed` and `LocalPlanPolicy` appear zero times in `crates/`. *Caveat added by adversarial review: this shows two symbols absent at one tag, not that no retention path exists.*
- **`ort-sys` caches by tarball SHA-256** under `dfbin/<target>/<hash>/`, and skips the download entirely when that directory exists — which is what made pre-seeding safe and verifiable.
- **APFS `cp -c` clones** share blocks until divergence: 22 GB backed up instantly at zero disk cost.

## Research Findings

- **The archive was never locked in.** It is a self-contained FTS5-indexed SQLite file. Verified: 10,328 full-text hits from `sqlite3` alone, no screenpipe process. Every "no importer" objection against every candidate evaporates — the need is a *viewer* (Datasette), not a migration.
- **Local storage inside a proprietary binary protects you from nothing meaningful.** Rewind was local-first; its final update acted as a **remote kill switch**, and a third party had to publish a bypass. Limitless geoblocked Israel 2025-12-05 and deleted regional accounts 2025-12-19. *(Corrected: the supportable claim is "local is insufficient", not "protects from nothing".)*
- **Popularity is anti-correlated with fitness.** Dayflow (6,830★) ships PostHog + Sentry with the comment `// Default ON per product decision`; rem (HN 555 pts) is abandoned and uses an API obsoleted in macOS 15; OpenRecall (2,914★) has a **broken main** — a duplicate function definition raises `NameError` and kills capture. Ten of fourteen candidates have bus factor 1. ActivityWatch is the only project with a healthy bus factor, and it captures no screen text.
- **Every OSS candidate stores the archive in plaintext.** LiveRecall advertises "real-time encryption" but only XORs the images with a passphrase and leaves OCR text in plaintext, while its docs say encryption is "coming soon".
- **Priorities were misordered.** Ranked properly: data minimisation and lifecycle, security boundaries, measured retrieval utility, operational durability — *then* telemetry, *then* OSI licence. The top threat is malicious same-user code (an npm lifecycle script, a local LLM agent with filesystem access), not the vendor.
- **Retrieval value is lower than assumed.** The funnel `captured × OCR-correct × queryable × ranked-usefully` ≈ 30%. A 90-day window plus 2% curation gives ~30 GB after ten years instead of ~660 GB.
- **Telemetry long predates the gate.** PostHog has been in the app since the 2024-07-17 commit that created it; account-level identification predates the gate by ~5 months. The gate didn't add tracking — it made the identity it attaches to mandatory.
- **Methodological hole, self-reported:** zero runtime network observation across all 30+ candidates. Every "no telemetry" verdict rests on grepping source. Reddit was blocked to the tooling, so "no reviews found" ≠ "none exist".

## Current Status & Next Steps

- **Resolved:** Fork updated to v2.5.194, built, signed with the local cert, installed. Recording verified live (frames advancing, `frame_status: ok`). TCC permissions preserved — no prompt across three installs. Telemetry verified silent at runtime (only a local listener; nothing reaching PostHog or Sentry IPs). Archive intact and independently queryable. Both branches pushed. Research complete: 108-source notebook, five documents preserved in `docs/research/`.
- **Pending:** The three-week retrieval experiment — log every genuine "when did I see X" moment and note whether browser history, shell history or `git log` would have answered it. Nobody in this field has published evidence that 24/7 capture beats targeted sources.
- **Next action:** Move the frozen archive onto an encrypted APFS volume or sparsebundle; point future capture storage at it. Neutralises the offline-copy threat for any tool. Then, optionally, run one afternoon of `lsof`/Little Snitch observation on whatever is adopted — the single piece of evidence the research could not supply.
- **Blockers:** None. Audio remains disabled — a pre-existing setting, not a regression.
- **Warnings:** (1) **Never install or launch an official prebuilt screenpipe 2.5.x on this machine** — signing in to export triggers the 7-day purge. The patched build is safe; the shipped one is not. (2) Do not point an old pinned CLI at the live database; no schema compatibility matrix exists for a downgrade. (3) Re-apply the six fork patches after every upstream sync; the 6 failing frontend tests are the tripwire.

## Related Topics

- macOS code signing, Designated Requirements and TCC
- Tauri architecture (Rust backend + webview, dual policy enforcement)
- Open-source relicensing and rug-pull risk
- Local-first tooling and personal data sovereignty
- Threat modelling for personal data archives
- Multi-agent research orchestration and adversarial review
