---
name: acumatica-customization
description: Use when the user mentions Acumatica, customization project, acumaticahelper, or wants to export, import, publish, validate, unpublish, or delete Acumatica ERP customization projects, check publish status, or toggle maintenance mode on an Acumatica instance.
license: MIT
compatibility: bash 4+, curl, jq, base64, python3; Acumatica ERP instance with Customizer role credentials required in acumatica.conf
---

# Acumatica Customization Helper

A bash script (`acumaticahelper.sh`) that manages Acumatica ERP customization
projects through the official `CustomizationApi` web API.

**Script location:** `acumaticahelper.sh` (run from its own directory)

---

## Configuration

The script reads `acumatica.conf` from the **same directory as the script**.
Copy `acumatica.conf.example` to `acumatica.conf` and fill in your values:

```ini
ACUMATICA_URL=http://host/instance    # base URL — no trailing slash
ACUMATICA_USERNAME=admin              # must have the Customizer role
ACUMATICA_PASSWORD=secret
```

Optional tuning (add to `acumatica.conf`):

| Variable               | Default | Description                                      |
|------------------------|---------|--------------------------------------------------|
| `EXPORT_PATH`          | `.`     | Default output directory for the `export` command |
| `RELEASE_PATH`         | —       | Directory containing release ZIPs for `release`  |
| `PUBLISH_POLL_INTERVAL`| `30`    | Seconds between `publishEnd` polls               |
| `PUBLISH_MAX_ATTEMPTS` | `10`    | Max polls before timeout (10 × 30s = 5 min max)  |

> **Note:** OAuth 2.0 is NOT supported by the `CustomizationApi`. Cookie-based
> session auth is used (`/entity/auth/login` / `/entity/auth/logout`).

---

## Commands

### `list`
List all published customization projects and their items.

```bash
./acumaticahelper.sh list
```

API: `POST /CustomizationApi/getPublished`

---

### `export`
Export a project as a local ZIP file. Validates the zip before saving and
auto-resolves file system conflicts.

```bash
./acumaticahelper.sh export <project-name> [output-dir]
# output-dir defaults to EXPORT_PATH, then current directory
```

API: `POST /CustomizationApi/getProject`

---

### `import`
Import a ZIP file as a customization project. Derives project name from the
filename if not specified. Replaces an existing project of the same name.

```bash
./acumaticahelper.sh import <file.zip> [project-name] [description]
```

API: `POST /CustomizationApi/import`

---

### `release`
Import a ZIP from `RELEASE_PATH` then publish it in one step. Looks for
`<project-name>.zip` in the configured `RELEASE_PATH`. Requires `RELEASE_PATH`
to be set in `acumatica.conf`.

```bash
./acumaticahelper.sh release <project-name> [description]
```

Equivalent to running `import <RELEASE_PATH>/<project-name>.zip` followed by
`publish <project-name>`.

---

### `validate`
Validate one or more projects without publishing them. Polls until complete.

```bash
./acumaticahelper.sh validate <project-name> [project-name2 ...]
```

API: `POST /CustomizationApi/publishBegin` (with `isOnlyValidation: true`)
then polls `POST /CustomizationApi/publishEnd`

---

### `publish`
Publish one or more projects. Polls until complete.

```bash
./acumaticahelper.sh publish <project-name> [project-name2 ...]
```

> **Important:** `publishEnd` must be called to complete publication — it
> triggers plug-in execution. The script handles this automatically.

API: `POST /CustomizationApi/publishBegin` → polls `POST /CustomizationApi/publishEnd`

---

### `unpublish`
Unpublish all projects. `tenantMode` controls scope.

```bash
./acumaticahelper.sh unpublish [Current|All]
# defaults to Current
```

API: `POST /CustomizationApi/unpublishAll`

---

### `delete`
Delete an **unpublished** project from the database.

```bash
./acumaticahelper.sh delete <project-name>
```

> **Warning:** The project must be unpublished first. Deletion removes project
> and item data but does NOT remove files/objects added to the site (e.g. site
> map nodes, reports).

API: `POST /CustomizationApi/delete`

---

### `status`
One-shot poll of `publishEnd` to check the current publish/validation state.

```bash
./acumaticahelper.sh status
```

API: `POST /CustomizationApi/publishEnd`

---

### `maintenance-on` / `maintenance-off`
Enable or disable maintenance mode (Lock endpoint V1).

```bash
./acumaticahelper.sh maintenance-on
./acumaticahelper.sh maintenance-off
```

API: `PUT /entity/Lock/1/ApplyUpdate/scheduleLockoutCommand` (on)
     `PUT /entity/Lock/1/ApplyUpdate/stopLockoutCommand` (off)

---

## API Endpoints Reference

| Endpoint                                       | Method | Used by              |
|------------------------------------------------|--------|----------------------|
| `/entity/auth/login`                           | POST   | all commands         |
| `/entity/auth/logout`                          | POST   | all commands         |
| `/CustomizationApi/getPublished`               | POST   | `list`               |
| `/CustomizationApi/getProject`                 | POST   | `export`             |
| `/CustomizationApi/import`                     | POST   | `import`             |
| `/CustomizationApi/publishBegin`               | POST   | `validate`, `publish`|
| `/CustomizationApi/publishEnd`                 | POST   | `validate`, `publish`, `status` |
| `/CustomizationApi/unpublishAll`               | POST   | `unpublish`          |
| `/CustomizationApi/delete`                     | POST   | `delete`             |
| `/entity/Lock/1/ApplyUpdate/scheduleLockoutCommand` | PUT | `maintenance-on`  |
| `/entity/Lock/1/ApplyUpdate/stopLockoutCommand`     | PUT | `maintenance-off` |

---

## Requirements

- `bash` 4+
- `curl` — HTTP requests
- `jq` — JSON building and parsing
- `base64` — encode/decode project ZIPs
- `python3` — ZIP validation on export (`import zipfile`)

---

## Common Workflows

**Backup before an upgrade:**
```bash
./acumaticahelper.sh export MyProject ./backups
```

**Promote from dev to prod (manual):**
```bash
# On dev — export
./acumaticahelper.sh export MyProject ./release

# On prod — import then publish
./acumaticahelper.sh import ./release/MyProject.zip
./acumaticahelper.sh publish MyProject
```

**Promote from dev to prod (using release path):**
```bash
# acumatica.conf on prod: RELEASE_PATH=/shared/releases
# Place MyProject.zip in /shared/releases, then on prod:
./acumaticahelper.sh release MyProject
```

**Safe publish with maintenance window:**
```bash
./acumaticahelper.sh maintenance-on
./acumaticahelper.sh publish MyProject
./acumaticahelper.sh maintenance-off
```

**Clean slate — unpublish everything then delete:**
```bash
./acumaticahelper.sh unpublish Current
./acumaticahelper.sh delete MyProject
```
