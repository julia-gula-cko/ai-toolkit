# Ways of Working — Reference

Julia's working patterns for discovery, RFC, and ARB-prep style projects. Mostly: how to use Claude Code as a partner on long-running, cross-team design work without losing context across sessions.

Source: [How to Run Project Discovery with Claude Code](https://checkout.atlassian.net/wiki/spaces/CLSE/pages/8441593921/How+to+Run+Project+Discovery+with+Claude+Code) (CLSE space).

---

## When to use this pattern

Use it when:
- Driving an RFC, ADR, or ARB submission that spans weeks and multiple sessions.
- The design touches more than one service or team.
- Diagrams + prose + decisions need to stay coherent across edits.

Skip it when:
- 1-hour ticket scoping — just open Jira.
- Work lives entirely in one repo — use that repo's `docs/`.

---

## One-time setup per project

### 1. Discovery folder outside any service repo

```shell
mkdir -p ~/Projects/<project-name>
cd ~/Projects/<project-name>
```

Keeping it neutral (not inside `service-a/` or `service-b/`) avoids biasing the framing toward one team's perspective. Discovery touches multiple repos; the folder should too.

### 2. Open it in Claude Code

```shell
claude
```

Sets the working directory and lets per-folder memory and `.claude/settings.local.json` accumulate context specific to this project.

### 3. Draft `CONTEXT.md` on day one

Most important step. **Don't backfill it later** — you lose the *why* on early decisions.

Prompt:
> "Draft a `CONTEXT.md` for this folder. The project is <one-sentence purpose>. Source of truth is <Confluence URL or Jira epic>. Use the template structure: purpose, file index, source of truth, key decisions, outstanding items, working style, working agreements, reference repos."

Aim for under 100 lines.

---

## Folder layout (worked example)

```
<project-name>/
├── CONTEXT.md                    # session restore file
├── RFC-<topic>.md                # working copy of the doc, ahead of Confluence
├── <topic>-guide.md              # any caller-facing or audience-specific cuts
├── OpenApispec                   # proposed API shape (if applicable)
├── <topic>.drawio(.png/.xml)     # main architecture diagram
├── uml/                          # PlantUML views — text-based diagrams
│   ├── 01-usecase.puml
│   ├── 02-component.puml
│   ├── 03-sequence-<flow>.puml
│   └── ...
├── screenshots/                  # pasted-in AWS console / dashboard captures
└── .claude/settings.local.json   # permission allowlist
```

Rules of thumb:
- **Local-first.** Markdown is the working surface. Confluence/Jira are publish targets.
- **Keep diagram sources AND exports.** `.drawio` for editing, `.png` for Claude to read visually and for embedding in the doc.
- **PlantUML for diagrams Claude should iterate on.** Pure text → Claude can edit in the same turn as the doc. Reserve drawio for the big "show this to the ARB" picture.

---

## `CONTEXT.md` template

```markdown
# Session Context — <Project Name>

> Paste this file into a new Claude Code session to restore context.

## What this folder is

<One paragraph. What's the project, who is it for, what's the deliverable.>

## Files in this folder

| File | Purpose |
| --- | --- |
| `RFC-...md` | Working copy. Ahead of / in sync with / behind Confluence. |
| `<diagram>.drawio` | Source for the architecture diagram. |
| ... | ... |

## Source of truth

* Confluence: <URL> (page id, space)
* Jira epic: <KEY>
* Note whether local is ahead/behind, and what hasn't been pushed.

## Key architectural decision(s)

**Option <X> chosen: <one-line summary>.**

Briefly: how it works, the alternatives considered, *why the others were rejected*.
This is the bit you don't want to re-derive in every session.

## Outstanding items

* [ ] ...
* [ ] ...

## How I work (style notes for next session)

* Terse vs verbose preference.
* Vocabulary that matters (e.g. "alerts" not "alarms").
* What you do/don't want Claude to volunteer.

## Working agreements

* Anything risky requires explicit per-turn instruction (e.g. Confluence writes).
* Anything that should NEVER happen wholesale (e.g. replacing a Confluence page body).
* Anything where Claude should ask before assuming (e.g. "check local repos before assuming SRE needs a ticket").

## Reference repos (local, useful)

* `/Users/<you>/Projects/<repo-a>/` — what it's the canonical source for.
* `/Users/<you>/Projects/<repo-b>/` — ditto.
```

The two highest-leverage sections: **Key architectural decision(s)** (rationale, not just outcome) and **Working agreements** (things Claude would otherwise do by default that you don't want).

---

## Per-session rhythm

1. Open the folder in Claude Code.
2. State the goal for *this session* in one sentence. ("Closing out Open Questions in section 7." "Drafting the failure-mode section." "Reviewing for ARB.")
3. Let Claude work in markdown. Iterate.
4. **Push to Confluence only at the end, only the changed sections.** Never wholesale-replace the page body — it silently drops embedded diagrams, macros, and inline comments.

Picking up an old project after a gap? Add: *"before suggesting anything, check that the files/repos referenced in `CONTEXT.md` still exist and that the facts in memory are still true."* Memories go stale — verify before recommending.

---

## Skills and agents worth knowing

| Skill | Use it for |
|---|---|
| `/architecture:review` | SDR / AWS Architecture Guidelines checklist over the RFC. Surfaces the questions the ARB will ask. Run once early (catches missing sections cheaply) and once near the end (catches what you hand-waved). |
| `/security:review` | Security design review. Threat model, trust boundaries, data flows. |
| `/security:pci` | PCI DSS scope review. Only if handling card data. |
| `/sre:review` | Operational readiness. Forces SLOs, monitors, runbooks. |
| `/ticket:ticket-create` | Convert converged discovery into JIRA tickets with structured confidence scoring. |
| `/ticket:ticket-review` | Score an existing ticket against 7 dimensions; identifies weak spots. |
| Atlassian MCP | Read Confluence/Jira freely. **Write only on explicit per-turn instruction.** |
| `Agent` with `subagent_type=Explore` | "Where is X defined across repos A and B?" Faster than grepping; keeps results out of main context. |

Use the architecture/security/SRE agents as **checklist generators**, not gates. They reliably catch the things you'd forget.

---

## Working agreements worth writing down

Put these in `CONTEXT.md` so they aren't renegotiated every session:

- **Atlassian writes require explicit per-turn instruction.** Even though it's in `CONTEXT.md`, restate "go ahead and push to Confluence" each time. Belt and braces.
- **Never wholesale-replace a Confluence page from markdown.** Pages contain things that don't round-trip: embedded drawio/Gliffy/Lucid, attached images, macros (info panels, status badges, ToCs), inline comments, mentions. Full-body replace silently drops them. Prefer targeted section edits.
- **Check local repos before assuming convention.** Most CKO infra is IaC PRs, not tickets to a team. If Claude proposes "raise a ticket with SRE", push back and ask it to check `~/Projects/<repo>/` first.
- **No generic best-practice padding.** Discovery docs are for an audience that knows the stack. The value is honest trade-offs, not 101-level advice.

---

## Tricks that compound

- **Screenshot-paste.** Drag AWS console / Datadog dashboard screenshots straight into a Claude turn. Faster than describing what you saw, and Claude reads the image.
- **OpenAPI as a real file, not inline.** Even at 500 lines, having it as a file means Claude can grep and cite line numbers when reviewing.
- **PlantUML for sequence diagrams.** Renders to PNG and lives as text → Claude can edit it in the same turn as the prose. If the sequence can't be drawn cleanly, the doc is hand-waving.
- **Cross-repo pointers in `CONTEXT.md`.** Naming `/Users/<you>/Projects/datadog-iac/` etc. means Claude greps existing CKO conventions instead of inventing generic AWS advice.
- **Memory for load-bearing facts only.** Save infra facts that are easy to forget and shape design ("we reuse this existing Kinesis stream — don't propose a new one"). Don't save code patterns (rederivable) or task state (that's what `CONTEXT.md` is for).

---

## Common pitfalls

- **Starting `CONTEXT.md` on day five.** Backfilling loses the rationale on early decisions. Start on day one even if it's three lines.
- **Letting Claude edit Confluence directly.** It will replace the page body and drop diagrams. Edit locally, push targeted sections.
- **Treating architecture review agents as gates.** They flag a lot; not every flag needs addressing. Use the output to find what's missing, not as pass/fail.
- **Drawing in ASCII for too long.** ASCII box diagrams are fine for the first 24 hours. Convert to PlantUML or drawio before they become load-bearing in the doc.
- **One mega-prompt at the start of every session.** Let `CONTEXT.md` do its job. State the session goal in one sentence and start.

Adapt freely — the pattern matters more than the file names.
