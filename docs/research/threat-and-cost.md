# The cost and threat model of 24/7 screen capture

Independent analysis, 2026-08-08, by a reasoning model with no web access — engineering
judgement and first principles, not citations. Where a number appears it is derived from the
user's actual corpus: ~140,000 frames and ~22 GB of database plus ~11 GB of media accumulated
over roughly four months on an Apple Silicon MacBook.

Read this alongside the tool inventory. It argues the tool choice is being made on the wrong
criteria.

## The threat model that was never stated

FileVault protects a **powered-off** machine. Once logged in, the mounted plaintext archive is
readable by anything running as that user. Ranked by realistic risk for a developer who is not
specifically targeted:

1. **Malicious or compromised same-user code.** An npm lifecycle script, a pip package, an
   editor or browser extension, or a local agent can read the entire archive recursively without
   root. **A local LLM agent with filesystem and network access is the worst case**, because
   prompt injection turns legitimate access into exfiltration.
2. **Accidental replication.** Cloud-synced folders, unencrypted backups, exports, temp files,
   debug logs and copied databases routinely create additional exposed copies.
3. **The capture tool's own update supply chain.** It already holds Screen Recording permission.
   "No telemetry today" does not constrain a compromised future update.
4. **Theft of an unlocked or sleeping laptop.** Catastrophic. Theft while fully shut down with
   FileVault is much less so.
5. **Legal discovery or subpoena.** Lower probability, but the archive converts forgotten,
   ephemeral material into cheaply searchable evidence. Encryption does not defeat compelled
   production.
6. **Border coercion.** Low baseline probability, very high impact if targeted. Encryption does
   not help against a person who can be compelled to unlock.

> "Vendor sees my data" is not automatically the largest risk. If the application is genuinely
> local and outbound traffic is blocked, ordinary product telemetry is considerably less
> threatening than every package and agent already running under the user account.

## Controls that actually help, beyond FileVault

- Keep 7–30 days of pixels online; 30–90 days of searchable text.
- **App allowlist by default, never a denylist.** Exclude password managers, banking,
  authentication, private messaging, healthcare, video calls, secret management.
- Deny the capture process outbound network access.
- Run development and untrusted package execution in a **separate OS account, container or VM**
  that cannot read the archive.
- Restrict local AI agents to explicit workspace roots — never arbitrary home-directory access.
- Keep long-term data in an independently encrypted cold archive whose key is normally
  unavailable. *Encryption adds little if its key is unlocked continuously.*
- Store in **monthly encrypted shards**: deleting a shard or its key is far more reliable than
  deleting rows from SQLite.
- Verify deletion covers FTS shadow tables, WAL files, embeddings, thumbnails, exports and
  backups. Deleted SQLite rows persist in free pages.
- Before risky travel, shut down and do not carry the archive. A travel device beats a clever
  encryption setting.

**The strongest control is retaining less. Everything else protects a target that need not
exist.**

## What to optimise for, in order

1. **Data minimisation and lifecycle** — app allowlists, pause controls, rolling retention,
   complete deletion, narrowly curated long-term storage.
2. **Security boundaries** — verified content egress, process isolation, key availability,
   updater security, backup behaviour, whether same-user tools can bulk-read the archive.
3. **Measured retrieval utility** — recall, ranking, metadata, deduplication, query speed.
4. **Operational durability** — resource use, exportability, sharding, repair, migration, and the
   ability to leave.
5. No ordinary telemetry.
6. OSI licence.

> An OSI licence helps auditability, forking and exit. It does not prove the distributed binary
> matches the source, that anyone audited it, that its dependencies are safe, or that its data
> model is secure. It should be a **tie-breaker, not the primary selection criterion**.

> His stated priorities are misordered. He is concentrating on who wrote the recorder while
> leaving a permanent plaintext dossier available to every process he chooses to run.

"No content egress" is a legitimate hard requirement and belongs under security. "No telemetry"
is a rough proxy — a content-free crash counter is less dangerous than an ostensibly
telemetry-free app whose updater and database are poorly isolated.

## Retrieval value, treated skeptically

A useful retrieval is a probability funnel:

```
captured × OCR-correct × queryable × ranked-usefully
0.8      × 0.9         × 0.6       × 0.7             ≈ 30%
```

Higher for a distinctive error message (70%+); much lower for "that thing I vaguely read months
ago". Fixed-rate capture misses transient events — a several-second popup lands around 40% —
which are exactly what people hope to recover. Change-triggered capture improves this.

Retrieval becomes genuinely valuable only with: app/window/URL/project/time context;
deduplication of repeated screens; **exact text from accessibility APIs with OCR as fallback**;
a "save the previous five minutes" control; and a tested set of real questions with measured
recall-at-10.

**Decision rule** (a rule of thumb, not a scientific threshold): if there are fewer than two
successful searches per month for material older than 90 days, permanent full capture is hard to
justify — a rolling archive preserves nearly all demonstrated value.

*This changes tool selection: prefer the tool with better metadata, deduplication, selective
retention and measurable retrieval quality — not the one promising to record everything.*

## The case for capturing less, with numbers

Architecture: **ephemeral by default, durable by exception.** Pixels 7–30 days; accessibility
text ~90 days; approved applications only; pin-an-event; move selected records into encrypted
monthly shards.

At the user's current rate:

| Strategy | Size |
|---|---|
| 30-day rolling archive | ~5.4 GB |
| 90-day rolling archive | ~16.3 GB |
| Permanent curation at 2% | +1.32 GB/year |
| **10 years, 90-day window + 2% curation** | **~30 GB, versus ~660 GB** — a 95% reduction |
| 10 years, same window + 5% curation | ~49 GB — a 92% reduction |

**Accessibility-tree capture reduces per-record storage by roughly 10–50×**, often improves exact
text quality, and avoids OCR compute entirely. What it loses: canvas, images, video, games,
remote desktops and poorly-accessible apps; exact visual arrangement; some terminal and custom
widget content; the evidentiary fidelity of "what was visibly on screen". It can also expose
*off-screen* accessibility content, so it still needs filtering.

Would he notice the loss? *Probably rarely* — unless he already retrieves old visual material
for a concrete recurring workflow. He would notice exclusions on the occasional failed search,
but not the absence of millions of never-queried duplicate frames. **At the current sampling
rate the archive is already incomplete; "capture everything" is more psychological than
literal.**

## The blunt recommendation

> Reject any replacement that cannot enforce app allowlists, bounded retention, verifiable
> local-only content handling, complete deletion, and export. Among tools meeting those
> requirements, prefer a **hybrid accessibility-text plus OCR system with a short pixel ring
> buffer**. Treat no telemetry as a security signal and OSI licensing as a useful tie-breaker —
> not as the goal.
