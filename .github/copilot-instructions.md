# Project Guidelines

This workspace uses [CLAUDE.md](../CLAUDE.md) as the canonical project guide.

Before making code changes, read [CLAUDE.md](../CLAUDE.md) for the platform overview, build commands, test device addresses, and image update workflow.

Use this file only as a thin entrypoint. Do not duplicate shared project instructions here. When common workflow or environment details change, update [CLAUDE.md](../CLAUDE.md) instead.

Treat `.claude/` as the home for modular supplemental docs. When the user says they are working on a specific area such as launcher3, QSSI, vendor, LE, or image update, look for a matching module note or skill under `.claude/skills/` and load that on demand.

If a needed module note does not exist yet, add it under `.claude/skills/` instead of expanding this file.