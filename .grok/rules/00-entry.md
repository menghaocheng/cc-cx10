# Grok Project Entry

This workspace uses [CLAUDE.md](../../CLAUDE.md) as the canonical project guide.

Grok auto-loads this file from `.grok/rules/` (project rules). Before making code changes, read [CLAUDE.md](../../CLAUDE.md) for the platform overview, dual-environment build/debug conventions, six-piece deploy workflow, and acceptance tests.

Use this file only as a thin entrypoint. Do not duplicate shared project instructions here. When common workflow or environment details change, update [CLAUDE.md](../../CLAUDE.md) instead.

## Multi-agent entry map

| Agent | Entry | Role |
|-------|-------|------|
| Claude Code | `CLAUDE.md` | Canonical full guide (SSOT) |
| Codex | `AGENTS.md` | Codex-facing entry (keep aligned with `CLAUDE.md`) |
| GitHub Copilot | `.github/copilot-instructions.md` | Thin pointer → `CLAUDE.md` |
| Grok | `.grok/rules/00-entry.md` (this file) | Thin pointer → `CLAUDE.md` |

Root-level `AGENTS.md` / `CLAUDE.md` may also be auto-loaded by Grok. If any instruction conflicts, **prefer `CLAUDE.md`**.

## Modular docs (load on demand)

Treat `.claude/` as the home for modular supplemental docs (not a second SSOT):

- Context / status: `.claude/context/` (e.g. `tango_hardening.md`)
- Build & debug: `.claude/instructions/`
- Skills / commands: `.claude/skills/`, `.claude/commands/`
- Project skills also under `.agents/skills/` when present

When the user is working on a specific area, look for a matching note or skill and load it on demand. If a needed module note does not exist yet, add it under `.claude/` (or a project skill under `.agents/skills/` / `.grok/skills/`) instead of expanding this file.

## Mainline code path

Primary work lives in `android10/vendor/hello/arm96/`. Nested module notes: `android10/vendor/hello/arm96/CLAUDE.md` and `debug.md`.
