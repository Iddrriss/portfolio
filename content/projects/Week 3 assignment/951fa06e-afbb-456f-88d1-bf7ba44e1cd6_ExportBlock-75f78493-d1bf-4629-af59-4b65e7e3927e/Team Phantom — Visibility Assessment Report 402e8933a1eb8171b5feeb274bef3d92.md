# Team Phantom — Visibility Assessment Report

*Attack-simulation* 

## 1. Executive Summary

Team Phantom's SOC simulated one trusted account being misused across two hosts to exercise the assigned scenarios — **File Creation and Modification** and **Simulated Insider Activity** — as a single story rather than two disconnected tests. The `oxseeker` account logged in after a burst of failed attempts, then used its access to create a hidden administrative account and to stage and change access to a file.

Each sequence ran twice per host — enrichment stopped, then running — to compare default vs enriched visibility on identical actions. Default logging reliably captured authentication and account-object activity but gave **zero visibility** into file staging or permission changes on either platform; enrichment closed that gap completely, adding full command detail, process ancestry, and, on Windows, several built-in heuristic detections (Section 6).

Confidence in reconstructing the incident from default logging alone is **PARTIAL**: account/privilege actions are well-supported on both platforms, but file-level tampering would have gone undetected without enrichment. See Section 9 for the identity-ambiguity and exfiltration gaps that limit a firmer verdict.

### The Incident, Step by Step

**Premise.** `oxseeker` is a legitimately provisioned account, and the session that followed is told as insider activity: a trusted user misusing real access. The logs cannot rule out that someone else held the credentials (Section 7), but the story commits to one reading: one account, one session, four moves.

1. **Logging in.** On the Linux host the account failed several SSH password attempts starting at 08:22, then got in at 08:25. The Windows host showed the same pattern at 18:07–18:08. Default logging captured both clearly.
2. **Creating a way back.** Using its access, the account created a second account named `backdoor` and gave it administrative rights (sudo group on Linux, Administrators on Windows). Default logs showed the account appear and who created it, but not the command used.
3. **Taking the data.** On Linux the account copied `/etc/passwd` to a hidden file, `/tmp/.staged_data` — in the scenario, a quick way to collect the host's account list for later targeting. On Windows it dropped a file in `C:\Users\Public`. Default logging recorded nothing for either.
4. **Changing who can touch it.** On Linux the account ran `chmod 600` so only it could read the staged copy. On Windows it ran `icacls` to grant Everyone full control of the staged file. Only the enriched pass (Auditd, Sysmon) recorded these commands.

**Intent note.** The motives in steps 3 and 4 are the scenario's script, not something the logs prove: they record the commands, never the intent.

## 2. Incident Scope and Questions

**Assigned scenarios (Team Phantom):**

- Scenario 1 — File Creation and Modification: create files and modify permissions to evaluate telemetry coverage.
- Scenario 2 — Simulated Insider Activity: perform a sequence of administrative actions and reconstruct the timeline.

**Systems in scope**

- Linux Wazuh agent "Evilcorp" (Ubuntu, Auditd-enabled)
- Windows 11 Wazuh agent "oxseeker-win" (Sysmon-enabled)
- Wazuh manager/indexer/dashboard (AWS, reached over a Tailscale private overlay)

**Investigation questions**

- Are the two assigned scenarios part of one incident, or separate, unrelated events?
- What is directly provable from the collected logs, versus reasonably inferred?
- What telemetry gap, if any, prevented a complete reconstruction?

**Assumptions** The account used for the initial login (`oxseeker`) is a legitimately provisioned account, and its activity is treated as insider misuse, as Scenario 2 intends. The failed-then-successful login leaves open that someone with weak or reused credentials produced the same logs; local logs alone cannot make that distinction (see Section 7, Correlation Analysis). The narrative commits to the insider reading, and the open question is carried into Section 9.

**Exclusions** No data left the lab network; "staging" actions are simulated and confined to `/tmp` and `/home` on the lab host; no production systems were touched; network-destination telemetry was not collected, so no claim of exfiltration is made.

## 3. Lab Architecture

The lab runs a Wazuh 4.14 all-in-one deployment (manager, Filebeat, indexer/OpenSearch, dashboard) on an AWS EC2 instance (Ubuntu). Both agents reach the manager over a Tailscale private overlay, which gives each host a stable virtual address independent of the underlying AWS or local network IP — this kept agent configuration stable across EC2 restarts and IP changes.

## 4. Methodology

Design. Both assigned scenarios were combined into a single continuous session per host, rather than tested as two disconnected activities: failed-then-successful login → short pause → backdoor account creation (Insider Activity) → file staging and permission change (File Creation and Modification). The same sequence was run on the Windows host with OS-equivalent commands. Between the default-logging and enriched passes, the team removed the backdoor account and staged file and re-ran the identical sequence from a clean state; those teardown commands (account removal, `/etc/passwd` edits) are visible in the raw Auditd stream around the same window and are test hygiene, not part of the simulated attacker's actions — they are excluded from the incident narrative in Section 5.

**Two-pass comparison.** Each sequence was executed twice per host under identical steps and fresh timestamps:

- **Pass 1 (baseline):** enrichment tool stopped (Auditd stopped on Linux; Sysmon stopped on Windows) — default logging only.
- **Pass 2 (enriched):** enrichment tool running with the custom ruleset active.

Running the identical steps twice, rather than once per condition with different actions, made the two passes directly comparable action-for-action.

**Evidence isolation.** Before each pass, the manager's event indices (`wazuh-alerts-*`, `wazuh-monitoring-*`, `wazuh-statistics-*`) were cleared so each pass's evidence set was unambiguous and not mixed with prior test data.

**Collection method.** Evidence was pulled from the Wazuh dashboard's Events view (Threat Hunting module), filtered by `agent.name` and `rule.id`, and cross-checked against raw Auditd records (`ausearch -k <key> -ts recent`) to confirm the full pipeline from syscall to alert. A `wazuh-logtest` check was used once to confirm a custom rule actually matched a captured log line before trusting dashboard results.

## 5. Timeline Reconstruction

All times below are as rendered by the Wazuh dashboard (WAT, UTC+1), 22 October 2026, across both hosts: the Linux agent "Evilcorp" (agent ID 006, Tailscale address 100.70.71.44) and the Windows agent "oxseeker-win" (agent ID 001, Tailscale address 100.73.170.33). Raw log lines are recorded in UTC, one hour behind the WAT times shown here (Section 3).

| Time | Host | User | Activity | Evidence source | Figure / Rule |
| --- | --- | --- | --- | --- | --- |
| 08:22:12 | Evilcorp | oxseeker | Failed login attempt (wrong password) | Linux default (syslog) | Fig. 2 / rule 2502 |
| 08:22:24 | Evilcorp | oxseeker | Failed login attempt | Linux default (unix_chkpwd) | Fig. 2 / rule 5557 |
| 08:22:26 | Evilcorp | oxseeker | Failed login attempt | Linux default (sshd) | Fig. 2 / rule 5760 |
| 08:25:26 | Evilcorp | oxseeker | Successful login after repeated failures | Linux default (sshd, PAM) | Fig. 2 / rule 5715, 5501 |
| 08:27:32 | Evilcorp | oxseeker (sudo) | Backdoor account “backdoor” created, added to sudo group | Linux default (PAM/sudo) | Fig. 2 / rule 5901, 5902, 5402 |
| 08:54:51 | Evilcorp | oxseeker | Session reopened for Pass 2 (enriched) | Linux default (PAM) | Fig. 3 |
| 08:55:10 | Evilcorp | oxseeker | File staged: `cp /etc/passwd /tmp/.staged_data` | Auditd (evilcorp_files/exec) | Fig. 3 / rule 100102 |
| 08:55:18 | Evilcorp | oxseeker | Permissions changed: `chmod 600 /tmp/.staged_data` | Auditd (evilcorp_perms) | Fig. 3 / rule 100102, 100104 |
| 08:55:26 | Evilcorp | oxseeker (sudo) | Backdoor account recreated (Pass 2 repeat) | Auditd (evilcorp_exec/files) | Fig. 3 / rule 100102, 100103 |
| 08:55:30 | Evilcorp | oxseeker (sudo) | `useradd`/`usermod` writes to `/etc/passwd`, `/etc/group`, `/etc/shadow` | Auditd (evilcorp_files/perms) | Fig. 3 / rule 100103, 100104 |
| 18:07:58–18:08:23 | oxseeker-win | Oxseeker | Repeated failed logon attempts (wrong password) | Windows default (Security log) | Fig. 4 / rule 60122 |
| 18:08:31 | oxseeker-win | Oxseeker | Successful logon after repeated failures | Windows default (Security log) | Fig. 4 / rule 60118, 67028 |
| 18:10:30–18:10:44 | oxseeker-win | Oxseeker | Backdoor account “backdoor” created, added to Administrators (Pass 1, Sysmon uninstalled) | Windows default (Security log) | Fig. 4 / rule 60154, 60109, 60110 |
| 18:30:17 | oxseeker-win | Oxseeker | Session reopened for Pass 2 (Sysmon reinstalled) | Windows default (Security log) | Fig. 5 |
| 18:31:24 | oxseeker-win | Oxseeker | File staged to `C:\Users\Public\staged.txt`; flagged by built-in rule as a malware-staging path | Sysmon | Fig. 5 / rule 92213, 100200 |
| 18:32:08 | oxseeker-win | Oxseeker | Permissions changed: `icacls.exe` invoked on staged file | Sysmon | Fig. 5 / rule 100200 |
| 18:38:56 | oxseeker-win | Oxseeker (net.exe) | Backdoor account recreated; `net.exe` discovery activity flagged | Sysmon | Fig. 5 / rule 60109, 60110, 92039, 92033, 100200 |

**Reading the timeline.** Pass 1 (08:22–08:27) shows only what default Linux logging captured: the failed-then-successful login and the account-creation commands — nothing about the file staging that happened alongside it in the full sequence, because that sequence was re-run clean in Pass 2 under enrichment. Pass 2 (08:54–08:55) repeats the identical action sequence with Auditd active, and is where the file-level evidence (staging, permission change) becomes visible for the first time.

## 6. Telemetry Analysis

### Default Linux logging (Pass 1, Auditd stopped)

[image: Figure 2 — Pass 1 baseline: default Linux/PAM/sshd events only]

*Figure 2.* With Auditd stopped, only authentication and account-object events appear — no reference anywhere to the file-staging activity from the same session (full rule mapping in the Section 5 timeline). The raw record behind one failed attempt is shown in Figure 2a.

[image: Figure 2a — sshd failed-login raw record]

*Figure 2a.* The raw record behind one of the failed-login alerts: `Failed password for oxseeker from 192.168.216.1 port 61792 ssh2`, captured via `journald` even with Auditd stopped. Default logging resolves the source IP of every login attempt — useful context Section 9 notes is not, by itself, enough to confirm whether the account holder or someone else was behind it.

### Auditd-enriched logging (Pass 2, Auditd running)

[image: Figure 3 — Pass 2 enriched: custom evilcorp_* Auditd rule hits]

*Figure 3.* With Auditd active, the identical sequence produces rule 100102/100103/100104 hits carrying full command arguments, working directory, and the real executable path for every step (see Section 5 for the complete mapping). Figures 3a and 3b below show what that resolves to at the raw-record level.

[image: Figure 3a — sudo useradd raw audit log]

*Figure 3a.* The raw `sudo` log line behind the backdoor-account alert: `oxseeker : TTY=/dev/pts/0 ; PWD=/home/oxseeker ; USER=root ; COMMAND=/usr/sbin/useradd -m backdoor`. This is the exact command an investigator would need, and that default logging alone does not surface.

[image: Figure 3b — cp raw audit record]

*Figure 3b.* The raw Auditd record for the staging command resolves the real binary behind `cp` (`exe=/usr/bin/gnucp`), the acting audit UID (`auid=1000`, which survives the later `sudo` to root), and the file's inode — forensic detail with no default-log equivalent.

### Windows: default Security log vs Sysmon

**Default Windows logging (Pass 1, Sysmon uninstalled via `Sysmon64.exe -u`, ≈ 18:07–18:10 WAT)**

[image: Figure 4 — Pass 1 baseline on Windows: logon and account-object events only, no Sysmon]

*Figure 4.* With Sysmon fully uninstalled (`Sysmon64.exe -u` — a plain `Stop-Service` call is blocked by its tamper protection), only authentication and account-object events appear. The Security log alone resolves the backdoor account's creation, its add to Administrators, and its add to the Users and Domain Users groups (rules 60109, 60110, 60154, 60170, 60160) — including the subject/target identity link shown in Figure 4a — but nothing about the `icacls`/`Copy-Item` commands from the same session (full rule mapping in Section 5).

[image: Figure 4a — Windows account-creation raw record]

*Figure 4a.* The raw Security-log record behind the account-creation alert resolves both ends of the action without any enrichment: `subjectUserName: Oxseeker` (the acting account, SID `S-1-5-21-2235569076-3001256017-3369850418-1001`) created `targetUserName: backdoor` (SID `...-1010`). Default Windows account-object logging is genuinely strong on *who created what* — the gap is entirely on the file/process side (Figures 4 vs. 5).

**Sysmon-enriched logging (Pass 2, Sysmon reinstalled, ≈ 18:30–18:39 WAT)**

[image: Figure 5 — Pass 2 enriched on Windows: Sysmon process creation plus built-in heuristic detections]

*Figure 5.* With Sysmon reinstalled, rule 100200 captures every process in the chain, including `icacls.exe` with its command line (Figure 5a). Two built-in Sysmon-based rules also fired with no custom authoring: 92213 (file dropped in a malware-staging path, rule level 15) and 92039/92033 (`net.exe`/PowerShell discovery activity) — see Section 5 for the complete mapping.

[image: Figure 5a — icacls command line]

*Figure 5a.* The full Sysmon command line behind the permissions alert: `"C:\WINDOWS\system32\icacls.exe" C:\Users\Public\staged.txt /grant Everyone:F` — granting the Everyone group full control, the same ACL-weakening pattern real intrusions use to keep a staged file accessible. Default Windows logging records that `icacls.exe` ran as `oxseeker`; it does not record what it was told to do.

## 7. Correlation Analysis

The two assigned scenarios are treated as **one correlated incident**, not two separate events, on the following evidence:

- **Same audit identity (`auid=1000`)** ties every action — the login, the backdoor account creation, and the file staging/permission changes — back to the same originating login session, even though several steps run with effective UID 0 via `sudo`. Auditd's `auid` field persists through privilege elevation specifically so this kind of chain can be proven.
- **Same host and session ID** (`ses=8` in the raw Auditd records) links every step of Pass 2 as part of one continuous terminal session.
- **Contiguous timeline** — no unexplained gap separates the account-creation activity from the file-staging activity; they occur within the same short window (Section 5).

**Competing explanation.** The account-creation and file-staging actions were performed under the `oxseeker` credential, which local logs show as the acting identity regardless of whether the real account holder performed them or an external actor used stolen credentials for that account. Available telemetry (local Auditd/Linux logs) **cannot distinguish** insider misuse of a legitimate session from external compromise of that account — that would require additional evidence such as login source/geolocation correlation or MFA logs, neither of which was in scope for this lab.

**Confidence.** The two scenarios correlating into a single incident is supported with **high confidence** (identity, host, and timeline all agree). Whether the actor behind that identity was an insider or an external party using stolen credentials remains **unresolved** by the available evidence — this is carried forward as an explicit gap in Section 9.

## 8. Visibility Comparison

| Activity / question | Windows Logs | Sysmon | Linux Logs | Auditd | Analyst note |
| --- | --- | --- | --- | --- | --- |
| Who performed it? | Yes | Yes | Yes | Yes | Both platforms' default logs name the acting account reliably (Windows: subjectUserName/targetUserName fields; Linux: auid persists through sudo). |
| What command ran? | No | Yes | No | Yes | Default logging on both platforms shows that privilege was used, not what command followed; Sysmon and Auditd both capture the full command line. |
| Which process was involved? | No | Yes | No | Yes | Neither default Windows nor default Linux logging exposes process ancestry; Sysmon and Auditd both close this gap identically. |
| What network connection occurred? | No | No | No | No | Not collected on either platform — no telemetry source in scope captured destination IPs or connection data; confirmed gap (Section 9). |
| What file or permission changed? | No | Partial | No | Yes | Default logging is silent on file activity on both platforms. Auditd (Linux) captured explicit permission-change syscalls; Sysmon (Windows) confirmed the `icacls.exe` command ran but, as configured in this lab, did not emit a dedicated permission-change event — the change is inferred from the captured command line rather than a typed file-permission event. |

## 9. Visibility Gap Analysis

| Unanswered question | Why evidence is missing | Required telemetry/configuration | Priority |
| --- | --- | --- | --- |
| Was the acting identity the genuine account holder, or an external actor using stolen credentials? | Both platforms' logs record the authenticated identity performing each action, but cannot verify who was physically behind the keyboard | Login source/geolocation correlation, MFA/conditional-access logs — none in scope for this lab | High |
| Did any data actually leave either host? | No network-destination telemetry (firewall, NetFlow, DNS, or a network-connection Sysmon/Auditd rule) was collected on either platform | Network connection logging on both hosts; the staged files never left `/tmp` (Linux) or `C:\Users\Public` (Windows) | High |
| Does Sysmon, as configured, distinguish a permission change from a mere process execution? | The Sysmon ruleset used here (100200–100203) has no dedicated “permission changed” event type the way Auditd's `evilcorp_perms` rule does | A Sysmon config/rule targeting registry or ACL-change events, or a FIM rule with permission-change detail | Medium |

## 10. Findings and Lessons Learned

- **Default host logging is authentication- and account-object-centric on both platforms.** Windows Security log and Linux PAM/sshd/syslog alike reliably answer who logged in and what account/group changed, but gave zero visibility into file creation, permission changes, or command-line detail — confirmed as a matched pair (Linux and Windows, each run twice) rather than assumed from one platform.
- **Enrichment value isn't limited to rules a team writes itself.** Auditd and Sysmon closed the file/process gap completely, and on Windows also triggered built-in heuristic detections (rules 92213, 92039, 92033) with no custom authoring required.
- **The detection pipeline has more than one place to silently fail.** Raw Auditd records were confirmed present while the corresponding Wazuh alert was still missing, because the manager-side custom rules file had been left disabled from an earlier pass — syscall-level logging working correctly is not proof the SIEM layer is alerting on it.
- **Identity in logs is not identity in fact.** `auid` (Linux) and the subject/target user fields (Windows) reliably prove which session performed an action, including through privilege elevation, but neither proves who was authenticated to that session originally.

## 11. Recommendations

**Immediate**

- Enable Auditd (Linux) and Sysmon (Windows) on every host that handles sensitive files, not only hosts already flagged for privileged-account monitoring — Pass 1 showed this is a complete blind spot otherwise.
- Alert on new local administrator/sudo-group membership changes in near-real time; this was one of the few signals default logging produced reliably and is cheap to act on.
- After any change to custom detection rules, verify end-to-end delivery (raw log → alert) with a test event before trusting the dashboard — this exercise found a real gap where the rule file had been left disabled.

**Longer-term**

- Collect network-connection telemetry (NetFlow, DNS, or Sysmon Event ID 3 / Auditd network-socket auditing) to close the exfiltration-visibility gap identified in Section 9.
- Pair host logins with an independent identity signal (MFA logs, login-source geolocation) so that "who performed this" can be answered with confidence rather than inferred from account name alone.
- Periodically audit which hosts have enrichment tooling active versus hosts running default logging only, since this gap is easy to lose track of as a fleet grows.

## 12. Conclusion and Management Brief

**What happened.** On both hosts, the legitimate `oxseeker` account logged in after several failed attempts, then used its access the way a malicious insider would: it created a hidden second account with administrative rights, copied a file to a staging location, and changed who could access it. Shared login identity, host, and timeline tie the account-creation and file activity together as one incident (Section 7).

**What remains uncertain.** Whether the acting party was the genuine account holder or someone using stolen credentials cannot be determined from the telemetry collected (Section 9). No network-destination telemetry was in scope, so the file activity observed is local staging, not proven exfiltration.

**Confidence level: PARTIALLY.** Both platforms support the narrative well once enrichment is active; default logging alone would have missed the file-tampering activity entirely on either one.

**Message for management.** Default host logging is not sufficient to investigate file-level tampering on either Windows or Linux — only account and authentication activity is reliably visible without enrichment. Auditd and Sysmon close that gap but require deliberate configuration and independent verification, not just installation (Section 10).