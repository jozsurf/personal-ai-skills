# personal-ai-skills

Personal AI skills and agents for use with the [allagents](https://github.com/WiseTechGlobal/cargowise-allagents) workspace.

## Plugin

This repo exposes a single plugin: `personal`

Reference it in `.allagents/workspace.yaml`:

```yaml
plugins:
  - personal@jozsurf/personal-ai-skills
```

## Contents

### Skills

| Skill | Description |
|-------|-------------|
| `skill-author` | Guide for writing high-quality skill documents and agent definitions |

### Agents

| Agent | Description |
|-------|-------------|
| `worktree-cleanup` | Safely inventory and clean up local git worktrees |
