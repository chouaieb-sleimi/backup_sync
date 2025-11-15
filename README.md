# backup_sync: Backup and Synchronization Scripts

Profile-based backup and synchronization tools built on `rsync`. Supports custom and multiple profiles, environment file preservation, and portable restores.

**Contents:**

- [Quick Start](#quick-start)
- [backup](#backup)
- [restore](#restore)
- [remote_sync](#remote_sync)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

```bash
# Backup (make sure to create/review config files first)
./backup -p full        # Full home directory backup
./backup -p lite        # Lite backup (tech folder only)

# Restore (make sure to review config files first)
./restore -p full all                    # Restore everything
./restore -p full vscode                 # Restore just VS Code settings
./restore -p full -n all                 # Preview restore (dry-run)

# Remote sync
./remote_sync user host /remote/path /local/path
```

---

## backup

Backs up files based on backup profiles. Each profile defines source folder, destination device, and environment files.

```bash
backup [-h] [-p PROFILE]

Options:
  -p PROFILE    Backup profile name (required)
  -h            Display help message
```

**What gets backed up:**

- System files (RPM package list + yum repos)
- VS Code settings and extensions list
- Main source folder (respects exclusion patterns)
- Files outside main source dir (env files)
- Backup configuration (for portable restores)

**How it works:**

1. **Profile Selection** — Load configuration from `$BACKUP_CONF/backup_${profile}_vars` and exclusion rules
2. **Device Prompt** — Ask for device name (USB drive, external HD, etc.)
3. **Environment Backup** — Back up files outside main source dir (repos, packages, VS Code, SSH keys, etc.)
4. **Config Packaging** — Copy the local configuration directory (`$BACKUP_CONF`, default `~/.backup_config/`) to the device under `$DEVICE_PATH/backup_config/`. This makes the backup self-contained: `restore` can read profile variables directly from the device if present.
5. **Main Backup** — rsync source folder to device, excluding specified patterns
6. **Logging** — Write detailed log to `backup_$TIMESTAMP.log`

**Configuration:**

Profiles are defined in `~/.backup_config/` with two files per profile:

1. `backup_${PROFILE}_vars` - Variables: `DEFAULT_DEVICE_NAME`, `DEV_DEST_FOLDER`, `VSCODE_PROFILE`, `SRC_FOLDER`, `ENV_FILES`
2. `backup_${PROFILE}_exclusion` - Exclusion patterns (rsync format)
   > see `samples/` for backup config samples

**Output structure:**

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

---

## restore

Restores files from backup devices with granular control.

```bash
restore -p PROFILE [-n] ACTION

Options:
  -p PROFILE    Backup profile name (required)
  -n            Dry-run mode (preview without making changes)

Actions:
  env           Restore environment files
  vscode        Restore VS Code settings
  extensions    Install VS Code extensions
  config        Restore backup config (writes to $BACKUP_CONF_DEFAULT)
  src           Restore main source folder
  all           Restore everything
```

**Features:**

- Dry-run mode to preview any restore without making changes
- Prompts before destructive operations
- Self-contained (reads config from backup device if available)
  - Uses `env_files.manifest` for portable restores

---

## remote_sync

Syncs files to/from remote hosts with preview.

```bash
remote_sync <USER> <HOST> <REMOTE_DIR> <LOCAL_DIR>
```

**Interactive:**

- Choose sync direction (get/send)
- Preview changes with configurable depth
- Confirm before syncing

---

## Configuration

1. Create config directory: `mkdir -p ~/.backup_config`
2. Create profile files:
   - `backup_${PROFILE}_vars` (variables)
   - `backup_${PROFILE}_exclusion` (exclusion patterns)
3. Make scripts executable: `chmod +x backup restore remote_sync`

## Troubleshooting

**Device not found:** Check if mounted (`mount | grep Backup`) or mount manually

**rsync not found:** Install with package manager (`dnf install rsync`)

**Permission denied:** Run with `sudo` or check file permissions

**VS Code issues:** Verify VS Code is installed and check backup paths
