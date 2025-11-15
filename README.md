# backup_sync: Backup and Synchronization Scripts

A collection of flexible, profile-based backup and synchronization tools built on `rsync`. Designed for reliable, reproducible backups with multiple profiles, environment file preservation, and portable restores.

**Table of Contents**

- [Quick Start](#quick-start)
- [Tools Overview](#tools-overview)
- [backup](#backup)
- [restore](#restore)
- [remote_sync](#remote_sync)
- [Configuration](#configuration)
- [Examples](#examples)
- [Advanced Features](#advanced-features)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

### Backup your system
```bash
./backup -p full        # Full home directory backup
./backup -p lite        # Lite backup (tech folder only)
```

### Restore from backup
```bash
./restore -p full all                    # Restore everything
./restore -p full vscode                 # Restore just VS Code settings
./restore -p full -n all                 # Preview restore (dry-run)
./restore -p full extensions             # Reinstall extensions
```

---

## Tools Overview

| Tool | Purpose | Use Case |
|------|---------|----------|
| **backup** | Create backups to storage devices | Daily/weekly backups of system files |
| **restore** | Restore from backup devices | Recover files, migrate to new machine |
| **remote_sync** | Sync files to/from remote hosts | Sync with remote servers or machines |

---

## backup

Backs up files based on customizable profiles. Each profile targets a specific source folder, destination device, and set of environment files (system configs, credentials, etc.).

### Usage

```bash
backup [-h] [-p PROFILE]

Options:
  -p PROFILE    Backup profile name (required)
  -h            Display help message
```

### How it works

1. **Profile Selection** — Load configuration from `$BACKUP_CONF/backup_${profile}_vars` and exclusion rules
2. **Device Prompt** — Ask for device name (USB drive, external HD, etc.)
3. **Environment Backup** — Back up files outside main source dir (repos, packages, VS Code, SSH keys, etc.)
4. **Config Packaging** — Copy the local configuration directory (`$BACKUP_CONF`, default `~/.backup_config/`) to the device under `$DEVICE_PATH/backup_config/`. This makes the backup self-contained: `restore` can read profile variables directly from the device if present.
5. **Main Backup** — rsync source folder to device, excluding specified patterns
6. **Logging** — Write detailed log to `backup_$TIMESTAMP.log`

### What gets backed up

**System Environment:**
- RPM package list
- Yum repository configuration (`/etc/yum.repos.d/`)
- Custom environment files (SSH keys, configs, bashrc, etc.)

**VS Code:**
- User settings, keybindings, snippets
- Extensions list (for quick reinstall)

**Main Source:**
- Files from `src_folder` (e.g., `$HOME` or `$HOME/tech/`)
- Respects exclusion patterns (`.cache/`, large files, etc.)

### Example profiles

Two profiles are provided:

#### Profile: `full`
Backs up entire home directory to a "Backup" device.

```bash
# NOTE: profile variables in the shipped profiles are UPPERCASE
SRC_FOLDER=$HOME
ENV_FILES=(
  /etc/hosts
)
DEFAULT_DEVICE_NAME="Backup"
DEV_DEST_FOLDER="home"
```

Usage:
```bash
./backup -p full
Device name [Backup]: 
[INFO] Starting backup of /home/chouaieb to /run/media/user/Backup/home
```

#### Profile: `lite`
Backs up only `~/tech/` folder to a "BackupLite" device.

```bash
# NOTE: profile variables in the shipped profiles are UPPERCASE
SRC_FOLDER="$HOME/tech/"
ENV_FILES=(
  /etc/hosts
  $HOME/.bashrc
  $HOME/.ssh
  $HOME/.backup_config
  # ... more files ...
)
DEFAULT_DEVICE_NAME="BackupLite"
DEV_DEST_FOLDER="tech"
```

Usage:
```bash
./backup -p lite
Device name [BackupLite]: 
[INFO] Starting backup of /home/chouaieb/tech to /run/media/user/BackupLite/tech
```

### Configuration

Backup profiles are defined in `$BACKUP_CONF` (default: `~/.backup_config/`).

Each profile requires two files:

#### 1. `backup_${PROFILE}_vars` — Profile configuration

A shell script that defines:

| Variable | Purpose | Example |
|----------|---------|---------|
| `SRC_FOLDER` | Source folder to backup | `$HOME` or `$HOME/tech/` |
| `ENV_FILES` | Array of files/folders to backup | `(/etc/hosts $HOME/.ssh)` |
| `VSCODE_PROFILE` | VS Code profile name | `"base"` |
| `DEFAULT_DEVICE_NAME` | USB/device name prompt default | `"Backup"` |
| `DEV_DEST_FOLDER` | Destination folder on device | `"home"` or `"tech"` |

**Example `backup_full_vars`:**
```bash
#!/usr/bin/env bash

# NOTE: variables used by the scripts are uppercase in the provided profiles
SRC_FOLDER=$HOME
ENV_FILES=(
  /etc/hosts
)

VSCODE_PROFILE="base"
DEFAULT_DEVICE_NAME="Backup"
DEV_DEST_FOLDER="home"
```

#### 2. `backup_${PROFILE}_exclusion` — Exclusion patterns

Defines files/folders to exclude from backup (rsync `--exclude-from` format).

**Example `backup_full_exclusion`:**
```
.cache/
.var/
.local/share/containers/storage/
Downloads/games/
Downloads/videos/
tech/vms/*
```

See `man rsync` for pattern syntax.

### Output structure

Backups are stored on the device with this structure:

```
$device_path/
├── backup_YYYY-MM-DD_HH-MM-SS.log    # Detailed log file
├── success_backup.log                # rsync success output
├── error_backup.log                  # rsync errors (if any)
├── backup_config/                    # Packaged copy of your $BACKUP_CONF (profiles, exclusions)
├── env_files/
│   ├── env_files.manifest            # Path mapping for env files
│   ├── hosts
│   ├── .bashrc
│   ├── .ssh/
│   ├── package_list
│   ├── yum.repos.d/
│   └── vscode/
│       ├── User/                     # VS Code settings, snippets, keybindings
│       └── code_extension_list       # Installed extensions
└── home/  (or other dest folder)
  ├── Documents/
  ├── Downloads/
  └── ... (entire src_folder)
```

### Advanced: env_files.manifest

New backups create `env_files.manifest` to enable **portable restores** without needing the profile file.

Format: `basename|original_path`

```
hosts|/etc/hosts
.bashrc|/home/chouaieb/.bashrc
.ssh|/home/chouaieb/.ssh
```

See [ENV_FILES_MANIFEST.md](ENV_FILES_MANIFEST.md) for details.

---

## restore

Restores files from a backup device created by the `backup` script. Supports granular restores (environment files, VS Code, extensions, main folder) or complete restores.

### Usage

```bash
restore -p PROFILE [-n] ACTION

Options:
  -p PROFILE    Backup profile name (required)
  -n            Dry-run mode (preview without making changes)

Actions:
  env           Restore environment files listed in profile
  vscode        Restore VS Code User settings (settings.json, keybindings, snippets, etc.)
  extensions    Install extensions from saved list
  src           Restore main source folder from backup device
  all           Restore env + vscode + extensions + src (in order)
```

### How it works

1. **Profile Selection** — Load configuration from `$BACKUP_CONF/backup_${profile}_vars`. If the mounted device contains a packaged copy of the config (`$DEVICE_PATH/backup_config/`), `restore` will prefer reading the profile vars from that packaged config (making restores self-contained and portable).
2. **Device Prompt** — Ask for device name (defaults to `default_device_name` from profile)
3. **Manifest/Path Reading** — Uses `env_files.manifest` if available, falls back to profile
4. **Selective Restore** — Restore only requested component(s)
5. **Logging** — Write detailed log to `restore_$TIMESTAMP.log`
6. **Confirmation** — Prompts before destructive operations (e.g., restoring src folder)

### Example workflows

#### Restore just VS Code settings
```bash
./restore -p full vscode
Device name [Backup]: 
[INFO] Profile: full
[INFO] Device: /run/media/chouaieb/Backup
======== RESTORING VS CODE USER SETTINGS ========
[INFO] Restored VS Code User settings to /home/chouaieb/.config/Code/User
```

#### Preview a full restore (dry-run)
```bash
./restore -p full -n all
Device name [Backup]: 
[INFO] Dry-run mode: previewing restore operations
[...operations listed with --dry-run...]
```

#### Restore environment files + reinstall extensions
```bash
./restore -p full env extensions
Device name [Backup]: 
======== RESTORING ENV FILES ========
[INFO] Restoring file: /etc/hosts
[INFO] Restoring directory: /home/chouaieb/.ssh
======== RESTORING VS CODE EXTENSIONS (from list) ========
[INFO] Installing extension: ms-python.python
[INFO] Installing extension: golang.go
```

#### Full restore (with confirmation)
```bash
./restore -p full all
Device name [Backup]: 
[...env files restored...]
[...VS Code settings restored...]
[...extensions installed...]
======== RESTORING SRC FOLDER ========
This will overwrite files in /home/chouaieb. Proceed? [y/N] y
[INFO] Restored source folder to /home/chouaieb/
```

### Restore actions explained

| Action | What it does | Use case |
|--------|-------------|----------|
| `env` | Restores environment files (SSH keys, configs, etc.) | Recover lost configs |
| `vscode` | Copies VS Code User folder back from backup | Restore editor settings |
| `extensions` | Installs VS Code extensions from saved list | Reinstall on new machine |
| `src` | Restores entire main source folder | Full data recovery |
| `all` | Runs all above actions in order | Complete system restore |

### Dry-run mode

Preview any restore operation without making changes:

```bash
./restore -p full -n src           # Preview source folder restore
./restore -p lite -n vscode        # Preview VS Code restore
./restore -p full -n all           # Preview everything
```

Useful for testing or understanding what will change before committing.

### Manifest-aware restores

New restore script uses `env_files.manifest` (created by new backup script) to restore env files to their **original paths**, without needing the profile file.

**Benefits:**
- ✓ Backups are self-contained and portable
- ✓ Restore works even if profile is deleted
- ✓ No basename collisions
- ✓ Can restore to different machine/user

**Fallback:** If manifest is missing, restore reads profile's `env_files` array (backward compatible with old backups).

See [ENV_FILES_MANIFEST.md](ENV_FILES_MANIFEST.md) for technical details.

### Output files

Restores create a log on the device:

```
$device_path/
├── restore_YYYY-MM-DD_HH-MM-SS.log    # Detailed restore log
├── [backup files preserved]
```

---

## remote_sync

This script syncs files from/to a remote host using the rsync tool. Before sending or fetching files it will first perform a dry run and display would-be-copied files with a custom folder tree depth.

After the dry run, User confirmation will be asked to synchronize files.

### Usage

```bash
remote_sync <REMOTE_USER> <REMOTE_HOST> <REMOTE_DIR> <LOCAL_DIR>
```

**Arguments:**
- `REMOTE_USER` — SSH username on remote host
- `REMOTE_HOST` — Remote hostname or IP
- `REMOTE_DIR` — Remote directory path
- `LOCAL_DIR` — Local directory path

**Interactive prompts:**
- Sync type: `get` (download from remote) or `send` (upload to remote)
- Dry-run depth: Folder tree depth to display (default: 3)
- Confirmation: Proceed with sync? (y/N)

### Example

Sync `/var/www/html` from server `prod.example.com`:

```bash
./remote_sync deploy prod.example.com /var/www/html ~/projects/myapp
Sync type? (get/send): get
Dry run depth? [3]: 
These modifications will be applied to "localhost"
    5 html/
    2 html/css/
    3 html/js/
Get files from prod.example.com? (y/N): y
Syncing files in 10 seconds...
```

---

## Configuration

### Setup: Create profiles

1. Create config directory:
```bash
mkdir -p ~/.backup_config
```

2. Create a profile (e.g., `full`):

**File: `~/.backup_config/backup_full_vars`**
```bash
#!/usr/bin/env bash

# NOTE: profile variables are uppercase in this project
SRC_FOLDER=$HOME
ENV_FILES=(/etc/hosts)
VSCODE_PROFILE="base"
DEFAULT_DEVICE_NAME="Backup"
DEV_DEST_FOLDER="home"
```

**File: `~/.backup_config/backup_full_exclusion`**
```
.cache/
.var/
Downloads/large_files/
tech/vms/*
```

3. Make scripts executable:
```bash
chmod +x backup restore remote_sync
```

### Customization

**Add more environment files:**
Edit `backup_${PROFILE}_vars` and add to `env_files` array:
```bash
env_files=(
    /etc/hosts
    /etc/yum.repos.d
    $HOME/.ssh
    $HOME/.bashrc
    $HOME/Documents
)
```

**Create a new profile:**
Copy existing profile files and modify:
```bash
cp ~/.backup_config/backup_full_vars ~/.backup_config/backup_dev_vars
cp ~/.backup_config/backup_full_exclusion ~/.backup_config/backup_dev_exclusion
# Edit the dev files
```

**Add custom environment backups:**
Edit `backup_env_files()` in `backup` script to add custom commands, e.g.:
```bash
# Backup database
mysqldump -u root mydb > "$env_files_dest/mydb.sql"
log_info "Backed up MySQL database"
```

---

## Examples

### Scenario 1: Weekly backup to USB drive

```bash
# Insert USB drive (should mount automatically)
# USB appears at /run/media/chouaieb/Backup

./backup -p full
Device name [Backup]: 
[INFO] Starting backup...
[INFO] Completed backup

# Check backup
ls -lh /run/media/chouaieb/Backup/
backup_2025-11-15_14-30-45.log  home/  env_files/
```

### Scenario 2: Restore after system failure

```bash
# Mount backup device
sudo mount /dev/sdb1 /mnt/backup

# Preview restore
./restore -p full -n all

# Restore everything (with confirmation)
./restore -p full all
Device name [Backup]: /mnt/backup
This will overwrite files in /home/chouaieb. Proceed? [y/N] y
```

### Scenario 3: Migrate to new machine

```bash
# Copy backup to new machine
scp -r user@oldmachine:/path/to/backup /mnt/newbackup

# On new machine, restore
./restore -p full all
Device name [Backup]: /mnt/newbackup
[...all restored...]

# VS Code is now configured with same settings + extensions!
```

### Scenario 4: Sync project with remote server

```bash
# Get latest code from production server
./remote_sync deploy prod.example.com /opt/myapp ~/projects/myapp
Sync type? (get/send): get
Dry run depth? [3]: 
[preview shows changes]
Get files from prod.example.com? (y/N) y

# Send local changes to staging
./remote_sync deploy staging.example.com /opt/myapp ~/projects/myapp
Sync type? (get/send): send
Dry run depth? [3]: 
[preview shows changes]
Send files to staging.example.com? (y/N) y
```

---

## Advanced Features

### Dry-run everything

Always preview before applying:

```bash
# Backup dry-run (not directly supported, but can test rsync manually)
rsync -avn --exclude-from=~/.backup_config/backup_full_exclusion \
  $HOME/ /mnt/test_backup/

# Restore dry-run
./restore -p full -n all
```

### Examine logs

Backups and restores create detailed logs:

```bash
# Latest backup log
tail -50 /run/media/chouaieb/Backup/backup_*.log

# Latest restore log
tail -50 /run/media/chouaieb/Backup/restore_*.log
```

### Restore individual files

Restore only specific env files:

```bash
# Manually restore SSH keys
rsync -av /run/media/chouaieb/Backup/env_files/.ssh/ ~/.ssh/
```

### Schedule automated backups

Use cron to run backups automatically:

```bash
# Edit crontab
crontab -e

# Example: backup every Sunday at 2 AM
0 2 * * 0 /home/chouaieb/tech/projects/sys_utilities/backup_sync/backup -p full
```

**Note:** May need to handle device mounting/unmounting in cron context.

---

## Troubleshooting

### Device not found

```
ERROR | Backup device "Backup" not found at /run/media/chouaieb/Backup
```

**Solutions:**
1. Check if device is mounted: `mount | grep Backup`
2. Manually mount: `sudo mount /dev/sdb1 /mnt/backup`
3. Update device name in prompt

### rsync: command not found

Install rsync:
```bash
# Ubuntu/Debian
sudo apt-get install rsync

# Fedora/RHEL
sudo dnf install rsync

# macOS
brew install rsync
```

### Permission denied on env files

```
rsync: send_files failed to open "...": Permission denied
```

**Solutions:**
1. Run with appropriate permissions: `sudo ./backup -p full`
2. Check file permissions: `ls -l ~/.ssh/`
3. Add user to group: `sudo usermod -aG group $USER`

### Log file says "backup_.log"

**Old bug (fixed):** This was caused by `unset timestamp` being called too early. Update to the latest version.

### Restore prompts for files but doesn't restore

Check the manifest:
```bash
cat /run/media/chouaieb/Backup/env_files/env_files.manifest
hosts|/etc/hosts
.bashrc|/home/chouaieb/.bashrc
```

If manifest is empty or missing, restore falls back to profile (backward compatible).

### VS Code settings not restoring

```
ERROR | VS Code user backup not found at: /run/media/chouaieb/Backup/env_files/vscode/User
```

**Solutions:**
1. Verify backup was created: `ls /run/media/chouaieb/Backup/env_files/vscode/`
2. Re-run backup: `./backup -p full`
3. Check for VS Code on system: `which code`

### Extensions not installing

```
ERROR | Failed to install ms-python.python
```

**Solutions:**
1. Verify VS Code is installed: `code --version`
2. Check internet connection (extensions download from marketplace)
3. Check extensions list: `cat /run/media/chouaieb/Backup/env_files/vscode/code_extension_list`
4. Install manually: `code --install-extension ms-python.python`

---

## License & Credits

These scripts are part of the `sys_utilities` collection for personal system administration.

Built with `rsync`, `bash`, and `code` CLI.
