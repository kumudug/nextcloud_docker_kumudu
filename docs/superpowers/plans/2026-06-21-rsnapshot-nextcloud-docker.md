# rsnapshot Backup of Dockerized Nextcloud — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Note:** This is an operations runbook executed by hand on two remote Arch
> Linux machines over SSH (with interactive `sudo`). There is no application
> code and no unit-test suite. Each task's "test" is a verification command with
> expected output. Do not skip the verification step — it is the gate.

**Goal:** Pull the Dockerized Nextcloud user files to the gembi file server over SSH and keep daily/weekly/monthly rsnapshot snapshots, driven by systemd timers.

**Architecture:** rsnapshot runs as root on the gembi server and pulls each Nextcloud user's `…/files/` directory over SSH from the Docker host. Read access on the Docker host is granted by adding a dedicated `syncusr` to the gid-33 group that owns the container-written data (no `chown`/`sgid` on the data). systemd timers fire the three rsnapshot intervals.

**Tech Stack:** rsnapshot, OpenSSH, systemd timers, rsync — all on Arch Linux.

## Global Constraints

- Docker host: `192.168.4.2`, SSH port `28`, admin login `kumudu`. Sync account: `syncusr`.
- Docker host data bind-mount: `/data/nextcloud/nextcloud_data` — files owned uid/gid **33**, perms dirs `0750` / files `0640`.
- Nextcloud users to back up: `kumudu` and `kumudu_amazon`.
- gembi `snapshot_root`: `/data/nextcld_docker/`.
- Retention: 7 daily / 4 weekly / 6 monthly.
- Read access is **read-only via gid-33 group membership** — never `chown` or change perms on the data directory.
- `rsnapshot.conf` fields are separated by **TABS**, never spaces.
- rsnapshot and the systemd units run as **root** on gembi.
- "Docker host" = `192.168.4.2`. "gembi" = the file server where snapshots live. Each step states which box it runs on.

---

### Task 1: Provision the syncusr SSH keypair on gembi

Create (or locate) the keypair that gembi's root-run rsnapshot will use to log in to the Docker host as `syncusr`. The private key stays on gembi; the public key is handed to Task 3.

**Files:**
- Create on gembi: `/home/syncusr/.ssh/id_rsa`, `/home/syncusr/.ssh/id_rsa.pub`

**Interfaces:**
- Produces: private key path `/home/syncusr/.ssh/id_rsa` (consumed by Task 5 `ssh_args`); public key contents (consumed by Task 3 `authorized_keys`).

- [ ] **Step 1: (gembi) Create the syncusr account that will hold the key**

```bash
sudo useradd -g users -m -s /bin/bash syncusr
```

- [ ] **Step 2: (gembi) Generate a dedicated keypair for syncusr**

```bash
sudo -u syncusr ssh-keygen -t ed25519 -N "" -f /home/syncusr/.ssh/id_rsa -C "syncusr@gembi rsnapshot nextcloud"
```

(Filename kept as `id_rsa` to match the `ssh_args` path in Task 5; the key type is ed25519.)

- [ ] **Step 3: (gembi) Verify the keypair exists and print the public key**

Run:
```bash
sudo ls -l /home/syncusr/.ssh/id_rsa /home/syncusr/.ssh/id_rsa.pub
sudo cat /home/syncusr/.ssh/id_rsa.pub
```
Expected: both files listed, private key mode `-rw-------`, and one `ssh-ed25519 …` line printed. **Copy that public-key line — Task 3 needs it.**

---

### Task 2: Create the gid-33 group and syncusr on the Docker host

Give the Docker host a `syncusr` whose group membership lets it read the container-owned data.

**Files:**
- Modify on Docker host: `/etc/group`, `/etc/passwd` (via `groupadd`/`useradd`)

**Interfaces:**
- Consumes: nothing.
- Produces: `syncusr` account on the Docker host, member of the gid-33 group (consumed by Task 4 read test and all backups).

- [ ] **Step 1: (Docker host) Check whether gid 33 is already assigned**

```bash
getent group 33
```
Expected: either no output (gid 33 free) or a line like `somegroup:x:33:`. Note the result for the next step.

- [ ] **Step 2: (Docker host) Ensure a group owns gid 33**

If Step 1 printed nothing, create one:
```bash
sudo groupadd -g 33 ncdata
```
If Step 1 printed an existing group (e.g. `http:x:33:`), skip `groupadd` and use that group's name in place of `ncdata` below.

- [ ] **Step 3: (Docker host) Create syncusr in the gid-33 group**

```bash
sudo useradd -g users -G ncdata -m -s /bin/bash syncusr
sudo passwd syncusr
```
(Replace `ncdata` with the existing gid-33 group name if you reused one.)

- [ ] **Step 4: (Docker host) Verify syncusr is in the gid-33 group**

Run:
```bash
id syncusr
```
Expected: output includes `33(ncdata)` (or the reused group name) among the groups.

---

### Task 3: Install SSH key trust and allow syncusr on the Docker host

Trust gembi's syncusr public key and permit the account to log in over SSH.

**Files:**
- Create on Docker host: `/home/syncusr/.ssh/authorized_keys`
- Modify on Docker host: `/etc/ssh/sshd_config`

**Interfaces:**
- Consumes: the public-key line from Task 1 Step 3.
- Produces: working key-based SSH login `syncusr@192.168.4.2 -p 28` (consumed by Task 4).

- [ ] **Step 1: (Docker host) Create the .ssh directory for syncusr**

```bash
sudo install -d -m 700 -o syncusr -g users /home/syncusr/.ssh
```

- [ ] **Step 2: (Docker host) Install the public key**

Append the `ssh-ed25519 …` line copied in Task 1 Step 3 into `/home/syncusr/.ssh/authorized_keys` (edit the file with `sudo`), then fix ownership/perms:
```bash
sudo chown syncusr:users /home/syncusr/.ssh/authorized_keys
sudo chmod 600 /home/syncusr/.ssh/authorized_keys
```

- [ ] **Step 3: (Docker host) Allow syncusr in sshd**

Edit `/etc/ssh/sshd_config`: ensure the `AllowUsers` line includes `syncusr` (e.g. `AllowUsers kumudu syncusr`). Remove any `AllowGroups` line that would otherwise exclude `syncusr`. Then:
```bash
sudo sshd -t && sudo systemctl restart sshd
```
Expected: `sshd -t` prints nothing (config valid) before the restart runs.

- [ ] **Step 4: (Docker host) Confirm the directives**

Run:
```bash
sudo sshd -T | grep -i '^allowusers'
```
Expected: a line listing `syncusr` (alongside `kumudu`).

---

### Task 4: Verify end-to-end read access (gate)

Prove that gembi's root, using the key, can log in as `syncusr` and read the data. This is the make-or-break checkpoint before touching rsnapshot.

**Files:** none.

**Interfaces:**
- Consumes: Tasks 1–3 outputs.
- Produces: confidence that the `backup` source paths in Task 5 will work.

- [ ] **Step 1: (gembi) SSH in as syncusr and list a user's files**

Run:
```bash
sudo ssh -i /home/syncusr/.ssh/id_rsa -p 28 syncusr@192.168.4.2 \
  'ls /data/nextcloud/nextcloud_data/kumudu/files/'
```
Expected: a directory listing of `kumudu`'s files (no `Permission denied`, no password prompt).

- [ ] **Step 2: (gembi) Confirm read access for the second user too**

Run:
```bash
sudo ssh -i /home/syncusr/.ssh/id_rsa -p 28 syncusr@192.168.4.2 \
  'ls /data/nextcloud/nextcloud_data/kumudu_amazon/files/'
```
Expected: a directory listing for `kumudu_amazon`.

- [ ] **Step 3: (gembi) Confirm it is read-only (write must be denied)**

Run:
```bash
sudo ssh -i /home/syncusr/.ssh/id_rsa -p 28 syncusr@192.168.4.2 \
  'touch /data/nextcloud/nextcloud_data/kumudu/files/__write_test 2>&1 || echo READONLY_OK'
```
Expected: `READONLY_OK` (the `touch` fails because `syncusr` only has group read, confirming the backup account cannot modify live data). If it instead succeeds, delete the file and investigate perms before continuing.

---

### Task 5: Install and configure rsnapshot on gembi

**Files:**
- Modify on gembi: `/etc/rsnapshot.conf`
- Create on gembi: `/data/nextcld_docker/` (snapshot_root)

**Interfaces:**
- Consumes: key path from Task 1; verified access from Task 4.
- Produces: a valid rsnapshot config defining `daily/weekly/monthly` and the two `backup` lines (consumed by Tasks 6–7).

- [ ] **Step 1: (gembi) Install rsnapshot and create the snapshot root**

```bash
sudo pacman -S --needed rsnapshot
sudo install -d -m 700 /data/nextcld_docker
```

- [ ] **Step 2: (gembi) Write /etc/rsnapshot.conf**

Replace the relevant lines so the file contains exactly these (back up the original first with `sudo cp /etc/rsnapshot.conf /etc/rsnapshot.conf.orig`). **Every gap shown below is a single TAB, not spaces:**

```
snapshot_root	/data/nextcld_docker/
cmd_rsync	/usr/bin/rsync
cmd_ssh	/usr/bin/ssh

retain	daily	7
retain	weekly	4
retain	monthly	6

ssh_args	-i /home/syncusr/.ssh/id_rsa -p 28
rsync_long_args	--delete --numeric-ids --relative --usermap=*:syncusr --groupmap=*:users

backup	syncusr@192.168.4.2:/data/nextcloud/nextcloud_data/kumudu/files/	nextcld/kumudu/
backup	syncusr@192.168.4.2:/data/nextcloud/nextcloud_data/kumudu_amazon/files/	nextcld/kumudu_amazon/
```

- [ ] **Step 3: (gembi) Validate the config**

Run:
```bash
sudo rsnapshot configtest
```
Expected: `Syntax OK`. If it complains about spaces/tabs, re-check that every field separator is a literal TAB.

---

### Task 6: Take and verify the first snapshot

**Files:**
- Creates on gembi: `/data/nextcld_docker/daily.0/…`

**Interfaces:**
- Consumes: Task 5 config.
- Produces: a populated `daily.0` proving the pull works (precondition for scheduling).

- [ ] **Step 1: (gembi) Dry-run the daily snapshot**

Run:
```bash
sudo rsnapshot -t daily
```
Expected: prints the `rsync`/`ssh` commands it *would* run, referencing both `backup` sources and `/data/nextcld_docker/daily.0/`. No errors.

- [ ] **Step 2: (gembi) Run the real daily snapshot**

```bash
sudo rsnapshot daily
```
Expected: completes with no error output.

- [ ] **Step 3: (gembi) Verify snapshot contents**

Run:
```bash
sudo ls /data/nextcld_docker/daily.0/nextcld/kumudu/ \
        /data/nextcld_docker/daily.0/nextcld/kumudu_amazon/
sudo du -sh /data/nextcld_docker/daily.0/
```
Expected: both user directories contain files, and `du` reports a non-trivial size matching roughly the live data.

---

### Task 7: Create the systemd service and timer units on gembi

Three oneshot services and three timers. Higher intervals fire earlier so rsnapshot rotation promotes a snapshot before the lower interval rotates.

**Files:**
- Create on gembi: `/etc/systemd/system/rsnapshot-daily.service`, `rsnapshot-weekly.service`, `rsnapshot-monthly.service`
- Create on gembi: `/etc/systemd/system/rsnapshot-daily.timer`, `rsnapshot-weekly.timer`, `rsnapshot-monthly.timer`

**Interfaces:**
- Consumes: working `rsnapshot daily` from Task 6 (and `weekly`/`monthly` use the same config).
- Produces: enabled timers (verified in Task 8).

- [ ] **Step 1: (gembi) Write the three service units**

`/etc/systemd/system/rsnapshot-daily.service`:
```ini
[Unit]
Description=rsnapshot daily snapshot of Nextcloud data

[Service]
Type=oneshot
ExecStart=/usr/bin/rsnapshot daily
```

`/etc/systemd/system/rsnapshot-weekly.service`:
```ini
[Unit]
Description=rsnapshot weekly snapshot of Nextcloud data

[Service]
Type=oneshot
ExecStart=/usr/bin/rsnapshot weekly
```

`/etc/systemd/system/rsnapshot-monthly.service`:
```ini
[Unit]
Description=rsnapshot monthly snapshot of Nextcloud data

[Service]
Type=oneshot
ExecStart=/usr/bin/rsnapshot monthly
```

- [ ] **Step 2: (gembi) Write the three timer units**

`/etc/systemd/system/rsnapshot-daily.timer`:
```ini
[Unit]
Description=Run rsnapshot daily

[Timer]
OnCalendar=*-*-* 03:00
Persistent=true

[Install]
WantedBy=timers.target
```

`/etc/systemd/system/rsnapshot-weekly.timer`:
```ini
[Unit]
Description=Run rsnapshot weekly

[Timer]
OnCalendar=Mon *-*-* 02:30
Persistent=true

[Install]
WantedBy=timers.target
```

`/etc/systemd/system/rsnapshot-monthly.timer`:
```ini
[Unit]
Description=Run rsnapshot monthly

[Timer]
OnCalendar=*-*-01 02:00
Persistent=true

[Install]
WantedBy=timers.target
```

- [ ] **Step 3: (gembi) Reload systemd and validate the units**

Run:
```bash
sudo systemctl daemon-reload
systemctl cat rsnapshot-daily.service rsnapshot-daily.timer >/dev/null && echo UNITS_OK
```
Expected: `UNITS_OK` (units parse without error).

---

### Task 8: Enable timers and verify scheduling

**Files:** none (enables Task 7 units).

**Interfaces:**
- Consumes: Task 7 units.
- Produces: live schedule.

- [ ] **Step 1: (gembi) Enable and start all three timers**

```bash
sudo systemctl enable --now rsnapshot-daily.timer rsnapshot-weekly.timer rsnapshot-monthly.timer
```

- [ ] **Step 2: (gembi) Confirm the timers are scheduled**

Run:
```bash
systemctl list-timers 'rsnapshot-*' --all
```
Expected: all three timers listed with sensible `NEXT` times (daily ~03:00, weekly next Monday 02:30, monthly the 1st at 02:00).

- [ ] **Step 3: (gembi) Smoke-test one service through systemd**

Run:
```bash
sudo systemctl start rsnapshot-daily.service
journalctl -u rsnapshot-daily.service -n 20 --no-pager
```
Expected: the unit runs to completion (`Deactivated successfully` / exit status 0) with no rsync errors in the log.

---

### Task 9: Record the runbook in the repo

Capture the as-built procedure alongside the existing numbered docs so it is discoverable next to `16_sync-data-from-nextcld.md`.

**Files:**
- Create in repo: `17_rsnapshot-backup-nextcloud-docker.md`

**Interfaces:**
- Consumes: the completed, verified setup from Tasks 1–8.
- Produces: committed runbook documentation.

- [ ] **Step 1: Write the runbook**

Create `17_rsnapshot-backup-nextcloud-docker.md` summarizing: the topology, the gid-33 read-access approach, the final `/etc/rsnapshot.conf`, the systemd units, the retention scheme, and the restore procedure (copy back from `/data/nextcld_docker/{interval}.N/nextcld/<user>/`). Link to the design spec at `docs/superpowers/specs/2026-06-21-rsnapshot-nextcloud-docker-design.md`.

- [ ] **Step 2: Verify the file references resolve**

Run:
```bash
ls 17_rsnapshot-backup-nextcloud-docker.md docs/superpowers/specs/2026-06-21-rsnapshot-nextcloud-docker-design.md
```
Expected: both paths listed (no "No such file").

- [ ] **Step 3: Commit**

```bash
git add 17_rsnapshot-backup-nextcloud-docker.md
git commit -m "Add rsnapshot backup runbook for Dockerized Nextcloud"
```

---

## Restore reference (no task — keep for operations)

To restore a user's files from any snapshot:
```bash
# From gembi, copy back to a staging area then push to the Docker host data dir
sudo rsync -a /data/nextcld_docker/daily.0/nextcld/kumudu/ /path/to/restore-staging/kumudu/
```
After restoring files into the live Nextcloud data directory (done as the data
owner / via the container), run `occ files:scan --all` inside the app container
so Nextcloud re-indexes the restored files.
