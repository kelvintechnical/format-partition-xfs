# Lab 116: Format Partition with XFS

**Series:** linux-ops-mastery — RHCSA Storage & Filesystems
**Subjects covered:** `lsblk`, `parted`, `wipefs`, `mkfs.xfs`, `blkid`, `xfs_info`, `mount`, `umount`, `df -hT`, `/mnt/lab116`, journal log internals, signature cleanup
**Career arcs covered:** RHCSA (Create, mount, unmount, and use file systems objective), RHCE (Ansible `filesystem` and `mount` modules), SRE (provisioning new data volumes), DevOps (CI runners with ephemeral data disks), AI/MLOps (formatting NVMe scratch volumes for training jobs)
**Prerequisite:** Comfort with `sudo`, block-device naming (`/dev/vdb`, `/dev/sda`), and the difference between a **disk**, a **partition**, and a **filesystem**
**Time Estimate:** 35 to 55 minutes
**Difficulty arc:** Tasks 1–3 pre-flight and partition prep · 4–7 the format and inspect cycle · 8–9 mount, write, verify · 10 capstone deliverable + cleanup

> ⚠️ **Lab VM only.** Every command in this lab writes to `/dev/vdb1` and runs `wipefs`/`mkfs` on a partition. **Never run this on a production host or on a device you cannot afford to lose.** Confirm with `lsblk` that `/dev/vdb` is your spare disk before you proceed.

---

## Objective

Take a spare partition (`/dev/vdb1` on the lab VM), put a fresh **XFS** filesystem on it with `mkfs.xfs`, confirm the format with `blkid` and `xfs_info`, mount it temporarily at `/mnt/lab116`, write a deliverable summary file, then unmount and optionally wipe signatures so the lab is fully repeatable.

By the end of this lab you can answer the exam-style ticket *"format /dev/vdb1 as XFS and mount it at /data"* in under two minutes, and you understand the **three layers** that every filesystem lab touches: the block device, the on-disk superblock, and the mount table.

---

## Concept: Three Layers Every Format Touches

```
        ┌────────────────────────────────────────────┐
        │ Layer 1 — Block device                     │
        │   /dev/vdb         (whole disk)            │
        │   /dev/vdb1        (one partition)         │
        └────────────────────┬───────────────────────┘
                             │ mkfs.xfs writes…
                             ▼
        ┌────────────────────────────────────────────┐
        │ Layer 2 — On-disk superblock + metadata    │
        │   UUID, label, block size, log size,       │
        │   sectsz, agcount, finobt, reflink, crc    │
        └────────────────────┬───────────────────────┘
                             │ mount + kernel attach
                             ▼
        ┌────────────────────────────────────────────┐
        │ Layer 3 — Mount table (live VFS)           │
        │   /mnt/lab116      target                  │
        │   xfs              fstype                  │
        │   rw,relatime,...  options                 │
        └────────────────────────────────────────────┘
```

`lsblk` and `parted` work at Layer 1. `mkfs.xfs`, `blkid`, `wipefs`, `xfs_info` work at Layer 2. `mount`, `umount`, `findmnt`, `df -hT` work at Layer 3. Mixing them up — for example, running `mkfs.xfs` on a mounted target — is the single most common destructive mistake in storage tickets, which is why Task 3 includes an explicit unmounted-check.

---

## 📚 Reference Table

| Goal | Command |
|---|---|
| List block devices | `lsblk` |
| Confirm partition exists | `lsblk /dev/vdb` |
| Create a partition (if missing) | `sudo parted -s /dev/vdb mklabel gpt mkpart primary 1MiB 100%` |
| Preview existing signatures | `sudo wipefs -n /dev/vdb1` |
| Erase signatures (destructive) | `sudo wipefs -a /dev/vdb1` |
| Format as XFS | `sudo mkfs.xfs /dev/vdb1` |
| Force overwrite if FS exists | `sudo mkfs.xfs -f /dev/vdb1` |
| Set a label at format time | `sudo mkfs.xfs -L LAB116 /dev/vdb1` |
| Show UUID and TYPE | `sudo blkid /dev/vdb1` |
| Inspect XFS geometry | `sudo xfs_info /mnt/lab116` |
| Mount once (not persistent) | `sudo mount /dev/vdb1 /mnt/lab116` |
| Human-readable usage + fstype | `df -hT /mnt/lab116` |
| Unmount | `sudo umount /mnt/lab116` |
| Detect leftover signatures | `sudo wipefs /dev/vdb1` |

---

## 🎯 Career Pathway

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Create, mount, unmount, and use file systems" is an explicit EX200 objective. Tickets typically say "format `/dev/vdb1` as XFS and mount it at `/data`" — this lab is the muscle memory. |
| **RHCE candidate** | The `community.general.filesystem` and `ansible.posix.mount` modules wrap `mkfs.xfs` and `mount` respectively; you cannot write idempotent storage roles without knowing the underlying flags. |
| **SRE / Platform** | New data volumes on cloud VMs (EBS, persistent disks) arrive raw; the first thing you do is partition, format, and mount — usually XFS on RHEL. |
| **DevOps** | CI runners often attach ephemeral data disks per job; an XFS format-and-mount step is a one-line `cloud-init` snippet that this lab teaches at the flag level. |
| **AI / MLOps** | Training nodes mount NVMe scratch volumes as XFS for the throughput and large-file performance; the format step is identical to this lab. |

---

## 🔧 The 10 Tasks

> Each task includes **Purpose**, the command block, **Human-Readable Breakdown**, **Reading it left to right**, **The story**, **Expected output**, and reference tables.

---

### Task 1 — Identify the spare disk and confirm the partition

**Purpose:** Confirm that `/dev/vdb` is the spare disk and that `/dev/vdb1` exists. If the partition is missing (fresh lab VM), create it.

```bash
lsblk
lsblk /dev/vdb
sudo parted -s /dev/vdb print 2>/dev/null || echo "no partition table yet"
if [ ! -b /dev/vdb1 ]; then
  sudo parted -s /dev/vdb mklabel gpt mkpart primary 1MiB 100%
  sudo partprobe /dev/vdb
  sleep 1
fi
lsblk /dev/vdb
```

**Human-Readable Breakdown:**
> "Hey shell, show me every block device on this VM, then narrow to `/dev/vdb`. Try to print its partition table — if it doesn't have one, the command errors and I print a friendly message. Then, *only if* `/dev/vdb1` does not yet exist as a block device, create a GPT label and one partition that fills the disk. Re-read the partition table with `partprobe` and confirm the partition is now present."

**Reading it left to right:**
- `lsblk` → "**l**i**s**t **bl**oc**k** devices in a tree."
- `lsblk /dev/vdb` → "scope to one disk."
- `sudo parted -s /dev/vdb print` → "non-interactive (`-s` = scripted) print of the partition table."
- `2>/dev/null` → "discard stderr (the 'unrecognised disk label' error)."
- `if [ ! -b /dev/vdb1 ]` → "if `/dev/vdb1` is **not** a block device file…"
- `mklabel gpt` → "write a GPT partition table."
- `mkpart primary 1MiB 100%` → "one partition from 1MiB to end-of-disk."
- `partprobe` → "ask the kernel to re-read the partition table."

**The story:** This pre-flight protects you from the worst storage mistake: formatting the wrong disk. On a typical lab VM (`qemu-kvm`/`libvirt`), `/dev/vda` is the boot disk and `/dev/vdb` is the spare. On AWS Nitro it's `/dev/nvme1n1`; on a bare-metal lab it might be `/dev/sdb`. Always verify with `lsblk` before you go further. The conditional `parted` block makes this lab repeatable on a fresh VM that has no partition yet — and skips the destructive step on a VM where the partition is already there.

**Expected output:**

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda    252:0    0   20G  0 disk
├─vda1 252:1    0    1M  0 part
├─vda2 252:2    0  200M  0 part /boot/efi
└─vda3 252:3    0 19.8G  0 part /
vdb    252:16   0    5G  0 disk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
vdb  252:16   0   5G  0 disk
no partition table yet
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
vdb    252:16   0   5G  0 disk
└─vdb1 252:17   0   5G  0 part
```

**Switches**

| Token | Meaning |
|---|---|
| `lsblk` | List block devices |
| `parted -s` | Scripted (non-interactive) mode |
| `mklabel gpt` | Write GPT partition table |
| `partprobe` | Re-read partition table |
| `-b FILE` | Test: file is a block device |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `lsblk` doesn't show `/dev/vdb` | Wrong disk name — try `/dev/sdb` or `/dev/nvme1n1`; substitute everywhere |
| `parted: unrecognised disk label` | First time use — `mklabel gpt` is the fix (already in this script) |
| `/dev/vdb1` exists but contains data | Stop. Don't proceed unless this is truly your lab disk |

---

### Task 2 — Install XFS tools and confirm the binary

**Purpose:** Make sure `mkfs.xfs` and `xfs_info` are present. On RHEL/Rocky the `xfsprogs` package is part of the base install, but verify rather than assume.

```bash
rpm -q xfsprogs || sudo dnf install -y xfsprogs
which mkfs.xfs xfs_info blkid wipefs
mkfs.xfs -V
```

**Human-Readable Breakdown:**
> "Hey RPM, do you already have `xfsprogs` installed? If not (the `rpm -q` returns nonzero), pipe into `dnf install` and grab it. Then show me the absolute paths of every binary I'll need in this lab and print the `mkfs.xfs` version banner."

**Reading it left to right:**
- `rpm -q xfsprogs` → "**q**uery the RPM database for the package."
- `||` → "logical OR — run the next command only if the previous failed."
- `sudo dnf install -y` → "install with `-y` (yes to prompts)."
- `which BIN1 BIN2 ...` → "absolute path of every named binary on `$PATH`."
- `mkfs.xfs -V` → "**V**ersion banner."

**The story:** Lab disasters often start with "the binary wasn't there and I `dnf install`'d a different one halfway through." Pinning the toolchain up-front means every later step uses known versions. On RHEL 9 you should see `xfsprogs 5.x`; on Rocky/Alma the version tracks RHEL closely.

**Expected output:**

```
xfsprogs-5.19.0-3.el9.x86_64
/usr/sbin/mkfs.xfs
/usr/sbin/xfs_info
/usr/sbin/blkid
/usr/sbin/wipefs
mkfs.xfs version 5.19.0
```

**Switches**

| Token | Meaning |
|---|---|
| `rpm -q PKG` | Query installed package |
| `dnf install -y` | Non-interactive install |
| `which BIN` | Path of binary on `$PATH` |
| `-V` | Version (common across xfsprogs tools) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `dnf: command not found` | Older system — use `yum install -y xfsprogs` |
| `Cannot open repo` | No internet on lab VM — preinstall in your base image |
| `mkfs.xfs: command not found` after install | Re-source PATH: `hash -r` or open a new shell |

---

### Task 3 — Preview existing signatures with `wipefs -n` (no-op)

**Purpose:** See what's currently on `/dev/vdb1` before you format. The `-n` flag is **dry-run** — it lists signatures without removing them.

```bash
sudo wipefs -n /dev/vdb1
sudo blkid /dev/vdb1 || echo "no existing filesystem signature"
findmnt /dev/vdb1 || echo "not mounted — safe to format"
```

**Human-Readable Breakdown:**
> "Hey wipefs, list every filesystem signature you find on `/dev/vdb1` but don't touch anything — the `-n` flag means 'preview only.' Then ask `blkid` to print the TYPE; if there's nothing there, print a friendly message. Finally, check the mount table — if the partition is currently mounted, abort the lab, because formatting a mounted filesystem corrupts it."

**Reading it left to right:**
- `sudo wipefs -n DEV` → "**n**o-op — list signatures, don't erase."
- `sudo blkid DEV` → "show **bl**ock-device **id**entifiers (UUID, TYPE, LABEL)."
- `findmnt DEV` → "look up `DEV` in the live mount table."
- `|| echo "..."` → "fallback message if the previous command returned nonzero."

**The story:** The single most destructive mistake in this lab is running `mkfs.xfs` on a mounted partition or on a partition that contains a filesystem you forgot was there. `wipefs -n` shows you what `mkfs.xfs` would clobber; `findmnt` confirms the partition is unmounted. If `findmnt` returns a row, **stop**. Unmount first. There is no recovery from "I `mkfs`'d over the wrong thing."

**Expected output (fresh partition — empty):**

```
no existing filesystem signature
not mounted — safe to format
```

**Expected output (re-running this lab with an existing XFS):**

```
DEVICE OFFSET TYPE UUID                                 LABEL
vdb1   0x0    xfs  9b41a2c6-7d2f-4a3b-8c1d-0e2f4a5b6c7d
/dev/vdb1: UUID="9b41a2c6-7d2f-4a3b-8c1d-0e2f4a5b6c7d" TYPE="xfs"
not mounted — safe to format
```

**Switches**

| Token | Meaning |
|---|---|
| `wipefs -n` | List signatures (dry-run) |
| `wipefs -a` | Erase **a**ll signatures (destructive) |
| `blkid` | Print block-device identifiers |
| `findmnt DEV` | Live mount table lookup by source |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `wipefs` shows multiple signatures | Old MBR + filesystem — `wipefs -a` to clean before format |
| `findmnt` returns a row | **STOP.** Unmount first: `sudo umount /dev/vdb1` |
| `blkid` empty but `wipefs` shows data | Stale superblock; `wipefs -a` then re-run `blkid` |

---

### Task 4 — Format the partition with `mkfs.xfs`

**Purpose:** Write a fresh XFS superblock, allocation groups, and journal log onto `/dev/vdb1`.

```bash
sudo mkfs.xfs -f -L LAB116 /dev/vdb1
echo "exit=$?"
```

**Human-Readable Breakdown:**
> "Hey `mkfs.xfs`, format `/dev/vdb1` as XFS. Use `-f` to force overwrite if an existing filesystem signature is found (we already previewed in Task 3, so we know what we're doing). Tag the new filesystem with label `LAB116` so I can identify it later by `blkid -L LAB116`. Print the exit status to prove the command succeeded."

**Reading it left to right:**
- `mkfs.xfs` → "**m**a**k**e **f**ile**s**ystem of type **xfs**."
- `-f` → "**f**orce — overwrite existing filesystem signature without prompting."
- `-L LAB116` → "**L**abel — human-readable name (max 12 chars on XFS)."
- `/dev/vdb1` → "target partition."
- `echo "exit=$?"` → "print the exit status of the previous command (`$?` = last exit code)."

**The story:** `mkfs.xfs` is intentionally fast — it writes the superblock and allocation-group headers and then exits, leaving the rest of the filesystem to be filled lazily. On a 5GB partition the entire command completes in well under a second. The `-f` flag is your "I know what I'm doing" override; without it, `mkfs.xfs` refuses to overwrite an existing filesystem. The `-L` flag is optional but high-value: a labeled filesystem is referenced as `LABEL=LAB116` in `/etc/fstab`, which is more readable than a 36-character UUID.

**Expected output:**

```
meta-data=/dev/vdb1              isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.
exit=0
```

**Switches**

| Token | Meaning |
|---|---|
| `-f` | Force overwrite |
| `-L LABEL` | Set filesystem label |
| `-b size=BYTES` | Block size (rare to change; default 4096) |
| `-l size=BYTES` | Log size override |
| `-n size=BYTES` | Directory block size |
| `-m crc=0` | Disable metadata CRC (do **not** in modern XFS) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mkfs.xfs: /dev/vdb1 appears to contain an existing filesystem` | Add `-f` (you previewed in Task 3) |
| `Error opening file: No such file or directory` | Wrong device path; recheck with `lsblk` |
| `Filesystem already mounted` | `umount` first; never format a mounted FS |
| Label `>12 chars` rejected | XFS label is **12-char max**; shorten |

---

### Task 5 — Inspect the new filesystem with `blkid`

**Purpose:** Confirm the UUID, TYPE, and LABEL that `mkfs.xfs` just wrote.

```bash
sudo blkid /dev/vdb1
sudo blkid -L LAB116
sudo blkid -U "$(sudo blkid -s UUID -o value /dev/vdb1)"
```

**Human-Readable Breakdown:**
> "Hey `blkid`, print everything you know about `/dev/vdb1` — UUID, TYPE, LABEL. Then prove that I can look up the same device by **label** with `-L`. Then prove I can look it up by **UUID** by first extracting the UUID with `-s UUID -o value` and feeding it back into `-U`."

**Reading it left to right:**
- `blkid DEV` → "show device's identifiers."
- `-L LABEL` → "lookup device by label; print path."
- `-s UUID` → "**s**elect tag name UUID."
- `-o value` → "**o**utput format: just the value, no `KEY=`."
- `$(...)` → "command substitution — embed the output of the inner command."
- `-U UUID` → "lookup device by UUID; print path."

**The story:** Every `/etc/fstab` entry references storage by **UUID** (recommended) or **LABEL** — never by `/dev/vdb1`, because kernel device naming can change across reboots (especially on cloud VMs that hot-attach disks). `blkid` is the bridge: it gives you the stable identifier from the live device path, and it can also reverse-lookup a device path from a stable identifier. Practice both directions now and the `/etc/fstab` lab in the next module will feel trivial.

**Expected output:**

```
/dev/vdb1: LABEL="LAB116" UUID="9b41a2c6-7d2f-4a3b-8c1d-0e2f4a5b6c7d" TYPE="xfs" PARTUUID="..."
/dev/vdb1
/dev/vdb1
```

**Switches**

| Token | Meaning |
|---|---|
| `blkid DEV` | All tags for a device |
| `-L LABEL` | Look up by label |
| `-U UUID` | Look up by UUID |
| `-s TAG` | Select one tag (UUID, LABEL, TYPE) |
| `-o value` | Print only the value (no `KEY=`) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `blkid` shows old UUID | Cache stale — `sudo blkid -g` (garbage-collect) |
| `blkid -L LAB116` empty | Label not written or typo; re-run with explicit label |
| Two devices share a UUID | Cloned disk — `xfs_admin -U generate /dev/vdb1` to mint a new UUID |

---

### Task 6 — Create the mount point and mount

**Purpose:** Attach the new XFS filesystem to the live VFS at `/mnt/lab116` for read-write use.

```bash
sudo mkdir -p /mnt/lab116
sudo mount /dev/vdb1 /mnt/lab116
findmnt /mnt/lab116
mount | grep /mnt/lab116
```

**Human-Readable Breakdown:**
> "Hey shell, make a mount point directory at `/mnt/lab116` if it doesn't already exist (the `-p` flag is idempotent — no error on re-run). Then attach `/dev/vdb1` there with the default mount options. Confirm the mount with `findmnt` (the modern, structured view) and again with classic `mount | grep` (the legacy view). Both should show the same fstype `xfs` and options `rw,relatime,...`."

**Reading it left to right:**
- `mkdir -p PATH` → "create directory; **p**arents — no error if it exists."
- `mount DEV PATH` → "attach device at path."
- `findmnt PATH` → "modern mount-table query (tree-formatted)."
- `mount | grep PATH` → "legacy mount-table view filtered by mount point."

**The story:** Mount points are just empty directories — the kernel doesn't care whether you call it `/mnt/lab116`, `/data`, or `/home/scratch`. Convention says: temporary or experimental mounts go under `/mnt/`, long-term mounts go under `/srv/`, `/data/`, or `/home/`. This lab uses `/mnt/lab116` to reinforce the "temporary" nature of the mount — Task 9 will unmount it cleanly. **No `/etc/fstab` edits** in this lab; that's a different lab. This is a live, in-memory mount only.

**Expected output:**

```
TARGET     SOURCE    FSTYPE OPTIONS
/mnt/lab116 /dev/vdb1 xfs    rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota
/dev/vdb1 on /mnt/lab116 type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

**Switches**

| Token | Meaning |
|---|---|
| `mount DEV TARGET` | Attach device at target |
| `mount -t xfs` | Explicit fstype (auto-detected if omitted) |
| `mount -o ro` | Read-only mount |
| `findmnt TARGET` | Modern mount-table query |
| `mount` (no args) | List every mounted filesystem |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `mount: special device /dev/vdb1 does not exist` | Path typo or partition not created |
| `mount: wrong fs type` | Format failed; re-run Task 4 |
| `mount point does not exist` | Forgot `mkdir -p`; create it |
| `/mnt/lab116 already mounted` | Already mounted — `findmnt /mnt/lab116` to confirm |

---

### Task 7 — Verify with `xfs_info` and `df -hT`

**Purpose:** Read the live XFS geometry and confirm capacity.

```bash
sudo xfs_info /mnt/lab116
df -hT /mnt/lab116
df -i /mnt/lab116
```

**Human-Readable Breakdown:**
> "Hey `xfs_info`, dump every parameter of the XFS filesystem mounted at `/mnt/lab116` — sector size, block size, allocation groups, internal log size, CRC enabled, reflink enabled, finobt enabled. Then `df -hT` for **h**uman-readable sizes plus the **T**ype column, and `df -i` for **i**node usage. Together these three lines answer 'is the filesystem healthy and how much space do I have?'"

**Reading it left to right:**
- `xfs_info PATH` → "report XFS geometry; **PATH** must be a mount point (not the block device)."
- `df -hT PATH` → "**h**uman-readable + **T**ype column."
- `df -i PATH` → "**i**node usage (free/used/percent)."

**The story:** `xfs_info` is the post-format truth: every parameter `mkfs.xfs` wrote is reflected here. Pay attention to `crc=1` (metadata checksums — RHEL 8+ default), `reflink=1` (lightning-fast copies, useful in container scenarios), and `bigtime=1` (timestamps that survive past 2038). These three features mark a "modern" XFS. The `df -i` line is the **forgotten metric**: XFS uses dynamic inode allocation, so it almost never runs out of inodes, but `df -i` will still report what's allocated.

**Expected output:**

```
meta-data=/dev/vdb1              isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/vdb1      xfs   5.0G   68M  4.9G   2% /mnt/lab116
Filesystem      Inodes IUsed   IFree IUse% Mounted on
/dev/vdb1       2621440     3 2621437    1% /mnt/lab116
```

**Switches**

| Token | Meaning |
|---|---|
| `xfs_info PATH` | Geometry of mounted XFS |
| `df -h` | Human-readable sizes |
| `df -T` | Show filesystem type column |
| `df -i` | Inode usage instead of bytes |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `xfs_info: must be mounted` | Pass the mount point, not `/dev/vdb1` |
| `df` shows wrong size | Mount the right device; re-check Task 6 |
| `Use%` already high on empty FS | XFS reserves metadata blocks — 68M overhead on 5G is normal |

---

### Task 8 — Write a test file and verify capacity

**Purpose:** Prove the filesystem accepts writes and reports usage correctly.

```bash
sudo install -d -m 0755 /mnt/lab116/lab
sudo tee /mnt/lab116/lab/hello.txt >/dev/null <<'EOF'
Lab 116 — XFS write test
EOF
sudo dd if=/dev/zero of=/mnt/lab116/lab/big.bin bs=1M count=100 status=progress
sync
ls -lh /mnt/lab116/lab
df -hT /mnt/lab116
```

**Human-Readable Breakdown:**
> "Hey shell, create a sub-directory `/mnt/lab116/lab` with mode 0755. Drop a tiny text file using `tee` (which reads from the heredoc and writes to the file). Then use `dd` to write a 100MB binary file from `/dev/zero` to prove the filesystem accepts large writes. `sync` forces in-flight writes to disk. Confirm both files exist with `ls -lh` and re-check capacity with `df -hT`."

**Reading it left to right:**
- `install -d -m 0755 PATH` → "create directory + set mode in one shot."
- `tee FILE` → "read stdin, write to **both** stdout and FILE."
- `>/dev/null` → "discard stdout (we want the file, not the terminal echo)."
- `<<'EOF' ... EOF` → "heredoc literal (single-quoted `EOF` = no variable expansion)."
- `dd if=SRC of=DST bs=BS count=N` → "**d**ata **d**uplicator: input file, output file, block size, count."
- `status=progress` → "show byte counter while writing."
- `sync` → "flush kernel write cache."
- `ls -lh` → "long listing with human-readable sizes."

**The story:** `dd` is the classic "write a lot of zeros and see if the filesystem complains" tool. 100MB on a 5GB filesystem is small enough to finish in well under a second but big enough to show up in `df`. `sync` is a habit, not a strict requirement — modern `dd` already returns only after the kernel has accepted the write, but on production servers (especially ones with battery-backed cache controllers) `sync` removes any ambiguity about whether the data is durable.

**Expected output:**

```
104857600 bytes (105 MB, 100 MiB) copied, 0.13 s, 807 MB/s
100+0 records in
100+0 records out
total 101M
-rw-r--r--. 1 root root 100M May 27 10:24 big.bin
-rw-r--r--. 1 root root   24 May 27 10:24 hello.txt
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/vdb1      xfs   5.0G  168M  4.9G   4% /mnt/lab116
```

**Switches**

| Token | Meaning |
|---|---|
| `install -d -m MODE` | Create dir with mode in one shot |
| `tee FILE` | Write stdin to FILE (and stdout) |
| `dd bs=1M count=100` | 100 blocks of 1MB = 100MB |
| `status=progress` | Live progress display |
| `sync` | Flush kernel write cache |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `No space left on device` | Filesystem already full; reduce `count=` |
| `dd: failed to open` | Wrong mount point or read-only mount |
| `Permission denied` | Need `sudo`; you cannot write to `/mnt/lab116` as a normal user without ACLs |

---

### Task 9 — Unmount and confirm

**Purpose:** Cleanly detach the filesystem from the VFS.

```bash
sudo sync
sudo umount /mnt/lab116
findmnt /mnt/lab116 || echo "not mounted"
mount | grep lab116 || echo "lab116 absent from mount table"
```

**Human-Readable Breakdown:**
> "Hey kernel, flush any in-flight writes with `sync`, then detach `/dev/vdb1` from `/mnt/lab116`. Confirm the unmount in two ways: `findmnt` (modern; returns nonzero when not mounted) and `mount | grep` (legacy; same idea). Both should return 'not mounted' messages."

**Reading it left to right:**
- `sync` → "flush write cache."
- `umount PATH` → "detach the filesystem at PATH."
- `findmnt PATH` → "modern check; nonzero exit when not mounted."
- `|| echo` → "fallback message when the previous command returns nonzero."

**The story:** A clean unmount writes the XFS log to the data area and marks the superblock as **clean**. An unclean unmount (power loss, force-removed device) leaves the log dirty, and the next mount will replay it automatically. Always `umount` before `wipefs` — wiping a mounted filesystem corrupts the in-memory state and can panic the kernel. The two-way confirmation (`findmnt` + `mount | grep`) is paranoia, but it's cheap paranoia and it catches the rare case of a stale mount entry.

**Expected output:**

```
not mounted
lab116 absent from mount table
```

**Switches**

| Token | Meaning |
|---|---|
| `sync` | Flush write cache |
| `umount PATH` | Detach mount at PATH |
| `umount -l PATH` | Lazy unmount (last resort) |
| `umount -f PATH` | Force unmount (NFS only, usually) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `umount: target is busy` | An open file or shell `cd`'d into the mount — `lsof +D /mnt/lab116`, `fuser -vm /mnt/lab116` |
| `not mounted` even though `mount` shows it | Pass the **mount point**, not the device |
| Hangs on unmount | NFS stale handle; `umount -f` or reboot the VM |

---

### Task 10 — Capstone: deliverable summary + full cleanup

**Task statement:** *"Write a deliverable summary file containing the UUID, fstype, size, and any test-file metadata, then perform full cleanup — unmount, wipefs the partition, remove the mount-point directory."*

```bash
sudo mkdir -p ~/lab116-deliverables
SUMMARY=~/lab116-deliverables/lab116-summary.txt
{
  echo "=== Lab 116 — Format Partition with XFS ==="
  echo "Date: $(date -Is)"
  echo "Host: $(hostname)"
  echo "Device: /dev/vdb1"
  echo
  echo "--- blkid ---"
  sudo blkid /dev/vdb1
  echo
  echo "--- partition table ---"
  sudo parted -s /dev/vdb print
  echo
  echo "--- end of summary ---"
} | sudo tee "$SUMMARY" >/dev/null

ls -l "$SUMMARY"
sudo cat "$SUMMARY"

cat <<'YAML'
- name: Format /dev/vdb1 as XFS and mount at /data
  block:
    - name: Create filesystem
      community.general.filesystem:
        fstype: xfs
        dev: /dev/vdb1
    - name: Mount filesystem
      ansible.posix.mount:
        path: /data
        src: /dev/vdb1
        fstype: xfs
        state: mounted
YAML

findmnt /mnt/lab116 && sudo umount /mnt/lab116
sudo wipefs -a /dev/vdb1
sudo rmdir /mnt/lab116
ls /mnt/lab116 2>/dev/null || echo "mountpoint removed"
sudo blkid /dev/vdb1 || echo "no signature on /dev/vdb1 — partition is clean"
```

**Human-Readable Breakdown:**
> "Hey shell, create a deliverable directory and a `lab116-summary.txt` file containing the date, host, device, `blkid` output, and `parted` table — that's the artifact a reviewer or autograder consumes. Print the equivalent Ansible playbook stanza so any teammate can reproduce this on demand. Finally, unmount (only if still mounted), `wipefs -a` to clear every signature, remove the mount point directory, and verify the partition is now truly empty."

**Reading it left to right:**

| Block | What it does |
|---|---|
| `{ ... } \| sudo tee FILE >/dev/null` | Run a sub-shell, pipe its combined stdout to `tee` which writes to FILE; discard the terminal echo |
| `date -Is` | ISO 8601 timestamp with seconds |
| `parted -s /dev/vdb print` | Scripted partition table dump |
| Heredoc YAML | Ansible equivalent for repeatability |
| `findmnt ... && umount` | Only unmount if currently mounted |
| `wipefs -a DEV` | Destructive: erase **a**ll filesystem signatures |
| `rmdir DIR` | Remove an **empty** directory (safer than `rm -rf`) |

**The story:** This is the lab's deliverable. The summary file is what an autograder, peer reviewer, or your future self consumes to confirm the lab ran. The Ansible stanza is the production-grade equivalent — once you can write it from memory, you've internalized this lab. The cleanup discipline matters because next week you'll run Lab 117 (ext4) on the **same partition**, and a lingering XFS signature would force `mkfs.ext4` to refuse. `wipefs -a` is the answer: it erases every filesystem signature byte at the start (and end) of the partition, returning the block device to a truly raw state.

**Expected output:**

```
-rw-r--r--. 1 root root 412 May 27 10:32 /home/ec2-user/lab116-deliverables/lab116-summary.txt
=== Lab 116 — Format Partition with XFS ===
Date: 2026-05-27T10:32:11-04:00
Host: lab1.example.com
Device: /dev/vdb1

--- blkid ---
/dev/vdb1: LABEL="LAB116" UUID="9b41a2c6-7d2f-4a3b-8c1d-0e2f4a5b6c7d" TYPE="xfs"

--- partition table ---
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 5369MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Number  Start   End     Size    File system  Name     Flags
 1      1049kB  5369MB  5368MB  xfs          primary

--- end of summary ---
/dev/vdb1: 4 bytes were erased at offset 0x00000000 (xfs): 58 46 53 42
mountpoint removed
no signature on /dev/vdb1 — partition is clean
```

**Switches**

| Token | Meaning |
|---|---|
| `tee FILE` | Capture sub-shell output to FILE |
| `>/dev/null` | Discard stdout |
| `wipefs -a` | Erase **a**ll signatures |
| `rmdir DIR` | Remove empty directory |
| `findmnt && umount` | Conditional unmount |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `tee: cannot create` | Wrong path or no write permission — `sudo` already used |
| `wipefs: probing initialization failed` | Device busy — re-check `findmnt`, `lsof` |
| `rmdir: failed: Directory not empty` | Files still inside; use `rm -rf` (only if mount confirmed gone) |
| Ansible stanza fails | Older collections; install with `ansible-galaxy collection install community.general ansible.posix` |

---

## ✅ Lab Checklist (10 Tasks)

- [ ] 01 Identify `/dev/vdb`, confirm or create `/dev/vdb1`
- [ ] 02 Install/verify `xfsprogs` and `wipefs`/`blkid`
- [ ] 03 Preview existing signatures with `wipefs -n`; confirm unmounted
- [ ] 04 `mkfs.xfs -f -L LAB116 /dev/vdb1`
- [ ] 05 `blkid` by device, by label, by UUID
- [ ] 06 Create `/mnt/lab116`, mount, confirm with `findmnt`
- [ ] 07 `xfs_info` + `df -hT` + `df -i`
- [ ] 08 Write `hello.txt` and a 100MB `big.bin`; confirm with `df`
- [ ] 09 `sync` + `umount`; confirm not mounted (twice)
- [ ] 10 Summary deliverable + `wipefs -a` + `rmdir` cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Format mounted partition | `mkfs.xfs: ... is mounted` | `umount` first, then `mkfs.xfs` |
| Wrong disk (`/dev/vda` not `/dev/vdb`) | **System destruction** | Always `lsblk` first; verify size and mountpoints |
| No `-f` when signature exists | `mkfs.xfs` refuses | Add `-f` only after `wipefs -n` confirms what's there |
| Label too long | `mkfs.xfs: invalid label` | XFS label is **12-char max** |
| Forgot `mkdir -p /mnt/lab116` | `mount: mount point does not exist` | Create directory before `mount` |
| `umount` while in directory | `target is busy` | `cd ~` first, then `umount` |
| `rmdir` non-empty mount point | `Directory not empty` | Confirm unmounted, then `rmdir` or `rm -rf` (carefully) |
| Skipped `wipefs -a` | Next `mkfs` refuses | Always end with `wipefs -a` for a clean lab restart |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the four-step pattern: `mkfs.xfs DEV` → `mkdir -p MOUNTPOINT` → `mount DEV MOUNTPOINT` → `df -hT MOUNTPOINT`. That's the entire exam ticket.
- Know that XFS has **12-char label max**, ext4 has **16**.

**RHCE candidate**
- `community.general.filesystem` + `ansible.posix.mount` together replace the manual flow. Use `state: mounted` (mounts now + adds fstab) vs `state: present` (fstab only) vs `state: ephemeral` (mount only, no fstab).

**SRE / Platform**
- Cloud volumes (EBS, GCE PD) arrive raw; cloud-init usually runs the same `mkfs.xfs` + mount sequence. Knowing the underlying flags lets you debug failed cloud-init.

**DevOps**
- CI ephemeral disks: `mkfs.xfs -K` (skip discard) shaves seconds off provisioning when the disk is already zeroed.

**AI / MLOps**
- Training scratch volumes: XFS with default block size is usually correct; consider `-b size=4096` (default) and large allocation groups for many concurrent writers.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 117 — Format Partition with Ext4 | Same partition, different filesystem; compare label limits and feature flags |
| Lab 118 — Check Filesystem Consistency (fsck) | Post-format integrity verification |
| Lab 119 — Inspect Filesystem Features (dumpe2fs) | Ext4 counterpart of `xfs_info` |
| Configure `/etc/fstab` persistence | Make this mount survive reboot — by UUID or LABEL |
| Mount with `nofail` and systemd `.mount` units | Production-grade mount handling |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
