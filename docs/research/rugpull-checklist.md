# Rug-pull risk: how to evaluate a local-first tool before adopting it

Research findings, 2026-08-08. Compiled after screenpipe v2.5.194 added a mandatory account
gate, four PostHog telemetry senders plus Sentry, and a free tier that force-deletes
recordings older than 7 days. Every factual claim below carries a source; interpretation is
marked as such.

## The single strongest finding

**Local storage inside a proprietary binary protects you from nothing.**

Rewind was local-first — recordings lived on the user's own Mac. Its final update acted as a
**remote kill switch**, permanently disabling capture. A third party had to publish a bypass
patch for users to keep their own software working.

- Meta acquires Limitless, 2025-12-05 — https://techcrunch.com/2025/12/05/meta-acquires-ai-device-startup-limitless/
- Kill switch / app sunset — https://9to5mac.com/2025/12/05/rewind-limitless-meta-acquisition/
- The 2024 cloud pivot in the founder's own words ("from local & desktop context to cloud &
  audio context") — https://techcrunch.com/2024/04/17/a16z-backed-rewind-pivots-to-build-ai-powered-pendant-to-record-your-conversations/
- Third-party bypass — https://gist.github.com/Saik0s/71c1a542f87c27fb8760e197174dd40c

The requirement is therefore not "local-first". It is **a licence and a codebase you hold**.

Regionally relevant: Limitless geoblocked Israel on 2025-12-05 and deleted regional accounts
on 2025-12-19.

## The trigger predicts the outcome

Across roughly 19 relicensing cases, **every durable fork followed a move to a licence the OSI
will not certify. Tightening within the OSI set produced no fork at all.**

- Grafana moved Apache → **AGPL** in 2021 and saw **zero significant forks in five years**. The
  cause is instructive: Amazon Managed Grafana launched December 2020, four months *before* the
  relicence — the one party with motive and budget to fork was already inside the tent.
- KeyDB forked Redis in 2019 for *technical* reasons under BSD-3 and is **dead** (last release
  2023-10-30, last push 2024-05-29).
- Bitwarden's 2024 SDK clause caused a **reproducible build failure**, not a preference
  complaint, and reversed in **8 days**.

**Rule: a move within the OSI set is survivable. A move out of it is the event.**

## Trademark custody is the decisive structural variable

**NATS / Synadia, 2025** is the clean proof. Synadia — which had donated NATS to the CNCF —
tried to withdraw it and relicense under BUSL, demanding the nats.io domain and GitHub org
within two weeks. The CNCF charter §11(a)(i) requires trademark transfer on acceptance, and
Synadia's CEO had committed to it in writing. Outcome: trademarks transferred to the Linux
Foundation, **no fork, NATS stays Apache-2.0**.
https://www.cncf.io/blog/2025/05/01/protecting-nats-and-the-integrity-of-open-source-cncfs-commitment-to-the-community/

Synadia could have forked its own code under BUSL; nothing stopped it. **What it could not take
was the name.**

Corollaries: Jenkins' founding act was trademarking "Jenkins" and handing it to the Software
Freedom Conservancy so that "the owner of the key assets cannot use that to unfairly influence
the project" — Hudson was terminated by Eclipse in 2019. The MariaDB *Foundation* never held the
mark (MariaDB Corp does; it went to private equity in 2024). Forgejo forked *over* trademark
capture and **still has not registered its own**.

## CLA versus DCO — the actual legal difference

The Apache ICLA's grant is **word-for-word identical to HashiCorp's**, `sublicense` and all.
Only the preamble differs:

> "In return, the Foundation shall not use Your Contributions in a way that is contrary to the
> public benefit or inconsistent with its nonprofit status and bylaws in effect at the time of
> the Contribution."

That clause, plus a nonprofit being legally barred from acting to benefit one company, is the
entire mechanism behind "foundations don't relicense".

The DCO grants nothing at all — the Linux Foundation says so outright: it is "**not** a
contract… **not** itself a license. **It does not contain a grant of license rights**", and
inbound==outbound means the project "receives the same rights as anyone else (**potentially
limiting ability to relicense in the future**)".
https://bestpractices.linuxfoundation.org/ip/contribution-mechanisms-dco.html

**Practical test: grep any CLA for the word `sublicense`.** It leaves you nominally owning your
copyright while granting the company the right to license your code under any terms forever —
functionally indistinguishable from assignment.

## Copyleft plus dispersed copyright is near-immunity

VLC relicensing *part* of itself to a *more permissive* licence — driven by its own founder,
with community support — took **three years and contacting 230+ developers**; non-responders'
code was rewritten. The founder used social media, phone directories, whois, and showing up at
workplaces. A *hostile* relicence would be impossible.
https://jbkempf.com/blog/How-to-properly-relicense-a-large-open-source-project/

The inversion: **GCC (2021-06-01) and glibc (2021-07-28) both dropped FSF copyright assignment
for the DCO** within eight weeks. Concentration buys enforcement power; dispersion buys
relicensing immunity. Same property, opposite ends.

## Non-profit status is the weakest protection

Mozilla FY2024 audited: **$1.4 billion net assets**, with "approximately 86% of revenues from
customers with contracts derived from **one customer**". It acquired ad-attribution startup
Anonym (founded by two ex-Meta executives); in February 2025 introduced Terms of Use claiming a
licence to user-input information and **deleted the FAQ entry "Does Firefox sell your personal
data?"**; and killed Pocket and Fakespot. A 501(c)(3) did all of that.

**Foundation form constrains who captures the surplus. It does not constrain what the
organisation does to users.**

## Two rug-pull categories beyond relicensing

**Access-and-contract** — Red Hat / CentOS, 2023. No GPL violation: sources remained available
*to subscribers*, while the subscription agreement's terminate-on-redistribute clause plus
trademark did the work. Mike McGrath, verbatim: "we do not find value in a RHEL rebuild and we
are not under any obligation to make things easier for rebuilders; **this is our call to
make**." Nothing walked back in three years. AlmaLinux's response is the generalisable move: it
abandoned 1:1 for **ABI compatibility**, converting a legal dependency into an engineering
standard and removing Red Hat's leverage.

**Channel capture** — WordPress / WP Engine, 2024–. The community's answer was not to fork the
code (GPL already guarantees that) but to fork the **distribution channel**: the Linux
Foundation's FAIR Package Manager, announced 2025-06-06, 100+ contributors from 10+ orgs who
"approached the Linux Foundation to ensure neutral governance". Automattic cut sponsored
WordPress contribution from ~3,988 to ~45 hours/week — a **~99% reduction**. Jury trial set for
February 2027.

## Abandonment is the likelier failure for small tools

**"Not archived" is not "maintained."** KeyDB has 12,506 stars, a live-looking README and is not
archived — but its last release was 2023-10-30 and its founder wrote issue #895 ("Goodbye")
directing effort to Valkey. Check the last *release*, not the last push, and check whether the
founder has said goodbye somewhere.

KeyDB also proves **permissive licensing and corporate backing do not prevent abandonment** —
Snap owned it and it died anyway.

Funding shapes the failure mode:
- **Jellyfin's entire founding leadership resigned effective 2026-07-19**; the project leader
  cited the toll on his health. It runs on ~$26,678/year with no company. Governance held — a
  release shipped two weeks later — because the constitution had a succession process. His own
  framing: succession triggered "rather than giving one person the ability to sell it,
  completely redirect it, or decide the project needs to start generating recurring revenue."
- **Thunderbird** took 539,000+ donations in 2024 averaging $18.88, with only 17 contributions
  ≥$1,000 — the least capturable funding base in the dataset.
- **KeePassXC**, security-critical, 28K stars, runs on ~$5,485/year with **no legal entity** —
  maximally unbuyable, maximally fragile; cadence visibly slowed 2024–25.
- **Signal** filed FY2024 with an $8.6M operating deficit and negative net assets against a
  stated need of ~$50M/year.

## Cautionary note on Audacity

The 2021 Audacity telemetry PR (audacity/audacity#835 — Google Analytics + Yandex Metrica,
1,091 comments) shows **`merged: false`**. Muse Group dropped it within four days and **no
release ever contained it**. Two forks were nonetheless founded over a proposal that was
withdrawn before shipping.

The original Tenacity: 6,617 stars and **zero releases, ever** — commits went 294 (Jul 2021) → 1
(Nov 2021) → archived. **Attention exceeded engineering by roughly 100×.** Today's Tenacity is a
different codebase (Saucedacity, rebranded Nov 2022).

*Interpretation: community outrage is a poor predictor of a fork surviving. Engineering capacity
is the predictor.*

## The checklist

Apply to any candidate before adopting it:

1. **Is the licence OSI-certified today?** If not, the event has already happened.
2. **Copyleft or permissive?** Copyleft plus dispersed copyright is near-immune to hostile
   relicensing.
3. **Is there a CLA? Does it contain `sublicense`?** If yes and the owner is a company, assume
   relicensing is available to them at will.
4. **Who owns the trademark?** The decisive variable. A foundation holding the mark is the
   strongest single signal; a company holding it means a fork cannot keep the name.
5. **Single-vendor or foundation?** And if a foundation, does its charter require trademark
   transfer?
6. **Funding model.** VC-backed implies an eventual return requirement. Donation-funded implies
   fragility. Neither is disqualifying; both are predictive of *which* failure arrives.
7. **Last *release*, not last push.** And search for a maintainer goodbye.
8. **Is there a succession process in writing?** Jellyfin survived its founders leaving because
   there was one.
9. **Telemetry present at all?** A sender that exists can be re-enabled; one that was never
   built cannot.

## Applied to the shortlist

*Interpretation, based on the above:*

OpenRecall (AGPL-3.0) and Windrecorder (GPL-2.0) have the structural immunity — copyleft, no
CLA, organizational owner — and both are roughly 11 months stale. Dayflow is active and
structurally exposed (MIT under an individual). Pensieve (Apache-2.0, never relicensed, zero
telemetry) was active as of 2026-07-19.

**In this category, liveness and rug-pull resistance are anti-correlated.** The resolution:
choose on copyleft, **mirror the repository on day one**, and budget for maintaining it
yourself. Anyone already maintaining a personal fork has proven that cost is payable.
