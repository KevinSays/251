# Week 7 Deep Dive — Reading Performance Tools for Real

*Companion to `IT251_Week7_Study_Page.html` (Week 7). That page introduces `journalctl`, `dmesg`, `ss`, `tcpdump`, `traceroute`, and the performance quartet (`top`/`vmstat`/`iostat`/`free`) at the glossary/command level. Troubleshooting carries the second-largest chunk of exam weight (22%, behind only System Management), and almost all of it is tested as "here's some real output, what's actually wrong" — so this is the longest of the six deep dives, and it's built entirely around reading annotated output rather than introducing new commands. Each section pairs the annotated output with an everyday analogy and a second, contrasting example — and a few include a "try it on your VM" you can run in two minutes. Have the Week 7 study page's command list and the "connection state" example down before this.*

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

> **Analogy — Checkout lanes at a grocery store.**
>
> Cores are the open registers. Load average is everyone being rung up **plus** everyone standing in line. Eight shoppers at a store with 4 open registers means every register is busy and there's a line of one behind each — the store is overloaded. The same 8 shoppers at a 16-register store leave half the registers idle. The shopper count alone tells you nothing until you know how many registers are open (`nproc`).
>
> The three numbers are a traffic report: "right now," "five minutes ago," and "fifteen minutes ago." Reading them together tells you whether the line is growing or shrinking.

### Another example: high load, idle CPU

```
$ uptime
 09:15:44 up 12 days,  2:03,  1 user,  load average: 11.87, 11.40, 10.95
$ nproc
4
$ top -bn1 | grep '%Cpu'
%Cpu(s):  1.2 us,  0.8 sy,  0.0 ni, 95.6 id,  2.4 wa,  0.0 hi,  0.0 si,  0.0 st
$ ps -eo stat,pid,comm | awk '$1 ~ /^D/'
D     2211 df
D     2305 ls
D     2388 rsync
...
```

Load near 12 on a 4-core box — yet the CPU is 95% idle. That's not a contradiction: on Linux, load average also counts processes in **uninterruptible sleep** (state `D`), which almost always means "stuck waiting on storage." The classic cause is a network mount (NFS) whose server has gone away: every process that touches it — even a plain `df` — hangs in `D` and adds 1 to the load.

Confirm with `dmesg | tail` (look for *nfs: server … not responding*) and `mount | grep nfs`. The fix path is the storage or the network, not the CPU. **Load average measures demand for CPU *and* for I/O — the CPU numbers in `top` tell you which.**

### Try it on your VM

```
$ nproc                 # say it prints 2
$ yes > /dev/null &     # one process that burns a full core
$ yes > /dev/null &     # a second one
$ watch -n 5 uptime     # watch the 1-minute number climb toward 2 (Ctrl+C to exit)
$ kill %1 %2            # stop both
```

Within about a minute the 1-minute number approaches your core count while the 15-minute number barely moves — that lag is exactly why the three numbers exist. After `kill`, watch the 1-minute number fall first.

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

> **Analogy — A restaurant kitchen.**
>
> `r` is orders waiting for a free cook. `b` is cooks standing at the oven waiting for it to finish. `wa` is the time cooks spend idle *because* they're waiting on the oven — the kitchen isn't short of cooks, it's short of oven. `si`/`so` is what happens when the counter (RAM) is full: cooks keep running to the walk-in freezer (disk) to put things away and fetch them back, and everything slows down even though nobody is slacking.

### Another example: three bottlenecks, side by side

```
# A) CPU-bound (4 cores)
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 9  0      0 2051200  60400 930100    0    0     0    12 4210 3890 88 11  1  0  0

# B) Disk-bound
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  6      0 1893400  60410 931200    0    0 48200   310 1650 2100  4  3 21 72  0

# C) Memory pressure (swapping)
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  4 1843200  52100   1200  40300  420 1250  1680  5200 2300 3900 10 12 20 58  0
```

**A:** `r` of 9 on 4 cores, `us`+`sy` at 99, `wa` at 0 — more work than cores. Find the hog with `top`.

**B:** `b` of 6, huge `bi` (blocks read in), `wa` at 72 — the CPU is waiting on disk reads. Next stop: `iostat -x` to find the device.

**C:** the trap. High `wa` makes it look like B, but `si`/`so` are non-zero, `free` is tiny, and `cache` has been squeezed to almost nothing — the kernel already gave up its cache and started swapping. The disk is busy *because* memory ran out. The root cause is memory: find the hog with `top` sorted by memory (`M`), not a faster disk. **When `wa` is high, check `si`/`so` before blaming the disk.**

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

> **Analogy — A drive-through window vs. a food court.**
>
> A spinning disk is a single drive-through window: if anyone is being served, the window is fully busy. An NVMe drive is a food court with 64 counters. `%util` only asks "was at least one customer being served somewhere?" — so one customer at one counter all day reads as "100% busy" while 63 counters sit idle. To know whether the food court is actually overwhelmed, ask how long people wait (`await`) and how long the lines are (`aqu-sz`).

### Another example: when 100% really is saturated

```
$ iostat -x 2 3
Device            r/s     w/s   rkB/s   wkB/s  r_await  w_await  aqu-sz  %util
sda              148.0    62.0   9472    3968    41.20    58.70   10.40   99.60
```

Same `%util` as the NVMe sample, opposite conclusion. This is a spinning disk (a few hundred random operations per second is about its ceiling), requests are waiting 40–60 ms instead of 0.3 ms, and the queue is 10 deep. This drive genuinely can't keep up.

**The rule that works on both kinds of hardware: `await` tells the truth; `%util` only tells the truth on spinning disks.** To see which kind you have: `lsblk -d -o NAME,ROTA` — `1` means rotational (spinning), `0` means SSD/NVMe. One catch in class: VirtualBox virtual disks usually report `ROTA=1` even when the host has an SSD, unless the disk is marked "Solid-state drive" in the VM's storage settings.

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

> **Analogy — Your desk and the filing cabinet.**
>
> RAM is your desk; the disk is a filing cabinet down the hall. `buff/cache` is folders you've left open on the desk because you'll probably need them again — the desk *looks* full, but you'd sweep them aside the moment you needed room. `available` is how much desk you could clear right now. Swap is moving work to a storage unit across town: you only do it when the desk is genuinely full, and every trip costs you.

### Another example: a real memory shortage

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       3.5Gi       112Mi        24Mi       210Mi       180Mi
Swap:          2.0Gi       1.9Gi       102Mi
$ sudo dmesg | grep -i "out of memory"
[81234.551203] Out of memory: Killed process 2314 (java) total-vm:4812340kB, anon-rss:2901244kB, ...
```

Compare with the healthy sample above. Here `available` is only 180Mi, `buff/cache` has already been squeezed down to 210Mi (the kernel has reclaimed almost everything it can), and swap is nearly full. That's a real shortage — and the kernel's last resort, the **OOM killer**, has started killing processes. This is where `dmesg` from the study page earns its place: it's the only place the OOM killer leaves its note.

### Try it on your VM

```
$ free -h                                     # note buff/cache
$ find / -type f > /dev/null 2>&1             # read a lot of filesystem metadata
$ free -h                                     # buff/cache grew; available barely moved
$ sync; echo 3 | sudo tee /proc/sys/vm/drop_caches   # ask the kernel to drop its cache
$ free -h                                     # buff/cache shrinks, free grows
```

Nothing was lost when the cache dropped — it was only ever a copy of what's already on disk. (Dropping caches is harmless but pointless on a real server; it just makes the next few reads slower. It's here to prove the point.)

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

> **Analogy — Phone calls.**
>
> `LISTEN` is a phone on the hook, waiting to ring. `ESTABLISHED` is two people talking. `SYN-SENT` is dialing and hearing it ring… and ring… with no answer and no voicemail — something is silently swallowing the call (a firewall dropping packets). Compare **connection refused**, which is the instant recording "the number you have dialed is not in service": the host is there and answered, there's just nothing listening on that port.
>
> `CLOSE-WAIT` is the other person saying goodbye and hanging up while you're still holding the phone to your ear — your side never hung up, which is why a pile of them means an application bug. `TIME-WAIT` is both of you having hung up, with the line held for a moment in case a delayed word arrives.

### Another example: turning a wall of sockets into one number

```
$ ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn
    412 CLOSE-WAIT
     58 ESTAB
     31 TIME-WAIT
      3 LISTEN
$ sudo ss -tanp state close-wait | head -3
Recv-Q Send-Q  Local Address:Port   Peer Address:Port  Process
1      0       10.0.0.5:8080        10.0.0.21:51344    users:(("java",pid=2314,fd=187))
1      0       10.0.0.5:8080        10.0.0.21:51362    users:(("java",pid=2314,fd=203))
```

The one-liner counts sockets by state: `sort` groups identical lines together, then `uniq -c` counts each group. 412 `CLOSE-WAIT` stands out immediately, and filtering by that state shows every one belongs to the same `java` process, with file-descriptor numbers (`fd=`) climbing into the hundreds — a descriptor leak in progress. (`ss` drops the State column when you filter to a single state.) Watch it grow with `ls /proc/2314/fd | wc -l`.

### Another example: refused vs. timed out

```
$ curl http://127.0.0.1:9999
curl: (7) Failed to connect to 127.0.0.1:9999 after 0 ms: Could not connect to server
$ curl --max-time 5 http://192.0.2.1
curl: (28) Connection timed out after 5001 milliseconds
```

The first fails in **0 ms** — the host answered "nothing here" (a TCP reset); older curl versions print "Connection refused" here. Check whether the service is running and listening (`ss -tln`). The second fails only when the timer runs out — nothing answered at all, so check routing, firewalls, and whether the name resolves to the right address. That second one is the same failure as Lab 6's name-resolution ticket. The exit codes (7 vs. 28) are the stable part; the wording varies between curl versions.

---

## 6. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| A "high" load average | An absolute, distro-independent threshold | Load average only means something **relative to core count** (`nproc`) — the same number is fine on one box and a real overload on another |
| High `wa` in `vmstat`/`top` | High CPU usage | `wa` is CPU time spent **idle while waiting on disk I/O** — it's a disk-bottleneck signal, not a compute-bottleneck one, even though it shows up in the CPU section |
| `iostat`'s `%util` at 100% on NVMe | The drive is saturated | On NVMe/SSD, `%util` measures "any request in flight," which parallel hardware queues make unreliable — trust `r_await`/`w_await`/`aqu-sz` instead |
| `free`'s low `free` column | Low available memory | `buff/cache` is reclaimable disk cache, not unavailable memory — check the `available` column and swap usage instead |
| `SYN-SENT` piling up in `ss` output | `CLOSE-WAIT` piling up | `SYN-SENT` points at a network/firewall problem preventing handshakes; `CLOSE-WAIT` points at an **application bug** not closing sockets — same "not ESTABLISHED" symptom, unrelated causes |
| A high load average | The CPU being busy | Linux load also counts processes stuck in uninterruptible sleep (`D` state) waiting on I/O — high load with an idle CPU points at storage or a hung network mount, not the CPU |
| High `wa` while swapping | A slow disk | If `si`/`so` are non-zero, the disk traffic *is* swap — the root cause is memory, and a faster disk only hides it |
| "Connection refused" | A connection timeout | Refused is instant: the host answered and nothing is listening on that port (check the service, `ss -tln`). A timeout is silence: packets are being dropped or going nowhere (check firewall, routing, name resolution) |

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

**Q4.** A server feels slow. Live `vmstat` rows show `wa` around 55%, `si` around 300, `so` around 900, `free` under 60 MB, and `cache` tiny. A teammate wants to order faster disks. What do you tell them?

<details><summary>Answer</summary>
Faster disks would treat the symptom, not the cause. Non-zero <code>si</code>/<code>so</code> means the system is actively swapping, and the tiny <code>free</code> and squeezed <code>cache</code> confirm memory has run out. The high <code>wa</code> is the CPU waiting on <em>swap</em> I/O. The root cause is memory: find what's using it (<code>top</code> sorted by memory, <code>free -h</code>, and <code>dmesg</code> for OOM-killer messages), then fix that process or add RAM.
</details>

**Q5.** A 4-core file server shows `load average: 14.02, 13.88, 13.50`, but `top` shows the CPU 96% idle. Running `df` hangs and never returns. What's the most likely cause, and where do you look?

<details><summary>Answer</summary>
Processes stuck in uninterruptible sleep (<code>D</code> state) — Linux counts them in the load average even though they use no CPU. <code>df</code> hanging is the giveaway: it's trying to read a mounted filesystem that isn't answering, most likely a network mount (NFS) whose server is down or unreachable. Look at <code>ps -eo stat,pid,comm | awk '$1 ~ /^D/'</code>, <code>dmesg | tail</code> for "server not responding", and <code>mount</code> to identify the network mounts. The fix is the storage server or the network path, not the CPU.
</details>

**Q6.** From the same client, `curl` to an internal API on port 8443 fails instantly, while `curl` to a second internal service only fails after two minutes. Where do you start on each?

<details><summary>Answer</summary>
The instant failure means the API host answered and refused — the host is up and reachable, but nothing is listening on 8443. Start on that host: is the service running (<code>systemctl status</code>), and is it listening on the port you expect (<code>ss -tln</code>)? The slow failure means nothing answered at all — packets are being dropped or sent somewhere wrong. Start with the path: does the name resolve to the right address (<code>getent hosts</code>), is there a route (<code>traceroute</code>), and is a firewall silently dropping the traffic?
</details>
