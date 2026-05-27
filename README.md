# ai-toolkit

A collection of generic AI skills and tools I want to share with the team. Anything I build for Claude Code (or other AI tooling) that's reusable beyond my own setup lives here.

PRs and suggestions welcome.

---

## Skills

Claude Code skills that auto-trigger based on what you're asking. Drop a skill folder into `~/.claude/skills/` and Claude Code picks it up.

### `ask-julia`

My brain dump for the CKO Credential Lifecycle platform. Auto-triggers when your question is in scope.

**Install:**

1. Copy the `skills/ask-julia` folder from this repo
2. Drop it into `~/.claude/skills/` so the path is `~/.claude/skills/ask-julia/`
   - If the folder doesn't exist: `mkdir -p ~/.claude/skills`
3. Restart Claude Code (or start a new session)

**Use it:** ask Claude naturally — no need to say "ask julia". If it doesn't trigger automatically, prefix with `ask julia ...`.
