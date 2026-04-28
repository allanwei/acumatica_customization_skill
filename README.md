# acumatica-customization

A skill + bash helper for managing **Acumatica ERP customization projects**
via the official `CustomizationApi` web API.

Compatible with **Claude Code**, **OpenClaw**, and **Hermes Agent**.

---

## Repository Structure

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
```

---

## Installation

### Claude Code

**Option A — Plugin (project-scoped):**

```bash
# Copy the plugin directory into your project root
cp -r claude/.claude-plugin /path/to/your/project/

# Or load it directly when launching Claude Code
claude --plugin-dir ./claude
```

**Option B — Personal skill (available in all projects):**

```bash
cp -r claude/.claude-plugin/skills/acumatica-customization ~/.claude/skills/
```

---

### OpenClaw

**Manual install:**

```bash
cp -r openclaw/acumatica-customization ~/.openclaw/skills/
```

Then restart the OpenClaw gateway so the skill is snapshotted.

**Via ClawHub** (once published):

```bash
clawhub install acumatica-customization
```

---

### Hermes Agent

**Manual install:**

```bash
cp -r hermes/acumatica-customization ~/.hermes/skills/
```

Skills in `~/.hermes/skills/` are picked up automatically — no restart needed.

**Via HermesHub / agentskills.io** (once published):

```bash
hermes skills install acumatica-customization
```

---

## Quick Start (all platforms)

### 1. Configure

```bash
cd ~/.claude/skills/acumatica-customization   # adjust path for your platform
cp acumatica.conf.example acumatica.conf
# Edit acumatica.conf:
#   ACUMATICA_URL=http://yourhost/instance
#   ACUMATICA_USERNAME=admin
#   ACUMATICA_PASSWORD=secret
```

> The config file must live in the **same directory** as `acumaticahelper.sh`.

### 2. Make executable

```bash
chmod +x acumaticahelper.sh
```

### 3. Run

```bash
./acumaticahelper.sh help
./acumaticahelper.sh list
```

---

## Commands at a Glance

```
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

Full documentation for every command: see **SKILL.md**.

---

## Configuration Reference

**Required** (in `acumatica.conf`):

| Key                  | Description                                            |
|----------------------|--------------------------------------------------------|
| `ACUMATICA_URL`      | Base URL of the instance, no trailing slash            |
| `ACUMATICA_USERNAME` | Login — must have the **Customizer** role              |
| `ACUMATICA_PASSWORD` | Password                                               |

**Optional:**

| Key                    | Default | Description                                        |
|------------------------|---------|----------------------------------------------------|
| `EXPORT_PATH`          | `.`     | Default output directory for the `export` command  |
| `RELEASE_PATH`         | —       | Directory containing ZIPs for the `release` command|
| `PUBLISH_POLL_INTERVAL`| `30`    | Seconds between `publishEnd` polls                 |
| `PUBLISH_MAX_ATTEMPTS` | `10`    | Max polls before timeout (default: 5 min)          |

---

## Requirements

- `bash` 4+
- `curl`
- `jq`
- `base64`
- `python3` (standard library only — used for ZIP validation on export)

> **Note:** OAuth 2.0 is not supported by the `CustomizationApi`. The script
> uses cookie-based session auth (`/entity/auth/login`). The session is always
> cleaned up via a `trap '_logout' EXIT`.

---

## Security Note

`acumatica.conf` contains plaintext credentials. Add it to `.gitignore` and
restrict file permissions:

```bash
chmod 600 acumatica.conf
```

---

## Common Workflows

**Backup a project:**
```bash
./acumaticahelper.sh export MyProject ./backups
```

**Promote dev → prod (manual):**
```bash
# dev
./acumaticahelper.sh export MyProject ./release

# prod
./acumaticahelper.sh import ./release/MyProject.zip
./acumaticahelper.sh publish MyProject
```

**Promote dev → prod (via release path):**
```bash
# acumatica.conf on prod: RELEASE_PATH=/shared/releases
./acumaticahelper.sh release MyProject
```

**Publish with maintenance window:**
```bash
./acumaticahelper.sh maintenance-on
./acumaticahelper.sh publish MyProject
./acumaticahelper.sh maintenance-off
```

---

## API Reference

All endpoints are from Acumatica's official
*"Managing Customization Projects by Using the Web API"* documentation.

| Command          | Endpoint                                             | Method |
|------------------|------------------------------------------------------|--------|
| `list`           | `/CustomizationApi/getPublished`                     | POST   |
| `export`         | `/CustomizationApi/getProject`                       | POST   |
| `import`         | `/CustomizationApi/import`                           | POST   |
| `release`        | `import` + `publish` (composite)                     | —      |
| `validate`       | `/CustomizationApi/publishBegin` + `publishEnd`      | POST   |
| `publish`        | `/CustomizationApi/publishBegin` + `publishEnd`      | POST   |
| `unpublish`      | `/CustomizationApi/unpublishAll`                     | POST   |
| `delete`         | `/CustomizationApi/delete`                           | POST   |
| `status`         | `/CustomizationApi/publishEnd`                       | POST   |
| `maintenance-on` | `/entity/Lock/1/ApplyUpdate/scheduleLockoutCommand`  | PUT    |
| `maintenance-off`| `/entity/Lock/1/ApplyUpdate/stopLockoutCommand`      | PUT    |
