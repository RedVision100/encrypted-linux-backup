# Encrypted Linux Backup

A simple Linux backup system for encrypted removable drives using Bash, rsync, LUKS, and systemd.

The backup runs automatically after an encrypted backup drive is unlocked and mounted. Each successful backup creates a timestamped snapshot while hard-linking unchanged files to the previous snapshot, providing multiple restore points without storing a complete duplicate every time.

## Features

- Automatic backup when the removable drive is mounted
- Designed for LUKS-encrypted backup drives
- Timestamped filesystem snapshots
- Hard-link deduplication with `rsync --link-dest`
- Optional filesystem UUID verification
- Dry-run mode
- Concurrent-run protection with `flock`
- Automatic cleanup of incomplete snapshots
- `latest` symlink pointing to the newest successful snapshot
- Selective backup of files and directories
- No LUKS passphrases or encryption keys stored by the backup script

## How It Works

The system has three main components:

1.  **LUKS** encrypts the removable drive.
2.  **systemd** detects changes to the user's removable-media directory after the drive is unlocked and mounted.
3.  **rsync** creates a new timestamped snapshot.

A typical workflow is:

``` text
Connect drive
    ↓
Unlock LUKS volume
    ↓
Filesystem mounts
    ↓
systemd detects mount-directory change
    ↓
laptop-backup runs
    ↓
New snapshot created
    ↓
latest points to successful snapshot
```

## Snapshot Layout

Example:

``` text
LAPTOP_BACKUP/
├── latest -> snapshots/2026-09-27_16-54-10
└── snapshots/
    ├── 2026-09-27_16-13-25/
    ├── 2026-09-27_16-20-27/
    └── 2026-09-27_16-54-10/
```

Each snapshot looks like an independent backup. Unchanged regular files can share disk blocks through hard links, so subsequent snapshots generally consume space only for changed or newly added data.

Deleting an old snapshot does not delete unchanged files that are still hard-linked from another snapshot.

## Requirements

- Linux
- Bash
- `rsync`
- `findmnt`
- `mountpoint`
- `flock`
- systemd user services
- A mounted backup filesystem

LUKS/`cryptsetup` is recommended when the backup contains private data.

## Installation

Clone the repository:

``` bash
git clone YOUR_REPOSITORY_URL
cd encrypted-linux-backup
```

Install the backup script:

``` bash
mkdir -p ~/.local/bin
cp bin/laptop-backup ~/.local/bin/laptop-backup
chmod +x ~/.local/bin/laptop-backup
```

Install the systemd user units:

``` bash
mkdir -p ~/.config/systemd/user
cp systemd/laptop-backup.service ~/.config/systemd/user/
cp systemd/laptop-backup.path ~/.config/systemd/user/
```

## Configuration

The script defaults to:

``` text
Backup label: LAPTOP_BACKUP
Mount point:  /run/media/$USER/LAPTOP_BACKUP
Source:       $HOME
```

These can be overridden with environment variables:

``` text
BACKUP_LABEL
BACKUP_DRIVE
BACKUP_UUID
SOURCE
```

### Filesystem UUID

Using the filesystem UUID prevents the script from writing a backup to the wrong mounted filesystem.

Find the UUID after mounting the backup drive:

``` bash
findmnt -no UUID /run/media/$USER/LAPTOP_BACKUP
```

Then edit:

``` text
~/.config/systemd/user/laptop-backup.service
```

and replace:

``` ini
Environment=BACKUP_UUID=YOUR_FILESYSTEM_UUID
```

with the actual filesystem UUID.

Do not use a LUKS passphrase here. `BACKUP_UUID` is only the filesystem identifier.

## Choose What Gets Backed Up

Edit the `BACKUP_ITEMS` array in:

``` text
~/.local/bin/laptop-backup
```

For example:

``` bash
BACKUP_ITEMS=(
    "Projects"
    "Documents"
    "Pictures"
    "Music"
    ".config"
)
```

Paths are relative to the source directory.

## Dry Run

Before the first real backup:

``` bash
~/.local/bin/laptop-backup --dry-run
```

A dry run performs rsync's comparison without copying files.

## Enable Automatic Backups

Reload the user systemd configuration:

``` bash
systemctl --user daemon-reload
```

Enable the mount-directory watcher:

``` bash
systemctl --user enable --now laptop-backup.path
```

Check it:

``` bash
systemctl --user status laptop-backup.path
```

It should report:

``` text
Active: active (waiting)
```

When the encrypted drive is subsequently unlocked and mounted, the backup service can run automatically.

## Check Backup Status

View the service status:

``` bash
systemctl --user status laptop-backup.service
```

View recent backup logs:

``` bash
journalctl --user -u laptop-backup.service -n 50 --no-pager
```

## Manual Backup

A backup can also be started manually:

``` bash
systemctl --user start laptop-backup.service
```

or:

``` bash
~/.local/bin/laptop-backup
```

## Restoring Files

No special restore program is required.

Browse the desired snapshot:

``` bash
cd /run/media/$USER/LAPTOP_BACKUP/snapshots
```

Files can be copied back with standard tools such as `cp` or `rsync`.

For example:

``` bash
rsync -a \
  /run/media/$USER/LAPTOP_BACKUP/latest/Documents/ \
  ~/Documents/
```

Always inspect the source and destination before performing a large restore.

## Encryption

This project does **not** create, unlock, or manage LUKS encryption.

Encryption should be configured separately using your Linux distribution's disk-management tools or `cryptsetup`.

The backup script does not need and should never contain your LUKS passphrase.

## Security

Backups may contain sensitive data such as SSH configuration, application data, shell history, and cloud configuration.

For that reason:

- Use full-disk or LUKS encryption on the backup device.
- Never commit backup contents to this repository.
- Never commit passwords, private keys, recovery phrases, API tokens, or encryption keys.
- Verify the destination filesystem before running backups.
- Test restoration periodically.

## Status

This project is currently a small personal Linux backup utility. Test it with your own environment and data before relying on it as your only backup.
