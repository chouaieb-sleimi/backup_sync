# Backup and Synchronization Scripts

This repository contains wrapper scripts for the rsync tool. These scripts are for file backups on storage devices or file synchronizations between hosts.

### backup

This script backs up files in 2 modes, **full** and **lite**. Files defined in *exclusion_file* var are excluded. The rsync tool to perform the backup.
Backup successes and errors are logged to *log_success* and *log_error* files under the backup device.

#### Usage

    backup [type]

    Backup types:
    - full
    - lite

### remote_sync

This script syncs files from/to a remote host using the rsync tool.

#### Usage

    remote_sync <USER> <REMOTE_HOST> <REMOTE_DIR> <LOCAL_DIR>
