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
- Optional secondary encrypted backup mirror
- No LUKS passphrases or encryption keys stored by the backup script

## How It Works

The system has three main components:

1.  **LUKS** encrypts the removable drive.
2.  **systemd** detects changes to the user's removable-media directory after the drive is unlocked and mounted.
3.  **rsync** creates a new timestamped snapshot.

An optional mirror adds a second encrypted backup filesystem:

``` text
laptop -> primary encrypted backup -> secondary encrypted backup
```

The mirror copies the completed snapshot structure from the primary backup to
the secondary backup. It preserves hard links, but deliberately does not use
`rsync --delete`: files and snapshots removed from the primary backup remain on
the secondary backup until you remove them there intentionally.

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

## Snapshot Retention

By default, the backup system keeps the newest 30 successful snapshots.

After a new backup completes successfully, snapshots older than the configured retention limit are automatically removed.

The retention limit can be changed with the `MAX_SNAPSHOTS` environment variable:

```text
MAX_SNAPSHOTS=60

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
git clone https://github.com/RedVision100/encrypted-linux-backup.git
cd encrypted-linux-backup
```

Install the backup script:

``` bash
mkdir -p ~/.local/bin
cp bin/laptop-backup ~/.local/bin/laptop-backup
chmod +x ~/.local/bin/laptop-backup
```

To install the optional secondary-backup mirror script:

``` bash
cp bin/backup-mirror ~/.local/bin/backup-mirror
chmod +x ~/.local/bin/backup-mirror
```

Install the systemd user units:

``` bash
mkdir -p ~/.config/systemd/user
cp systemd/laptop-backup.service ~/.config/systemd/user/
cp systemd/laptop-backup.path ~/.config/systemd/user/
```

The optional mirror uses separate generic user units:

``` bash
cp systemd/backup-mirror.service ~/.config/systemd/user/
cp systemd/backup-mirror.path ~/.config/systemd/user/
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

## Secondary Backup Mirror

`backup-mirror` mirrors a primary encrypted backup drive onto a distinct
secondary encrypted backup drive. It is intended to run only after both drives
are unlocked and mounted.

Create `~/.config/backup-mirror.env` with your own mountpoints and filesystem
UUIDs. Do not add this file to the repository.

``` ini
SOURCE_MOUNT=/path/to/primary-backup
DEST_MOUNT=/path/to/secondary-backup
SOURCE_UUID=PRIMARY_FILESYSTEM_UUID
DEST_UUID=SECONDARY_FILESYSTEM_UUID
```

All four values are required. Both mount paths must be absolute. Find each
filesystem UUID after mounting it:

``` bash
findmnt -no UUID /path/to/primary-backup
findmnt -no UUID /path/to/secondary-backup
```

The mirror script refuses to run unless both configured mountpoints are
mounted, their UUIDs match, and source and destination are different locations
and devices. It preserves snapshot hard links with `rsync -H`, excludes each
filesystem's `lost+found`, and excludes the primary `latest` symlink. It
requires the resolved source `snapshots` directory to be directly under the
resolved source mount, then validates that the primary `latest` symlink refers
to one of those snapshot directories before creating a relative `latest`
symlink on the secondary backup.

The mirror does not use `rsync --delete`. Deletions from the primary backup are
intentionally not propagated to the secondary backup.

### Enable the Mirror

`backup-mirror.service` checks both configured mountpoints every two seconds
for up to 60 seconds so a desktop mount/unlock race does not start the script
too early. UUID validation inside `backup-mirror` remains the authoritative
safety check.

The path unit watches the common per-user removable-media directory
`/run/media/%u`. If your system mounts removable filesystems elsewhere, edit
the installed `PathChanged` directory to match the parent directory containing
your configured mountpoints.

``` bash
systemctl --user daemon-reload
systemctl --user enable --now backup-mirror.path
systemctl --user status backup-mirror.path
```

Run it manually after mounting both filesystems:

``` bash
SOURCE_MOUNT=/path/to/primary-backup \
DEST_MOUNT=/path/to/secondary-backup \
SOURCE_UUID=PRIMARY_FILESYSTEM_UUID \
DEST_UUID=SECONDARY_FILESYSTEM_UUID \
~/.local/bin/backup-mirror
```

View logs with:

``` bash
journalctl --user -u backup-mirror.service -n 50 --no-pager
```

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

To restore ordinary files from either encrypted backup, copy from its `latest/`
directory or from a chosen older directory under `snapshots/`. For example:

``` bash
cp -a /path/to/backup/latest/Documents/example.txt ~/Documents/
```

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

This project is a small Linux backup utility. Test it with your own environment and data before relying on it as your only backup.
