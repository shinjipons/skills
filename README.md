# Skills

Personal Claude skills. Each skill is a folder under `skills/` with a `SKILL.md`.

## Install

Copy a skill folder into `~/.claude/skills/` (on Windows, `C:\Users\<you>\.claude\skills\`), or symlink it:

```bash
cp -r skills/ship ~/.claude/skills/
```

Start a new session afterwards.

## Skills

- `ship`: commit, push, open a PR, wait for CI, then merge and delete the branch. Runs only when you type `/ship`.
