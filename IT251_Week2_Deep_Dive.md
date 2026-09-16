# Week 2 Deep Dive — Redirection Order, Sync Semantics, and Name Resolution

*Companion to `IT251_Week2_Study_Page.html` (Week 2). That page covers stdin/stdout/stderr as three streams and shows the basic `2>&1` merge. This goes one layer deeper on exactly the part that trips people up hardest: redirection *order*, plus two more real-world traps in the rsync and DNS material the study page introduces operationally but doesn't fully unpack. Read the Week 2 study page first if `stdout`/`stderr`/`2>&1` isn't already familiar.*

---

## 1. The redirection-order trap: `2>&1 >file` vs `>file 2>&1`

This is arguably the single best "why" in all of shell redirection, and it's a genuine exam trap because the two forms *look* almost identical but do opposite things.

Bash processes redirections **left to right**, and each one is really saying "point this file descriptor at whatever the target currently points to" — not "permanently link these two forever." That distinction is the whole trick.

### Case A: `command > file.log 2>&1` (the one you want)

```
$ ls /nope > out.log 2>&1
$ cat out.log
ls: cannot access '/nope': No such file or directory
```

Step by step:
1. `> file.log` — stdout (fd 1) now points at `file.log`.
2. `2>&1` — stderr (fd 2) is redirected to point at **wherever fd 1 currently points**, which is `file.log`.

Result: both streams end up in the file. This is the pattern to memorize.

### Case B: `command 2>&1 > file.log` (the one that silently breaks)

```
$ ls /nope 2>&1 > out.log
ls: cannot access '/nope': No such file or directory
$ cat out.log
(empty)
```

Step by step:
1. `2>&1` — stderr is redirected to point at **wherever fd 1 currently points right now**, which at this moment is still the terminal (nothing's redirected fd 1 yet). So stderr now goes to the screen.
2. `> out.log` — *then* stdout gets pointed at the file.

Result: stderr already "locked in" the terminal as its target before stdout got reassigned, so the error prints to the screen and the file stays empty (assuming the command produced no stdout). This is why order matters — `2>&1` doesn't create a permanent bond between the streams, it's a one-time snapshot of where fd 1 pointed *at that instant*.

**How to never get this wrong again:** always redirect stdout first, then merge stderr into it. If you remember nothing else from this section, remember the working order is `> file 2>&1`, not the other way around.

---

## 2. `rsync` trailing-slash behavior

The study page's backup example correctly recommends `rsync -a` for incremental backups. What it doesn't show is that `rsync`'s behavior changes depending on whether the **source path** ends in a slash — and this is one of the most common real-world `rsync` mistakes.

```
$ rsync -a project/  /backup/project/
# copies the CONTENTS of project/ into /backup/project/
# i.e. /backup/project/file1, /backup/project/file2, ...

$ rsync -a project   /backup/project/
# copies project/ ITSELF into /backup/project/
# i.e. /backup/project/project/file1, /backup/project/project/file2, ...
```

Read it as: a trailing slash on the source means "copy what's *inside* this directory"; no trailing slash means "copy this directory itself, including its own name, into the destination." The destination path's trailing slash doesn't have this effect — it's specifically a source-side behavior.

**Why this matters in practice:** a script that runs nightly with the wrong form doesn't error out — it "succeeds" every time while silently nesting a nearly-identical copy one level deeper than intended, or clobbering just the contents instead of accumulating full snapshots. Because there's no error message, this can run wrong for months before anyone notices. Always test a new `rsync` invocation with `--dry-run` (or `-n`) first and read the planned file list before trusting it against real data.

---

## 3. Name resolution order: `/etc/nsswitch.conf` vs. `/etc/resolv.conf`

The study page's DNS worked example correctly points at `/etc/resolv.conf` for "which nameserver does this box ask." That's the right first move, but it's only half the picture — `resolv.conf` only matters at all if the system is even configured to *use* DNS for a given lookup in the first place, and that's a separate file's job.

`/etc/nsswitch.conf` controls, per lookup type, *which sources get checked and in what order*. The line that matters for hostname resolution looks like:

```
hosts: files dns
```

Read left to right as *precedence order*: check `files` first (meaning `/etc/hosts`), then fall back to `dns` (meaning whatever's configured in `resolv.conf`) only if `files` didn't have an answer.

**The trap this explains:** a student edits `/etc/resolv.conf`, confirms the nameserver line looks correct, and a hostname *still* resolves to the wrong IP. The actual cause is frequently a stale or wrong entry sitting in `/etc/hosts` — because `files` is checked first per the `nsswitch.conf` order above, a hardcoded `/etc/hosts` entry wins before DNS is ever consulted, no matter how correct `resolv.conf` is. This is exactly the kind of "I fixed the file everyone talks about and it still didn't work" scenario the Troubleshooting domain likes to test — the fix requires knowing there's an ordering file upstream of the nameserver file, not just knowing the nameserver file exists.

---

## 4. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| `2>&1 >file` | `>file 2>&1` | Redirections apply left-to-right; putting `2>&1` **before** the stdout redirect merges stderr into the *old* target (the terminal), not the file |
| A trailing slash on `rsync`'s destination | A trailing slash on `rsync`'s source | Source-side trailing slash controls whether the directory's **contents** or the **directory itself** gets copied; destination slash doesn't have this effect |
| Editing `/etc/resolv.conf` | Fixing all hostname-resolution problems | `/etc/nsswitch.conf`'s `hosts:` line decides whether `/etc/hosts` is checked *before* DNS at all — a stale hosts entry wins regardless of what `resolv.conf` says |
| `curl`/`wget` failing on a URL | A DNS problem by default | Could be DNS, but could just as easily be TLS, a firewall drop, or the service simply not listening — `dig`/`nslookup` isolates whether resolution itself is the failing step before you look elsewhere |

---

## 5. Scenario quiz

**Q1.** A student runs `find / -name "*.conf" 2>&1 > results.txt` expecting a clean terminal and a full results file. Instead, a wall of "Permission denied" errors prints to the screen, and `results.txt` only has the successful matches. Explain exactly what happened.

<details><summary>Answer</summary>
Redirections apply left to right. <code>2>&1</code> ran first, while stdout (fd 1) still pointed at the terminal — so stderr got pointed at the terminal too, permanently for the rest of that command. <em>Then</em> <code>&gt; results.txt</code> redirected stdout to the file, but stderr had already locked onto the terminal in the previous step. The fix is reordering to <code>find / -name "*.conf" &gt; results.txt 2&gt;&amp;1</code>.
</details>

**Q2.** A nightly `rsync -a nightly_data /mnt/backup/` job has been running for six months with no errors, but a teammate notices `/mnt/backup/` now contains `nightly_data/nightly_data/` nested inside itself. What caused this, and is data being lost?

<details><summary>Answer</summary>
No trailing slash on the source (<code>nightly_data</code> instead of <code>nightly_data/</code>) means rsync copies the directory itself into the destination, not just its contents — so each run copies <code>nightly_data</code> as a subdirectory of the target. No data is being silently lost, but the structure isn't what was intended, and depending on how it's been running repeatedly, it's worth checking whether re-runs have been creating further nested copies rather than just updating one.
</details>

**Q3.** `/etc/resolv.conf` lists a nameserver that responds correctly when queried directly with `dig @nameserver hostname`, but a plain `ping hostname` on the same box still resolves to an old, wrong IP. `/etc/resolv.conf` is not the problem — what should you check next, and why?

<details><summary>Answer</summary>
<code>/etc/hosts</code> and the <code>hosts:</code> line in <code>/etc/nsswitch.conf</code>. If <code>nsswitch.conf</code> lists <code>files</code> before <code>dns</code> (the common default), a stale static entry in <code>/etc/hosts</code> is checked and matched first, before DNS is ever consulted — which explains why a working, correctly-configured nameserver never gets a chance to answer for that particular hostname.
</details>
