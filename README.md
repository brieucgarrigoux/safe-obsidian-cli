# Safe Obsidian CLI

An [Agent Skill](https://agentskills.io) that lets Claude read, write, search and reorganize your Obsidian vault through the official [Obsidian CLI](https://help.obsidian.md/cli), **without breaking it**.

> This is a skill for Claude (and other agents that support Agent Skills), **not an Obsidian community plugin**. Nothing is installed inside Obsidian.

## Why this skill

- **A safety protocol before every write.** Before creating, moving or deleting a note, Claude reads your vault's `CLAUDE.md`, searches for duplicates, checks which backlinks will be affected, and verifies afterwards that no link is broken.
- **Your vault's rules, not the skill's.** Templates, frontmatter, folders and naming come from your vault.
- **Field-tested pitfalls.** `vault=` targeting, notes resolved by name with `file=`, commands that silently write into the open note, single-line `eval`, `template:insert`, list properties.
- **Windows, Git Bash and headless Linux** covered, with fixes.
- **Never stale.** `obsidian help` stays the source of truth for commands and flags, so Claude never invents one.

Compared with [kepano/obsidian-skills](https://github.com/kepano/obsidian-skills), which also covers plugin and theme development, this skill focuses on day-to-day vault management and on not damaging your notes.

## Example

> **You:** Create a note for my new project Atlas.

Claude reads your `CLAUDE.md`, searches for an existing "Atlas" note, picks your project template, creates `Projects/Atlas.md`, links it from today's daily note, and checks that no link is broken. If a note already exists, it asks you first.

Other things you can ask:
- *"What are my open tasks this week?"*
- *"Move every note tagged #archive into Archive/ without breaking links."*
- *"Which notes have no incoming links?"*

## Requirements

- Obsidian desktop **1.12+**
- CLI enabled in **Settings → Command line interface**
- **Obsidian must be running**: the CLI talks to the open app

## Installation

### Claude Code (marketplace)

```
/plugin marketplace add brieucgarrigoux/safe-obsidian-cli
/plugin install safe-obsidian-cli@safe-obsidian-cli
```

### npx skills

```
npx skills add https://github.com/brieucgarrigoux/safe-obsidian-cli
```

### Manual

Copy `skills/safe-obsidian-cli/` into `~/.claude/skills/` (all projects) or into the `.claude/skills/` folder of your vault.

## Skills

| Skill | Description |
|---|---|
| [safe-obsidian-cli](skills/safe-obsidian-cli/SKILL.md) | Interact with an Obsidian vault through the official CLI: read, write, search, tasks, properties, links, with a safety protocol before any write. |

## Limitations

- The Obsidian app must be open. On a server, run it under `xvfb` (see the skill).
- One vault at a time: targeting another vault opens it in the app, so Claude asks first.
- The safety protocol relies on the agent following it; keep backups or Obsidian Sync / File Recovery enabled.
- Tested on Obsidian 1.13.7 (Windows). Behavior may differ on other versions: `obsidian help` wins.

## Feedback

Found a command that behaves differently on your setup, or a pitfall worth adding? [Open an issue](https://github.com/brieucgarrigoux/safe-obsidian-cli/issues) with your Obsidian version (`obsidian version`) and OS.

See [CHANGELOG.md](CHANGELOG.md) for version history.

## License

MIT
