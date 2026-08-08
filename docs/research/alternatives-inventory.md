# Alternatives to screenpipe — verified inventory

Research findings, 2026-08-08. Criteria: runs fully locally on macOS Apple Silicon; no account
required to record; telemetry absent or off by default; a real OSI licence rather than
"source-available"; no vendor-imposed deletion of the user's own data.

## Source retraction — read this first

The research initially cited `rewind.ai/what-happened-to-rewind` as a primary source. **That
citation is retracted.** whois shows the domain lapsed and was re-registered from scratch on
**2026-05-01** by an anonymous registrant (NameCheap, "Privacy service provided by Withheld for
Privacy ehf"). The site is an unaffiliated SEO/AI-tools farm. Its facts happen to match the real
record, but it must not be used as a source. Replacements are cited in the rug-pull document.

*This is worth keeping in the notebook as a reminder that a plausible-looking domain in a
dead product's name is a predictable place for content farms to appear.*

## Ranked candidates

### Structurally strongest, but stale

**OpenRecall** — AGPL-3.0. Copyleft, no CLA, organizational owner: the structural immunity
profile that resists relicensing. 2,914 stars; last push 2025-09-24, i.e. roughly 11 months
stale at time of writing.

**Windrecorder** — GPL-2.0. Same structural profile. Also roughly 11 months stale.

### Active and clean, the practical fallback

**Pensieve / Memos** (arkohut) — Apache-2.0, **never relicensed**, **zero telemetry**, last
commit 2026-07-19. The recommended independent option to test.

### Active but structurally exposed

**Dayflow** — MIT under an individual owner. The risk here is not relicensing; MIT under an
individual rarely gets that far. The risk is **abandonment**. Mitigation: mirror the repository
immediately — the MIT licence guarantees you can carry it yourself, and that guarantee *is* the
protection.

### Disqualified, with reasons worth keeping

| Tool | Why it's out |
|---|---|
| **rem** (jasonjmcghee) | MIT, individual owner, 2,481★, last push **2024-05-30**. Abandoned mid-2024. Has the exact shape of the KeyDB trap — looks alive, isn't. |
| **anarlog** (ex-Hyprnote) | Relicensed **GPL-3.0 → MIT** on 2026-04-26 (commit ab4063fd). `telemetry_consent` defaults to **`true`** — opt-out, not opt-in. Sentry includes **session replay**. Its macOS `Info.plist` declares **no `NSScreenCaptureUsageDescription`**, so the shipped app *cannot* capture the screen — audio only, proven rather than inferred. YC S25, ~$2.2M seed (medium confidence; Crunchbase blocked). |
| **Littlebird** | Closest commercial analogue to screenpipe. **$11M seed, March 2026**. Proprietary, account mandatory, AWS-stored. https://techcrunch.com/2026/03/23/littlebird-raises-11m-to-capture-context-from-your-computer-so-you-can-query-your-data/ |
| **Highlight AI** | Markets as local-first, but the policy states raw audio "is processed locally if you are on the free tier or in the cloud if you are on a paid plan." **Paying makes privacy worse.** $40M Series A. |
| **Cluely** | CEO admitted on 2026-03-05 to fabricating the $7M ARR figure given to TechCrunch. Unauditable privacy claims from a company that fakes financials are worthless. https://techcrunch.com/2026/03/05/cluely-ceo-roy-lee-admits-to-publicly-lying-about-revenue-numbers-last-year/ |
| **Amurex** | AGPL-3.0, 2,867★ — but last commit 2025-05-27. Stale. |
| **Memex** (memex.garden) | **Relicensed to PolyForm Noncommercial** — source-available, not OSI. Another instance of the pattern. |
| **Granola** | Sends audio to **four cloud subprocessors** (Deepgram, AssemblyAI, OpenAI, Anthropic). Individual users are **opted in by default** to Granola training its own models on de-identified data, which "may be retained indefinitely". Free tier's 30-day limit **hides** older notes rather than deleting them — they remain on Granola's servers. https://docs.granola.ai/help-center/policies/privacy-policy (updated 2026-07-24) |
| **Rewind / Limitless** | Acquired by Meta 2025-12-05; final update acted as a remote kill switch. **Geoblocked Israel 2025-12-05 and deleted regional accounts 2025-12-19** — never an option here regardless. |
| **Pieces for Developers** | Proprietary but the best-documented local-first commercial option; 9-month retention. Account requirement unverified. |
| **"Zapa"** | **Does not exist.** zapa.ai is a parked for-sale domain. |

## Windows Recall as a rubric, not a candidate

Worth keeping for its *design*, which is what a privacy-respecting recall tool looks like when
someone is forced to build one under scrutiny: opt-in and off by default, on-device NPU
inference, TPM-bound keys, default sensitive-content filtering, a DRM flag other applications
can set to opt themselves out, and full removability.

And its cautionary half: the April 2026 "TotalRecall Reloaded" extraction works after a single
Hello prompt with no admin rights — "It's sending decrypted content to an unprotected process" —
and Microsoft closed it as **"Not a Vulnerability"**.
https://www.thurrott.com/windows/windows-11/335005/microsoft-denies-a-new-recall-security-vulnerability-claim

## The top recommendation

A **pinned MIT-licensed screenpipe CLI** (`0.3.327`, or a fork of `app-v2.5.27`), with
**Pensieve** as the independent fallback to test now.

The reasoning is architectural rather than sentimental: the account gate and the destructive
free-plan retention live **entirely in the Tauri desktop app**, not in the recording engine.
Verified by grep against v2.5.194 — `recording_access_allowed` and `LocalPlanPolicy` appear zero
times in `crates/`, and the only code that ever arms `enforce_free_plan_retention` is in
`apps/screenpipe-app-tauri/src-tauri/src/server_core.rs`. Dropping to the CLI removes that whole
layer by not running it, rather than by patching it.

The CLI does still carry PostHog and Sentry — but its documented kill switch,
`SCREENPIPE_DISABLE_TELEMETRY=1`, actually works for a shell-launched process, whereas it is
useless for a GUI app launched from Finder, which inherits no shell environment.

Pinning to a version published under MIT matters because a licence already granted cannot be
retroactively revoked for that version.
