# Week 1 Deep Dive — Reading the Storage Stack for Real

*Companion to `IT251_Study_Page.html` (Week 1). That page gave you the boot-sequence diagram, the FHS tree, and the storage-stack diagram (Disk → Partition → LVM → Filesystem → Mount). This picks up where the diagram stops being enough: actual command output, the failure modes that only show up once a box has been alive for a while, and the traps in this material that show up on practice exams. If you haven't got the boot sequence and FHS basics down cold from the study page, go do that first — this assumes you do.*

---

## 1. Reading real output: `lsblk`, `findmnt`, `blkid`, `fstab`

The study page's storage-stack diagram shows the *shape* — Disk → Partition → LVM → Filesystem → Mount. In the real world you diagnose that shape by reading four tools' output together. None of them alone tells the whole story.

### `lsblk` — the disk's physical/logical layout

```
$ lsblk
NAME            MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda               8:0    0    40G  0 disk
├─sda1            8:1    0     1G  0 part /boot
└─sda2            8:2    0    39G  0 part
  └─vg_data-lv_root
                253:0    0    35G  0 lvm  /
  └─vg_data-lv_var
                253:1    0     4G  0 lvm  /var
```

Read this top to bottom as a tree, not a list:

- `sda` is the whole disk (`disk`). Its size (40G) is the ceiling everything below it has to fit inside.
- `sda1` and `sda2` are partitions carved out of that disk (`part`). Notice `sda1` mounts directly at `/boot` — no LVM layer at all for the boot partition. That's deliberate and common: GRUB has to be able to find and read `/boot` before the kernel and any LVM tooling exist to assemble a logical volume, so `/boot` is almost always a plain partition.
- `sda2` doesn't mount anywhere itself — it feeds into LVM (`lvm` type), which then presents `lv_root` and `lv_var` as the things that actually get mounted.
- The indentation under `sda2` is the giveaway that LVM is doing something with a partition here — a flat, no-LVM box would show partitions mounting directly, the way `sda1` does.

**Exam-trap version of this:** a question describes a box where `/boot` mounts fine but the rest of the system won't come up, and asks you to explain why LVM tooling isn't the problem. The `lsblk` layout above is exactly why — `/boot` never depended on LVM being available in the first place.

### `findmnt` — what's actually mounted, right now, with real options

```
$ findmnt /var
TARGET SOURCE                FSTYPE OPTIONS
/var   /dev/mapper/vg_data-lv_var
                              ext4   rw,relatime
```

`lsblk` tells you the block-device layout; `findmnt` tells you the *live mount table* — what filesystem type is actually in use and what options it mounted with. This matters because a filesystem can exist and be perfectly healthy while mounted read-only (`ro` instead of `rw` in the options column) — a classic "I have permission on paper but writes fail anyway" symptom that `ls -l` alone won't explain, because the block on write is coming from the mount, not the file's permission bits.

### `blkid` — matching a device node to the UUID `fstab` actually uses

```
$ blkid /dev/sda1
/dev/sda1: UUID="3a91-2C4F" TYPE="vfat" PARTUUID="a1b2c3d4-01"
```

This is the missing link between `lsblk`'s device names and `/etc/fstab`'s entries, which almost never reference `/dev/sda1` directly — they reference the UUID. Why UUID instead of the device name? Because `/dev/sda1` isn't a stable identity — if you add a second disk, or the boot order changes, what the kernel calls `sda` and `sdb` can swap. A UUID is generated once, baked into the filesystem itself, and doesn't drift when device enumeration order changes. `blkid` is how you find out what UUID a given partition actually carries, so you can cross-check it against what `fstab` claims.

### `/etc/fstab` — field by field, and what happens when one is wrong

```
UUID=3a91-2C4F  /boot  vfat  defaults        0  2
UUID=7f2e-91aa  /      ext4  defaults        0  1
UUID=b81c-44de  /var   ext4  defaults,nofail 0  2
```

Six fields, in order: **device** (by UUID here), **mount point**, **filesystem type**, **mount options**, **dump flag** (legacy, almost always `0` now), **fsck pass order** (`0` = don't check, `1` = check first — root only gets `1`, everything else gets `2` or `0`).

**The trap that actually bites students:** if a UUID in `fstab` doesn't match any real device — a drive was removed, cloned without regenerating its UUID, or someone fat-fingered a digit — the boot process will, on most distros, drop you into an **emergency shell** (sometimes called a maintenance/rescue shell) instead of completing the boot. This looks alarming the first time you see it — a text prompt, no desktop, an error about a filesystem or `fsck` failing — but it's not a kernel panic and it's not data loss. The fix is almost always: boot into that emergency shell (it's usually a normal root shell), edit `/etc/fstab` to correct or remove the bad line, and reboot. This is precisely why the `nofail` option exists on the `/var` line above — for a non-critical mount, `nofail` tells the boot process "continue normally even if this one doesn't mount," trading a working-but-incomplete boot for a hard stop. Root's own entry should never carry `nofail` — if root can't mount, there's no functioning system to continue booting into anyway.

---

## 2. Edge cases: where the simple rules break

- **GPT still needs a small BIOS boot partition on legacy-BIOS+GPT combos.** The study page correctly says GPT removes MBR's 2TB/4-partition limits — but if a system boots via legacy BIOS (not UEFI) and uses GPT, GRUB needs a tiny (~1MB) unformatted "BIOS boot partition" to embed its second-stage code, because legacy BIOS has nowhere else to put it. Pure UEFI+GPT systems don't need this — it's specifically the BIOS+GPT combination that does. If you see an unlabeled, tiny partition with no filesystem and no mount point in `lsblk` output on an otherwise-normal GPT disk, this is almost certainly it, not an error.
- **`resize2fs` can grow *and* shrink ext4 — `xfs_growfs` can only grow.** This is a real, frequently-tested asymmetry, not a minor footnote: XFS has no supported way to shrink a filesystem once created, full stop. If a student's plan is "make the LV smaller, then shrink the filesystem to match," that plan only exists on ext4. On XFS, the only path to a smaller filesystem is backup → recreate smaller → restore. Verify current at authoring time if you're teaching a specific distro's default (this asymmetry itself is stable and long-standing).
- **A VG showing free space doesn't guarantee an LV can use all of it.** The study page's "disk is full, VG has 200GB free" example is the common case, but if that VG spans multiple physical volumes and the LV isn't set up as `--type striped` or similar, `lvextend` still works fine — LVM handles this transparently for a simple linear LV. The rarer edge case is a *thinly provisioned* LV pool: you can technically over-commit a thin pool beyond the VG's real physical space, and running out of actual backing space then causes writes to fail even though `lvs`/`vgs` "space available" numbers looked fine. Worth knowing this distinction exists even if thin provisioning itself is out of scope here.

---

## 3. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| MBR's 2TB limit | A hardware limitation of the drive | MBR's limit is a property of the **partition table format**, not the disk — the same physical drive shows its full capacity once repartitioned with GPT |
| `resize2fs` failing to shrink | A generic "filesystem is full" error | `resize2fs` shrinks ext4 fine when there's room; the real asymmetry is that **XFS can't shrink at all**, regardless of free space |
| An emergency/rescue shell at boot | A kernel panic or hardware failure | An emergency shell usually means **`fstab` or `fsck` found a mount it couldn't satisfy** — it's a config problem you fix from a live root shell, not a hardware failure |
| `/dev/sda1` in `fstab` | A stable, safe way to reference a partition | Device names can renumber when disks are added/removed; **UUID** (from `blkid`) is the stable identity `fstab` should reference instead |
| A VG with free space | A guarantee any LV in it can grow | True for a simple linear LV; **thin-provisioned pools** can report space that isn't really backed by physical capacity |

---

## 4. Scenario quiz

**Q1.** A server won't finish booting — it drops to a shell prompt with no desktop, and the last visible message before that mentions a device UUID that "does not exist." What's your first move, and is this a hardware failure?

<details><summary>Answer</summary>
Not a hardware failure — this is the classic bad-<code>fstab</code>-entry emergency shell. From that shell (it's a functioning root shell), run <code>blkid</code> to see what UUIDs actually exist on the box, compare against <code>/etc/fstab</code>, fix or remove the stale line (adding <code>nofail</code> if it's a non-critical mount), and reboot.
</details>

**Q2.** You're told to shrink an LV that holds an XFS filesystem by 10GB to free up space for another LV in the same VG. Walk through why this specific request, as stated, can't be done directly.

<details><summary>Answer</summary>
XFS has no supported shrink operation — <code>xfs_growfs</code> only grows. The LV's block device could theoretically be shrunk with <code>lvreduce</code>, but doing that while an XFS filesystem still occupies the old size risks corrupting it, since the filesystem has no way to first shrink its own metadata down to match. The real path is: back up the data, recreate the filesystem at the smaller size, restore. If shrinking without a rebuild is a hard requirement, that's an argument for provisioning ext4 instead of XFS on volumes expected to need this later.
</details>

**Q3.** `lsblk` shows `/boot` mounted directly from a partition (`sda1`), with no LVM layer, while `/` and `/var` both sit on top of an LVM volume group. Is this inconsistent or misconfigured?

<details><summary>Answer</summary>
Not misconfigured — this is the normal, expected layout. GRUB needs to read <code>/boot</code> before any LVM tooling is available to assemble a volume group, so <code>/boot</code> almost always stays a plain partition even when everything else on the disk uses LVM.
</details>
