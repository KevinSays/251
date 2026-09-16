# Week 4 Deep Dive — Cron's OR Trap, Umask Arithmetic, and Reading `ls -l` for Real

*Companion to `IT251_Week4_Study_Page.html` (Week 4). That page covers the cron 5-field syntax, the `/etc/passwd` fields, the zombie-process worked example, and the `/etc/login.defs` UID/GID lookup. This goes deeper on three things the study page correctly kept brief: what happens when cron's day-of-month and day-of-week fields fight each other, the actual arithmetic behind umask (it isn't subtraction), and how to read the parts of `ls -l` output the study page's permission coverage doesn't reach — setuid/setgid/sticky bits and the link-count column. Have the cron 5-field syntax and basic `chmod`/`ls -l` reading down before this.*

---

## 1. The cron day-of-month/day-of-week trap (verified against `man 5 crontab`)

The study page teaches the 5-field syntax (`minute hour day-of-month month day-of-week`) and step/range/list values. What it doesn't cover — because it's an edge case, not the core mental model — is what happens when a cron line restricts **both** the day-of-month field and the day-of-week field at the same time.

The intuitive assumption is "AND" — run only when both conditions are true. **That assumption is wrong**, and it's wrong specifically, per the crontab man page's documented behavior:

> If both fields are restricted (i.e., don't start with `*`), the command will be run when **either** field matches the current time.

So:

```
30 4 1,15 * 5
```

A student reading this as "4:30am, but only on the 1st or 15th, and only if that day happens to be a Friday" would be wrong. The actual behavior: this fires at 4:30am on the 1st of the month, on the 15th of the month, **and** on every Friday — three independent trigger conditions OR'd together, not one narrow AND'd condition.

**The rule that resolves the confusion:** the AND/OR behavior hinges on whether a field is "restricted" — meaning it doesn't start with `*`. If *either* the day-of-month or day-of-week field is unrestricted (a plain `*`, or something like `*/2` which the man page also treats as unrestricted since it starts with `*`), the two fields behave normally and combine with AND, same as every other field pair in the line. It's specifically the case where **both** are restricted simultaneously that flips the relationship to OR. This is exactly the kind of rule that's easy to get backwards under exam pressure because "restricted fields should be more selective, not less" is the natural (wrong) intuition — restricting both fields actually *widens* the set of trigger times compared to restricting just one.

**Why this matters for real jobs, not just the exam:** a backup script scheduled with `0 2 1 * 1` (intending "2am on the 1st, if it's a Monday" as an extra safety check) will actually run at 2am on the 1st of every month *and* every Monday — almost certainly more often than intended. If you genuinely need a true AND — "only the first Monday of the month" — cron's 5-field syntax can't express that directly; the common workaround is a day-of-month range plus a script-level day-of-week check (`[ "$(date +%u)" -eq 1 ]`), not a cleverer cron expression.

---

## 2. Umask arithmetic — why it isn't subtraction

The study page's `/etc/login.defs` example shows students reading policy files, but umask itself — the thing that decides a *new* file or directory's default permissions — isn't covered there. It's easy to memorize "umask 022 means subtract 022 from the default," and that shortcut happens to give the right answer for files but breaks for directories if you don't know why.

**The real mechanism: umask is subtracted using bitwise math, not decimal subtraction**, and there's a different starting maximum for files versus directories:

- New **files** start from a maximum of `666` (rw-rw-rw-) — never executable by default, even with a permissive umask, because the kernel won't set the execute bit on a freshly created file just because umask allows it.
- New **directories** start from a maximum of `777` (rwxrwxrwx) — directories need the execute bit to be enterable at all, so it's part of their default maximum.

With `umask 022`:

```
File:      666 & ~022  =  644  (rw-r--r--)
Directory: 777 & ~022  =  755  (rwxr-xr-x)
```

Both examples happen to look like decimal subtraction here, which is exactly the trap — it *coincidentally* works for the common umask values (`022`, `002`) because those don't force any bit-borrowing. Try `umask 027` on a file, though:

```
File:      666 & ~027 = 640   (not 639 — decimal subtraction would give the wrong digit)
```

`666` in binary per-bit is `110 110 110`; `027` is `000 010 111`. The bitwise AND-with-complement clears exactly the bits umask has set, per bit, independent of decimal borrowing rules. The practical takeaway: don't do the arithmetic as decimal subtraction in your head and hope it's right — know the two starting maximums (666 for files, 777 for directories) and think in bits, or just trust `umask -S` to show you the symbolic result directly.

```
$ umask 027
$ umask -S
u=rwx,g=rx,o=
```

---

## 3. Reading `ls -l` output completely: setuid, setgid, sticky, and the link-count column

The study page covers hard links vs. symlinks conceptually. Here's the rest of what a real `ls -l` line is telling you.

```
-rwsr-xr-x  1 root root   68208 Mar  2 09:14 /usr/bin/passwd
drwxrwxrwt  9 root root    4096 Sep 14 22:01 /tmp
drwxr-sr-x  3 root devs    4096 Sep 10 11:02 /srv/shared-project
drwxr-xr-x  4 alice alice  4096 Sep  1 08:00 /home/alice/scripts
```

**The permission string's special-bit positions** — where you'd expect `x`, you sometimes see `s` or `t` instead:

- `passwd`'s owner-execute slot shows `s` (setuid). While it runs, the process temporarily runs **as the file's owner (root)**, not as whoever launched it. This is precisely how an unprivileged user runs `passwd` and is still able to write to `/etc/shadow` (which ordinary users can't touch directly) — the setuid bit is the mechanism, not a special case in `passwd`'s code. A capital `S` instead of lowercase `s` would mean the setuid bit is set but the underlying execute bit for the owner is *not* — usually a misconfiguration, since setuid without execute permission does nothing useful.
- `/srv/shared-project`'s group-execute slot shows `s` (setgid) **on a directory**. This is a different, equally real mechanism: any new file created inside that directory inherits the directory's group (`devs`) automatically, instead of the creating user's primary group. This is the standard fix for "everyone on the team needs group-shared access to files their teammates create," without needing every user to manually `chgrp` every new file.
- `/tmp`'s other-execute slot shows `t` (sticky bit). In a world-writable directory like `/tmp`, this restricts deletion/renaming of a file to that file's own owner (or root) — otherwise any user with write access to `/tmp` could delete anyone else's files sitting there, since directory write permission alone normally governs delete rights, not the file's own permissions.

**The link-count column (the number right after the permission string):** for a **directory**, this number is not arbitrary — it equals **2 plus the number of immediate subdirectories** (the `2` accounts for the directory's own `.` entry and the `..` entry inside each of its subdirectories pointing back up to it). `/home/alice/scripts` showing `4` means it has exactly 2 subdirectories inside it, whether or not you can see them all in a quick listing (a hidden dotdir still counts). For a **regular file**, the same column instead means "how many hard-linked directory entries point at this same inode" — normally `1`, and only higher if you deliberately created hard links to it, tying directly back into the Week 4 study page's hard-link material.

---

## 4. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| Restricting both cron day fields | Making the schedule more selective (AND) | Per `man 5 crontab`, restricting **both** flips the relationship to **OR** — the job fires on either match, which usually *widens* the schedule, not narrows it |
| Umask subtraction | Simple decimal subtraction | It's a **bitwise** AND-with-complement against a starting maximum (666 for files, 777 for dirs) — decimal subtraction only coincidentally matches for common umask values |
| setuid on an executable | A generic "special permission" flag | Specifically means the process runs with the **file owner's** privileges while executing — it's the exact mechanism behind `passwd` writing to `/etc/shadow` |
| setgid on a file vs. on a directory | The same behavior in both cases | On a directory, setgid makes **new files inherit the directory's group**; on an executable file, it's the group analogue of setuid (runs as the file's group) |
| A directory's link count | An arbitrary/cosmetic number | It's always **2 + number of immediate subdirectories** — a reliable, checkable fact about the directory tree beneath it |

---

## 5. Scenario quiz

**Q1.** A junior admin writes `0 6 15 * 1` intending "6am on the 15th, but only if the 15th falls on a Monday." What will this cron line actually do, and why?

<details><summary>Answer</summary>
It will run at 6am on the 15th of every month <em>and</em> at 6am every Monday — two independent conditions OR'd together, not one AND'd condition — because both the day-of-month field (<code>15</code>) and the day-of-week field (<code>1</code>) are restricted (neither starts with <code>*</code>). To get the intended "only if the 15th is a Monday" behavior, cron's fields alone can't express it; the fix is a broader schedule plus a script-level date check.
</details>

**Q2.** With `umask 027` active, a script creates both a new file and a new directory. Predict the resulting permissions for each and show the bitwise reasoning.

<details><summary>Answer</summary>
File: <code>666 &amp; ~027 = 640</code> (rw-r-----). Directory: <code>777 &amp; ~027 = 750</code> (rwxr-x---). Both come from clearing exactly the bits set in <code>027</code> against each type's starting maximum — not decimal subtraction, which would give the wrong result for the file case.
</details>

**Q3.** A shared project directory has the setgid bit set and belongs to group `devs`. A user whose primary group is `sales` creates a new file inside it. What group ends up owning that file, and why?

<details><summary>Answer</summary>
<code>devs</code> — not the creating user's own primary group (<code>sales</code>). Setgid on a directory overrides the normal "new file inherits the creator's primary group" rule and forces inheritance of the directory's own group instead, which is exactly the mechanism that keeps a shared team directory's files group-consistent without manual <code>chgrp</code> after every creation.
</details>
