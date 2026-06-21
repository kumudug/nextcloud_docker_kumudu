# Design: rsnapshot backup of Dockerized Nextcloud (user files)

Date: 2026-06-21

## Goal

Replace the old native-Nextcloud rsync/rsnapshot backup (documented in
`16_sync-data-from-nextcld.md`) with an equivalent approach for the new
**Dockerized** Nextcloud stack (`compose.yaml`). Pull the Nextcloud **user
files** to the gembi file server over SSH and keep **daily / weekly / monthly**
rsnapshot snapshots there.

## Scope decision

**User files only** — same as the previous setup. This is a file-level backup
for retrieving documents/photos, *not* a full instance restore. The MariaDB
database and `config.php` are intentionally **out of scope**; there is no DB to
reconcile on restore.

## Topology

```
┌──────────────────────────────┐      SSH pull (key, port 28)   ┌──────────────────────────┐
│ Docker host (Arch) 192.168.4.2│  <──── read-only ───────────── │ gembi file server (Arch) │
│ Nextcloud Docker stack         │                                │ rsnapshot runs here      │
│ /data/nextcloud/nextcloud_data │                                │ snapshot_root:           │
│   └ <user>/files/  (uid/gid 33)│                                │   /data/nextcld_docker/  │
│ syncusr ∈ gid-33 group (ro)    │                                │ syncusr owns the SSH key │
└──────────────────────────────┘                                └──────────────────────────┘
```

rsnapshot on gembi pulls each user's `…/files/` over SSH and keeps hardlinked
daily/weekly/monthly snapshots locally. Same model as the old doc; the
differences are the **source path** and the **read-access mechanism** (Docker
ownership instead of the old `syncgrp`/`sgid` dance).

## Parameters (confirmed)

| Parameter            | Value                                             |
|----------------------|---------------------------------------------------|
| Docker host          | `192.168.4.2`, SSH port `28`, admin login `kumudu`|
| Docker host OS       | Arch Linux                                         |
| Data bind-mount path | `/data/nextcloud/nextcloud_data`                  |
| Nextcloud users      | `kumudu`, `kumudu_amazon`                          |
| gembi OS             | Arch Linux                                         |
| `snapshot_root`      | `/data/nextcld_docker/`                            |
| Scheduler            | systemd timers                                     |
| Retention            | 7 daily / 4 weekly / 6 monthly                     |
| Read access          | Option A — group membership on gid 33              |

## Read-access approach (Option A)

Nextcloud in the container writes data as uid/gid **33** with group-readable
perms (dirs `0750`, files `0640`). On the Docker host, `syncusr` is added to the
group that owns gid 33, so it can **read** the data without any change to
ownership or perms — Nextcloud keeps writing as 33, and access is read-only by
construction. This is the Docker-native equivalent of the old `syncgrp` trick,
but simpler: **no `chown`, no `sgid`, no fighting the container over ownership.**

Rejected alternatives:
- **Option B (passwordless sudo rsync):** grants `syncusr` root-equivalent read,
  the broad privilege the old doc deliberately avoided.
- **Option C (`docker exec` / read from inside the container):** more moving
  parts, couples backup to container lifecycle, breaks when the stack is down.

Gotcha addressed: every parent dir on the path to `nextcloud_data` must be
traversable by `syncusr`. `/data` is traversable, so no fix is needed here.

## Source-side setup (Docker host, Arch)

```bash
# 1. Group pinned to gid 33 (Nextcloud container www-data); syncusr in it
sudo groupadd -g 33 ncdata          # or reuse the existing gid-33 group if one exists
sudo useradd -g users -G ncdata -m -s /bin/bash syncusr
sudo passwd syncusr

# 2. SSH key trust (gembi's syncusr pubkey installed here)
sudo install -d -m 700 -o syncusr -g users /home/syncusr/.ssh
#   append gembi's syncusr public key into /home/syncusr/.ssh/authorized_keys, then:
sudo chown syncusr:users /home/syncusr/.ssh/authorized_keys
sudo chmod 600 /home/syncusr/.ssh/authorized_keys

# 3. Allow over SSH
#   /etc/ssh/sshd_config -> AllowUsers kumudu syncusr   (drop any AllowGroups line)
sudo systemctl restart sshd
```

## Destination-side setup (gembi, Arch)

Install rsnapshot (`sudo pacman -S rsnapshot`). `syncusr`'s private key lives at
`/home/syncusr/.ssh/id_rsa` (matching the pubkey trusted on the Docker host).

`/etc/rsnapshot.conf` — **fields separated by TABS, not spaces**:

```
snapshot_root   /data/nextcld_docker/
cmd_rsync       /usr/bin/rsync
cmd_ssh         /usr/bin/ssh

retain  daily    7
retain  weekly   4
retain  monthly  6

ssh_args        -i /home/syncusr/.ssh/id_rsa -p 28
rsync_long_args --delete --numeric-ids --relative --usermap=*:syncusr --groupmap=*:users

backup  syncusr@192.168.4.2:/data/nextcloud/nextcloud_data/kumudu/files/         nextcld/kumudu/
backup  syncusr@192.168.4.2:/data/nextcloud/nextcloud_data/kumudu_amazon/files/  nextcld/kumudu_amazon/
```

Notes:
- `--numeric-ids` — uid/gid 33 has no name on either box.
- `--usermap/--groupmap` — remaps the stored copy to `syncusr:users` so the
  snapshots are browsable, matching the old setup.
- rsnapshot runs as **root** on gembi (needed for hardlinks and the remap).
- Run `sudo rsnapshot configtest` after editing.

## Scheduling — systemd timers (root) on gembi

Three oneshot services + timers. Higher intervals fire at **earlier** times so
rsnapshot rotation promotes a snapshot before the lower interval rotates.

`rsnapshot-daily.service` (and `-weekly`, `-monthly` analogous):

```ini
[Unit]
Description=rsnapshot daily snapshot of Nextcloud data

[Service]
Type=oneshot
ExecStart=/usr/bin/rsnapshot daily
```

Timers:

```ini
# rsnapshot-daily.timer
[Timer]
OnCalendar=*-*-* 03:00
Persistent=true

# rsnapshot-weekly.timer
[Timer]
OnCalendar=Mon *-*-* 02:30
Persistent=true

# rsnapshot-monthly.timer
[Timer]
OnCalendar=*-*-01 02:00
Persistent=true
```

Enable: `sudo systemctl enable --now rsnapshot-{daily,weekly,monthly}.timer`.
`Persistent=true` catches up runs missed while the server was off. Logs go to
journald (`journalctl -u rsnapshot-daily`).

## Result and restore

- Rolling **7 daily + 4 weekly + 6 monthly** snapshots, hardlink-deduped under
  `/data/nextcld_docker/` (only changed files cost disk space).
- Layout: `/data/nextcld_docker/{daily,weekly,monthly}.N/nextcld/<user>/`.
- Restore = copy back from any snapshot dir (`cp`/`rsync`) — pure file restore,
  no database to reconcile.

## Verification before trusting it

1. Read access:
   `ssh -i /home/syncusr/.ssh/id_rsa -p 28 syncusr@192.168.4.2 'ls /data/nextcloud/nextcloud_data/kumudu/files/'`
2. Config: `sudo rsnapshot configtest`
3. First run: `sudo rsnapshot daily`, then inspect
   `/data/nextcld_docker/daily.0/nextcld/kumudu/`.
4. Confirm timers scheduled: `systemctl list-timers 'rsnapshot-*'`.

## Out of scope / future

- Full restorable backup (data + MariaDB dump + `config.php`) — could be a later
  enhancement if a bare-metal rebuild capability is wanted.
- Off-site copy of the snapshot_root.
