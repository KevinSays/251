# Week 5 Deep Dive — Ordering vs. Requirement, and Reading `systemctl status` for Real

*Companion to `IT251_Week5_Study_Page.html` (Week 5). That page's active/enabled two-axis diagram is, by design, the best diagram in this whole course — it teaches "currently running" and "starts at boot" as independent axes instead of one linear state. This deep dive applies the exact same move to a second systemd trap the study page's glossary only touches in passing (`Wants=`/`Requires=`), and then does the annotated-output pass the study page doesn't have room for: reading an actual `systemctl status` block line by line. Have the active/enabled diagram and the `mask` vs. `disable` distinction down before this — this builds directly on both.*

---

## 1. `After=`/`Before=` vs. `Wants=`/`Requires=` — two independent axes, again

The study page's glossary says "`Wants` is a soft suggestion, `Requires` is a hard dependency." True, but that's only *one* of the two axes a unit file's dependency directives control, and conflating the two axes is a real, common mistake.

**Axis 1 — ordering:** `After=` and `Before=` control *when* a unit starts relative to another, and nothing else. `After=network-online.target` in a unit file means "don't start me until network-online.target has started" — it says nothing about whether that target actually has to succeed, or even exist.

**Axis 2 — requirement:** `Wants=` and `Requires=` control whether one unit's success/failure *matters* to another, and nothing about timing. `Wants=` pulls in the target unit as a soft dependency — if it fails or is missing, the unit with `Wants=` still starts anyway. `Requires=` makes it a hard dependency — if the required unit fails, systemd won't start (or will stop) the unit that required it.

**The trap:** these two axes are independent, and a unit file commonly needs *both* directives together to get the intended behavior, because neither one implies the other:

```ini
[Unit]
Description=My web app
Wants=network-online.target
After=network-online.target
```

`After=` alone here would mean "wait until network-online.target has *started* (or failed and given up) before launching" — but without `Wants=`, systemd has no reason to make sure `network-online.target` is even *pulled in* as part of the boot sequence at all if nothing else already wants it. And `Wants=` alone, without `After=`, would mean "pull this dependency in, but don't guarantee it starts first" — the app could still race ahead and try to bind a socket before the network is actually up, purely because nothing told systemd to sequence them. Ordering and requirement are genuinely orthogonal — you can have either without the other, and most real service files pair `After=` with `Wants=` (or the stronger `Requires=`) specifically because one directive alone leaves a gap the other closes.

**Where this shows up as a real bug:** a service that "usually works, but sometimes fails right after boot" — intermittent, not consistent — is a classic symptom of a missing `After=` on a service that assumes some other resource (network, a database, a mount) is already up. It works most of the time because that other unit usually finishes first anyway, by luck of however fast that particular boot happened to go, and fails intermittently when timing shifts. This is exactly the same shape of bug as the active/enabled trap the study page already teaches — a real behavior driven by two independent settings, where changing only one half doesn't fix the underlying gap.

> **Analogy — Planning a dinner party.**
>
> `Wants=` is sending the caterer an invitation. `Requires=` is "if the caterer cancels, the dinner is cancelled." `After=` is "don't seat the guests until the caterer has arrived."
>
> Inviting the caterer doesn't mean anyone waits for them: without `After=`, guests sit down and start asking for food while the van is still on the highway. Waiting for a caterer you never invited means waiting for nobody: `After=` on its own doesn't pull anything in. It only matters if something else already invited that unit. You need both: invite them, *and* wait for them.

### Another example: all four combinations, side by side

Say `app.service` depends on `db.service`. What each combination of directives in `app.service` actually does:

| In app.service | Is db started too? | Does app wait for db? | If db fails or is stopped... |
|---|---|---|---|
| *(nothing)* | Only if something else starts it | No | app doesn't care |
| `Wants=db.service` | Yes | **No**. They start in parallel | app keeps running |
| `After=db.service` | **No**. Only orders it *if* something else starts db | Yes, if db is starting at all | app doesn't care |
| `Wants=` + `After=` | Yes | Yes | app still starts or keeps running |
| `Requires=` + `After=` | Yes | Yes | app won't start if db fails to start; `systemctl stop db` stops app too |

One row catches people: `Requires=` **without** `After=`. The two units start in parallel, so if db fails a moment later, app may already be up. That's why the man page's advice is to pair `Requires=` with `After=` too.

### Try it on your VM

```
$ systemctl show -p Wants -p Requires -p After cron.service    # Fedora: crond.service
$ systemctl list-dependencies cron.service                    # what it pulls in (Wants/Requires)
$ systemd-analyze critical-chain                              # the boot's ordering chain, with timings
```

`show` prints both axes for a real unit: its requirement lists and its ordering list, separately. (Expect a long `After=` line. systemd adds many ordering dependencies automatically.) `critical-chain` draws the ordering axis for the whole boot: `@` is when a unit became active, `+` is how long it took to start. You'll also see units marked `static` in `systemctl status`. Those have no `[Install]` section, so they can't be enabled. They only start because some other unit `Wants=` or `Requires=` them.

---

## 2. Reading `systemctl status` output, field by field

The study page shows `systemctl start/stop/restart/status` as commands to run, but doesn't walk through what a real status block is telling you. Here's an annotated one:

```
$ systemctl status nginx
● nginx.service - A high performance web server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
     Active: active (running) since Sun 2026-09-14 08:02:11 UTC; 2h 14min ago
       Docs: man:nginx(8)
   Main PID: 1042 (nginx)
      Tasks: 3 (limit: 4683)
     Memory: 5.2M
        CPU: 89ms
     CGroup: /system.slice/nginx.service
             ├─1042 nginx: master process /usr/sbin/nginx
             ├─1043 nginx: worker process
             └─1044 nginx: worker process

Sep 14 08:02:11 web01 systemd[1]: Starting A high performance web server...
Sep 14 08:02:11 web01 systemd[1]: Started A high performance web server.
```

Reading it top to bottom:

- **The colored dot** (`●`) is a fast visual summary before you read anything else — green for active/running, red for failed, white/grey for inactive or stopped. Worth knowing this exists even in a text-only environment where color may not render, because the word after `Active:` carries the same information regardless.
- **`Loaded:`** answers "does systemd know about this unit and where did its definition come from" — the file path tells you whether you're looking at a package-provided unit (`/usr/lib/systemd/system/`) or a local override (`/etc/systemd/system/`, which the study page's glossary already notes takes precedence). The `enabled`/`disabled` word here is the **boot-time axis** from the active/enabled diagram — it's telling you the same fact `systemctl is-enabled` would.
- **`Active:`** is the **current-state axis** — and it's actually two pieces of information in one line: the high-level state (`active`, `inactive`, `failed`, `activating`) and, in parentheses, a lower-level sub-state (`running`, `exited`, `dead`) that clarifies what that high-level state actually looks like for this unit type. A one-shot script-running unit with `RemainAfterExit=yes` can show `active (exited)` — meaning it ran, succeeded, and correctly isn't still running anything, which is entirely normal for that unit type and not evidence of a stalled service, unlike a long-running daemon unit where `exited` instead of `running` usually does mean something's wrong.
- **`Main PID:`** is the process systemd is actually tracking and will signal on `stop`/`restart` — useful when a service forks child workers, since this tells you which PID is the one systemd considers authoritative, distinct from the worker PIDs shown further down in the CGroup tree.
- **The `CGroup:` tree** shows every process systemd considers part of this unit, not just the main one — this is how systemd can cleanly stop a multi-process service (like nginx's master + worker model shown here) as one unit instead of hunting down each PID by hand, and it's the same cgroups mechanism the study page's container material already connects to resource limits.
- **The last lines** are a tail of that unit's own journal entries (the same data `journalctl -u nginx` would show in full) — a quick, unscoped preview without needing a separate command, useful for a fast first look before reaching for the full Week 7 `journalctl` toolkit if something looks wrong.

> **Analogy — A patient's chart at the foot of a hospital bed.**
>
> `Loaded:` is the admission paperwork: who this is, where their file lives, and whether they're scheduled to come back (`enabled`). `Active:` is the current vitals. `Main PID:` is the attending physician, the one person systemd deals with. The `CGroup:` tree is everyone else in the room. The log lines at the bottom are the latest nurse's notes. You read the chart top to bottom *before* you start poking the patient, and you read it the same way whether the patient is healthy or not.

### Another example: a failed unit, read the same way

```
$ systemctl status myapp
× myapp.service - My web app
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; preset: disabled)
     Active: failed (Result: exit-code) since Mon 2026-09-28 09:14:03 UTC; 12s ago
   Duration: 88ms
    Process: 2210 ExecStart=/usr/bin/python3 /opt/myapp/app.py (code=exited, status=1/FAILURE)
   Main PID: 2210 (code=exited, status=1/FAILURE)
        CPU: 143ms

Sep 28 09:14:03 web01 python3[2210]: OSError: [Errno 98] Address already in use
Sep 28 09:14:03 web01 systemd[1]: myapp.service: Main process exited, code=exited, status=1/FAILURE
Sep 28 09:14:03 web01 systemd[1]: myapp.service: Failed with result 'exit-code'.
```

Same chart, bad news. The dot is now `×` (red). `Loaded:` shows a local unit in `/etc/systemd/system/`. It's still `enabled`, so it will try again at the next boot and fail again. `Active: failed` gives the reason in parentheses. The `Process:` line is the most useful line on the page: the program *did* run (it lasted 88 ms) and exited with status 1, its own error. So the answer is in the program's own output. The last log lines show that output: another process already has the port. That's Lab 6, Ticket 2.

| Exit status in the Process: line | What it means | Where to look |
|---|---|---|
| `status=1/FAILURE` (or any small number) | The program ran and exited with an error | The program's own log lines: `journalctl -u <unit>` |
| `status=203/EXEC` | systemd couldn't launch the program at all | The `ExecStart=` path and the file's execute permission, not the app's logs (there are none) |
| `status=217/USER` | The `User=` in the unit doesn't exist | The `User=` line; `id <username>` |
| `status=200/CHDIR` | The `WorkingDirectory=` doesn't exist or isn't accessible | The `WorkingDirectory=` path |
| `code=killed, status=9/KILL` | Something sent it SIGKILL, often the kernel's OOM killer | `dmesg` or `journalctl -k` for "Out of memory" |

Statuses in the 200s are systemd's own codes for "I couldn't even set this up." Small numbers come from the program itself. That one distinction tells you whether to read the unit file or the application's logs.

### Another example: two healthy one-shot units that look different

```
# backup.service, WITH RemainAfterExit=yes
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
RemainAfterExit=yes

$ systemctl status backup
● backup.service - Nightly backup
     Active: active (exited) since Mon 2026-09-28 02:00:04 UTC; 7h ago
    Process: 881 ExecStart=/usr/local/bin/backup.sh (code=exited, status=0/SUCCESS)

# Same unit WITHOUT RemainAfterExit=yes
$ systemctl status backup
○ backup.service - Nightly backup
     Active: inactive (dead) since Mon 2026-09-28 02:00:09 UTC; 7h ago
    Process: 881 ExecStart=/usr/local/bin/backup.sh (code=exited, status=0/SUCCESS)
Sep 28 02:00:09 web01 systemd[1]: backup.service: Deactivated successfully.
```

Both runs succeeded. `status=0/SUCCESS` is the proof in both. `RemainAfterExit=yes` tells systemd to keep calling the unit "active" after its program exits, which suits setup-style units ("the firewall rules are loaded"). Without it, a successful one-shot unit goes back to `inactive (dead)` with *Deactivated successfully* in the log. **Neither `active (exited)` nor `inactive (dead)` means failure on its own. The exit status in the `Process:` line is what tells you.**

### Try it on your VM

```
$ systemctl --failed                                           # anything broken right now?
$ systemctl list-units --type=service --state=exited           # healthy units in active (exited)
$ systemctl status systemd-tmpfiles-setup                      # read one: Loaded, Active, Process
```

`--state=exited` lists the one-shot units that ran at boot and stayed "active." Every system has a dozen or more, and none of them is a problem. Pick one and read its status top to bottom. Notice `static` in its `Loaded:` line.

---

## 3. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| `After=` | `Wants=`/`Requires=` | `After=`/`Before=` control **ordering only**; `Wants=`/`Requires=` control **whether failure matters** — a unit can have either without the other, and usually needs both together |
| `Wants=` without `After=` | A guarantee the wanted unit starts first | `Wants=` only ensures the dependency is pulled in — it says nothing about sequencing; without `After=`, the two can start in either order |
| `active (exited)` | A service that crashed or stopped unexpectedly | Normal for a **one-shot** unit with `RemainAfterExit=yes` that ran to completion and correctly has nothing left running — the sub-state in parentheses is what tells you which case you're in |
| Killing worker PIDs shown in the CGroup tree | The correct way to stop a multi-process service | `systemctl stop` operates on the whole cgroup as one unit via the **Main PID**; manually killing worker PIDs can leave systemd's tracked state inconsistent with reality |

| `Requires=` | A guarantee the required unit starts first | `Requires=` alone starts both in parallel, so the dependent can already be running when the required unit fails. Pair it with `After=` |
| `After=` on its own | Starting the other unit | `After=` only orders units that are *already* being started. It pulls nothing in. If nothing else starts the other unit, `After=` does nothing |
| `status=203/EXEC` | A bug in the application | A 200-range status is systemd failing to set up or launch the program (203 = couldn't execute it). The program never ran, so check `ExecStart=` and permissions, not the app's logs |
| `inactive (dead)` after a one-shot job | The job never ran, or failed | Without `RemainAfterExit=yes`, a successful one-shot unit ends as `inactive (dead)`. Check the `Process:` line for `status=0/SUCCESS` |

---

## 4. Scenario quiz

**Q1.** A service's unit file has `Wants=postgresql.service` but no `After=postgresql.service`. The service occasionally fails to connect to its database right after a reboot, but works fine after a manual restart. Diagnose why, precisely.

<details><summary>Answer</summary>
<code>Wants=</code> only guarantees <code>postgresql.service</code> gets pulled into the boot sequence — it does not guarantee it starts <em>before</em> the dependent service. Without a matching <code>After=postgresql.service</code>, systemd is free to start both units in either order (or in parallel), so the dependent service sometimes wins the race and tries to connect before the database is listening. Adding <code>After=postgresql.service</code> alongside the existing <code>Wants=</code> fixes the ordering without changing the failure-tolerance behavior <code>Wants=</code> already provides.
</details>

**Q2.** `systemctl status` for a nightly-backup unit shows `Active: active (exited)`. A junior admin flags this as a failure needing investigation. Are they right?

<details><summary>Answer</summary>
Not necessarily — for a one-shot unit (the typical type for a backup job that runs once and finishes), <code>active (exited)</code> means it ran successfully to completion and correctly has no ongoing process, which is the expected end state. The distinguishing check is whether the unit is <code>Type=oneshot</code> (with <code>RemainAfterExit=yes</code>, which is what keeps it showing as active) and whether the exit code was 0 (visible via <code>systemctl status</code>'s recent journal lines or a full <code>journalctl -u</code> check) — <code>exited</code> alone isn't evidence of failure, only the combination of unit type and exit code tells you that.
</details>

**Q3.** A multi-worker service is misbehaving, and an admin runs `kill` directly against one worker PID visible in the `CGroup:` tree instead of using `systemctl restart`. What's the risk with this approach?

<details><summary>Answer</summary>
systemd tracks the unit as a whole via its Main PID and cgroup membership, not by any single worker PID — manually killing one worker can leave systemd's view of the unit's state (active/running) out of sync with what's actually happening (a service now missing a worker, possibly respawning it unpredictably depending on the unit's <code>Restart=</code> policy, or left in a degraded state systemd doesn't detect). <code>systemctl restart</code> operates through systemd's own tracking, ensuring the whole unit's process group is stopped and started coherently instead of leaving a partial state behind.
</details>

**Q4.** After a package upgrade, a service fails and `systemctl status` shows `Process: 4410 ExecStart=/opt/tool/bin/tool-server (code=exited, status=203/EXEC)`. A teammate starts reading the application's log files. Is that the right place to look?

<details><summary>Answer</summary>
No. <code>203/EXEC</code> means systemd couldn't execute the program at all, so it never ran and wrote no logs. The upgrade most likely moved or renamed the binary, or it lost its execute bit. Check the path from <code>ExecStart=</code> with <code>ls -l /opt/tool/bin/tool-server</code> (does it exist? is it executable?), and look for where the new version put it (e.g. <code>rpm -ql</code> or <code>dpkg -L</code> for the package). Then fix <code>ExecStart=</code> with <code>systemctl edit</code>, run <code>daemon-reload</code> if you edited the file directly, and restart.
</details>

**Q5.** `app.service` has `Requires=db.service` and `After=db.service`. An admin runs `systemctl stop db` for maintenance, then `systemctl start db` twenty minutes later. What state is `app.service` in, and would `Wants=` have behaved differently?

<details><summary>Answer</summary>
Stopping <code>db</code> also stopped <code>app</code>, because <code>Requires=</code> propagates an explicit stop (or restart) of the required unit to the units that require it. Starting <code>db</code> again does <strong>not</strong> start <code>app</code> back up. It stays stopped until someone starts it (<code>systemctl restart db</code> instead of stop/start would have restarted both). With <code>Wants=</code>, <code>app</code> would have kept running through the maintenance window and hit connection errors until <code>db</code> returned. Which is better depends on whether <code>app</code> can survive without its database.
</details>

**Q6.** A teammate checks `systemctl status backup` the morning after, sees `Active: inactive (dead)`, and concludes last night's backup never ran. What do you check before agreeing?

<details><summary>Answer</summary>
Look at the <code>Process:</code> line and the log tail in that same output. A line like <code>(code=exited, status=0/SUCCESS)</code> plus <em>Deactivated successfully</em> means it ran and succeeded. It's a <code>Type=oneshot</code> unit without <code>RemainAfterExit=yes</code>, so <code>inactive (dead)</code> is its normal resting state. <code>journalctl -u backup --since yesterday</code> shows the full run. If there's no <code>Process:</code> line and no log entries at all, it really didn't run, and the next question is what was supposed to trigger it (a timer: <code>systemctl list-timers</code>; or cron).
</details>
