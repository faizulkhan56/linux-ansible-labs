# Linux Lab 09 — Disk add, filesystem mount, and LVM (VirtualBox)

## Objective

- Add a new 2 GB VirtualBox disk and use it as a normal filesystem
- Format (`mkfs`), mount, and make the mount persistent
- Add another 2 GB disk and use it for LVM
- Create `pv`, `vg`, `lv`, format, mount, and persist
- Extend and shrink LVM safely
- Clean up everything after the lab (filesystem, LVM, and disks)

## Lab assumptions

- You are using Ubuntu in VirtualBox
- VM is reachable and you have `sudo`
- Existing OS disk is usually `/dev/sda` (do not touch it)

> Always verify disk names with `lsblk` before running destructive commands.

---

## Part A — Add Disk #1 (2 GB) in VirtualBox for normal filesystem

### Step A1: Power off VM and add disk in VirtualBox

1. Power off the VM.
2. VirtualBox -> VM **Settings** -> **Storage**.
3. Under SATA Controller, click **Add Hard Disk**.
4. Create new disk:
   - Type: VDI
   - Storage: Dynamically allocated
   - Size: **2 GB**
5. Start VM.

### Step A2: Identify the new disk in Ubuntu

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
sudo fdisk -l
```

Find the new 2 GB disk (example: `/dev/sdb`).

For safety, set a variable (replace if your disk is different):

```bash
DISK1=/dev/sdb
echo "$DISK1"
```

### Step A3: Create ext4 filesystem and mount

```bash
sudo mkfs.ext4 -F "$DISK1"
sudo mkdir -p /mnt/data-test
sudo mount "$DISK1" /mnt/data-test
df -h | grep data-test
```

Write a test file:

```bash
echo "Disk1 filesystem test at $(date -Is)" | sudo tee /mnt/data-test/hello.txt
ls -l /mnt/data-test
```

### Step A4: Permanent mount for Disk #1

Get UUID:

```bash
sudo blkid "$DISK1"
```

Add to `/etc/fstab` (replace UUID value with yours):

```bash
echo "UUID=<DISK1_UUID> /mnt/data-test ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

Validate:

```bash
sudo umount /mnt/data-test
sudo mount -a
df -h | grep data-test
```

### `/etc/fstab` — theoretical concepts

Each non-comment line tells the system **what** to mount, **where**, **how**, and **when** (at boot). One-line memory: *fstab = what to mount, where, how, and when.*

#### Structure of `/etc/fstab`

General format (six fields, whitespace-separated):

```text
<device>   <mount_point>   <filesystem>   <options>   <dump>   <fsck>
```

Example (matches Step A4 above; replace the UUID with yours):

```text
UUID=<DISK1_UUID>   /mnt/data-test   ext4   defaults   0   2
```

#### Field-by-field explanation

**1. Device (`UUID=…` or `/dev/…`)**

Identifies the block device. Prefer **`UUID=…`** from `blkid`: device names such as `/dev/sdb` can change to `/dev/sdc` after reboot or hardware changes; the UUID stays stable.

```bash
sudo blkid "$DISK1"
```

Example line from `blkid` output:

```text
/dev/sdb1: UUID="1234-ABCD-..." TYPE="ext4"
```

In `/etc/fstab` you copy the `UUID="..."` value as `UUID=1234-ABCD-...` (no quotes in fstab).

**2. Mount point**

Directory where the filesystem is attached. File and directory paths on that filesystem appear under this path (for example disk content under `/mnt/data-test`).

The directory **must exist** before mount (as in Step A3):

```bash
sudo mkdir -p /mnt/data-test
```

**3. Filesystem type**

Examples:

| Type   | Typical use              |
|--------|--------------------------|
| `ext4` | Common default on Linux  |
| `xfs`  | Large files, performance |
| `nfs`  | Network filesystem       |
| `vfat` | Removable / Windows USB  |

**4. Mount options**

`defaults` is a shortcut for: `rw`, `suid`, `dev`, `exec`, `auto`, `nouser`, `async`.

| Option   | Meaning                                      |
|----------|-----------------------------------------------|
| `rw`     | Read/write                                    |
| `suid`   | Honor set-user-ID / set-group-ID bits         |
| `dev`    | Interpret device files on this filesystem     |
| `exec`   | Allow executing binaries from this mount      |
| `auto`   | May be mounted by `mount -a` (e.g. at boot)   |
| `nouser` | Only root may mount (unless other rules apply)|
| `async`  | Writes may be deferred (not fully synchronous)|

Other entries often use extra options (for example `nofail`, `_netdev` for NFS or network-backed storage); those are not required for this lab’s simple disk line.

**5. Dump (backup flag)**

- `0` — do not include this filesystem in **`dump`(8)**-style backups.
- `1` — eligible for inclusion when `dump` is configured to use `/etc/fstab`.

On modern systems **`dump` is rarely used**; backups are usually done with tools such as `rsync`, snapshots, cloud backup, or orchestration-specific backup (for example Velero on Kubernetes, EBS snapshots on AWS). The field remains for **historical Unix compatibility**. **Practical rule: use `0`** unless you have a deliberate `dump` setup.

Setting `1` has **no effect** unless you actually run and configure `dump`.

**6. Fsck pass order (filesystem check at boot)**

Tells the boot process **whether** to run `fsck` on this filesystem and **relative order**:

| Value | Meaning                                      |
|-------|----------------------------------------------|
| `0`   | Do not run `fsck` on this entry at boot    |
| `1`   | Check **first** — reserved for **root** (`/`) |
| `2`   | Check **after** root; use for other local disks |

Example roles:

- Root (`/`) should use pass **`1`** so it is checked before other filesystems that may depend on it.
- Normal data volumes (such as `/mnt/data-test`) typically use **`2`**.
- Entries that must not be fsck’d at boot use **`0`**: NFS, `tmpfs`, `proc`, `sysfs`, and many network or virtual filesystems.

**Why use `0` (skip fsck)?**

- Some filesystem types are not checked this way (NFS, `tmpfs`, etc.).
- Skipping fsck on **non-critical** or **large** disks can shorten boot time (trade-off vs. repair at boot).
- Ephemeral or externally managed storage may not need a boot-time check.

**Mistakes to avoid**

- Root as **`0`** in the last column (e.g. `… / ext4 defaults 0 0` for the sixth field) — root should normally be **`1`** so it is checked first.
- A non-root disk with **`1`** — can **compete** with root ordering and cause confusing boot behavior; non-root local ext4 volumes should usually be **`2`**.

**Correct pattern (illustrative)**

```text
UUID=<ROOT_UUID>     /           ext4   defaults   0   1
UUID=<DATA_UUID>     /data       ext4   defaults   0   2
server:/export       /mnt/nfs    nfs    defaults   0   0
tmpfs                /run        tmpfs  defaults   0   0
```

**Boot flow (simplified)**

Boot continues to multi-user; local filesystems with pass `1` then `2` are candidates for `fsck` ordering (implementation details depend on init system). Pass **`0`** entries are **skipped** for this ordering.

#### Combined meaning (Step A4 line)

| Part        | Role                                      |
|------------|-------------------------------------------|
| `UUID=…`  | Find the disk reliably                    |
| `/mnt/…`  | Attach it here                          |
| `ext4`    | Filesystem driver / type                 |
| `defaults`| Normal read/write mount behavior         |
| `0`       | Not included in legacy `dump` backups    |
| `2`       | Fsck after root at boot                  |

#### Verify after editing `/etc/fstab`

Always run:

```bash
sudo mount -a
```

If this prints **no errors**, the new lines are syntactically valid and mountable under current conditions.

**Common mistakes**

- Wrong or stale UUID after recreating the partition.
- Mount point directory missing.
- Typo in filesystem type (`ext4` vs `ext3`, etc.).
- Wrong **fsck** pass number for root vs. other disks.

**Where `/etc/fstab` matters in practice**

Persistent local disks, NFS mounts, cloud block volumes attached to a VM, and static mounts on nodes that run containers or stateful workloads often use `/etc/fstab` or an equivalent generated by the distro (systemd mount units can express the same idea).

---

### Disk partition test — How `/dev/sdb1` and `/dev/sdb2` are defined

Partitions such as `/dev/sdb1` and `/dev/sdb2` are not predefined by the kernel. You create them with a partitioning tool, for example:

- `fdisk`
- `parted`

#### Example with `fdisk`

Suppose `/dev/sdb` is a new empty 10 GB disk.

Start:

```bash
sudo fdisk /dev/sdb
```

Inside `fdisk`, create the **first partition**:

1. Press **n** for new.
2. Choose **p** for primary.
3. Partition number: **1**
4. First sector: press **Enter** (accept default).
5. Last sector: set size, for example **+5G**

This creates `/dev/sdb1`.

Create the **second partition**:

1. Press **n**
2. Choose **p**
3. Partition number: **2**
4. First sector: press **Enter**
5. Last sector: press **Enter** for the remaining space

This creates `/dev/sdb2`.

**Write changes** to disk:

- Press **w**

After that, on the VM:

```bash
lsblk
```

You may see something like:

```text
sdb    10G
├─sdb1  5G
└─sdb2  5G
```

---

## Part B — Add Disk #2 (2 GB) for LVM

### Step B1: Add second 2 GB disk in VirtualBox

1. Power off VM.
2. VM **Settings** -> **Storage** -> **Add Hard Disk**.
3. Create another **2 GB** VDI disk.
4. Start VM.

### Step B2: Identify second disk

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

Find the second new 2 GB disk (example: `/dev/sdc`), then:

```bash
DISK2=/dev/sdc
echo "$DISK2"
```

Install LVM tools if needed:

```bash
sudo apt-get update
sudo apt-get install -y lvm2
```

### Step B3: Create PV, VG, and LV

Create physical volume:

```bash
sudo pvcreate "$DISK2"
sudo pvs
```

Create volume group:

```bash
sudo vgcreate vg_lab "$DISK2"
sudo vgs
```

Create logical volume (example: 1.5 GB):

```bash
sudo lvcreate -L 1500M -n lv_data vg_lab
sudo lvs
```

### Step B4: Create filesystem on LV and mount

Use ext4 for this lab (supports grow and shrink):

```bash
sudo mkfs.ext4 /dev/vg_lab/lv_data
sudo mkdir -p /mnt/lvm-data
sudo mount /dev/vg_lab/lv_data /mnt/lvm-data
df -h | grep lvm-data
```

Test write:

```bash
echo "LVM test file at $(date -Is)" | sudo tee /mnt/lvm-data/lvm.txt
ls -l /mnt/lvm-data
```

### Step B5: Permanent mount for LVM LV

Get UUID:

```bash
sudo blkid /dev/vg_lab/lv_data
```

Add to `/etc/fstab` (replace UUID value):

```bash
echo "UUID=<LV_UUID> /mnt/lvm-data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

Validate:

```bash
sudo umount /mnt/lvm-data
sudo mount -a
df -h | grep lvm-data
```

---

## Part C — Increase and decrease LVM

## C1: Extend LVM logical volume

Check free space in VG:

```bash
sudo vgs
```

Extend LV by 400 MB:

```bash
sudo lvextend -L +400M /dev/vg_lab/lv_data
sudo resize2fs /dev/vg_lab/lv_data
df -h | grep lvm-data
```

## C2: Shrink LVM logical volume (ext4 only)

> Shrink is riskier than extend. Keep backup before production changes.
> Do not shrink XFS (XFS cannot be shrunk).

Example: shrink LV to `1200M`.

1) Unmount:

```bash
sudo umount /mnt/lvm-data
```

2) Filesystem check:

```bash
sudo e2fsck -f /dev/vg_lab/lv_data
```

3) Shrink filesystem first:

```bash
sudo resize2fs /dev/vg_lab/lv_data 1000M
```

4) Shrink LV (keep LV size slightly larger than filesystem):

```bash
sudo lvreduce -L 1200M /dev/vg_lab/lv_data
```

5) Re-check and mount:

```bash
sudo e2fsck -f /dev/vg_lab/lv_data
sudo mount /dev/vg_lab/lv_data /mnt/lvm-data
df -h | grep lvm-data
```

---

## Part D — Cleanup and remove everything (in safe order)

Use this when lab is complete.

If this is a new shell session, set disk variables again based on your actual device names:

```bash
DISK1=/dev/sdb
DISK2=/dev/sdc
echo "$DISK1 $DISK2"
```

### Step D1: Remove persistent mounts from `/etc/fstab`

Edit:

```bash
sudo nano /etc/fstab
```

Delete the two lines you added for:
- `/mnt/data-test`
- `/mnt/lvm-data`

### Step D2: Unmount filesystems

```bash
sudo umount /mnt/lvm-data 2>/dev/null || true
sudo umount /mnt/data-test 2>/dev/null || true
```

### Step D3: Remove LVM objects (LV -> VG -> PV)

```bash
sudo lvremove -y /dev/vg_lab/lv_data
sudo vgremove -y vg_lab
sudo pvremove -y "$DISK2"
```

Optional metadata wipe on LVM disk:

```bash
sudo wipefs -a "$DISK2"
```

### Step D4: Remove normal filesystem signatures

```bash
sudo wipefs -a "$DISK1"
```

### Step D5: Remove mount directories (optional)

```bash
sudo rmdir /mnt/lvm-data 2>/dev/null || true
sudo rmdir /mnt/data-test 2>/dev/null || true
```

### Step D6: Detach disks from VirtualBox

1. Power off VM.
2. VirtualBox -> VM **Settings** -> **Storage**.
3. Remove the two 2 GB test disks from controller.
4. Optionally delete disk files from host if not needed.

---

## Verification checklist

- `lsblk` shows expected devices and mount points
- `df -h` shows `/mnt/data-test` and `/mnt/lvm-data` while mounted
- `pvs`, `vgs`, `lvs` show LVM objects during LVM steps
- After cleanup, no custom entries remain in `/etc/fstab`

