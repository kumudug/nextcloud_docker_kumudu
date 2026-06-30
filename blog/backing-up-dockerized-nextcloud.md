# Time Travel for Your Files: Snapshot Backups

**Status:** Design + runbook done; pull-over-SSH backup with daily/weekly/monthly snapshots, files-only.

---

A plain backup answers one question: "is my data safe somewhere else?" A
*snapshot* backup answers a far more useful one: "what did my data look like at
any point in the past?" The difference is the dimension of *time*. A sync or
mirror only ever knows the present — it makes a second copy look exactly like the
source *right now*. Snapshots instead keep many frozen points in time, so
yesterday's version stays untouched even after today's backup runs. That turns "a
copy" into a time machine: a way to step back to last Tuesday — before the
mistake, before the corruption, before the ransomware. It's the whole idea, and
it's worth understanding before any of the Docker plumbing.

**Why this matters, in short:**

- **History** — every run is a separate point in time you can browse, not one
  moving copy that forgets the past.
- **Undo** — a bad change or deletion doesn't erase the good version; it's set
  aside, not overwritten.
- **Version control for everything** — you can actually answer "what did this
  file look like last Tuesday?"
- **Damage doesn't propagate** — deletions, corruption, and ransomware don't
  silently flow downstream into the backup.

Here's why each of those matters, and why a sync can't give you any of them. A
plain mirror — `rsync` with `--delete`, Dropbox, Nextcloud's own sync client —
has exactly one job: make the destination look like the source *right now*.
That's the trap. The moment you delete a file, fat-finger a "save," let an app
corrupt a document, or a piece of ransomware encrypts the lot, the sync does its
job perfectly and faithfully propagates the damage to the copy. The "backup" now
matches the broken source, and the good version is gone from both sides.

That single failure mode is really three missing things. There is **no history**:
a synced copy only knows the present, so the moment you discover the mistake,
there's nothing earlier to reach back to. There is **no undo**: the previous-good
state was overwritten, not set aside. And there is **no version control**: you
can't ask "what did this file look like last Tuesday?" because last Tuesday was
overwritten by Wednesday. Snapshots add back exactly the dimension a sync throws
away — *time*. Keeping multiple points in time, where yesterday's snapshot stays
frozen even after today's runs, is what turns "a copy" into "a way to go back."

The rest of this post explains how that actually works in practice, using my own
setup. I used to run Nextcloud on bare metal; it's now a Docker Compose stack,
and that one change quietly broke the mental model behind my old backup. So this
is less "here's what I did today" and more "here's the handful of technologies
involved and why moving Nextcloud into containers makes the backup question more
interesting than it looks." If you want the copy-paste steps, those live in a
separate runbook — this is the *why*.

## The shape of the problem

Nextcloud is really three things glued together:

1. **User files** — the actual photos and documents, on disk.
2. **A database** — MariaDB, holding file metadata, shares, users, etc.
3. **Config** — `config.php` and friends.

My old backup only ever copied #1 — the user files — over SSH to a file server,
and snapshotted them there. That was a deliberate, modest goal: "if a drive dies,
I can still get my documents back." It is *not* a "rebuild the whole instance"
backup, because the database can drift out of sync with the files you copied. I
kept that same modest scope this time. Worth being honest about that up front:
files-only means restore is "copy the files back and re-scan," not "boot a
byte-perfect clone."

## Where does the data even live now? (Docker bind mounts)

On bare metal the answer was easy: `/data/nextcloud/<user>/files/`, owned by the
web server user.

Under Docker, the Compose file has this:

```yaml
volumes:
  - ./nextcloud_data:/var/www/html/data
```

That's a **bind mount**: a directory on the host (`./nextcloud_data`) mapped
straight into the container at `/var/www/html/data`. So the files are still plain
files on the host — good, that's exactly what a file backup wants. The path is
just different now (`/data/nextcloud/nextcloud_data/<user>/files/`).

The subtle part is *ownership*. Inside the official Nextcloud image, the web
server runs as user `www-data`, which is **uid/gid 33**. Bind mounts don't
translate ownership — the container writes files as uid 33, so on the host those
files are *also* owned by uid 33, regardless of whether the host even has a user
named "www-data." On my Arch host, uid 33 maps to nothing in particular. The
files just show up owned by a numeric ID. Nextcloud also writes them with tight
permissions: directories `0750`, files `0640`. Group-readable, but not
world-readable.

That permission detail is the whole ballgame for the backup.

## The actual hard part: letting a backup user read the data

My backup pulls files over SSH using a dedicated, unprivileged account
(`syncusr`). For that account to read the data, it has to get past `0640`/`0750`.
There were three honest options:

- **Option A — join the owning group.** Add `syncusr` to the group that owns
  gid 33. Because the files are group-readable, membership alone grants read
  access. Nothing about the data changes — Nextcloud keeps writing as uid 33,
  permissions stay exactly as the app intends, and the backup account is
  *read-only by construction* (it has group **read**, not write).
- **Option B — passwordless `sudo rsync`.** Let `syncusr` run the remote copy as
  root. Works against any permissions, but it hands a backup account
  root-equivalent read of the whole box. That's a lot of blast radius for a
  convenience.
- **Option C — read from inside the container** (`docker exec` / `docker cp`).
  Couples the backup to the container being up and to Docker's lifecycle. More
  moving parts, more ways to fail silently.

I went with **A**. It's the smallest, most boring mechanism that does the job,
and it preserves a nice property: the backup user *cannot modify the live data*
even if it's compromised. On bare metal I'd achieved something similar with a
`syncgrp` group plus `chown` and the setgid bit — a whole dance to make new files
inherit the right group. Docker actually made this *simpler*: the container
already stamps everything with gid 33 for me, so I skip the `chown`/setgid
gymnastics entirely and just add `syncusr` to a group pinned at gid 33.

There's one gotcha that bites people here and it has nothing to do with the data
itself: **every parent directory on the path has to be traversable** by
`syncusr`. Group-read on the files is useless if `/home/someone/` in the path is
`0700`. Putting the data under `/data/...` (traversable) instead of a locked-down
home directory sidesteps it.

## Why pull over SSH instead of push

The backup server reaches *into* the Nextcloud host and pulls, rather than the
Nextcloud host pushing out. This is a small security posture choice that matters:
the machine exposed to the internet (Nextcloud) holds no credentials to the
backup server. If it's ever popped, the attacker can't reach the backups, can't
delete them, can't ransomware them. Credentials and initiative live on the
quieter, internal backup box. Combined with the read-only access above, the
Nextcloud host can neither reach nor corrupt its own backups.

## rsnapshot and the magic of hardlinks — cheap time travel

So far the time machine is just a promise. Hardlinks are what make it affordable
enough to actually keep. The retention engine is
[rsnapshot](https://rsnapshot.org/), and the thing worth understanding is *how it
keeps "7 daily + 4 weekly + 6 monthly" points in time without using 17× the
disk*.

Each snapshot is a full, browsable directory tree — `daily.0/`, `daily.1/`, and
so on — one self-contained destination in time you can walk into and copy from.
But under the hood, when a file hasn't changed between snapshots, rsnapshot
doesn't copy it again — it creates a **hardlink** to the existing copy. A
hardlink is just a second name pointing at the same bytes on disk. So a file that
never changes across six months of snapshots costs disk space *once*, while still
appearing in every single point in time as if it were really there. You only pay
for what actually changed between runs. That's what makes "keep a year of
monthly snapshots" — a whole year of dates you can travel back to — affordable
instead of absurd.

The retention itself is declarative:

```
retain  daily    7
retain  weekly   4
retain  monthly  6
```

Run `rsnapshot daily` and it rotates `daily.0 → daily.1 → … → daily.6` and drops
the oldest. `rsnapshot weekly` promotes the oldest daily into the weekly chain,
and so on up to monthly.

## The non-obvious scheduling detail

I drive the three intervals with **systemd timers** rather than cron (journald
logging, `Persistent=true` to catch up runs missed while the box was off). But
the genuinely non-obvious bit is *ordering*: the higher-frequency interval has to
run **last**, not first.

```
monthly  → 02:00 on the 1st
weekly   → 02:30 on Mondays
daily    → 03:00 every day
```

rsnapshot's promotion works by reaching into the *lower* interval's oldest
snapshot. If `daily` ran first and rotated away its oldest before `weekly` got to
look at it, the weekly chain would grab the wrong thing. Running the bigger
intervals at earlier times guarantees they sample the daily chain *before* it
rotates. It's the kind of detail that "works fine" for weeks and then quietly
produces a wrong snapshot on the one day all three fire together (the 1st of a
month that lands on a Monday).

## Two small things that waste an afternoon if you miss them

- **`rsnapshot.conf` is tab-delimited.** Not whitespace — *tabs*. Spaces produce
  a config error that doesn't obviously say "you used spaces." `rsnapshot
  configtest` is the thing that saves you.
- **`--numeric-ids` on rsync.** Because uid 33 has no name on either machine, you
  want rsync to preserve the raw numeric IDs rather than trying (and failing) to
  map them to names. I also remap the *stored* copy to a real local user so the
  snapshots are pleasant to browse, but the wire transfer stays numeric.

## Honest take

The interesting realization was that "dockerize it" didn't make the backup harder
so much as it *relocated* the difficulty. The old setup's pain was all in
file ownership — groups, setgid, making sure new files inherited the right group.
Docker took that specific pain away (the container stamps gid 33 on everything,
consistently) and replaced it with a smaller, cleaner version of the same
question: "which group is gid 33 on the host, and is `syncusr` in it?" The AI
collaboration was at its best when it refused to let me hand-wave the scope —
pinning down "files-only, not a full restore" early meant I didn't accidentally
design a half-baked database backup I'd have trusted and shouldn't have. The least
glamorous decisions — pull not push, read-only by group, files-only and *say so*
— are the ones I'll be glad about if I ever actually have to restore.
