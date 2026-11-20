# Raspberry Pi Automated Backups and Disaster Recovery

## Overview

This guide documents an automated, encrypted backup workflow for the Raspberry Pi 5 (FrankiesPi). The Pi backs up selected service data and configuration files to an SMB share hosted by the Unraid NAS, using Restic for encrypted, incremental snapshots.

The workflow is designed around an important homelab constraint: Unraid is not powered on continuously. A systemd automount allows the Pi to boot without depending on the NAS being available. The backup script checks whether Unraid is reachable and skips the run cleanly when the NAS is offline.

This is a configuration and application-data backup, not a full bare-metal disk image.

## Architecture

```text
Raspberry Pi 5 (FrankiesPi)
  |
  +-- systemd weekly timer (Sunday, 02:00 local time)
        |
        +-- raspberry-pi-weekly-backup.service
              |
              +-- raspberry-pi-weekly-backup.sh
                    |
                    +-- Check Unraid reachability
                    +-- Trigger and verify SMB automount
                    +-- Refresh application/configuration staging files
                    +-- Create encrypted incremental Restic snapshot
                          |
                          +-- SMB share on Unraid
                                |
                                +-- Restic repository
```

## Backup destination and storage

- NAS: Unraid
- SMB share: `raspberry-pi-backups`
- Pi mount point: `/mnt/unraid-backups`
- Restic repository: `/mnt/unraid-backups/restic-repository`
- Local staging directory: `/var/backups/raspberry-pi`

The SMB share is mounted on demand by systemd. The `_netdev`, `nofail`, `x-systemd.automount`, and idle-timeout options help prevent an unavailable NAS from blocking normal Pi startup and allow the mount to become idle after use.

Example `/etc/fstab` entry (credentials are stored separately):

```fstab
//UNRAID_IP/raspberry-pi-backups /mnt/unraid-backups cifs credentials=/root/.smbcredentials-unraid,vers=3.0,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=300 0 0
```

Replace `UNRAID_IP` with the NAS address used on the local network. Do not put an SMB password directly in `/etc/fstab`.

## What is backed up

The script refreshes the staging directory before creating each snapshot.

| Component | Backup method | Staged content |
|---|---|---|
| Uptime Kuma | SQLite online backup using `.backup` | Consistent copy of `kuma.db` |
| Pi-hole | Built-in FTL Teleporter export | Configuration archive (ZIP) |
| CUPS | Attribute-preserving directory copy | `/etc/cups`, including printer configuration and related files |
| Health reporting | File copies | Push environment file, systemd service/timer, and Python script |
| Uptime Kuma deployment | File copy | Docker Compose file |

The live Uptime Kuma SQLite database is not copied by simply copying the database and WAL files. The script uses SQLite's online backup command to create a consistent database copy while the service is running.

The Pi-hole Teleporter ZIP is tested before Restic runs. If the export or integrity test fails, the script exits instead of creating a new snapshot from an incomplete staging set.

The Pi-hole Teleporter export is a configuration backup; this workflow does not claim to preserve the complete DNS query history.

## Security

- Restic encrypts repository data using a password stored locally in `/root/.restic-password`.
- The Restic password must also be kept in a separate, secure location outside the Pi and the Unraid backup repository. Losing the password means the encrypted repository cannot be decrypted.
- SMB credentials are stored in `/root/.smbcredentials-unraid`.
- Both credential files are restricted to root (mode `600`).
- The staging directory `/var/backups/raspberry-pi` is owned by root and has mode `700`; it can contain private configuration, certificates, and monitoring credentials.
- Never commit credential files, real Push URLs, passwords, private keys, or unredacted secrets to Git.

## 1. Install and initialize Restic

Install Restic from the Debian package repository:

```bash
sudo apt install restic
restic version
```

Create a root-only password file, then initialize the repository on the mounted Unraid share:

```bash
sudo chmod 600 /root/.restic-password
sudo restic -r /mnt/unraid-backups/restic-repository \
  --password-file /root/.restic-password init
```

The password itself is intentionally not documented or stored in this repository.

## 2. Prepare the staging directory

The staging directory is restricted to root because it contains sensitive service configuration:

```bash
sudo install -d -o root -g root -m 700 /var/backups/raspberry-pi
```

The weekly script refreshes the staged application and configuration files before invoking Restic.

## 3. Weekly backup script

The script is installed at:

```text
/usr/local/sbin/raspberry-pi-weekly-backup.sh
```

Its workflow is:

1. Check whether the Unraid host responds to ping.
2. Skip the run successfully if Unraid is offline.
3. Trigger the systemd automount with a bounded timeout.
4. Verify that the expected CIFS share is mounted at the backup path.
5. Refresh the Uptime Kuma SQLite backup.
6. Generate a Pi-hole Teleporter archive and validate the ZIP.
7. Refresh the CUPS and selected system configuration copies.
8. Create an encrypted Restic snapshot tagged `weekly`.

The script uses strict Bash error handling so a failed preparation step prevents Restic from creating a new snapshot.

Check script syntax:

```bash
sudo bash -n /usr/local/sbin/raspberry-pi-weekly-backup.sh
```

## 4. systemd service and timer

The oneshot service is stored at:

```text
/etc/systemd/system/raspberry-pi-weekly-backup.service
```

```ini
[Unit]
Description=Weekly Raspberry Pi Backup to Unraid
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/raspberry-pi-weekly-backup.sh
```

The timer is stored at:

```text
/etc/systemd/system/raspberry-pi-weekly-backup.timer
```

```ini
[Unit]
Description=Weekly Raspberry Pi Backup Schedule

[Timer]
OnCalendar=Sun *-*-* 02:00:00
Persistent=true
Unit=raspberry-pi-weekly-backup.service

[Install]
WantedBy=timers.target
```

Enable the timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now raspberry-pi-weekly-backup.timer
```

The timer is configured for Sundays at 02:00 in the Pi's local timezone. The Pi uses `America/Los_Angeles`.

`Persistent=true` allows systemd to trigger a missed calendar event after the Pi starts. If the Pi is running but Unraid is powered off at the scheduled time, the script skips that run; it does not continuously retry until the NAS becomes available. The next scheduled weekly run will try again.

Verify the timer:

```bash
systemctl status raspberry-pi-weekly-backup.timer --no-pager
systemd-analyze calendar 'Sun *-*-* 02:00:00'
```

## 5. Manual execution and log review

Run the backup manually for a controlled test:

```bash
sudo /usr/local/sbin/raspberry-pi-weekly-backup.sh
```

After systemd has run the service, inspect its journal:

```bash
sudo journalctl -u raspberry-pi-weekly-backup.service --no-pager
```

Check the timer's next execution:

```bash
systemctl list-timers raspberry-pi-weekly-backup.timer --all
```

## 6. Verify snapshots and repository integrity

List saved snapshots:

```bash
sudo restic -r /mnt/unraid-backups/restic-repository \
  --password-file /root/.restic-password snapshots
```

Check repository structure and data integrity:

```bash
sudo restic -r /mnt/unraid-backups/restic-repository \
  --password-file /root/.restic-password check
```

A successful repository check ends with:

```text
no errors were found
```

## 7. Restore test

A restore test was completed using the first snapshot:

```bash
sudo restic -r /mnt/unraid-backups/restic-repository \
  --password-file /root/.restic-password \
  restore SNAPSHOT_ID --target /var/tmp/restic-restore-test
```

Replace `SNAPSHOT_ID` with the desired snapshot ID from the `snapshots` command.

The test restored 45 files and directories, including CUPS configuration and PPDs, the Uptime Kuma database, the Pi-hole Teleporter archive, and the selected health-monitoring and deployment files. The temporary restore directory was inspected and then removed.

A restore test is important: a successful backup command alone does not prove that the data can be recovered.

## 8. Verified implementation results

Initial implementation and manual test results (November 19, 2026):

| Verification | Result |
|---|---|
| Restic repository initialized | Successful |
| First configuration snapshot | `f24db75a` (`system-config`) |
| First repository integrity check | No errors |
| Restore test | 45 files/directories restored |
| Weekly script manual execution | Successful |
| Pi-hole Teleporter ZIP validation | Passed |
| Second incremental snapshot | `3765f40e` (`weekly`) |
| Second repository integrity check | No errors |
| systemd service and timer validation | Passed |
| Weekly timer | Enabled and active |
| Scheduled execution | Sunday at 02:00 Pacific |

The manual weekly run used the first snapshot as its parent and added approximately 168.5 KiB of compressed repository data. Restic reported 27 changed files and one new file in the staging set.

<img width="854" height="295" alt="Screenshot 2026-11-19 at 4 27 11 PM" src="https://github.com/user-attachments/assets/6bb9bfd3-da48-4b3a-a1fd-add058c79c8c" />


## Operational considerations

- Unraid must be powered on and reachable at the scheduled time for a backup to occur.
- If Unraid is offline, the script logs a skip and exits without attempting Restic operations against an unavailable share.
- Keep an independent copy of the Restic password. Do not store the only copy on the Pi or in the repository it unlocks.
- This workflow backs up selected service data and configuration; it is not a complete operating-system image.
- Review snapshots and perform periodic restore tests as the configuration evolves.
- No automatic snapshot-retention policy is configured in this implementation.

## Skills demonstrated

- Linux system administration on Debian
- SMB/CIFS networking and credential-file permissions
- systemd automounts, oneshot services, and calendar timers
- Bash scripting and error handling
- SQLite online database backups
- Pi-hole configuration export and archive validation
- CUPS configuration backup
- Restic encrypted, incremental backups
- Repository integrity verification
- File restoration and disaster-recovery testing
- Secure handling of credentials and sensitive configuration
- Operational logging and technical documentation
