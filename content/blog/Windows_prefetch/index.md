
---
title: "Windows Prefetch Files"
date: 2026-09-20
draft: false
tags: ["Digital forensics"]
summary: "Windows Internals- prefetch files"
---


![Prefetch image(Prefetch.png)


What they are Prefetch (.pf) files are a Windows Memory Manager feature (since XP) that logs data about application execution to speed up subsequent launches. Located at "C:\Windows\Prefetch\." Only enabled by default on non-SSD/non-server systems traditionally, though Win10/11 behavior varies (Superfetch/SysMain service controls it — check if enabled, as it affects evidentiary reliability).

What they do On execution, Windows monitors the first ~10 seconds of a process’s disk I/O — which files/DLLs get loaded, in what order. This data is cached so next launch, Windows can pre-load those resources into memory ahead of time, reducing load time. Forensically, this behavior is a side effect Windows exposes as an execution artifact — not designed for forensics but heavily used for it.

Naming convention EXECUTABLENAME-HASH.pf

Hash = 8-char hash derived from the file path (and on Win10+, sometimes additional entropy like command-line args in some versions) — used to differentiate same-named binaries run from different locations.
Same exe run from two different paths = two separate .pf files.
Formation / structure

Created on first execution if it doesn’t exist; updated (not recreated) on subsequent runs.
Max count historically capped (128 on XP/7, 1024 on Win8+) — once cap hit, oldest gets purged/overwritten, so this is a rolling window, not infinite history.
File is compressed (Win10+ uses MAM compression — need libscca or tools like PECmd to parse; raw parsing won’t work without decompression).
Version differs by OS (17 = XP, 23 = Win7, 26 = Win8.1, 30 = Win10, 31 = Win11) — parser needs to handle the right version.
Forensic value — what’s inside

Executable name and original file path
Run count
Last execution timestamp (single timestamp; Win8+ stores up to 8 last-run timestamps — useful for establishing a run pattern/frequency, not just “last seen”)
Creation timestamp of the .pf file itself ≈ first execution time (with caveats)
Volume information (serial number, path referenced) — useful for identifying source volume, including removable media
List of referenced files/directories — DLLs loaded, config files touched, and directories accessed during that init window. This is the big one for showing what a binary interacted with — e.g. a malware .pf referencing a dropped payload path, a staging directory, or a C2 config file is direct evidence of behavior even if those artifacts are later deleted.

Become a Medium member
How referenced files/directories are recorded Prefetch tracks resource access (file reads via NTFS $MFT resolution) during the trace window and stores them as a list of full paths in the file, tagged by volume. This is why prefetch is gold for showing execution context — you don’t just know mimikatz.exe ran, you may see it referenced sekurlsa.dll-equivalent modules or output file paths it touched, giving lateral evidence beyond a simple execution timestamp.

Forensic capabilities (summary)

Proves program execution ( “did X run” and when ) even after the binary is deleted
Approximates first-run and last-run time(s)
Shows execution frequency (run count)
Reveals original file path — useful when binary later renamed/moved/deleted
Indicates volume/device it ran from (USB, network share, etc. via volume serial)
Can reveal supporting files/directories touched at launch — sometimes recovers evidence of files no longer present on disk
Forensic limitations

Not enabled everywhere — historically disabled by default on server SKUs, and SysMain/Superfetch can be disabled by admin/malware to defeat this artifact deliberately (common AV-evasion/anti-forensic step — absence of expected .pf is itself a finding).
Rolling cap — high-volume systems purge old entries; can’t assume full history survives.
Only captures first ~10 seconds of execution — doesn’t capture everything the process does over its lifetime, only init-phase I/O.
Timestamp “last run” — can’t be used to establish frequency, only most recent.
.pf itself can be deleted/tampered by an attacker with sufficient privilege — should be cross-validated (Amcache, ShimCache/AppCompatCache, SRUM, EventLogs, MFT timestamps) rather than trusted alone.
Doesn’t prove successful execution or what the program did afterward — only that it launched and touched certain resources early on. No argument/command-line capture in most versions (some newer builds add limited info but don’t rely on it).
Volume serial can help but path normalization/interpretation of embedded directory strings needs parser support — raw strings inside the compressed blob aren’t always straightforward to string-grep.
Tools PECmd (Eric Zimmerman) is the standard, it parses version, run count, last-run(s), volume info, and full referenced file/directory list into CSV. libscca/scca.py (Python) for cross-platform/scripted parsing. Always verify parser matches file version format before trusting field-level output.
