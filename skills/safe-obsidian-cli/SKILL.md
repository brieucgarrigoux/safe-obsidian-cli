---
name: safe-obsidian-cli
description: >
  Read, write, search and reorganize an Obsidian vault through the official
  Obsidian CLI, with a safety protocol before any write. Use whenever the user
  wants Claude to act on their vault: read a note or the daily note; create,
  append, move, rename or delete notes; search the vault; manage tasks,
  properties, tags, bookmarks or templates; find orphans or broken links; query
  Bases; restore versions from Sync or file history; automate vault workflows;
  run JavaScript against the Obsidian API. Treat "go into my vault and do X" as
  a trigger. Do not use for conceptual questions about the Obsidian app, its
  settings, theme or plugin installation through the UI, Dataview syntax, or
  third-party sync conflicts, where the user needs an explanation rather than
  an action on the vault.
license: MIT
compatibility: >
  Requires the Obsidian desktop app 1.12+ running, with the command line
  interface enabled in its settings, and shell access. Designed for Claude Code;
  works with any agent that supports Agent Skills.
metadata:
  version: "0.1.0"
  tested-on: "Obsidian 1.13.7"
---

# Safe Obsidian CLI

The official Obsidian CLI (v1.12+) controls a **running** Obsidian desktop app from the terminal, over IPC.

## Rule 1: `obsidian help` is the source of truth

This skill deliberately does **not** list every command and flag, because they change between Obsidian versions.

- `obsidian help` lists every command and flag of the installed version.
- `obsidian help <command>` shows the flags of one command.

**Before using a command or flag that is not in the memo below, run `obsidian help <command>`.** Never guess a flag. Official docs: https://help.obsidian.md/cli

---

## Rule 2: protocol before any write

For any **create / append / prepend / move / rename / delete / property change**, follow these three steps. They set the *method*; the concrete conventions (templates, frontmatter, folders, naming) come from the **active vault**, never from an assumption.

**1. Pre-check (always)**
- Read the vault's `CLAUDE.md` and its index or home note if any, unless already done in this session. They define the conventions to apply.
- `obsidian search query="title or concept"` to make sure a similar note does not already exist.
- Before a move, rename or deletion: `obsidian backlinks path="..."` to see which notes will be affected.

**2. Follow the vault's conventions**
- Use the vault's templates (`obsidian templates`, then `create ... template="..."`). Never rebuild a frontmatter by hand if a template exists.
- Respect the vault's own filing and linking system (folders, `parent` property, tags...).
- If the vault has no explicit rule: stay minimal and consistent with existing notes, impose nothing.

**3. Post-check (after a move or a multi-file change)**
- `obsidian unresolved total` must return `0` (or the same number as before).
- Update the vault's index note **if it exists** and if the structure changed.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Obsidian Desktop | v1.12.0+ (`obsidian version`) |
| CLI enabled | Settings → Command line interface |
| Obsidian running | The app **must be open**, otherwise commands hang or return nothing |

## Core syntax

```bash
obsidian <command> [key=value ...] [flags] [vault=<name>]
```

- **Parameters** are `key=value`. Quote values with spaces: `content="hello world"`.
- **Flags** are bare words: `total`, `overwrite`, `permanent`, `case`...
- **New lines in content**: use `\n` (and `\t` for tabs) inside `content=`.
- **`file=` vs `path=`**:
  - `file="Note name"` resolves the note **by name**, like a wikilink. No folder or extension needed.
  - `path="folder/note.md"` is the exact vault-relative path.
  - With neither, most commands act on the **file currently open** in Obsidian. Always pass one of them to avoid writing in the wrong note.
- **Targeting a vault**: `vault="My Vault"`. Without it, the CLI uses the most recently active vault.

## Memo: most common commands (verified on 1.13.7)

```bash
# Read & write
obsidian read file="Note name"
obsidian create path="folder/note" content="# Title\n\nBody"
obsidian create path="folder/note" template="Template name"
obsidian append path="folder/note.md" content="New paragraph"
obsidian prepend path="folder/note.md" content="Goes after the frontmatter"
obsidian move path="old/note.md" to="new/folder"      # to= accepts a folder or a full path
obsidian rename path="folder/note.md" name="New name"
obsidian delete path="folder/note.md"                  # to trash; add `permanent` to skip it

# Daily note
obsidian daily:read
obsidian daily:append content="- [ ] New task"
obsidian daily:path

# Search & links
obsidian search query="project alpha" limit=10 format=json   # JSON array of paths
obsidian search:context query="project alpha"                # matching lines
obsidian backlinks path="folder/note.md"
obsidian unresolved total
obsidian orphans                                              # notes with NO incoming link

# Properties & tasks
obsidian properties path="folder/note.md"
obsidian property:set path="folder/note.md" name="status" value="active"
obsidian tasks todo                                           # incomplete tasks
obsidian task path="folder/note.md" line=12 done
```

For everything else (Bases, Sync, history, diff, plugins, themes, bookmarks, workspace, dev tools): `obsidian help`.


## Example: create a project note safely

User: *"Create a note for my new project Atlas."*

```bash
obsidian read path="CLAUDE.md"                      # 1. vault conventions (folder, template, naming)
obsidian search query="Atlas" limit=5               # 2. does a note already exist?
obsidian templates                                  # 3. which template to use
obsidian create path="Projects/Atlas" template="Project"
obsidian property:set path="Projects/Atlas.md" name="status" value="planning"
obsidian daily:append content="- Started [[Atlas]]"
obsidian unresolved total                           # 4. still no broken link
```

If the search finds an existing "Atlas" note, stop and ask the user instead of creating a duplicate. Folder and template names above are examples: always take them from the vault.

---

## Known pitfalls

1. **Vault name as first argument fails.** `obsidian "My Vault" read ...` returns `Command "My Vault" not found`. Use `vault="My Vault"`.
2. **Targeting another vault opens it** in the Obsidian app. Ask the user before targeting a vault other than the active one.
3. **`create` on an existing file**: replacing it requires the `overwrite` flag. Read the file first and only overwrite when it is clearly intended; otherwise use `append` / `prepend`.
4. **`create` path**: the `.md` extension is added automatically, so omit it.
5. **`template:insert` writes into the file currently open** in the Obsidian UI and takes no `path=`. To create a note from a template, use `create ... template="..."`.
6. **`property:set` and lists**: by default the value is stored as text, so `value="a, b"` becomes one string. Pass `type=list` for a list property, then check the result with `obsidian property:read`.
7. **`eval` needs single-line JavaScript.** For a multi-line script, write it to a temp file and pass it with command substitution:
   ```bash
   cat > /tmp/obs.js << 'JS'
   var files = app.vault.getMarkdownFiles();
   files.length;
   JS
   obsidian eval code="$(cat /tmp/obs.js)"
   ```
8. **`dev:console`** only captures messages after `obsidian dev:debug on`.
9. **`dev:screenshot path=`** must be vault-relative.
10. **Output formats differ by command** (`text`, `json`, `tsv`, `csv`, `yaml`, `md`, `tree`...). Check `obsidian help <command>` before parsing output.

## Platform notes

- **macOS / Linux**: enabling the CLI in settings registers `obsidian` in PATH.
- **Windows**: needs the `Obsidian.com` redirector next to `Obsidian.exe` (shipped with recent installers). Run from a **normal, non-admin** terminal: admin terminals fail silently.
- **Windows + Git Bash / MSYS2**: Bash may resolve `obsidian` to `Obsidian.exe` (the GUI) instead of `Obsidian.com`, so commands with a colon and parameters fail with exit code 127. Fix: create `~/bin/obsidian` containing
  ```bash
  #!/bin/bash
  "/c/path/to/Obsidian.com" "$@"
  ```
  then add `export PATH="$HOME/bin:$PATH"` to `~/.bashrc`. PowerShell is not affected.
- **Headless Linux**: install the `.deb` package (snap confinement breaks IPC), run Obsidian under `xvfb` (`DISPLAY=:5 obsidian &`), prefix commands with `DISPLAY=:5`, set `PrivateTmp=false` in a systemd service. GPU warnings on stderr are harmless (`2>/dev/null`).

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Empty output or hang | Obsidian not running, or admin terminal (Windows) | Open Obsidian; use a non-admin terminal |
| `obsidian: command not found` | CLI not registered | Re-enable the CLI in settings, restart the terminal |
| `Command "X" not found` | Vault name passed as first argument, or typo | Use `vault="X"`; check `obsidian help` |
| Wrote in the wrong note | No `file=` / `path=`, so the active file was used | Always pass `file=` or `path=` |
| Exit 127 on colon commands (Windows) | `Obsidian.com` missing, or Git Bash resolving the `.exe` | Reinstall from obsidian.md/download, or use the wrapper above |
| IPC socket not found (Linux) | `PrivateTmp=true` or snap package | `PrivateTmp=false`; use the `.deb` |
