# Skills

A collection of custom [Agent Skills](https://agentskills.io) for automating development workflows with [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

# About This Repository

This repository contains skills focused on development workflow automation. Each skill is self-contained in its own folder with a `SKILL.md` file containing the instructions and metadata that Claude uses.

## Skills

| Skill | Description |
|-------|-------------|
| [worktree-setup](./skills/worktree-setup) | Set up parallel Git worktree-based development environments with shared config files via symlinks |
| [worktree-setup-ja](./skills/worktree-setup-ja) | Same as above, Japanese version |

# Installation

## Claude Code

Register this repository as a Claude Code Plugin marketplace:

```
/plugin marketplace add akifumi/skills
```

Then install a skill:

```
/plugin install worktree-setup@akifumi-agent-skills
```

After installing, you can use the skill via its slash command:

```
/worktree-setup /path/to/repo 3
```

# Structure

```
skills/
  worktree-setup/         # English version
    SKILL.md
  worktree-setup-ja/      # Japanese version
    SKILL.md
.claude-plugin/
  marketplace.json        # Plugin marketplace definition
```

# License

[MIT](LICENSE)
