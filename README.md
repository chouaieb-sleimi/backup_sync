# Backup and Synchronization Scripts

This repository contains wrapper scripts for the rsync tool.
**Tools:**

- **backup:** are for file backups on storage devices
- **remote_sync:** file synchronizations between hosts

### backup

#### :bulb: Description

This tool backs up files based on different backup profiles. Each backup profile has one backup source and destination folders and multiple environment files outside the source and destination folders.

Each backup profile uses 2 variable files under the folder defined in `$backup_conf` variable; `backup_[PROFILE]_vars` and `backup_[PROFILE]_exclusion`.
`backup_[PROFILE]_vars` is a shell script that defines variables specific to each backup profile. `backup_[PROFILE]_exclusion` file defines files and/or folders that are excluded from the backup source.

`backup_[PROFILE]_vars` example:

    src_folder=$HOME
    env_files=(/etc/hosts)
    device_path=/run/media/dave/backupUSB
    dest_folder=$device_path/homeBackup

#### :wrench: Usage

    backup [-h] [-p PROFILE]

### remote_sync

#### :bulb: Description

This script syncs files from/to a remote host using the rsync tool. Before sending or fetching files it will first perform a dry run and display transfered files with a variabilised folder tree depth.

After the dry run, User confirmation will be asked to synchronize files.

#### :wrench: Usage

    remote_sync <REMOTE_USER> <REMOTE_HOST> <REMOTE_DIR> <LOCAL_DIR>
