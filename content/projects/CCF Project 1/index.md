---
title: "Cyber career Fair SOC bootcamp Assignment 1"
date: 2026-10-10
draft: false
featured: true
summary: "setting up a controlled virtual environment."
tags: ["Wireshark", "Ubuntu server", "Baseline test", "Linux", "Windows", "Monitoring"]
---







# Executive Summary
For this project I built a home lab consisting of two virtual machines (an Ubuntu server and a Kali Linux testing machine) via VM8NET NAT and a Windows 11 workstation ,. There was communication between them tested with ssh and ping

Established the normal operating baseline between them before testing to understand what normal looks like on each one. On Windows, I looked at processes, services, open ports and the firewall. On Ubuntu, I made users and a group, set file permissions for both users and groups,  normal and failed logins, revoked and maintained access and set up a ssh. Mapped network traces with wireshark and auth.log. for the security testing ran accepted and failed login attempt between users with evidence in the auth.log 
My biggest take away is how processes are interconnected and you would have to understand what normal looks like before making security decisions





# 1. Project Scenario

Your team has been asked to set up and assess a small organization's IT environment before security monitoring is introduced. You will build the environment, establish what normal looks like, perform controlled activities, and find the evidence they leave behind. The goal is not simply to build virtual machines. Your team should be able to answer:

|                                                                                                                                                                                                                            |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The question this project answers<br><br>- What systems do we have, how do they communicate, what does normal activity look like, what changes when we perform an action, and where can we find evidence of that activity? |

## Required Environment:

### Windows
This would be my host machine, running windows 11 pro workstation core i5 8th gen 8gb and 256gb 
LAB IPv4 Address --  192.168.216.1 / 255.255.255.0 (VMnet8)
Mac Address -- F4-D1-08-E8-02-29  
LAB Mac -- 00-50-56-C0-00-08
IP / mask  --192.168.43.209 / 255.255.255.0 (DHCP)|
Default gateway --192.168.43.1
DHCP server --192.168.43.1
DNS -- 192.168.43.1

![Pasted image 20261007154049](Pasted%20image%2020261007154049.png)

![Pasted image 20261007154230](Pasted%20image%2020261007154230.png)

### Ubuntu
This running on VMware allocated system specs 20gb (minimum required), 2gb ram and 2 processor cores 
LAB ip v4 address --> 192.168.216.134 
Mac address --> 00:0c: 29:d3:03:17
Default gateway --> 192.168.216.2

![Pasted image 20261007155303](Pasted%20image%2020261007155303.png)

![Pasted image 20261007155446](Pasted%20image%2020261007155446.png)

### Kali 
This running on VMware allocated system specs 50gb (minimum required), 2gb ram and 2 processor cores 
LAB ip v4 address --> 192.168.216.131 
Mac address --> 00:0c:29:4e:63:fd 
Default gateway --> 192.168.216.2

![Pasted image 20261007160002](Pasted%20image%2020261007160002.png)

![Pasted image 20261007160111](Pasted%20image%2020261007160111.png)


### Network diagram
![566](Pasted%20image%2020261007235329.png)

| Machine                   | ip address      | MAC address        | LAB role                     | Network type | Gateway       |
| ------------------------- | --------------- | ------------------ | ---------------------------- | ------------ | ------------- |
| Windows 11 pro            | 192.168.216.1   | F4-D1-08-E8-02-29  | Windows endpoint and Host OS | VM8net       | 192.168.216.2 |
| Ubuntu server (Evil corp) | 192.168.216.134 | 00:0c: 29:d3:03:17 | LAB server                   | en33         | 192.168.216.2 |
| KALI (forensics)          | 192.168.216.131 | 00:0c:29:4e:63:fd  | Attack machine               | eth0         | 192.168.216.2 |

#### Evidence of Connectivity

![Pasted image 20261008000626](Pasted%20image%2020261008000626.png)
Ubuntu Server (Evil corp)

![Pasted image 20261008001149](Pasted%20image%2020261008001149.png)

Windows endpoint (0xseeker)

![Pasted image 20261008001529](Pasted%20image%2020261008001529.png)
KALI (Forensics)

SSH login from Windows end point to Ubuntu server
![Pasted image 20261008002611](Pasted%20image%2020261008002611.png)

## Windows running processes
| Process name                      | what it does                                                         | is it Expected?                           | Flag as Suspicious if                                                    |
| --------------------------------- | -------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------ |
| agent_ovpnconnect                 | Background helper for OpenVPN Connect, handling the VPN service side | Yes, I run some apps that require openvpn | It runs from outside C:\Program Files\OpenVPN Connect, is unsigned,      |
| AlpsAlpine Pointing-device Driver | Touchpad driver service, for gestures and settings                   | Yes, any  laptop with a touchpad has them | It sends network traffic, runs from a user folder like AppData or  Temp  |
| Antimalware Service Executable    | The Microsoft Defender scanning engine                               | Yes                                       | It runs outside `C:\ProgramData\Microsoft\Windows Defender\Platform\...` |
![Pasted image 20261008004442](Pasted%20image%2020261008004442.png)

Active Listening services ![Pasted image 20261008004956](Pasted%20image%2020261008004956.png)

Firewall service 
![Pasted image 20261008005045](Pasted%20image%2020261008005045.png)

Active connection
![Pasted image 20261008005139](Pasted%20image%2020261008005139.png)

### Linux Administration

Users --> jane_dev and dave_researcher
Group --> build_team
![Pasted image 20261008192407](Pasted%20image%2020261008192407.png)

Successfully created the users, added them to a group and removed one of them from the group

![Pasted image 20261008193718](Pasted%20image%2020261008193718.png)
tested their access with a file and revoked and gave access back to dave_researcher
![Pasted image 20261008193819](Pasted%20image%2020261008193819.png)

ssh to successfully login as jane
![Pasted image 20261008194513](Pasted%20image%2020261008194513.png)

controlled failed login
![Pasted image 20261008194734](Pasted%20image%2020261008194734.png)

evidence of failed login attempts
![Pasted image 20261008195101](Pasted%20image%2020261008195101.png)

user creation evidence
![Pasted image 20261008195331](Pasted%20image%2020261008195331.png)
group membership
![Pasted image 20261008195627](Pasted%20image%2020261008195627.png)

![Pasted image 20261008213859](Pasted%20image%2020261008213859.png)
logs for failed and accepted ssh login

access and modified time
![Pasted image 20261008214047](Pasted%20image%2020261008214047.png)


## Wireshark
Capturing normal traffic: ping, DNS lookup, web/HTTPS, a TCP connection, SSH where available.
![Pasted image 20261008223437](Pasted%20image%2020261008223437.png)

DNS lookup `nslookup example.com`
![Pasted image 20261008223823](Pasted%20image%2020261008223823.png)

TCP handshake

![Pasted image 20261008224016](Pasted%20image%2020261008224016.png)

TLS

![Pasted image 20261008224405](Pasted%20image%2020261008224405.png)

HTTP

![Pasted image 20261008224435](Pasted%20image%2020261008224435.png)

ARP

![Pasted image 20261008224526](Pasted%20image%2020261008224526.png)

## ICMP ping (VM → Windows host)

|Field|Evidence|
|---|---|
|**Source IP**|192.168.216.1 (Windows host) in the selected reply (packet 17)|
|**Destination IP**|192.168.216.134 (Ubuntu VM). Requests go the opposite way.|
|**Protocol**|ICMP over IPv4|
|**Ports**|None (ICMP is network layer, with no TCP/UDP ports)|
|**Key packet info**|4 echo request/reply pairs, ID 0x515a, seq 1 to 4. TTL 64 in requests (Linux) and 128 in replies (Windows). 98 bytes per packet.|

**Activity:** `ping -c 4 192.168.216.1` run on the Ubuntu VM. The host replied to each request.

**Why it appears in Wireshark:** the capture was on the VMnet8 adapter, which is the host's own interface on the VM network. The ping's destination was the host, so every packet crossed it. ICMP is unencrypted, so all fields are readable.


## Controlled security activity

### Port scan

 Using the command `nmap -sS -p 22 192.168.216.134.` This Nmap scan is used to find out the running services or open port of a target machine. I scanned using kali -> `192.168.216.131` against ubuntu `192.168.216.134` 

![Pasted image 20261008231040](Pasted%20image%2020261008231040.png)


reply to show that the port is opened. This shows that port 22 is open


![Pasted image 20261008231312](Pasted%20image%2020261008231312.png)

filter with  `tcp.flags.syn==1 && tcp.flags.ack==1` to find the port that responded
![Pasted image 20261008231514](Pasted%20image%2020261008231514.png)

### Benign Scan

Using the command `nmap -sV -p 22 192.168.216.134.` This Nmap scan is an harmless reconnaissance command used to gather information about a target. I scanned using kali -> `192.168.216.131` against ubuntu `192.168.216.134` 

![Pasted image 20261008232103](Pasted%20image%2020261008232103.png)

The resulting out on wireshark shows that after Nmap established the connection, it probes to find the running service info then disconnected gracefully as though it's a normal connection.

![Pasted image 20261008232000](Pasted%20image%2020261008232000.png)

Proof of benigh scan on the auth.log
![Pasted image 20261009000722](Pasted%20image%2020261009000722.png)

## Before and After comparison

For an active server ssh is crucial to be able unable for easier and remote access, but something it is best to restrict the ssh access to certain levels or to certain individuals. For this, I would playing a scenario where the user `jane_dev` is an high profile account carry company secret that shouldn't be accessed outside it's environment regardless of the user. Here i would be disabling ssh access for the user `jane_dev`. 

#### Before 
![Pasted image 20261009002100](Pasted%20image%2020261009002100.png)

**CHANGES** -->  Heading over to `sudo nano /etc/ssh/sshd_config` and adding the user i want to restrict  `jane_dev` 

![Pasted image 20261009002409](Pasted%20image%2020261009002409.png)


#### After
![Pasted image 20261009002801](Pasted%20image%2020261009002801.png)

The access changed immediately I added the rule and reload the server, one important caveat the rule added doesn't revoke any current access until the person logs out and tries to login


## Mini Investigation
Investigating a suspicious activities on the network;

**Observation** 
First finding a burst of `SYN` traffic coming from an unsolicited ip address communicating across multiple ports 
![Pasted image 20261009004347](Pasted%20image%2020261009004347.png)

**Interpretation** 
I checked wireshark to view the source information `192.168.216.131` and since it was probing multiple port at the same time in milli-seconds I can say it was definitely a port scanning attempt. an alternate reason would be maybe a service mal-functioning. 

**Conclusion** 
This is a reconnaissance attempt and it needs further investigation to find out what services responded to the prob and understand the damage that could be done.

**Additional Evidence** 
Checking the /var/log/auth.log to find support evidence

**Recommendation** is to escalate to medium rated incident but most be checked


## Assignment conclusion 
This is probably the first time I had to set up and "Document" the process of setting up VMs and lab environments. Established and documented the normal baseline against abnormal, ran and detected reconnaissance attempts, investigated failed and accepted logons.  
Overall this practical assignment gave me that knowledge refresh I needed and made me understand how things run in a real security environment.
