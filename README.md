# Backup and Synchronization Scripts

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [Backup and Synchronization Scripts](#backup-and-synchronization-scripts)
    - [backup](#backup)
    - [remote_sync](#remote_sync)

<!-- /code_chunk_output -->

This repository contains wrapper scripts for the rsync tool.
**Tools:**

- **backup:** are for file backups on storage devices
- **remote_sync:** file synchronizations between hosts

---

### backup

This tool allowes you to define multiple backup profiles backs up files based on them.
Backup profiles can be customized to suit different needs, for example, targeting different backup devices, backup specific folders/files and so on.

**Usage:**

```bash
backup [-h] [-p PROFILE]
```

**Configuration and Setup:**

This tool is meant to be flexible and allows a good degree of customization.
Each backup profile has:

- **one source folder**
- **one destination folder**
- **multiple environment files**/folders (files to backup outside the source backup folder.)

These information can be defined in configuration files under the `$backup_conf` configuration folder, this fodler is defined as a variable that can/should be modified in the `backup` script.

Custom backup commands should be added to the `backup_env_files()` function in the `backup` script. for example:

- backing up vscode extension list using the command: `code --list-extensions --profile $vscode_profile`

Under `$backup_conf` there should be 2 files:

- `backup_[PROFILE]_vars`

  - a shell script
  - executed to define variables needed in the `backup script execution`
  - these vars are specific to each backup profile, example:
    - source folder
    - destination folder
    - environment files
    - Backup device name and path
    - other variables needed in running custom backup commands.
  - example `backup_[PROFILE]_vars` file:

    ```bash
    #!/usr/bin/env bash

    echo

    # src and exclusion files
    src_folder=$HOME
    env_files=(
        /etc/hosts
        $HOME/.bashrc
        $HOME/.bashrc.d
        $HOME/.bash_profile
        $HOME/.ssh
        $HOME/.backup_config
    )

    # dest and device paths
    default_device_name="Backup"
    read -p "Device name [$default_device_name]: " device_name
    device_name=${device_name:-"$default_device_name"}
    device_path="/run/media/$USER/$device_name"
    dest_folder="$device_path/home"

    vscode_profile="base"
    ```

- `backup_[PROFILE]_exclusion`
  - defines files and/or folders that are excluded from the backup source.
  - conforms to the `rsync --exclude-from=FILE` command FILE format
    - for more on this format, execute `man rsync` and go to the section talking about the `--exclude-from` option

---

### remote_sync

This script syncs files from/to a remote host using the rsync tool. Before sending or fetching files it will first perform a dry run and display would-be-copied files with a custom folder tree depth.

After the dry run, User confirmation will be asked to synchronize files.

**Usage:**

```bash
remote_sync <REMOTE_USER> <REMOTE_HOST> <REMOTE_DIR> <LOCAL_DIR>
```
