# meiko-superpowers

Meiko development skills plugin — conventions, hooks, and workflows for the Meiko multi-service repo (Python/Flask/SQLAlchemy backend + React/Cypress frontend). Mirrors the [superpowers](https://github.com/obra/superpowers) format so it works across Claude Code, Codex, Gemini CLI, and other agent harnesses.

## What's inside

- **Skills** (principles, not rules) — entry skill plus future skills for backend conventions, frontend conventions, testing, code review, and a self-refinement loop
- **Hooks** — block `git push` until you approve, auto-format Python on edit (`black`), stub for uncertainty capture
- **`settings/settings.json`** — a permissions template you copy into project `.claude/settings.json` so common read-only commands don't prompt
- **`CLAUDE.md` / `AGENTS.md` / `GEMINI.md`** — Meiko-wide conventions doc, auto-loaded by each harness

## Install (Claude Code)

This repo ships its own marketplace (`.claude-plugin/marketplace.json`), so add the marketplace first, then install the plugin by name. No manual `git clone` needed.

```bash
# 1. register the marketplace (pick one source form)
/plugin marketplace add assoumaaa/meiko-superpowers                          # owner/repo shorthand
/plugin marketplace add https://github.com/assoumaaa/meiko-superpowers.git   # full git URL
/plugin marketplace add /path/to/meiko-superpowers                           # local checkout (dev)

# 2. install the plugin (format is <plugin>@<marketplace>)
/plugin install meiko-superpowers@meiko-superpowers-dev
```

The repo is **public**, so the shorthand/URL forms clone anonymously over HTTPS — no GitHub auth, SSH key, or credential helper needed.

### Pre-register for a team

To make it always available (e.g. via a shared `settings.json`), skip the marketplace add step and declare it in your Claude Code settings instead:

```json
{
  "extraKnownMarketplaces": {
    "meiko-superpowers-dev": {
      "source": { "source": "git", "url": "https://github.com/assoumaaa/meiko-superpowers.git" }
    }
  },
  "enabledPlugins": {
    "meiko-superpowers@meiko-superpowers-dev": true
  }
}
```

Plugins load at **session start** — restart Claude Code after installing or enabling for the skills to appear.

## Install (Codex)

Copy or symlink this directory into Codex's plugins folder. Codex reads `.codex-plugin/plugin.json`.

## Install (Gemini CLI)

Add the plugin directory; Gemini reads `gemini-extension.json` and auto-loads `GEMINI.md`.

## Layout

```
.claude-plugin/        Claude Code manifest
.codex-plugin/         Codex manifest
hooks/                 PreToolUse + PostToolUse hooks
skills/                One folder per skill, each with SKILL.md
settings/              Permissions template to copy into projects
CLAUDE.md              Meiko-wide conventions (Python, JS/React, testing, git, security)
AGENTS.md              Symlink → CLAUDE.md
GEMINI.md              Gemini context file
```

## Philosophy

**Principles, not rules.** Each SKILL.md tells the model *how to think*, not *what to do*. Rules go stale; principles transfer.

**Learning loop.** When the model surprises you (good or bad), log it in `skills/_journal.md`. Periodically fold the journal back into the skills. The plugin gets sharper every week.

**Cross-harness.** Same skill files work in Claude Code, Codex, Gemini CLI, and any agent that loads markdown context files.
