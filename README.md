# agiskills

Shared agent skills for Claude Code and Codex. Each folder in `skills/` is one skill with a `SKILL.md`.

## Use on your machine

Clone this repo, then symlink each skill so both tools read the same files:

```bash
for s in skills/*/; do
  name=$(basename "$s")
  ln -s "$PWD/skills/$name" ~/.claude/skills/$name
  ln -s "$PWD/skills/$name" ~/.codex/skills/$name
done
```

## Skills

- `sqlalchemy-alembic-engineer`: SQLAlchemy 2.x models, queries, and Alembic migrations
- `external-api-integration`: safe third-party API and webhook integrations
- `check-code-standards`: readability, layering, and maintainability review
- `backend-api-performance-qa`: API performance, limits, and load-test planning
