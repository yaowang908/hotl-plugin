# Contributing to HOTL

Thank you for your interest in contributing to HOTL. This guide explains the repository architecture, how each piece fits together, and how to extend or improve the system.

---

## What Is This Repository?

HOTL (Human-on-the-Loop) is a **plugin and rule distribution system** — not a compiled application. It ships:

- **Skill files** — Markdown documents that teach AI coding tools how to follow structured workflows
- **Command definitions** — Slash command wrappers for Claude Code
- **Rule files** — Cline global rules that apply to all projects
- **Adapter templates** — Starting points for Cursor, GitHub Copilot, and AGENTS.md integration
- **Utility scripts** — Bash scripts for linting, installation, and updates
- **Tests** — BATS smoke tests validating skill structure and workflow parsing

There is no compilation step. Installing HOTL means copying Markdown files to the right places and registering them with the target tool.

---

## Repository Layout

```text
hotl-plugin/
├── skills/                 # Core AI behavior definitions (one directory per skill)
│   ├── brainstorming/
│   │   └── SKILL.md
│   ├── writing-plans/
│   │   └── SKILL.md
│   └── ...                 # 15 skills total
├── commands/               # Claude Code slash command entry points
│   ├── brainstorm.md
│   ├── write-plan.md
│   └── ...
├── cline/
│   └── rules/              # Cline global rule files (one per skill area)
│       ├── hotl-brainstorming.md
│       └── ...
├── hooks/
│   ├── hooks.json          # Claude Code SessionStart hook registration
│   └── run-hook.cmd        # Hook invocation script
├── adapters/               # Templates users copy into their own projects
│   ├── AGENTS.md.template
│   ├── .clinerules.template
│   ├── cursor-rules.template
│   └── copilot-instructions.template
├── workflows/              # Reusable workflow templates
│   ├── feature.md
│   ├── bugfix.md
│   └── refactor.md
├── scripts/
│   ├── document-lint.sh    # Structural validation for workflow files
│   └── dev-setup.sh
├── docs/                   # User and developer documentation
│   ├── how-it-works.md
│   ├── workflow-format.md
│   ├── architecture.md
│   ├── README.cline.md
│   ├── README.codex.md
│   └── contracts/
│       └── pr-review-output.md
├── test/
│   ├── smoke.bats          # BATS test suite
│   └── fixtures/
├── .claude-plugin/
│   ├── plugin.json         # Claude Code plugin registration
│   └── marketplace.json
├── .cursor-plugin/
│   └── plugin.json         # Cursor adapter configuration
├── install.sh              # Claude Code installation script
├── install-cline.sh        # Cline installation script
└── update.sh               # Unified update script
```

---

## How Skills Work

Each skill lives in `skills/<skill-name>/SKILL.md`. The frontmatter names and describes the skill:

```markdown
---
name: brainstorming
description: "Use before any feature work — explores intent, requirements, and design."
---

# HOTL Brainstorming
...detailed instructions...
```

Claude Code loads skills from the `skills/` directory declared in `.claude-plugin/plugin.json`. Codex discovers them through the `~/.agents/skills/hotl` symlink. Cline receives equivalent rules via `cline/rules/`.

### Adding a New Skill

1. Create `skills/<new-skill-name>/SKILL.md`
2. Add frontmatter with `name` and `description`
3. Write the skill body — follow the structure used by existing skills (overview, process steps, contracts, examples)
4. Add a row to the skill index in `skills/using-hotl/SKILL.md`
5. Create a matching `commands/<new-skill>.md` entry for Claude Code slash command support (optional)
6. Add a corresponding rule file under `cline/rules/hotl-<new-skill>.md` for Cline users
7. Update `README.md` Skills table

### Modifying an Existing Skill

Edit the relevant `SKILL.md` directly. The skill files are the source of truth — all other integration layers (`cline/rules/`, `commands/`, `hooks/`) reference or summarize them.

---

## How Commands Work (Claude Code)

Files in `commands/` define Claude Code slash commands. Each file contains a short description and typically delegates to the matching skill:

```markdown
# /hotl:brainstorm

Invoke `hotl:brainstorming` to design this change before writing any code.
```

Commands are registered via `.claude-plugin/plugin.json`:

```json
{
  "skills": "./skills/",
  "commands": "./commands/"
}
```

---

## How Hooks Work (Claude Code)

`hooks/hooks.json` registers a `session-start` hook that runs `hooks/run-hook.cmd` when a Claude Code session begins. This hook loads `using-hotl` — the skill index and operating principles — automatically at the start of every session.

---

## How Cline Rules Work

Cline reads global rule files from `~/Documents/Cline/Rules/`. `install-cline.sh` copies every file from `cline/rules/` to that directory. Each rule file is a self-contained Markdown document covering one HOTL skill area. Changes to `cline/rules/` take effect for Cline users after they re-run the installer or `update.sh`.

---

## Workflow File Format

When HOTL executes work, it creates `hotl-workflow-<slug>.md` files in the user's project. The format is defined in [`docs/workflow-format.md`](docs/workflow-format.md). When modifying any execution skill, ensure the step format it generates and parses remains consistent with that specification.

---

## Running Tests

Tests use [BATS](https://github.com/bats-core/bats-core):

```bash
bats test/smoke.bats
```

The smoke tests validate:
- Skill file discovery (all expected skills are present and parseable)
- Workflow file parsing (step extraction, frontmatter fields)
- Branch name derivation logic
- Document-lint script correctness

When adding a skill or changing the workflow format, add a corresponding test in `test/smoke.bats`.

---

## Linting Workflow Files

`scripts/document-lint.sh` validates `hotl-workflow-*.md` files structurally:

```bash
bash scripts/document-lint.sh hotl-workflow-example.md
```

This is the deterministic gate that runs before AI-driven review in the `hotl:document-review` skill. Keep it in sync with `docs/workflow-format.md` whenever the format changes.

---

## Adapter Templates

Files in `adapters/` are templates users copy into their own projects via `/hotl:setup`. They are not loaded automatically. When the HOTL operating model changes (new skills, updated gate logic, etc.), update the relevant template so new projects start with current guidance.

---

## Version and Changelog

The version is kept in `.claude-plugin/plugin.json` under the `version` field. `CHANGELOG.md` documents every release. When preparing a release:

1. Update `version` in `.claude-plugin/plugin.json`
2. Add an entry to `CHANGELOG.md` following the existing format
3. Tag the release

---

## Design Principles

These principles guide what belongs in HOTL and how skills should be written:

- **Design before code** — no skill should generate implementation code before requirements are explicitly clear
- **Verification before claiming done** — every skill that changes state must end with verification steps
- **Human gates for high-risk changes** — `risk_level: high` always requires human approval regardless of `auto_approve`
- **Hard-fail on dirty state** — execution skills must not silently stash or mutate state; surface the problem and ask
- **Structural determinism** — `document-lint.sh` validates structure without AI — keep it deterministic and fast
- **Minimal surface area** — each skill does one thing well; cross-cutting concerns belong in `using-hotl`

---

## Reporting Issues

Open an issue at [github.com/yimwoo/hotl-plugin/issues](https://github.com/yimwoo/hotl-plugin/issues). Include:

- Which AI tool you are using (Claude Code, Codex, Cline, etc.)
- The skill or command that behaved unexpectedly
- What you expected vs. what happened
