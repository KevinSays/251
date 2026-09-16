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
- **`Active:`** is the **current-state axis** — and it's actually two pieces of information in one line: the high-level state (`active`, `inactive`, `failed`, `activating`) and, in parentheses, a lower-level sub-state (`running`, `exited`, `dead`) that clarifies what that high-level state actually looks like for this unit type. A one-shot script-running unit can show `active (exited)` — meaning it ran, succeeded, and correctly isn't still running anything, which is entirely normal for that unit type and not evidence of a stalled service, unlike a long-running daemon unit where `exited` instead of `running` usually does mean something's wrong.
- **`Main PID:`** is the process systemd is actually tracking and will signal on `stop`/`restart` — useful when a service forks child workers, since this tells you which PID is the one systemd considers authoritative, distinct from the worker PIDs shown further down in the CGroup tree.
- **The `CGroup:` tree** shows every process systemd considers part of this unit, not just the main one — this is how systemd can cleanly stop a multi-process service (like nginx's master + worker model shown here) as one unit instead of hunting down each PID by hand, and it's the same cgroups mechanism the study page's container material already connects to resource limits.
- **The last lines** are a tail of that unit's own journal entries (the same data `journalctl -u nginx` would show in full) — a quick, unscoped preview without needing a separate command, useful for a fast first look before reaching for the full Week 7 `journalctl` toolkit if something looks wrong.

---

## 3. Exam traps: paired confusions

| Students conflate... | ...with | What actually discriminates them |
|---|---|---|
| `After=` | `Wants=`/`Requires=` | `After=`/`Before=` control **ordering only**; `Wants=`/`Requires=` control **whether failure matters** — a unit can have either without the other, and usually needs both together |
| `Wants=` without `After=` | A guarantee the wanted unit starts first | `Wants=` only ensures the dependency is pulled in — it says nothing about sequencing; without `After=`, the two can start in either order |
| `active (exited)` | A service that crashed or stopped unexpectedly | Entirely normal for a **one-shot** unit that ran to completion and correctly has nothing left running — the sub-state in parentheses is what tells you which case you're in |
| Killing worker PIDs shown in the CGroup tree | The correct way to stop a multi-process service | `systemctl stop` operates on the whole cgroup as one unit via the **Main PID**; manually killing worker PIDs can leave systemd's tracked state inconsistent with reality |

---

## 4. Scenario quiz

**Q1.** A service's unit file has `Wants=postgresql.service` but no `After=postgresql.service`. The service occasionally fails to connect to its database right after a reboot, but works fine after a manual restart. Diagnose why, precisely.

<details><summary>Answer</summary>
<code>Wants=</code> only guarantees <code>postgresql.service</code> gets pulled into the boot sequence — it does not guarantee it starts <em>before</em> the dependent service. Without a matching <code>After=postgresql.service</code>, systemd is free to start both units in either order (or in parallel), so the dependent service sometimes wins the race and tries to connect before the database is listening. Adding <code>After=postgresql.service</code> alongside the existing <code>Wants=</code> fixes the ordering without changing the failure-tolerance behavior <code>Wants=</code> already provides.
</details>

**Q2.** `systemctl status` for a nightly-backup unit shows `Active: active (exited)`. A junior admin flags this as a failure needing investigation. Are they right?

<details><summary>Answer</summary>
Not necessarily — for a one-shot unit (the typical type for a backup job that runs once and finishes), <code>active (exited)</code> means it ran successfully to completion and correctly has no ongoing process, which is the expected end state. The distinguishing check is whether the unit's <code>ExecStart</code> is a one-shot type and whether the exit code was 0 (visible via <code>systemctl status</code>'s recent journal lines or a full <code>journalctl -u</code> check) — <code>exited</code> alone isn't evidence of failure, only the combination of unit type and exit code tells you that.
</details>

**Q3.** A multi-worker service is misbehaving, and an admin runs `kill` directly against one worker PID visible in the `CGroup:` tree instead of using `systemctl restart`. What's the risk with this approach?

<details><summary>Answer</summary>
systemd tracks the unit as a whole via its Main PID and cgroup membership, not by any single worker PID — manually killing one worker can leave systemd's view of the unit's state (active/running) out of sync with what's actually happening (a service now missing a worker, possibly respawning it unpredictably depending on the unit's <code>Restart=</code> policy, or left in a degraded state systemd doesn't detect). <code>systemctl restart</code> operates through systemd's own tracking, ensuring the whole unit's process group is stopped and started coherently instead of leaving a partial state behind.
</details>
