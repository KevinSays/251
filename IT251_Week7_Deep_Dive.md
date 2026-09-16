# Week 7 Deep Dive — Reading Performance Tools for Real

*Companion to `IT251_Week7_Study_Page.html` (Week 7). That page introduces `journalctl`, `dmesg`, `ss`, `tcpdump`, `traceroute`, and the performance quartet (`top`/`vmstat`/`iostat`/`free`) at the glossary/command level. Troubleshooting carries the single largest chunk of exam weight of any domain (22%), and almost all of it is tested as "here's some real output, what's actually wrong" — so this is the longest of the six deep dives, and it's built entirely around reading annotated output rather than introducing new commands. Have the Week 7 study page's command list and the "connection state" example down before this.*

---

## 1. Load average — read against core count, not against zero

```
$ uptime
 14:32:07 up 3 days,  6:12,  2 users,  load average: 8.42, 6.10, 3.05
```

Three numbers: 1-minute, 5-minute, and 15-minute rolling averages of the number of processes that were either actively running or waiting for CPU/uninterruptible I/O. The number that matters isn't "is this high" in the abstract — it's **how it compares to the number of CPU cores**.

```
$ nproc
4
```

On a 4-core box, a load average of `8.42` means, roughly, twice as much runnable/waiting work as the machine has cores to run it concurrently — a real, current overload. That exact same `8.42` on a 16-core box would indicate the machine is comfortably under half-utilized. **The trap:** a student who's only ever seen load average discussed as "low is good, high is bad" without the core-count context will misjudge both directions — flagging a healthy 16-core box as overloaded, or missing a genuinely struggling 2-core box because "8" didn't sound alarming in isolation.

**The shape of the three numbers together is its own diagnostic.** `8.42, 6.10, 3.05` — rising from the 15-minute number toward the 1-minute number — describes a load spike that's actively getting worse right now. The reverse pattern (`3.05, 6.10, 8.42`, high 15-minute, low 1-minute) describes a spike that already happened and is now resolving. Same three raw numbers in different order tell an admin whether to be worried about *now* or just documenting what happened *earlier*.

---

## 2. `vmstat` — reading a sampled table, column by column

```
$ vmstat 2 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  2   1024  81920  30456 512300    0    2   180   340 1102 2450 22  9 12 57  0
 5  1   1024  79880  30460 512310    0    0    64   120  980 2210 18  7  5 70  0
```

`vmstat 2 5` means "take 5 samples, 2 seconds apart" — the **first row is always a since-boot average**, not a live sample, and should generally be discarded when diagnosing a current problem; the rows after it are the real live samples.

- **`r`** (procs): processes currently runnable, waiting for CPU time. Read this alongside `nproc` the same way as load average — `r` consistently exceeding core count means CPU contention right now.
- **`b`** (procs): processes blocked, waiting on I/O (usually disk). A high `b` with low `r` points away from a CPU problem and toward a storage/I/O one.
- **`si`/`so`** (swap): pages swapped **in**/**out** per second. Any sustained non-zero `so` (swapping *out* to disk) is a real red flag — it means the system is under real memory pressure and actively pushing memory pages to disk to make room, which is dramatically slower than RAM and often the actual root cause behind a system that "feels slow" for no obvious reason in `top`.
- **`wa`** (cpu): percent of CPU time spent idle specifically *while waiting on disk I/O* — not general idle time. This is the column that answers "is my slowness a CPU problem or a disk problem" better than almost anything else in this table: high `wa` with otherwise-low `us`/`sy` means the CPU has work it could be doing but is stalled waiting on storage, not that the CPU itself is overloaded.

**Exam-relevant read of the sample above:** `wa` at 57–70% with `us`+`sy` only around 25–31% describes a box where the CPU is mostly idle-but-blocked, not busy — that's a disk bottleneck signature, not a compute-bound one, and the fix path is entirely different (check disk I/O with `iostat`, not add CPU or optimize code).

---

## 3. `iostat -x` and the NVMe `%util` trap

```
$ iostat -x 2 3
Device            r/s     w/s   rkB/s   wkB/s  r_await  w_await  aqu-sz  %util
nvme0n1          420.0   180.0  53760   23040     0.31     0.42    3.10  100.00
```

The extended (`-x`) view is what you want for real diagnosis — the plain `iostat` output without `-x` doesn't give you `%util`, `await`, or queue depth at all.

**Here's the trap, confirmed against current sources (Red Hat's own support documentation flags this explicitly for NVMe):** `%util` is calculated as "percentage of time the device had at least one I/O request in flight." That definition is a genuinely reliable busy-signal for a traditional spinning disk, which can only service one request at a time — if it's doing *anything*, it's fully occupied. It is **not** reliable for NVMe/SSD devices, which service many requests in parallel across hardware queues. An NVMe drive comfortably handling a queue depth of 16 out of a possible 1000+ can show `%util: 100.00` the entire time, because *some* request was always in flight — while the drive itself is nowhere near actually saturated.

**What to trust instead, on NVMe/SSD:** `r_await`/`w_await` (average time, in milliseconds, a read/write request actually waited) and `aqu-sz` (average queue depth). In the sample above, `%util` reads a flat 100%, which looks alarming — but `r_await`/`w_await` under 1ms and a modest `aqu-sz` of 3.10 (far below a modern NVMe drive's real queue capacity) both say the drive is running fast and lightly loaded. This is a real, current gotcha worth stating explicitly because it inverts the intuitive reading: on this specific hardware, **the number that looks the scariest is the one to trust least**, and the two numbers that look the most boring (sub-millisecond awaits) are the ones actually telling you the truth.

---

## 4. `free` — the buff/cache column students misread as a memory shortage

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       2.1Gi       1.8Gi       128Mi        11Gi        12Gi
Swap:          2.0Gi          0B        2.0Gi
```

The trap here is almost always the same: a student sees `free: 1.8Gi` out of 15Gi total, panics about "the system is nearly out of memory," and misses the two columns that actually answer the question.

- **`buff/cache`** is memory the kernel is using to cache recently-read disk data and filesystem metadata — this is Linux doing exactly what it's supposed to do, using otherwise-idle RAM to make future reads faster. It is **not** memory that's unavailable; the kernel will drop cached pages instantly and without any performance penalty the moment a real application actually needs that RAM.
- **`available`** is the column that answers the real question — "how much memory could a new process actually get right now, including what would be reclaimed from cache if needed." Here, `available` (12Gi) is far closer to the true free capacity than the raw `free` column (1.8Gi) suggests.
- **`Swap: used 0B`** is the number that would actually indicate real memory pressure if it were climbing — a healthy system with plenty of headroom, even one showing a small raw `free` number, will have swap sitting at or near zero, exactly as shown here. This is the same signal as `vmstat`'s `so` column from a different angle — if swap usage is flat at zero, the low raw `free` number is a caching artifact, not a shortage.

**Rule to memorize:** never diagnose memory pressure from the `free` column alone — check `available` first, and cross-check against swap usage before concluding anything about real memory shortage.

---

## 5. `ss` connection-state reference

The study page already covers `ss -tuln` (what's listening) and `ss -tanp` (connection state + owning process) as two distinct, deliberately different queries. Here's the connection-state vocabulary itself, since the study page names the concept but doesn't spell out every state:

| State | Meaning | What a pile of these usually indicates |
|---|---|---|
| `LISTEN` | Socket is open and waiting for incoming connections | Normal for a server process; absence here means the service isn't listening at all |
| `ESTABLISHED` | A full, working two-way connection | Normal steady-state traffic |
| `SYN-SENT` | This host sent a SYN and is waiting for a reply | A pile of these stuck (not quickly resolving) usually means the remote host or a firewall in between is silently dropping the connection attempt — different from an active refusal |
| `SYN-RECV` | This host received a SYN and sent a reply, waiting for the final handshake step | A pile of these can indicate a SYN flood, or a client-side network issue preventing the handshake from completing |
| `TIME-WAIT` | Connection closed, socket held briefly to catch any delayed/duplicate packets | Normal and expected in bulk after a busy server closes many short-lived connections — not itself a problem unless the volume is exhausting available ephemeral ports |
| `CLOSE-WAIT` | The remote end closed the connection, but this host's application hasn't closed its side yet | A large, growing pile of these usually points at an **application bug** — the app isn't calling close() on sockets it's done with, a real and common cause of a slow file-descriptor leak |

**The diagnostic pattern worth internalizing:** `SYN-SENT` piling up says "something between me and the destination is silently dropping packets" (firewall, dead route) — this is the "not connection refused, just hanging" scenario the study page's own worked example already describes, and this table is the vocabulary behind that diagnosis. `CLOSE-WAIT` piling up says something completely different — "the network is fine, but my own application has a bug." Same symptom category (a socket table full of something that isn't `ESTABLISHED`), two unrelated root causes, and the state name is what tells them apart.

---

## 6. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| A "high" load average | An absolute, distro-independent threshold | Load average only means something **relative to core count** (`nproc`) — the same number is fine on one box and a real overload on another |
| High `wa` in `vmstat`/`top` | High CPU usage | `wa` is CPU time spent **idle while waiting on disk I/O** — it's a disk-bottleneck signal, not a compute-bottleneck one, even though it shows up in the CPU section |
| `iostat`'s `%util` at 100% on NVMe | The drive is saturated | On NVMe/SSD, `%util` measures "any request in flight," which parallel hardware queues make unreliable — trust `r_await`/`w_await`/`aqu-sz` instead |
| `free`'s low `free` column | Low available memory | `buff/cache` is reclaimable disk cache, not unavailable memory — check the `available` column and swap usage instead |
| `SYN-SENT` piling up in `ss` output | `CLOSE-WAIT` piling up | `SYN-SENT` points at a network/firewall problem preventing handshakes; `CLOSE-WAIT` points at an **application bug** not closing sockets — same "not ESTABLISHED" symptom, unrelated causes |

---

## 7. Scenario quiz

**Q1.** A 4-core VM shows `load average: 3.80, 3.95, 4.10` in `uptime`. A teammate says this is fine because "it's under 5." Evaluate that reasoning.

<details><summary>Answer</summary>
The reasoning is coincidentally close to right but for the wrong reason — "under 5" isn't a meaningful threshold on its own. What matters is load average relative to core count: on a 4-core box, a load average near 4 means the system is running at roughly full CPU capacity, right at the edge of being genuinely saturated, not comfortably fine. The declining trend (4.10 → 3.95 → 3.80 from 15-min to 1-min) does suggest the load is easing rather than climbing, which is the actually useful piece of information here — but "under 5" as a rule has no basis without knowing <code>nproc</code>.
</details>

**Q2.** `iostat -x` on a database server's NVMe volume shows `%util: 100.00` continuously, and a junior admin wants to escalate this as a critical disk-saturation incident. What should you check before agreeing, and why?

<details><summary>Answer</summary>
Check <code>r_await</code>/<code>w_await</code> and <code>aqu-sz</code> before treating <code>%util</code> as conclusive. NVMe drives service many requests in parallel across hardware queues, so <code>%util</code>'s "at least one request in flight" definition can pin at 100% on a lightly-loaded NVMe drive simply because some request is always active — it does not mean the drive is out of capacity the way it would for a spinning disk. Sub-millisecond await times and a queue depth well below the drive's real hardware queue capacity would indicate the drive is actually fine despite the alarming <code>%util</code> reading.
</details>

**Q3.** `ss -tanp` on a web server shows dozens of connections stuck in `CLOSE-WAIT`, slowly increasing over several hours, with no corresponding firewall changes or network issues reported. What's the most likely root cause, and is this a network problem?

<details><summary>Answer</summary>
Not a network problem — a growing pile of <code>CLOSE-WAIT</code> connections almost always points at an application bug: the remote end already closed its side of the connection, but the local application process isn't calling close() on its own socket, leaving it stuck. Left unaddressed, this is a slow file-descriptor leak that will eventually exhaust the process's available file descriptors. The fix is in the application code (or a restart as a stopgap), not the network configuration — contrast this with <code>SYN-SENT</code> piling up, which would actually point at a network/firewall issue.
</details>
