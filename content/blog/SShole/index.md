---
title: "SSHole (CVE-2026-123158)"
date: 2026-11-15
draft: false
tags: ["CVE", "Vulnerability", "Alphine", "Open SSH", "RCE"]
summary: "A 2016 Security Fix Became a 2026 Authentication Bypass"
---

![HTB sshole diagram](sshole.png)

An account with no password should never accept one. SSHole breaks that rule, and the reason is more interesting than the name.

Disclosed on 5 November 2026, CVE-2026-123158 is a high-severity authentication bypass (CVSS 8.1) in OpenSSH deployments that use musl libc without PAM. The discovery is credited to Özgün Kültekin of Wakefield.

## The bug in one chain

Four behaviors combine, and none of them is a bug on its own.

1. **A disabled account stores `*` in `/etc/shadow`.** This is normal for key-only and service accounts.
2. **sshd still hashes the password for accounts with no usable hash.** In 2016, OpenSSH fixed a timing leak that let attackers tell which usernames exist (CVE-2016-6210). The fix made sshd borrow another account's salt and hash the submitted password anyway.
3. **musl's `crypt()` fails on long input.** When the password exceeds 256 bytes, it returns `*` as an error value instead of a hash. The accepted length range is 257 to 1024 characters.
4. **sshd treats the error as a match.** The returned `*` equals the stored `*`, and the comparison passes. The weakness class is CWE-394, an unexpected return value.

The server does check the password. It gets an error back, and the error happens to equal the stored value. The 2016 fix is what made this path reachable.

## Who is exposed

All of these must be true:

- OpenSSH older than 10.6p2, built against musl, without PAM, with `PasswordAuthentication yes`
- An SSH-eligible account with `*` in its password field
- At least one other local account with a `$5$` or `$6$` hash, which supplies the borrowed salt

Dropbear, glibc-based distributions, non-Linux systems and PAM-based password checks are not affected. Reported affected distributions include Alpine 3.16 to 3.24, OpenWrt and Buildroot. Accounts locked with `!` are not susceptible through the `*` comparison. That distinction between `*` and `!` is worth a line in any audit.

Alpine's cloud images deserve special attention. Official images from 3.18.11 through 3.24.3 ship an `alpine` account with a disabled password and passwordless `doas`. They are safe at first boot. They become exploitable once any `$6$` hash exists on the host, for example when an admin sets a console password. A routine admin action creates the missing precondition, and nothing in the SSH configuration changes. Bypass to root takes one command.

## Exploitation was immediate

One researcher reported the first exploitation attempt less than nine hours after the CVE was published, and confirmed successful root access on an Alpine honeypot. Treat any exposed musl host without PAM as potentially compromised until you have checked it.

## What it looks like to an investigator

This is the part most coverage skips. The exploit leaves a narrow footprint, so the investigation has to be disciplined.

**The log gap.** Standard SSH logs generally don't record password length. You will not see the oversized password. What you see is a successful password login to an account that should not have been able to log in with a password. That mismatch is your anchor.

**Questions in order.** Each answer narrows the next:

- **What is the host?** The distribution and version tell you whether it is musl-based and which package version is installed. Alpine backports fixes to older OpenSSH branches, so the OpenSSH version number alone proves nothing.
- **Is the CVE reachable?** Check PAM, `PasswordAuthentication`, and whether a `$5$` or `$6$` hash exists on the box.
- **When did it happen?** Look for the first successful password authentication on a key-only account. Then check whether sshd was restarted after patching. Upgrading the package does not replace the running daemon. A patched package with an unrestarted daemon is still vulnerable.
- **Which account was abused?** It will be the one with `*` in shadow.
- **What happened after?** Treat the login as the start of the timeline. Pivot to shell history, process activity, outbound connections and network scanning from the compromised host. A jumpbox is valuable because of what it can reach, so the post-access activity matters more than the bypass itself.

**Detection logic.** A hunt should key on one condition: a successful password authentication for an account whose shadow entry is `*`. Correlate it with the account's password state at login time, with what ran afterward, and with changes to SSH keys or accounts. Network restrictions reduce exposure but don't replace patching.

## Fix and hardening

1. Upgrade to OpenSSH 10.6p2 or your distribution's fixed package. For Alpine, run `apk update && apk upgrade`, then restart sshd.
2. If you can't patch immediately, set `PasswordAuthentication no`.
3. Audit service and deployment accounts that can open SSH sessions, and any account with passwordless `doas` or `sudo`.
4. Restrict SSH to management networks, VPNs or a bastion.

## The lesson

The root cause is a trusted error path. Code written to prevent username enumeration assumed the hash function would always return a hash. A less common C library broke that assumption ten years later. Review the glue code between components, because that is where assumptions go unchecked.

*Credit: Özgün Kültekin for the discovery. Primary sources to cite: OpenSSH 10.6p2 release notes and the Alpine Linux security advisory of 5 November 2026.*
