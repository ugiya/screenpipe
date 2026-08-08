# Corrections to the earlier research — adversarial audit

An independent reasoning model was given the two research documents and asked to attack them.
Most of its criticisms are correct. This file records them so the errors do not propagate.
Read it before acting on `alternatives-inventory.md` or `rugpull-checklist.md`.

## Errors in the earlier documents

**1. "Local storage protects you from nothing" is rhetorically useful but literally wrong.**
Local storage does protect confidentiality and offline possession. What it fails to guarantee is
continuing capture, software access, portability and recoverability. The supportable claim is
**"local is insufficient"**, not "protects from nothing".

**2. The static-code proof was overclaimed.** Grepping found two symbol names
(`recording_access_allowed`, `LocalPlanPolicy`) absent from `crates/` at one tag, plus one call
site. That is evidence about **one identified mechanism**. It does not prove the absence of
generic deletion functions, configuration-driven gates, helper processes, or future schema
behaviour. The defensible conclusion is narrower: *not running the inspected Tauri path avoids
the two policy mechanisms that were identified there.*

**3. The preservation order is backwards.** The only irreplaceable asset is the ~22 GB corpus,
yet neither document opens with backup, checksums, integrity verification, schema capture, export
and a restore drill. **Tool experimentation should happen only after that asset is independently
safe.** (In this case a full clone of databases and media was taken before any fork build ran —
but the documents do not lead with it, and they should.)

**4. "Every factual claim carries a source" is visibly false.** Numerous funding, regional
deletion, contributor-count, star and commit-date claims appear without a direct citation. They
may have been verified elsewhere, but the written artifact is not independently auditable.

**5. The empirical law is unauditable.** "Roughly 19 cases" supplies no dataset, no selection
method, no definition of "durable" or "significant", and no counterexample search. The Grafana
causal explanation is post-hoc. Server-infrastructure fork history is also weak external evidence
for a macOS application dependent on TCC and native frameworks.

**6. Trademark custody is not decisive *for this user*.** It matters for community continuity and
adoption. A single developer preserving a local recorder can simply rename it. Data format,
builds, OS compatibility and security updates matter far more here. The NATS case does not
isolate trademark from CNCF governance, infrastructure and contractual commitments.

**7. Several legal claims are too absolute.** A sublicensable CLA is *not* "functionally
indistinguishable from assignment" — ownership, enforcement rights and the contributor's ability
to license elsewhere all remain different. Nonprofits are not simply prohibited from anything
that benefits a company. Dispersed copyleft makes unilateral relicensing *difficult*, not
impossible.

**8. "Structural immunity" was overstated.** "Organizational owner" may mean a GitHub org
controlled by one person. No CLA plus copyleft does not by itself establish dispersed copyright.
Establishing this needs a contributor/copyright graph and ownership-concentration evidence.

**9. Mirroring conflates availability with maintainability.** Mirroring guarantees the bytes stay
legally obtainable. It does not prove the developer can build, sign, debug, secure or port them.
"Anyone maintaining a personal fork has proven that cost is payable" is a **survival-biased non
sequitur**.

**10. The anarlog `Info.plist` conclusion exceeds its evidence.** Apple does instruct
ScreenCaptureKit apps to include `NSScreenCaptureUsageDescription`, so the omission is adverse
evidence — but checking one source plist does not prove the signed bundle cannot capture.
Build-time injection, helpers, child executables, the capture API in use, the deployment target
and an actual launch all need checking. "Proven rather than inferred" was unjustified.
https://developer.apple.com/documentation/screencapturekit

## The recommendation itself is weaker than presented

### Against the pinned MIT screenpipe CLI

> **Pinning preserves legal permission by abandoning operational maintenance. It directly
> conflicts with the maintainability requirement.**

- A macOS update changes capture, audio, TCC, signing, sleep/wake or display behaviour, and the
  user now owns the Rust and native-media stack.
- Security vulnerabilities accumulate in a process exposing an extraordinarily sensitive database.
- `SCREENPIPE_DISABLE_TELEMETRY=1` is **fail-open operational configuration, not structural
  removal**. A launchd plist, wrapper or alternate sender can omit or bypass it. No packet-capture
  evidence was gathered.
- **No evidence shows CLI `0.3.327` can safely open data last touched by app `v2.5.194`.**
  Downgrading a live 22 GB database without a schema compatibility matrix is dangerous.
- The version was named without commit SHAs, checksums, reproducible build instructions,
  migration compatibility or an update policy.
- MIT covers released source. It does not preserve binary hosting, signing certificates, model
  weights, codecs, dependencies, package registries or Apple compatibility.
- Dropping the UI may also drop health reporting, pause/exclusion controls, launch-at-login,
  timeline navigation and agent UX. Untested.

**Most damaging:** recent releases fixed a macOS SQLite WAL-index `SIGBUS` failure
(https://github.com/screenpipe/screenpipe/releases/tag/app-v2.5.172) and a macOS capture stall
(https://github.com/screenpipe/screenpipe/releases/tag/app-v2.5.176). **Pinning old code means
declining exactly the fixes a 24/7 recorder needs.**

### Against Pensieve

> **Pensieve is a screenshot sampler and indexing service, not a demonstrated screenpipe
> replacement.**

Per its own documentation: captures every 5 seconds, but **default processing indexes only 1 in
12 screenshots**, adjusting further when it cannot keep up. That yields either **minute-scale
searchable gaps** despite retaining images, or a substantially heavier compute and storage
burden under full indexing. Python upgrades can invalidate macOS screen-capture permission. Some
Python SQLite builds lack required extension support. PostgreSQL, pgvector and Docker are
recommended at scale. **No audio capture or transcription.**

Its own figures: ~400 MB per ten-hour single-display day, ~8 GB per work month, ~2.2 GB of
database per 100,000 indexed screenshots. Two years of workday use ≈ **192 GB of screenshots plus
~76 GB of index** if fully indexed, before deduplication and backups. True 24/7 approaches 1 TB.

Also: "zero telemetry" does not establish zero network use (model downloads, remote
OpenAI-compatible configuration); a local web service plus optional PostgreSQL expands the attack
and backup surface; and **the documented Docker example publishes PostgreSQL port 5432 with the
password `mysecretpassword`** — not a privacy-safe copy-paste default.

**Pensieve deserves a pilot, not the title "fallback".**

## The meta-criticism, which is the sharpest

> **It evaluated governance and then recommended a product.**

The inventory's declared criteria omit search quality, audio, agent integration, energy, storage,
reliability, migration, security, maintainability and reviews. **The conclusion does not follow
from the evaluation actually performed.**

## What survives

Narrow but real: retracting the compromised source was correct; an already-granted MIT licence
for that exact old code cannot be retroactively withdrawn; and avoiding the inspected Tauri code
path likely avoids the two identified policy mechanisms.

Those facts justify keeping a pinned build as an **emergency escape hatch**. They do not justify
calling it the best two-year solution.
