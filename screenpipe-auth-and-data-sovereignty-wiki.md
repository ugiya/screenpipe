---
type: session-summary
project: "screenpipe"
topic: "Screenpipe Auth & Data-Sovereignty Wiki"
session_type: "infrastructure"
created: 2026-08-10
time_periods:
  - start_time: "10:37"
    end_time: "14:00"
    duration_minutes: 202
  - start_time: "21:02"
    end_time: "23:28"
    duration_minutes: 146
duration_minutes: 348
agent: "claude"
agent_version: "2.1.225"
harness: "native"
model: "claude-opus-5"
effort: "high"
---

# Screenpipe Auth & Data-Sovereignty Wiki

**Date:** 2026-08-10 (Monday)
**Time periods:** 10:37–14:00, 21:02–23:28 (~348 active min, project-scoped for the day)
**Type:** personal

## Objective

How do I store the screenpipe API key so it isn't readable by every process, and where does the
research knowledge live so it survives the repo?

## Summary

Two halves, both about durability. The first was **where yesterday's research should live**. The
answer was not tk: ThreadKeeper indexes sessions and owns validated `kb/` memory through its own
`distill → inbox → approve` pipeline, and the wiki's `AGENTS.md` rule 3 explicitly forbids the
wiki writing `Harness/kb/` at all. The wiki is the right home, and it **cites** tk rather than
being built by it. Reading the schema first surfaced a design constraint I would have got wrong by
guessing: `wiki/index.md` states subjects are *"not courses, not repos"*, so naming a subject
`screenpipe` would have been repo-shaped and needed re-homing the moment the tooling changed.
Created **`wiki/data-sovereignty/`** instead, with screenpipe as the running case study — grown to
four concept pages (`designated-requirement`, `rug-pull`, `retention-as-security-control`,
`archive-portability`), registered in both `wiki/index.md` and the Subjects section of
`AGENTS.md`, linking to canonical paths rather than copying, and citing four tk threads by name.

The second half was the **API key**. The starting state was clean — no env var, nothing in the
Keychain, no plaintext in dotfiles — which made it worth doing properly rather than expediently.
The design constraint that shaped everything: the key grants full read of the entire screen
archive, and yesterday's threat modelling ranked *malicious same-user code* as the top risk. So
the goal was that the key never lands anywhere another process can read opportunistically.
Verified the mechanics before committing to them: a Keychain round-trip with a throwaway value,
and then the discovery that `security add-generic-password` **reads the password from stdin when
`-w` is last with no value** — Apple's own man page says *"Put at end of command to be prompted
(recommended)"*. That gives an argv-free write path, which matters because command arguments are
visible via `ps` to any process running as the user.

Built **`~/.local/bin/sp-api`** around that: Keychain storage, on-demand read into a local
variable, and the bearer header passed to curl over **stdin via `curl -K -`** rather than as a
`-H` argument. Nothing is exported, so child processes don't inherit it. Two real bugs surfaced
during testing and were fixed: `die` inside a command substitution only exited the subshell, so a
missing key produced curl running anyway and a confusing second error; and the `get` subcommand
dropped trailing arguments, so `-o file` was silently ignored and a large response dumped inline.

The key itself was fetched via `bun x screenpipe@latest auth token` (the Homebrew CLI is v0.3.278
and predates the `auth` subcommand) and piped straight into `security` in a single shell
invocation, so the value never entered a transcript, argv, or shell history. `sp-api check`
confirms HTTP 200.

Finally, on the question of whether to wrap `/screenpipe-api`: **no**, and the investigation
produced a decisive reason beyond the skill's own instruction not to modify vendor skills.
`skills.rs:281` shows the desktop app fetches skills from
`raw.githubusercontent.com/screenpipe/screenpipe/main/…/skills-registry.json`, with the comment
*"Re-installing the same name refreshes it"* — and the local copies were rewritten at 21:03 today.
Any edit would be silently overwritten. Wrote a narrow user-owned **`screenpipe-auth`** skill
instead, which corrects exactly one thing (the vendor skill's `export SCREENPIPE_LOCAL_API_KEY=…`
line) and defers all endpoint knowledge to the vendor skill.

Also ran the API in anger to answer "what have I done today?" — 7h 23m of active screen time,
4,399 frames, dominated by ~3¼ hours of linear algebra (3Blue1Brown plus an iTerm2 study session).

## Work Log

1. Read `wiki/AGENTS.md` (120 lines) and `wiki/index.md` before touching anything; listed existing subjects (`dullms`, `misgeret`) and inspected `misgeret` as the no-feeds template.
2. Established that tk is the wrong tool: rule 3 forbids the wiki writing `Harness/kb/`; tk owns validated memory via its own pipeline. Wiki cites, tk stores.
3. Surfaced the naming constraint ("not courses, not repos") and offered three framings rather than deciding unilaterally; Uri chose `data-sovereignty`.
4. Confirmed the four tk thread filenames to cite: `screenpipe-main`, `screenpipe-vision-allowlist-reconcile-on-wake`, `screen_capture`, `salvage-c2-screen-context-admission`.
5. Copied five research documents out of the ephemeral session scratchpad into `docs/research/` in the repo and committed them, so `file://` links survive the session (`76e73b076`).
6. Created `wiki/data-sovereignty/` with `index.md`, `log.md`, `concepts/`, `sources/`, `synthesis/`; wrote four concept pages; registered in `wiki/index.md` and `AGENTS.md`; appended the setup log entry.
7. Carried two adversarial-audit contradictions into `concepts/rug-pull.md` as `⚠` flags rather than silently folding them in (the "~19 cases" dataset is unpublished; trademark custody governs community continuity, not an individual's ability to keep using software).
8. Checked key storage state: no `SCREENPIPE_LOCAL_API_KEY`, nothing in Keychain, no plaintext in `.zshrc`/`.zprofile`/`.zshenv`/`.bashrc`/`.profile`.
9. Found `screenpipe` v0.3.278 on PATH via Homebrew — no `auth` subcommand. Noted the `vault` subcommand (encrypt/lock data at rest) as directly relevant to the plaintext-archive gap.
10. Verified a Keychain round-trip with a throwaway value, then deleted it.
11. Read `man security`: `-w` last with no value prompts, and reads from stdin. Verified with a dummy fed as `printf '%s\n%s\n'` (it asks twice) — argv-free store confirmed working.
12. Wrote `~/.local/bin/sp-api` with `setup|check|get|raw`; `chmod +x`; `bash -n` syntax check; confirmed `~/.local/bin` is on PATH.
13. Found bug 1: `die` inside `$( )` only exits the subshell, so a missing key printed two conflicting errors. Restructured to capture the key first and abort before curl runs.
14. Fetched the token via `bun x screenpipe@latest auth token` and piped it into `security` in one invocation — never printed. `sp-api check` → HTTP 200.
15. Ran `/activity-summary`; the ISO form `2026-08-10T00:00:00` was rejected (needs `Z` or relative), so switched to a relative window.
16. Found bug 2: `sp-api get` passed only `$2` to curl, so `-o` was dropped and a large JSON response landed inline. Fixed to forward `"$@"`; verified `-o` now writes to file (1,471 bytes, HTTP 200).
17. Answered "what have I done today?" from the data, flagging two caveats: window-title attribution is unreliable for background tabs (an OpenRecall snippet was recorded under a watch-video window), and audio is disabled so nothing is transcribed.
18. On the wrap-vs-skill question: confirmed via `skills.rs:281` that skills are fetched from a remote registry and refreshed on re-install, and that the local copies were rewritten at 21:03 today — so edits would be reverted.
19. Wrote `~/.claude/skills/screenpipe-auth/SKILL.md`; validated frontmatter parses; re-ran `bash -n` and `sp-api check`.

## Key Identifiers

- **Keychain item:** service `screenpipe-local-api`, account `$USER`, login keychain
- **Helper:** `~/.local/bin/sp-api` (`setup` | `check` | `get` | `raw`)
- **Remove key:** `security delete-generic-password -s screenpipe-local-api -a "$USER"`
- **Wiki subject:** `~/obsidian/wiki/data-sovereignty/`
- **Skill:** `~/.claude/skills/screenpipe-auth/`
- **Vendor skill registry:** `https://raw.githubusercontent.com/screenpipe/screenpipe/main/apps/screenpipe-app-tauri/src-tauri/skills-registry.json` (`skills.rs:281`)
- **Commit:** `76e73b076` — research docs preserved into `docs/research/`
- **Local API:** `localhost:3030` (engine), `localhost:11435` (Tauri sidecar)
- **Homebrew CLI:** screenpipe 0.3.278 (too old for `auth token`)

## Deliverables

### Obsidian Note
- [Session note in Obsidian](file:///Users/uri/obsidian/work_notes/2026/08_August/week_3_09_15/10_Monday/screenpipe-auth-and-data-sovereignty-wiki.md)

### Wiki (subject knowledge)
- [data-sovereignty — subject index](file:///Users/uri/obsidian/wiki/data-sovereignty/index.md)
- [log](file:///Users/uri/obsidian/wiki/data-sovereignty/log.md)
- [concept — Designated Requirement](file:///Users/uri/obsidian/wiki/data-sovereignty/concepts/designated-requirement.md)
- [concept — Rug-pull](file:///Users/uri/obsidian/wiki/data-sovereignty/concepts/rug-pull.md)
- [concept — Retention as a security control](file:///Users/uri/obsidian/wiki/data-sovereignty/concepts/retention-as-security-control.md)
- [concept — Archive portability](file:///Users/uri/obsidian/wiki/data-sovereignty/concepts/archive-portability.md)
- [wiki index (registered here)](file:///Users/uri/obsidian/wiki/index.md)
- [wiki AGENTS.md (Subjects entry added)](file:///Users/uri/obsidian/wiki/AGENTS.md)

### Tooling
- [sp-api helper](file:///Users/uri/.local/bin/sp-api)
- [screenpipe-auth skill](file:///Users/uri/.claude/skills/screenpipe-auth/SKILL.md)

### Task Folder
- [Previous session — fork degate & alternatives](file:///Users/uri/projects/ai/evaulate/screenpipe/screenpipe-fork-degate-and-alternatives.md)
- [Research: final alternatives report](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/final-alternatives-report.md)
- [Research: rug-pull checklist](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/rugpull-checklist.md)
- [Research: threat model & cost](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/threat-and-cost.md)
- [Research: adversarial corrections](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/research/corrections-adversarial.md)
- [macOS signing & TCC manual](file:///Users/uri/projects/ai/evaulate/screenpipe/docs/macos-signing-and-tcc.md)

## Key Decisions

- **Wiki, not tk.** tk owns validated `kb/` memory through a gated pipeline and the wiki is forbidden from writing it. The wiki cites kb facts and tk threads by name.
- **Subject named `data-sovereignty`, not `screenpipe`.** The wiki's stated principle is subjects "not courses, not repos". Screenpipe is the case study, so Pensieve or a self-built pipeline files under the same roof later without re-homing.
- **Keychain over env var.** `export` makes every child process inherit the key; the Keychain does not. Accepted the residual limit rather than overselling it.
- **`-w` last, value on stdin.** Passing a secret as a command argument exposes it in `ps` to same-user processes. Apple recommends the prompt form; it also accepts stdin, which makes it scriptable.
- **`curl -K -` instead of `-H`.** Same argv reasoning applied to the retrieval side.
- **A narrow user-owned skill, not a wrapper.** Wrapping would duplicate ~27KB of endpoint docs that drift, and — decisively — vendor skills are refreshed from a remote registry, so edits are reverted.
- **Did not patch the remote-support-logs poller** in the fork: it cannot fire without an account token, so a patch would add rebase burden for no real gain.

## Technical Context

- **`security add-generic-password -w` with no value reads stdin** and asks twice, so `printf '%s\n%s\n' "$K" "$K" | security … -w` is an argv-free, scriptable store. Confirmed with a throwaway before use.
- **`die` inside `$( )` does not abort the caller.** Command substitution runs in a subshell; the outer pipeline continues. Capture first, test, then act.
- **`lsof -p PID -i` ORs its predicates** — it needs `-a` to AND them, otherwise it lists every process. This produced a misleading network reading earlier in this project and is worth remembering.
- **Vendor skills are remotely managed.** `skills.rs:281` fetches `skills-registry.json` from upstream `main`; re-install overwrites. `~/.claude/skills/screenpipe-api/` and `screenpipe-cli/` were rewritten at 21:03 today. This is an inbound channel: the vendor can change the instructions an agent follows, with no local action.
- **`/activity-summary` rejects bare ISO without `Z`** — `2026-08-10T00:00:00` fails; use `…Z` or a relative window like `22h ago`.
- **`screenpipe vault`** exists in the CLI (`setup|lock|unlock|status`) — first-party at-rest encryption, directly relevant to the plaintext-archive gap the research found. Locking stops recording, so it suits the frozen archive rather than the live one. Not yet evaluated.

## Research Findings

- No screenpipe **MCP tools** are exposed to this Claude Code session — the running `screenpipe-mcp` processes belong to Claude Desktop. So curl + key is the only route here.
- The MCP server itself (`packages/screenpipe-mcp`, v0.19.1, registry name `io.github.screenpipe/screenpipe-mcp`) exposes ~25 tools including `get-frame-elements` (accessibility tree), `keyword-search`, `activity-summary`, `control-recording`. Two transports: stdio and HTTP.
- **The MCP layer is not account-gated** — zero entitlement checks — consistent with the earlier finding that the gate lives only in the Tauri app. It does require the local API key.
- Today's usage from the API: **7h 23m active, 4,399 frames**. Brave 5h 32m, iTerm2 1h 47m. ~3¼ hours on linear algebra (3Blue1Brown chapters 1–2 plus an 83-minute iTerm2 session titled "Learn matrix and vector multiplication operations"). Audio: **0 segments** — last audio 2026-04-26, consistent with the disabled setting.
- **Window-title attribution is unreliable for background tabs**: an OpenRecall repo description was captured under a watch-video window title. Do not trust title-to-content pairing without checking the frame.

## Current Status & Next Steps

- **Resolved:** Wiki subject `data-sovereignty` created, registered in both required places, with four concept pages and a log entry. Research documents preserved into the repo so links survive. API key stored in the Keychain and authenticating (HTTP 200). `sp-api` helper working, two bugs fixed. `screenpipe-auth` skill written and discoverable. Recording still healthy — last frame 21:14 today, three days after the patched install.
- **Pending:** The three-week retrieval experiment from the previous session. Evaluating `screenpipe vault` for at-rest encryption of the frozen archive.
- **Next action:** Move the frozen 22 GB archive onto an encrypted APFS volume or sparsebundle — still the single highest-value unaddressed control, since Keychain protects the key but not the archive.
- **Blockers:** None.
- **Watch:** `~/.claude/skills/screenpipe-*` are refreshed from upstream `main`. Periodically `git diff` them if the instructions matter — they are not pinned to a release.

## Related Topics

- macOS Keychain and credential handling
- Secret hygiene: argv, environment inheritance, shell history
- Knowledge wiki schema (subjects vs repos; link, never copy)
- ThreadKeeper vs wiki: session index vs subject knowledge
- Vendor-managed skills and remote instruction channels
- Local-first tooling and data sovereignty
