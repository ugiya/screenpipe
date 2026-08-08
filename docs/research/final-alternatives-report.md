ary": "Deep multi-angle research into local-first screen-recall alternatives: discovery across ecosystems, per-candidate repo health + social reviews + security, then adversarial completeness",
  "agentCount": 19,
  "logs": [
    "discovery: 82 raw, 74 unique candidates",
    "deep-diving 10 OSI-licensed candidates; 64 others kept for the record",
    "evidence gathered on 10 candidates"
  ],
  "result": {
    "candidateCount": 74,
    "deepDived": 10,
    "report": "# Replacing screenpipe: a verified evaluation for a privacy-conscious macOS developer

**Date of research: 2026-08-08.** Every "last release" figure below is a release date, not a push date. Claims are marked DOCUMENTED FACT (read from source, API, or vendor doc) or INFERENCE (my reasoning).

---

## 1. Bottom line

**Split the problem in two and stop looking for a single replacement.** Your 22 GB archive is already a self-contained, FTS5-indexed SQLite file that any `sqlite3` binary can search in milliseconds without screenpipe, so preserving it requires no migration and no new tool â freeze it read-only on an encrypted volume and you are done. For *going forward*, the honest ranking is: (1) keep running your own already-patched screenpipe fork, which is the only option that preserves accessibility-tree capture and audio transcription and costs zero migration; (2) **Pensieve** (Apache-2.0) if you want a genuinely OSI-licensed, no-account tool and can live without audio; (3) roll your own from macOS-native parts, which is ~300 lines and beats every candidate on licence, telemetry, and control.

**Single strongest argument against my own recommendation:** the archive you are protecting is a plaintext, pre-OCR'd, pre-indexed record of everything you have seen â including credentials, client material, recruiter conversations, and third parties on video calls â sitting in a non-TCC-protected dotfolder readable by any process running as you, in the year that macOS infostealers began specifically targeting developer credentials through AI tooling ([Sophos on AMOS](https://www.sophos.com/en-us/blog/why-amos-matters-the-macos-malware-stealing-data-at-scale), [Trend Micro, Feb 2026: 800+ malicious skills distributing AMOS](https://www.trendmicro.com/en_us/research/26/b/openclaw-skills-used-to-distribute-atomic-macos-stealer.html)). Retention limits are a security control. Screenpipe's coercive 7-day deletion was implemented indefensibly, but the *posture* it enforced was the safer one. If you keep this archive forever, you are accepting a permanently growing liability in exchange for retrieval value that decays within days â and the best public evidence on this product category says users abandoned it precisely because retrieval "always walked the line of the accuracy needed for reliable use" ([Schreiber on Rewind, 2024](https://andrewschreiber.substack.com/p/an-early-adopters-thoughts-on-rewindais)).

---

## 1a. Read this before touching anything

**DOCUMENTED FACT â verified in your working tree today.** Your fork at `/Users/uri/projects/ai/evaulate/screenpipe` (branch `update/v2.5.194-uri`, HEAD `2fab59a10 fix(fork): bypass the unknown-plan consumer gate too`) has *already* neutralised both hazards. `apps/screenpipe-app-tauri/src-tauri/src/sync.rs` now opens with:

```rust
const FORK_DISABLE_AUTO_RETENTION: bool = true;
if FORK_DISABLE_AUTO_RETENTION {
    tracing::info!("fork-local: automatic retention disabled; nothing will be deleted");
    return;
}
```

Immediately below it, the *upstream* logic is still visible and still does exactly what you feared: `let is_free_plan = local_plan_policy == LocalPlanPolicy::VerifiedFree;` â `let enabled = is_free_plan || â¦` â `let days = if is_free_plan { 7 }`. The free-plan branch **short-circuits the user's own `localRetentionEnabled` setting**, and `RetentionMode::All` removes frames, audio transcriptions and UI events plus the media under `~/.screenpipe/data/`.

**The operational consequence, which no listicle will tell you:** on an *official* prebuilt screenpipe 2.5.x binary, the single most destructive action available to you right now is *signing in so you can export your data*. Signed out, the plan is `Unknown`, recording is denied and **retention never runs â nothing is deleted**. Signed in on free, the 7-day purge activates. Do not install or launch an official 2.5.x build on the machine holding this archive. Your patched build is safe; the shipped one is not.

**Your archive, measured today, not estimated:**

| Metric | Value |
|---|---|
| `~/.screenpipe/db.sqlite` | 23,673,200,640 bytes (22 GiB) |
| `~/.screenpipe/data/` | 11 GB (7,210 entries) |
| `frames` | 141,089 |
| `elements` (accessibility tree) | 3,349,826 |
| `audio_transcriptions` | 3,648 |
| Span | 2026-04-11 â 2026-08-08 (~4 months) |
| Growth rate (INFERENCE, from the above) | ~5.5 GB/month DB + ~2.8 GB/month media |

FTS5 indexes are materialised *inside the file* (`frames_fts`, `ocr_text`, `audio_transcriptions_fts`, `ui_events_fts`, `elements_fts`). A read-only immutable query against it returns thousands of hits in single-digit milliseconds with no screenpipe process running and no vendor code involved. **You are not locked in and never were.**

---

## 2. Ranked shortlist

### The table

| # | Option | Licence | Last **release** | Bus factor | Account | Telemetry | LLM integration | Real-world sentiment | Verdict |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **Your patched screenpipe fork (v2.5.194)** | Screenpipe Commercial â **source-available, fails OSI gate**; Â§2 permits personal non-commercial use | upstream `app-v2.5.176` 2026-08-04 | 1 (you) | patched out | patched out | best-in-class: MCP, HTTP API, a11y tree, Whisper | n/a | **Keep as the incumbent.** Zero migration, only option with a11y + audio |
| 2 | **Pensieve** (`arkohut/pensieve`) | Apache-2.0 â | **v0.37.0, 2026-06-19** | **1** â arkohut 913 commits, next human 9 | none | none (see caveat) | HTTP API + Claude Code agent skill; **no MCP** | one HN thread (129 pts, Nov 2024); 0 AlternativeTo reviews | **Conditional.** Best OSI option. Patch the CORS bug first. **No audio at all** |
| 3 | **Roll your own** (screencapture + Vision + FTS5 + launchd) | your code / OS APIs | n/a | you | none | none | you write ~80-line MCP server | n/a | **Strongest on requirements.** ~300 lines, one focused day |
| 4 | **Dayflow** | MIT â | v2.0.3, 2026-07-17 | 1 (99.3%) | none | **PostHog + Sentry, `// Default ON per product decision`** | Ollama/LM Studio inbound only | 6,830â, HN 480 pts | **Reject.** Wrong category (AI journal, no OCR index) + the exact telemetry stack you're fleeing |
| 5 | **TimeScroll** | GPL-3.0 â | v1.0.1, **2026-03-25** | 1 | none | none (full outbound audit: Sparkle, Ollama, GitHub only) | **real first-party MCP server** | none found | **Watch.** Only polished native app + MCP; ~4.5 months dormant; retention defaults 30d (user-editable) |
| 6 | **activity-frames + nocta-recorder** | MIT â | v0.2.2, 2026-07-28 | 1 | none (grep-verified) | none (grep-verified) | **MCP, 6 tools** | 551â / 1â | **Watch.** Closest architectural match (a11y primary, OCR fallback, whisper.cpp). OSS layer under a commercial product â same structure that just burned you |
| 7 | **mnema** | AGPL-3.0 â | v0.1.17, 2026-07-24 | **1** (1,525/1,526 commits) | **no account, but mandatory online activation**; capture pauses after 7 offline days | none | MCP + Claude plugin + RAG | **1 star; zero reviews anywhere** | **Reject.** Licence churn MITâBUSLâAGPL in 12 days; server-gated |
| 8 | **retrace** | MIT â | **v0.8.7, 2026-04-03** (GitHub Releases empty; via Sparkle appcast) | 1, already exercised | none | none | **none** | 20 Homebrew installs/90d; zero HN/AlternativeTo presence | **Reject.** README claims encryption it doesn't do; export is a "Coming Soon" card; 50-70 GB/month |
| 9 | **OpenReLife** | GPL-2.0 â | v1.1.0, 2026-05-06 | 1 | none | none | none | zero reviews; 184 total downloads | **Reject.** Search silently drops history past ~600k frames â the exact 22 GB case |
| 10 | **hyprmnesia** | AGPL-3.0 â | v0.6.1, 2026-06-02 (**v0.7.0 blocked by a failed Windows CI job**) | 1 | none | none | **MCP-first, 6 tools** | **0 stars, 0 forks** | **Reject.** Release pipeline broken 7 weeks; macOS marked "(TODO)" |
| 11 | **backscroll** | MIT â | **never** | 1 | none | none | hand-rolled MCP | zero | **Reject.** Two commits, both on one day |
| 12 | **shadow** | MIT â | v1.0.0, **2026-03-16** | 1, dead | none | none | "MCP" is a repo *topic* and a wanted-contribution â **not implemented** | HN Show: **2 points, 0 comments** | **Reject.** Hardcodes 7-day video deletion with no config â the same failure you're leaving |
| 13 | **OpenRecall** | AGPL-3.0 â | **never** (0 tags) | 1 | none | none | none | 2,914â but 25 watchers | **Reject. Main is broken:** duplicate function definition â `NameError` kills capture on first changed frame |
| 14 | **rem** | MIT â | v0.1.14, **2024-05-30** | 1, abandoned | none | **verifiably zero â no networking code at all** | clipboard copy of last 15 frames | HN 555 pts (Jan 2024) | **Reject as software; study as blueprint.** Uses `CGDisplayCreateImage`, obsoleted in macOS 15 |
| â | ActivityWatch | MPL-2.0 | v0.13.2, 2024-10-05 (tag) | healthy, 18.5kâ | none | none | 3rd-party MCP bridges | strong | **Complement only.** Window titles/URLs, no screen text |
| â | Meetily / Anarlog | MIT | 2026-06-05 / 2026-08-06 | small | none / **unverified** | unverified | Ollama | 28kâ / 9kâ | **Complement.** Covers the audio half Pensieve lacks |
| â | Memento Capture, Littlebird, Pieces, Highlight, LUCI | proprietary | â | â | varies | **Pieces: opt-out telemetry + 9-month free-tier cap; Highlight: local only on free tier; Littlebird: cloud-stored** | â | â | **All fail the OSI gate.** Littlebird's own pricing page contradicts every review site claiming "stays on your Mac" |

### Why the top three, criticisms first

**#1 â Your own fork.** *It fails your OSI hard requirement.* The current tree is v2.5.194, which is under the Screenpipe Commercial License; `LICENSE.md` Â§2 does grant free "personal, non-commercial use" including modification, and Â§3 confirms this licence governs binaries *you* build (official prebuilts are governed by the ToS instead) â so running it is permitted, but it is source-available, not open source. If you ever use this archive to support paid work, Â§4 requires a commercial licence. The alternative "pin at the last MIT commit" is worse than it sounds: the MIT boundary is commit [`81e412ff5`](https://github.com/screenpipe/screenpipe/commit/81e412ff5) (2026-06-10), and pinning there means (a) telemetry is **on by default** in that build (`disable_telemetry: bool` defaults to `false`, PostHog key hardcoded), and (b) you deliberately adopt a known SQLite WAL-reset corruption bug, fixed only afterwards in [`632e75660`](https://github.com/screenpipe/screenpipe/commit/632e75660) (2026-07-16) â a fix that required vendoring 266k lines of SQLite and is not cheaply backportable â plus you forgo the low-disk guard ([`416c01dbe`](https://github.com/screenpipe/screenpipe/commit/416c01dbe)) and unverified-binary-download SHA-256 checks ([`788e41247`](https://github.com/screenpipe/screenpipe/commit/788e41247)). **INFERENCE:** for a database you cannot replace, pinning to a build with a known WAL-corruption bug is the worst option on the table. Staying on your patched 2.5.194 keeps those fixes. In its favour: it is the only option that keeps 3.35 M accessibility-tree rows, audio transcription, MCP, and your existing archive in one system. No community fork exists to join â a GitHub search for `screenpipe in:name,description` returns 164 repos, every third-party one at 0â1 stars.

**#2 â Pensieve.** *Blocking defect, verified in source:* the local server sets `allow_origins=["*"]` with `allow_credentials=True` (`memos/server.py:98-103`) and `GET /api/files/{file_path:path}` resolves `Path("/") / file_path.strip("/")` with **no containment check** (`server.py:932-941`). While `memos serve` runs, any web page you visit can read your entire archive *and* any file your account can read â SSH keys, `~/.aws/credentials`. Binding to 127.0.0.1 does not help; the request originates in your browser. **CAVEAT ON CONFIDENCE:** this was reproduced by copying the route and middleware into a standalone Starlette app, *not* executed against a real Pensieve install. Because it alone demotes the top OSI candidate, treat it as high-probability but single-sourced. It is ~20 lines to patch locally. Second defect: `logfire` (Pydantic's hosted observability SaaS) is a hard runtime dependency wired into `server.py:84-96` with `send_to_logfire="if-token-present"` â no egress by default, but the instrumented spans carry filepaths and query content. Honest framing: telemetry-off-by-default, not telemetry-free code. Third: **no audio, ever** â the maintainer declined it on HN and closed feature request #91 unimplemented. Fourth: encryption was publicly promised in Nov 2024 and, 20 months on, grepping `encrypt|fernet|aes|keyring|keychain` across the package returns one comment. Fifth: default `processing_interval: 12` means only ~1 frame in 12 is OCR'd unless you reconfigure. In its favour: Apache-2.0, no account, **no retention logic anywhere in the codebase**, PyPI artifacts carry PEP 740 Sigstore attestations, and the AprâJun 2026 work was exactly right for a macOS recorder â `memos doctor`, capture heartbeat, stale detection, screen-recording-permission probes, and a fix for the watcher dying when `CGWindowListCopyWindowInfo` returns None.

**#3 â Roll your own.** Measured on this machine (M-series, macOS 26.5.1, 3456Ã2234): `screencapture -x -t jpg -D 1` = ~0.14 s/frame at 1.10 MB; Vision accurate-level OCR = 0.76â0.86 s/frame returning 75 text lines from 15 lines of Swift. System `sqlite3` is 3.51.0 with FTS5 compiled in. Ollama 0.31.1 with `nomic-embed-text` and `bge-m3` is already installed. Components that remove most of the work: [`mac-ocr`](https://github.com/privatenumber/mac-ocr) (MIT, v1.1.1 released 2026-08-04, JSON with bounding boxes), [`yap`](https://github.com/finnvoor/yap) (CC0, v1.2.1 2026-07-20, Apple SpeechAnalyzer, macOS 26 only â you qualify), [`sqlite-vec`](https://github.com/asg017/sqlite-vec) (Apache-2.0), [`AudioCap`](https://github.com/insidegui/AudioCap) (BSD-2, the reference for Core Audio process taps). The expensive missing piece is a timeline UI, not the capture-and-search core. **Negative finding worth recording:** there is no well-maintained, properly-licensed off-the-shelf MCP server for a local SQLite FTS5 DB â the official reference server is [archived](https://github.com/modelcontextprotocol/servers-archived) and the popular fork carries no licence. Write your own ~80 lines.

---

## 3. The unknown unknowns

**(a) Your archive is already the product. You need a viewer, not a migration.** Every candidate report penalised its subject for "no screenpipe importer." No importer is needed. Point [Datasette](https://github.com/simonw/datasette) (Apache-2.0, 11,367â, **0.65.3 released 2026-08-06**) at the frozen file and you get browsable, facetable search plus a JSON HTTP API plus auth in one command â the "local HTTP API / RAG substrate" strong-plus, delivered against the history you already have. Add [`sqlite-utils`](https://github.com/simonw/sqlite-utils) (4.1.1, 2026-07-12) and [`dogsheep-beta`](https://github.com/dogsheep/dogsheep-beta) for a unified personal-search index. **This entire ecosystem was absent from the candidate sweep** and it solves the half of the problem that actually matters.

**(b) You would be silently downgrading from accessibility-tree capture to pixels.** Screenpipe is a11y-first â this repo's own CLAUDE.md says "captures accessibility trees, with OCR as fallback," and you hold 3,349,826 `elements` rows. **Every** top-tier alternative (Pensieve, retrace, Dayflow, OpenReLife, backscroll) is pixels+OCR only. A11y text is exact, structured, carries element roles and focus, and costs almost no CPU. Nobody framed this as a cross-cutting regression. If you build your own, the a11y path is ~150 lines of Swift over `AXUIElementCopyAttributeValue`; [`Moly-OpenAppshots`](https://github.com/gaowanqi08141999/Moly-OpenAppshots) (MIT, 16â) is a small reference, and [`OpenAdapt`](https://github.com/OpenAdaptAI/OpenAdapt) (MIT, 1,669â, **v1.12.0 released 2026-08-05**, six contributors with 33+ commits each) is the only multi-contributor, actively-releasing screen+a11y+input recorder in the neighbourhood â but note honestly that it has pivoted to GUI automation and is *not* a recall product. Use its recorder as a component, not as a replacement.

**(c) macOS already OCRs images and indexes the text â and the limit is instructive.** Verified on this machine: `mdfind "(kMDItemTextContent == '*'cdw) && (kMDItemContentTypeTree == 'public.image'cd)"` returns 5,628 images with indexed Live Text. So `screencapture` on a launchd timer plus `mdfind` is a searchable screen history in ~10 lines. **Three hard limits:** indexing lag is hours-to-days (`mediaanalysisd` runs at low priority); `mdls -name kMDItemTextContent` on a specific PNG returns `(null)` â you can *match* text but not *read it back*, so this path can never feed an LLM; and there is no dedup, timeline, or app attribution. Verdict: macOS built-ins give ~40% of the value, not 80%.

**(d) Event-scoped capture instead of 24/7.** Meeting-only recording eliminates the battery, thermal, disk *and* blast-radius problems simultaneously while keeping where the value concentrates. [`Meetily`](https://github.com/Zackriya-Solutions/meetily) (MIT, v0.4.0 2026-06-05, 28kâ, speaker diarization + Ollama) or [`talkthrough-mcp`](https://github.com/korovin-aa97/talkthrough-mcp) (MIT, v0.2.5 2026-07-31, ffmpeg + faster-whisper + RapidOCR, zero LLM in the server) cover this. Note that [Anarlog](https://github.com/fastrepl/anarlog) â formerly Hyprnote â is a YC-backed company that just *renamed*, which is often when licence and account terms move; verify before trusting it.

**(e) The whole category is pre-PMF, so pick for data portability, not for a winner.** Ten of the fourteen candidates have a bus factor of exactly 1. Three independent generations of this product have taken user history to the grave: Atlas Recall folded in under a year (2017, [TechCrunch](https://techcrunch.com/2017/10/18/atlas-informatics-calls-it-quits-after-less-than-a-year/)); Rewind/Limitless was acquired by Meta and disabled all capture on 2025-12-19 with 14 days' notice â and in **Israel**, along with the EU, UK, Brazil, China, South Korea and Turkey, accounts and all associated data were **permanently deleted** with no continuation path ([9to5Mac](https://9to5mac.com/2025/12/05/rewind-limitless-meta-acquisition/)); ChatGPT Atlas stops working **2026-08-09 â tomorrow** ([TNW](https://thenextweb.com/news/openai-chatgpt-atlas-browser-shutdown-superapp)). Humane's own FAQ stated all customer data including PII would be permanently deleted from its servers on the 2025-02-28 shutdown date. Bee's pre-acquisition policy promised audio was never stored; on acquisition Amazon declined to confirm that commitment would survive ([TechCrunch](https://techcrunch.com/2025/07/22/amazon-acquires-bee-the-ai-wearable-that-records-everything-you-say/)). **The lesson for your rubric: "their policy says they don't" is worth strictly less than "the code cannot, and I can read the code" â and less again than "my data is in a format that outlives the code."**

**(f) The rubric column nobody scored: does the app accept *inbound* remote configuration?** Verified in this repo: `components/desktop-remote-control.tsx:69` lets PostHog **feature flags mutate local settings and restart the recorder**. That is remote control, not telemetry. Only Pensieve's unauthenticated local API received comparable scrutiny anywhere in the sweep. Ask this question of anything you adopt.

**(g) "Stays on your machine" is necessary and nowhere near sufficient.** Microsoft did everything right on Recall after the 2024 backlash â opt-in, TPM-sealed keys, VBS enclave, just-in-time biometric decryption â and Alexander Hagenah's TotalRecall Reloaded still extracts the data from in-process memory with "no admin required, standard user, no kernel exploit." Microsoft reviewed it for a month and **closed the case**, saying it "does not represent a bypass of a security boundary" ([CSO Online](https://www.csoonline.com/article/4159643/microsofts-windows-recall-still-allows-silent-data-extraction.html), [GovInfoSecurity](https://www.govinfosecurity.com/microsoft-recall-again-spills-secrets-to-totalrecall-tool-a-31083)). Signal ships a DRM-flag hack so its window renders as a black rectangle to Recall. Ask of any candidate: what is the at-rest encryption, what reads the index, and can an unprivileged same-user process walk it?

---

## 4. What 24/7 capture actually costs

**Disk â measured, not estimated.** Your own archive: 22 GB DB + 11 GB media over ~4 months â **8.3 GB/month combined**. Pensieve's own README documents ~40â400 KB per screenshot, **~400 MB/day / ~8 GB/month** on a single 2560Ã1440 display, plus ~2.2 GB of SQLite per 100,000 screenshots â so Pensieve is roughly break-even to slightly worse, and scales linearly per monitor. retrace's README admits **~50â70 GB/month**, "4â5Ã less efficient than Rewind AI," while its homepage simultaneously advertises "~4000x Compressionâ¦ only 10-15 GB per month." Those cannot both be true; the README is the honest one, and that pattern of vendor self-contradiction is itself the finding. Dayflow: ~410 KB per 1080p JPEG Ã 360/hour â **1.2 GB per 8-hour day** (INFERENCE, arithmetic from issue #338's reporter's own figures of ~46,000 screenshots / ~19 GB).

**Battery and thermals â one honest measurement exists, and it is mnema's.** The maintainer instrumented a fanless M4 Air on macOS 26.5.2 and published it verbatim in commit "perf(ocr): pace OCR cooler": *"OCR is the whole of Mnema's CPU costâ¦ the app's total marginal draw was 1.4 W, 9.1 W recording vs 7.7 W with the queue drained, 57.5 Â°C vs 52.8 Â°C. On a fanless chassis that heat has no fan to leave by."* The same commit fixes a bug where **pausing made the app run hotter than recording** (67% OCR duty cycle paused vs 29% recording) â and that fix landed *after* v0.1.17, so the current release still has it. This is the only quantified thermal figure I found anywhere in the field. Everything else is either a marketing claim (Shadow's "under 3% CPU average," unverifiable, no CI, no user willing to corroborate) or an issue title (Dayflow #218, "High CPU usage and excessive battery drain since version 1.8," MacBook Air M4, 22 comments; Pensieve #44, sustained 100% CPU; Pensieve #76, ~3.5 GB resident with OCR disabled). Rewind's early-adopter account puts the historical figure at 20â40% battery reduction and calls the result a "toaster."

**Security â the real cost.** Data at rest is **unencrypted in every single candidate except mnema and TimeScroll**. Pensieve: plain `.webp` files plus an unencrypted SQLite DB with full OCR text, window titles and browser URLs. retrace: verified plaintext, and its own README claimed encryption until a user proved otherwise in issue #27 and the maintainer conceded â *"Apologies, you are correctâ¦ Encryption is currently in progress, not fully implemented"* â then fixed it by editing the README. Dayflow: unencrypted SQLite + plain JPEGs, plus an `llm_calls` table retaining request and response bodies. Shadow's SECURITY.md is at least candid: *"Any user with access to your home directory can read this data."* mnema's own ADR 0057 is the most honest document in the entire field, explaining that its original keychain design meant *"any process running as the user can read the key back silentlyâ¦ The encryption therefore protects an offline copyâ¦ but is worthless against software already running on the machine"* â and that self-built binaries without a paid Apple Developer ID **fall back to that weak path**.

**The mitigation nobody offered, four separate reports having raised encryption as a disqualifier:** put the archive on an encrypted APFS volume or an encrypted sparsebundle. It is free, OS-level, and neutralises the offline-copy threat for every candidate at once. It does not touch the same-user-process threat, which no product in this category solves.

**Prompt injection is a live risk class, not a hypothetical.** Anything that pipes your screen into an agent inherits it: hostile text rendered on screen becomes agent input. OpenAI's own Chronicle preview is architected as local capture â **cloud analysis** â unencrypted local markdown memories that never expire, and is geofenced out of the EU, UK and Switzerland ([Help Net Security](https://www.helpnetsecurity.com/2026/04/21/openai-chronicle-codex-screen-context-memories/)).

**What I could not verify, and it is the biggest methodological hole in this sweep:** **zero runtime network observation across all 30+ candidates.** Every "no telemetry" verdict here rests on grepping source for `posthog|sentry|mixpanel|amplitude`. That misses transitive dependencies, vendored blobs, update checks, and anything in a shipped binary that differs from the tree. For a user whose entire complaint is telemetry, not one candidate was run for an hour under `lsof`/pcap/Little Snitch. Do this yourself for whatever you adopt â it is one afternoon and it is the only evidence that actually answers your question.

Also unverified: Reddit sentiment for essentially every candidate (reddit.com is blocked to this environment's tooling â treat "no Reddit discussion found" as *not confirmed*, not as *confirmed absent*); OCR accuracy benchmarks for any candidate (none exist, from anyone); macOS 15/26 compatibility for rem, retrace, shadow and OpenRecall (nobody has publicly run them on a current OS).

---

## 5. What to do in the next week

**Day 1 â protect the asset (30 minutes, no decisions required).**
1. Do not install or launch any official screenpipe 2.5.x prebuilt binary on this machine. Your patched fork is fine; the shipped one purges on sign-in.
2. Stop the recorder, then `sqlite3 ~/.screenpipe/db.sqlite ".backup /Volumes/<encrypted>/screenpipe-2026-08-08.sqlite"` â **not** `cp`, because WAL mode means a naive copy can catch a partial state (`db.sqlite-wal` is currently 16 MB).
3. `rsync -a ~/.screenpipe/data/ /Volumes/<encrypted>/screenpipe-data/`.
4. Create the target as an encrypted APFS volume or sparsebundle (Disk Utility â 256-bit AES).
5. Verify independence from the vendor: `sqlite3 "file:/Volumes/<encrypted>/screenpipe-2026-08-08.sqlite?mode=ro&immutable=1" "select count(*) from frames_fts where frames_fts match 'anthropic'"`. If that returns a number, you own your history outright and every "no import path" objection in section 2 evaporates.

**Day 2 â make it queryable without screenpipe (1 hour).** `uv tool install datasette`, point it at the frozen copy, confirm faceted search and the JSON API work. That is your permanent archive viewer regardless of what you record with next. If you want agent access, write the ~80-line MCP server over `frames_fts` + `audio_transcriptions_fts` â the schema is yours, so the tool surface is trivial and exactly shaped to your queries.

**Days 3â7 â decide going-forward capture, and consider *not* deciding.**
- Keep your patched fork running as the default. It is the only thing that preserves a11y + audio + MCP + your existing index, and it costs nothing to keep.
- **Run the cheap experiment before you commit to anything else:** for 3â4 weeks, log every genuine "when did I see X" moment and note whether browser history, shell history, `git log`, or your notes would have answered it. If most were answerable otherwise, drop to event-scoped meeting capture and delete the always-on pipeline entirely. Nobody in this field has published evidence that 24/7 capture beats targeted sources for real recall tasks, and the one substantive user account we have says retrieval accuracy was a top-two reason people quit.
- If you want an OSI-licensed second system running in parallel on *new* data only: install Pensieve, patch `memos/server.py` (pin `allow_origins` to the app origin, drop `allow_credentials`, confine `get_file`/`get_thumbnail`/`get_video_frame` to `settings.resolved_base_dir`), unset any `LOGFIRE_*` env vars, populate the **empty-by-default** `app_blacklist` with your password manager and banking apps, and pair it with Meetily for audio. Do not migrate anything into it â there is no importer and you no longer need one.
- Adopt voluntarily what the free tier was imposing coercively: a self-chosen 30â90 day window on *new* capture, with the frozen 22 GB archive as your one permanent, encrypted, offline exception.

**Do not** adopt retrace, shadow, backscroll, hyprmnesia, OpenReLife, OpenRecall, rem, mnema, Dayflow, or Windrecorder. Each is disqualified above for a specific, verified reason.

---

## 6. Sources

**Your local evidence (verified today):** `/Users/uri/.screenpipe/db.sqlite` Â· `/Users/uri/.screenpipe/data/` Â· `apps/screenpipe-app-tauri/src-tauri/src/sync.rs` Â· `LICENSE.md` Â· `docs/local-build-account-gate-analysis.md` Â· `CLAUDE.md` Â· HEAD `2fab59a10`

**screenpipe:** [repo](https://github.com/screenpipe/screenpipe) Â· [MITâcommercial commit 81e412ff5](https://github.com/screenpipe/screenpipe/commit/81e412ff5) Â· [WAL corruption fix 632e75660](https://github.com/screenpipe/screenpipe/commit/632e75660) Â· [low-disk guard 416c01dbe](https://github.com/screenpipe/screenpipe/commit/416c01dbe) Â· [disk-fill stop 8fc3d770e](https://github.com/screenpipe/screenpipe/commit/8fc3d770e) Â· [SHA-256 for downloads 788e41247](https://github.com/screenpipe/screenpipe/commit/788e41247) Â· [PII token redaction 793271027](https://github.com/screenpipe/screenpipe/commit/793271027) Â· [CPU governor 23a8883e8](https://github.com/screenpipe/screenpipe/commit/23a8883e8)

**Top candidates:** [Pensieve](https://github.com/arkohut/pensieve) Â· [server.py](https://github.com/arkohut/pensieve/blob/master/memos/server.py) Â· [SKILL.md](https://github.com/arkohut/pensieve/blob/master/skills/pensieve-search/SKILL.md) Â· [issue #79 (0.0.0.0, no auth)](https://github.com/arkohut/pensieve/issues/79) Â· [#91 (audio declined)](https://github.com/arkohut/pensieve/issues/91) Â· [#96 (unanswered first-launch crash)](https://github.com/arkohut/pensieve/issues/96) Â· [HN thread](https://news.ycombinator.com/item?id=42163978) Â· [PyPI provenance](https://pypi.org/integrity/memos/0.37.0/memos-0.37.0-py3-none-any.whl/provenance)

**Rejected:** [retrace](https://github.com/haseab/retrace) + [issue #27 (encryption claim retracted)](https://github.com/haseab/retrace/issues/27) + [#26 (OCR stalls on shipping build)](https://github.com/haseab/retrace/issues/26) + [appcast](https://retrace.to/appcast.xml) Â· [Dayflow](https://github.com/JerryZLiu/Dayflow) + [#239 (analytics opt-out ineffective)](https://github.com/JerryZLiu/Dayflow/issues/239) + [#338 (hourly straggler cleanup deletes fresh frames)](https://github.com/JerryZLiu/Dayflow/issues/338) Â· [mnema](https://github.com/shaik-zeeshan/mnema) + [ADR 0057](https://github.com/shaik-zeeshan/mnema/blob/main/docs/adr/0057-capture-index-key-moves-to-a-shared-keychain-access-group.md) + [licensing CONTEXT.md](https://github.com/shaik-zeeshan/mnema/blob/main/docs/licensing/CONTEXT.md) Â· [shadow](https://github.com/ghostwright/shadow) + [retention.rs](https://github.com/ghostwright/shadow/blob/main/shadow-core/src/retention.rs) + [Show HN, 2 points](https://news.ycombinator.com/item?id=47504048) Â· [OpenRecall](https://github.com/openrecall/openrecall) + [issue #107 (capture crash, unanswered 13 months)](https://github.com/openrecall/openrecall/issues/107) + [launch HN](https://news.ycombinator.com/item?id=40644040) Â· [OpenReLife](https://github.com/porech/openrelife) + [issue #12 (search evicts old history)](https://github.com/porech/openrelife/issues/12) Â· [hyprmnesia](https://github.com/hyprmnesia/hyprmnesia) + [failed release run](https://github.com/hyprmnesia/hyprmnesia/actions/runs/27622446209) Â· [backscroll](https://github.com/AnthonyDavidAdams/backscroll) Â· [rem](https://github.com/jasonjmcghee/rem) + [issue #99 (zero-byte MP4s)](https://github.com/jasonjmcghee/rem/issues/99) + [HN 555 pts](https://news.ycombinator.com/item?id=38787892) Â· [TimeScroll](https://github.com/XInTheDark/TimeScrollApp) Â· [activity-frames](https://github.com/nossa-y/activity-frames) + [nocta-recorder](https://github.com/nossa-y/nocta-recorder) Â· [Windrecorder](https://github.com/yuka-friends/Windrecorder)

**Components / roll-your-own:** [Vision RecognizeTextRequest](https://developer.apple.com/documentation/vision/recognizetextrequest) Â· [mac-ocr](https://github.com/privatenumber/mac-ocr) Â· [yap](https://github.com/finnvoor/yap) Â· [AudioCap](https://github.com/insidegui/AudioCap) Â· [sqlite-vec](https://github.com/asg017/sqlite-vec) Â· [Datasette](https://github.com/simonw/datasette) Â· [sqlite-utils](https://github.com/simonw/sqlite-utils) Â· [dogsheep-beta](https://github.com/dogsheep/dogsheep-beta) Â· [OpenAdapt](https://github.com/OpenAdaptAI/OpenAdapt) Â· [Moly-OpenAppshots](https://github.com/gaowanqi08141999/Moly-OpenAppshots) Â· [archived MCP SQLite server](https://github.com/modelcontextprotocol/servers-archived) Â· [Live Text via mdfind](https://eclecticlight.co/2025/08/13/how-to-search-spotlight-for-live-text-and-objects-in-images/)

**Complements:** [ActivityWatch](https://github.com/ActivityWatch/activitywatch) Â· [Meetily](https://github.com/Zackriya-Solutions/meetily) Â· [Anarlog (ex-Hyprnote)](https://github.com/fastrepl/anarlog) Â· [talkthrough-mcp](https://github.com/korovin-aa97/talkthrough-mcp) Â· [ArchiveBox](https://github.com/ArchiveBox/ArchiveBox) Â· [Promnesia](https://github.com/karlicoss/promnesia) Â· [Karakeep](https://github.com/karakeep-app/karakeep)

**Shutdowns, acquisitions, and security precedent:** [Rewind/Limitless â Meta](https://9to5mac.com/2025/12/05/rewind-limitless-meta-acquisition/) Â· [ChatGPT Atlas shutdown 2026-08-09](https://thenextweb.com/news/openai-chatgpt-atlas-browser-shutdown-superapp) Â· [Atlas Recall, 2017](https://techcrunch.com/2017/10/18/atlas-informatics-calls-it-quits-after-less-than-a-year/) Â· [Humane AI Pin](https://www.howtogeek.com/humane-ai-pin-shutdown/) Â· [Amazon acquires Bee](https://techcrunch.com/2025/07/22/amazon-acquires-bee-the-ai-wearable-that-records-everything-you-say/) Â· [Windows Recall, one year on](https://www.geekwire.com/2026/one-year-after-its-rocky-launch-microsofts-windows-recall-still-raises-security-red-flags/) Â· [TotalRecall Reloaded, case closed](https://www.csoonline.com/article/4159643/microsofts-windows-recall-still-allows-silent-data-extraction.html) Â· [GovInfoSecurity on the same](https://www.govinfosecurity.com/microsoft-recall-again-spills-secrets-to-totalrecall-tool-a-31083) Â· [OpenAI Chronicle architecture](https://www.helpnetsecurity.com/2026/04/21/openai-chronicle-codex-screen-context-memories/) Â· [Granola link-sharing incident (The Verge, Apr 2026)](https://docs.granola.ai/help-center/policies/privacy-policy) Â· [Pieces privacy/telemetry docs](https://docs.pieces.app/products/privacy-security-your-data) Â· [Highlight privacy policy (local is a free-tier property)](https://highlightai.com/privacy) Â· [Littlebird pricing (cloud-stored, contradicting review sites)](https://littlebird.ai/pricing) Â· [Rewind early-adopter account](https://andrewschreiber.substack.com/p/an-early-adopters-thoughts-on-rewindais) Â· [AMOS / Atomic Stealer](https://www.sophos.com/en-us/blog/why-amos-matters-the-macos-malware-stealing-data-at-scale) Â· [Picus on AMOS](https://www.picussecurity.com/resource/blog/atomic-stealer-amos-macos-threat-analysis) Â· [800+ malicious skills distributing AMOS](https://www.trendmicro.com/en_us/research/26/b/openclaw-skills-used-to-distribute-atomic-macos-stealer.html) Â· [macOS 27 ~Sept 2026](https://www.macrumors.com/roundup/macos-27/)"
  },
  "workflowProgress": [
    {
      "type": "workflow_phase",
      "index": 1,
      "title": "Discover"
    },
    {
      "type": "workflow_phase",
      "index": 2,
      "title": "Evidence"
    },
    {
      "type": "workflow_phase",
      "index": 3,
      "title": "Critique"
    },
    {
      "type": "workflow_phase",
      "index": 4,
      "title": "Synthesize"
    },
    {
      "type": "workflow_agent",
      "index": 1,
      "label": "discover:oss-en",
      "phaseIndex": 1,
      "phaseTitle": "Discover",
      "agentId": "a7ddd714a9cede765",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786198783974,
      "queuedAt": 1786198779342,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "METHOD: All repo metadata (stars, licence SPDX, last push, â¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199196041,
      "tokens": 72674,
      "toolCalls": 27,
      "durationMs": 412067,
      "resultPreview": "{"candidates":[{"name":"Pensieve (aka memos)","url":"https://github.com/arkohut/pensieve","licence":"Apache-2.0 (OSI-approved) â https://github.com/arkohut/pensieve","category":"TOP TIER â best overall fit","platform":"macOS (incl. Apple Silicon) + Windows; Linux in development. Installed via `pip install memos` / `uv tool install memos`, not a native .app","last_release":"v0.37.0 published 2026-0â¦"
    },
    {
      "type": "workflow_agent",
      "index": 2,
      "label": "discover:non-english",
      "phaseIndex": 1,
      "phaseTitle": "Discover",
      "agentId": "ade02b74fd1e4723b",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786198783761,
      "queuedAt": 1786198779342,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "**[19:42 IDT] Bottom line: the non-English sweep yields exaâ¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199220612,
      "tokens": 71162,
      "toolCalls": 35,
      "durationMs": 436851,
      "resultPreview": "{"candidates":[{"name":"Pensieve (PyPI package name: memos; Chinese media: å¼æºç\"å¥æ³ç\")","url":"https://github.com/arkohut/pensieve","licence":"Apache-2.0 (confirmed via GitHub API license.spdx_id and PyPI metadata for package `memos` 0.37.0)","category":"Direct screenpipe replacement (screen only, no audio)","platform":"macOS (incl. Apple Silicon, Metal acceleration for embeddings per README) + Wiâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 3,
      "label": "discover:adjacent-shapes",
      "phaseIndex": 1,
      "phaseTitle": "Discover",
      "agentId": "af7d42dc24e0c6452",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786198783856,
      "queuedAt": 1786198779342,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "RECOMMENDATION No single OSI-licensed project does screenpiâ¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199263383,
      "tokens": 74419,
      "toolCalls": 32,
      "durationMs": 479527,
      "resultPreview": "{"candidates":[{"name":"Pensieve (arkohut/pensieve, PyPI package \"memos\"/\"pensiev\")","url":"https://github.com/arkohut/pensieve","category":"direct-replacement (screenshot + OCR + index + search)","licence":"Apache-2.0 (OSI-approved) â confirmed via GitHub API","platform":"macOS + Windows (Linux in development). macOS setup docs cover Screen Recording permission prompt; picks GPU/Metal accelerâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 4,
      "label": "discover:assemble",
      "phaseIndex": 1,
      "phaseTitle": "Discover",
      "agentId": "aa5642eadd4c1b5eb",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786198784734,
      "queuedAt": 1786198779342,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "SWEEP: build-it-from-parts + what macOS itself provides. Evâ¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199597157,
      "tokens": 99735,
      "toolCalls": 66,
      "durationMs": 812423,
      "resultPreview": "{"candidates":[{"name":"Roll-your-own stack: screencapture(1) + Vision RecognizeTextRequest + SQLite FTS5 + launchd","url":"https://developer.apple.com/documentation/vision/recognizetextrequest","licence":"N/A (your own code; all components are OS-provided or public-domain APIs)","category":"build-it-from-parts (the assembled pipeline)","platform":"macOS 15+ (new Swift Vision API); macOS 10.15+ ifâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 5,
      "label": "discover:llm-native",
      "phaseIndex": 1,
      "phaseTitle": "Discover",
      "agentId": "aae20c5041d399186",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786198783693,
      "queuedAt": 1786198779342,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "METHOD. I did not rely on search-engine summaries for any lâ¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199533116,
      "tokens": 106083,
      "toolCalls": 47,
      "durationMs": 749423,
      "resultPreview": "{"candidates":[{"name":"activity-frames + nocta-recorder (Nocta)","url":"https://github.com/nossa-y/activity-frames","licence":"MIT (both packages; verified via GitHub API + LICENSE in nocta-recorder repo, which ships full Rust source â not a binary-only drop)","category":"Screen capture engine + deterministic agent-memory compiler + MCP server","platform":"macOS only (capture engine assets pinnedâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 6,
      "label": "discover:commercial-rubric",
      "phaseIndex": 1,
      "phaseTitle": "Discover",
      "agentId": "a73877f9b293d9ae6",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786198783903,
      "queuedAt": 1786198779342,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "SCOPE: commercial/proprietary sweep only. No open-source caâ¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199349705,
      "tokens": 77917,
      "toolCalls": 38,
      "durationMs": 565802,
      "resultPreview": "{"candidates":[{"name":"Memento Capture","url":"https://mementocapture.com/","licence":"Proprietary freeware (no licence published) â FAILS the OSI-licence hard requirement","category":"commercial â closest genuine fit, but licence-disqualified","platform":"macOS 14.0+ (signed/notarized Developer ID DMG, direct download, not Mac App Store)","account_required":"No â site states the archive and searâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 7,
      "label": "evidence:Pensieve (aka memos)",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a10725476a3ab2a9f",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199602511,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "Pensieve (aka memos / pensiev) â github.com/arkohut/pensieve",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200426191,
      "tokens": 115850,
      "toolCalls": 58,
      "durationMs": 823680,
      "resultPreview": "{"name":"Pensieve (aka memos / pensiev) â github.com/arkohut/pensieve","verdict":"CONDITIONAL CANDIDATE â passes every hard requirement on paper, but ships an unauthenticated arbitrary-file-read reachable from any website, and cannot replace screenpipe's audio.\n\nPASSES all hard requirements: fully local (data in ~/.memos, no server-side component); NO account gate anywhere in the codebase (no auâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 8,
      "label": "evidence:retrace",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a0cbfe06470d3644a",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199602149,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "retrace (haseab/retrace)",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200235487,
      "tokens": 104249,
      "toolCalls": 52,
      "durationMs": 633338,
      "resultPreview": "{"name":"retrace (haseab/retrace)","verdict":"DO NOT ADOPT as a screenpipe replacement â not today. Retrace passes every one of your HARD requirements on paper and is genuinely the cleanest licence/privacy story I checked (MIT, \"Zero cloud. Zero login.\", no telemetry SDK in the source, retentionDays default 0 = keep forever, no pricing page at all â https://retrace.to returns 404 on /pricing). Bâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 9,
      "label": "evidence:Dayflow",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "aebc30aad5b550649",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199601761,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "Dayflow",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200304997,
      "tokens": 116792,
      "toolCalls": 60,
      "durationMs": 703236,
      "resultPreview": "{"name":"Dayflow","verdict":"NOT A VIABLE SCREENPIPE REPLACEMENT for this user â recommend rejecting as a migration target. Two hard requirements fail outright and the core capability match is a category error.\n\nCATEGORY MISMATCH (the decisive finding). Dayflow is an AI work-journal, not a searchable screen-history archive. It captures 1080p JPEGs every 10s (default), sends batches to an LLM, anâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 10,
      "label": "evidence:OpenReLife",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a5b04f17d0533d30e",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199601836,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "OpenReLife (porech/openrelife)",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200172482,
      "tokens": 86373,
      "toolCalls": 44,
      "durationMs": 570646,
      "resultPreview": "{"name":"OpenReLife (porech/openrelife)","verdict":"DO NOT ADOPT as the screenpipe replacement â but keep on a watchlist. OpenReLife is real, verifiably exists, and passes every HARD requirement on paper: 100% local, no account gate of any kind, zero telemetry (verified by source grep, not just by claim), OSI-approved licence, retention defaults to -1 = keep forever with no vendor-imposed deletionâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 11,
      "label": "evidence:mnema",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a4f75d4324169b69c",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199602589,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "mnema (shaik-zeeshan/mnema)",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200226513,
      "tokens": 122881,
      "toolCalls": 58,
      "durationMs": 623923,
      "resultPreview": "{"name":"mnema (shaik-zeeshan/mnema)","verdict":"NOT RECOMMENDED as a primary screenpipe replacement for this user â though it is technically the closest thing to what they asked for, and worth watching.\n\nIt satisfies most hard requirements on paper: fully local capture, AGPL-3.0 (OSI-approved, real licence text in LICENSE), no telemetry in the app, user-controlled retention defaulting to \"neveâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 12,
      "label": "evidence:hyprmnesia",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a6713cc2e850e3a54",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199607940,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "hyprmnesia",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200076376,
      "tokens": 88899,
      "toolCalls": 34,
      "durationMs": 468436,
      "resultPreview": "{"name":"hyprmnesia","verdict":"REJECT as a screenpipe replacement for this user â though it is not a bad project, and it is worth bookmarking.\n\nOn paper it is close to a perfect requirements match: every HARD requirement passes. Fully local (`~/.hyprmnesia/`), no account gate of any kind, telemetry verifiably absent (grep for posthog/sentry/telemetry/analytics/mixpanel/amplitude/segment finds nâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 13,
      "label": "evidence:backscroll",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a56948bd811fced1c",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199602027,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "backscroll (AnthonyDavidAdams/backscroll)",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786199987549,
      "tokens": 73167,
      "toolCalls": 27,
      "durationMs": 385522,
      "resultPreview": "{"name":"backscroll (AnthonyDavidAdams/backscroll)","verdict":"NOT RECOMMENDED as a replacement for screenpipe for this user â though it fails for the opposite reason most candidates do.\n\nIt PASSES every hard requirement, cleanly and verifiably:\n- Fully local: I grepped the entire source tree for `urllib|http|socket|requests|upload|fetch` â the only matches are the Apple DTD URL inside the launâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 14,
      "label": "evidence:shadow (ghostwright)",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a13858864dc8804c2",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199601918,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "shadow (ghostwright)",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200172786,
      "tokens": 116918,
      "toolCalls": 47,
      "durationMs": 570868,
      "resultPreview": "{"name":"shadow (ghostwright)","verdict":"DO NOT ADOPT as a screenpipe replacement. Shadow is real, MIT-licensed, and its privacy claims survive a source audit â but it is an abandoned single-author code drop that FAILS the hard requirement on data retention.\n\nHard requirements scorecard:\n- Fully local: PASS (verified in source, not just claimed)\n- No mandatory account: PASS (verified â no autâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 15,
      "label": "evidence:OpenRecall",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "a234cfe5af42e21d7",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199603402,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "OpenRecall",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200187444,
      "tokens": 91840,
      "toolCalls": 45,
      "durationMs": 584042,
      "resultPreview": "{"name":"OpenRecall","verdict":"REJECT â do not adopt. OpenRecall passes the licence test (AGPL-3.0, OSI-approved, confirmed via GitHub API) and the no-account/no-telemetry/no-retention tests, but it fails on the two things that matter more: it is effectively abandoned, and the code on its only installable ref does not work.\n\nThree disqualifying findings, each independently sufficient:\n\n(1) ABâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 16,
      "label": "evidence:rem",
      "phaseIndex": 2,
      "phaseTitle": "Evidence",
      "agentId": "af3615037b82ed0b4",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786199602998,
      "queuedAt": 1786199599798,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "rem (jasonjmcghee/rem)",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200374796,
      "tokens": 101400,
      "toolCalls": 61,
      "durationMs": 771797,
      "resultPreview": "{"name":"rem (jasonjmcghee/rem)","verdict":"REJECT â do not adopt as the screenpipe replacement. rem passes every one of your HARD requirements on paper (MIT/OSI, no account, zero telemetry verified in source, no retention policy, OCR search, notarized signed binary), but it is an ABANDONED ALPHA. Last commit 2024-05-16, last release v0.1.14 on 2024-05-30 â 26 months before today. Zero commits andâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 17,
      "label": "critic:completeness",
      "phaseIndex": 3,
      "phaseTitle": "Critique",
      "agentId": "ab1b18858f8d9e5e8",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786200429968,
      "queuedAt": 1786200427929,
      "attempt": 1,
      "lastToolName": "Bash",
      "lastToolSummary": "date "+%H:%M %Z"; TZ=Asia/Jerusalem date "+%H:%M IDT"",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786200756130,
      "tokens": 78939,
      "toolCalls": 19,
      "durationMs": 326162,
      "resultPreview": "**[17:51 IDT]** COMPLETENESS CRITIQUE â six gaps, ranked. Findings below are verified in the live repo at `/Users/uri/projects/ai/evaulate/screenpipe` and via the GitHub API today (2026-08-08).

---

## GAP 1 (largest) â The incumbent was never evaluated as a candidate, and the premise was never checked

Thirty-plus alternatives were audited; screenpipe itself was audited by nobody. It is the onlyâ¦"
    },
    {
      "type": "workflow_agent",
      "index": 18,
      "label": "critic:devils-advocate",
      "phaseIndex": 3,
      "phaseTitle": "Critique",
      "agentId": "a5256ab6b9ee42848",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786200429770,
      "queuedAt": 1786200427929,
      "attempt": 1,
      "lastToolName": "Bash",
      "lastToolSummary": "grep -n -A12 "fn performance()\|fn balanced()\|fn saver()\|â¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786201002386,
      "tokens": 85491,
      "toolCalls": 47,
      "durationMs": 572616,
      "resultPreview": "**[17:58 IDT]** Evidence gathered. Here is the case against.

---

# Devil's advocate: against both recommendations, and against the framing

## 0. Two premises I checked before arguing

Both hold, but one changes the shape of the problem:

- The account gate and forced retention are real and worse than described. On `VerifiedFree`, `sync.rs:457-487` **ignores** `localRetentionEnabled` and forces â¦"
    },
    {
      "type": "workflow_agent",
      "index": 19,
      "label": "synthesis",
      "phaseIndex": 4,
      "phaseTitle": "Synthesize",
      "agentId": "aa5ec050324ade20e",
      "model": "claude-opus-5[1m]",
      "state": "done",
      "startedAt": 1786201009070,
      "queuedAt": 1786201004284,
      "attempt": 1,
      "lastToolName": "Bash",
      "lastToolSummary": "ls docs/local-build-account-gate-analysis.md && head -60 LIâ¦",
      "promptPreview": "TARGET USER: privacy-conscious developer, macOS Apple Silicon, 2026-08. Replacing screenpipe
after it added a mandatory account gate, PostHog+Sentry telemetry, and a free tier that
force-deletes recordings older than 7 days. Holds ~22GB of irreplaceable history.

HARD REQUIREMENTS: fully local; NO mandatory cloud account; telemetry absent or off by default;
OSI-approved licence (MIT/Apache/GPL/AGPâ¦",
      "lastProgressAt": 1786201232798,
      "tokens": 236630,
      "toolCalls": 4,
      "durationMs": 223728,
      "resultPreview": "# Replacing screenpipe: a verified evaluation for a privacy-conscious macOS developer

**Date of research: 2026-08-08.** Every "last release" figure below is a release date, not a push date. Claims are marked DOCUMENTED FACT (read from source, API, or vendor doc) or INFERENCE (my reasoning).

---

## 1. Bottom line

**Split the problem in two and stop looking for a single replacement.** Your 22 GBâ¦"
    }
  ],
  "totalTokens": 1921419,
  "totalToolCalls