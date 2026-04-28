# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A skill + bash helper for managing Acumatica ERP customization projects via the official `CustomizationApi` web API, packaged for Claude Code, OpenClaw, and Hermes Agent. There are no build steps, tests, or package managers.

## Repo Structure

```
claude/                          # Claude Code plugin install
  .claude-plugin/
    plugin.json
    skills/
      acumatica-customization/
        SKILL.md
        acumaticahelper.sh
        acumatica.conf.example
openclaw/                        # OpenClaw install
  acumatica-customization/
    SKILL.md
    acumaticahelper.sh
    acumatica.conf.example
hermes/                          # Hermes Agent install
  acumatica-customization/
    SKILL.md
    acumaticahelper.sh
    acumatica.conf.example
README.md
CLAUDE.md
```

All three distributions share identical `SKILL.md`, `acumaticahelper.sh`, and `acumatica.conf.example` — only the wrapping directory structure differs.

## Running the Script

```bash
cd claude/.claude-plugin/skills/acumatica-customization   # or openclaw/acumatica-customization etc.
chmod +x acumaticahelper.sh
./acumaticahelper.sh help
./acumaticahelper.sh list
./acumaticahelper.sh export   <project-name> [output-dir]
./acumaticahelper.sh import   <file.zip> [project-name] [description]
./acumaticahelper.sh release  <project-name> [description]
./acumaticahelper.sh validate <project-name> [project-name2 ...]
./acumaticahelper.sh publish  <project-name> [project-name2 ...]
./acumaticahelper.sh unpublish [Current|All]
./acumaticahelper.sh delete   <project-name>
./acumaticahelper.sh status
./acumaticahelper.sh maintenance-on
./acumaticahelper.sh maintenance-off
```

## Configuration

The script reads `acumatica.conf` from the **same directory as the script** (not the caller's working directory). The conf file must use bare `KEY=VALUE` lines — the script's parser (`declare -g`) does not handle the `export KEY="VALUE"` syntax.

```ini
ACUMATICA_URL=http://host/instance
ACUMATICA_USERNAME=admin
ACUMATICA_PASSWORD=secret
# Optional:
EXPORT_PATH=/path/to/exports        # default output dir for export
RELEASE_PATH=/path/to/releases      # source dir for release command
PUBLISH_POLL_INTERVAL=30
PUBLISH_MAX_ATTEMPTS=10
```

Add `acumatica.conf` to `.gitignore` and `chmod 600` it — it contains plaintext credentials.

## Architecture

The script is a single file with three logical sections:

1. **Config + auth** — loads `acumatica.conf`, provides `_login`/`_logout`. Login stores a session cookie in a temp file; a `trap '_logout' EXIT` guarantees cleanup on every exit path. OAuth 2.0 is not supported by the `CustomizationApi`.

2. **Commands** (`cmd_*` functions) — one function per CLI command. Each calls `_login` then makes one or more `curl` POST/PUT requests against the Acumatica instance. `cmd_release` is a composite: it calls `cmd_import` then `cmd_publish` (each calls `_login` internally; the double-login is acceptable — the cookie file is shared and the session refreshes).

3. **Publish polling** (`_poll_publish_end`) — shared by both `publish` and `validate`. Both commands call `publishBegin` then poll `publishEnd` until `isCompleted=true`.

## Key Behaviours to Preserve

- `export` decodes the base64 ZIP to a temp file, validates it with `python3 zipfile`, then atomically `mv`s it — ensures no corrupt file is left on failure.
- `_require_json` guards every API response; non-JSON triggers a descriptive error.
- `delete` requires the project to be unpublished first; it removes DB records only — files/objects added to the site (site map nodes, reports) are **not** removed.
- `unpublish` accepts `Current`, `All`, or `List` as `tenantMode`; defaults to `Current`.

## Skill File

`SKILL.md` is identical across all three distributions. The frontmatter uses the agentskills.io standard (single-line `description`, `license`, `compatibility`) which is valid for Claude Code, OpenClaw, and Hermes Agent. When editing `SKILL.md`, edit one copy and sync the others — the canonical source is `claude/.claude-plugin/skills/acumatica-customization/SKILL.md`.
