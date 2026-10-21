# skills

Claude Code skills.

## Skills

- [adversarial-review](skills/adversarial-review/SKILL.md) - three agents review a GitLab MR against [principles.md](skills/adversarial-review/principles.md), then try to refute each other. You get only the findings that survive.

## Install

Copy a skill folder into `~/.claude/skills/` (all projects) or `<repo>/.claude/skills/` (one project).

```sh
cp -R skills/adversarial-review ~/.claude/skills/
```

Run it with `/adversarial-review <GitLab MR URL>`.
