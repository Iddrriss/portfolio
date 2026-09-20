
---
title: "DETECTING RECON ATTEMPTS"
date: 2026-09-20
draft: false
tags: ["Networking", "Digital forensics"]
summary: "Learning to detect network reconnaissance"
---

Your writing here, referencing images the same way:

![Network scan](network_recon.png)


Network reconnaissance (network recon) is the process of gathering information about a network to discover its devices, services, ports, and possible weaknesses before taking any further action.

The Goals of recon:

    Is anything active at this address at all
    Which ports, on a given host are open, closed or filtered
    What specific Software and version is running behind an open port?

TCP connections begins with a three-way handshake, SYN from the intiator, SYN.ACK from the listener if the port is open then comes a final ACK from the initiator.

RFC 793 defines how a TCP stack must respond to segment arriving with any given connection of control flag(SYN or ACK) for a part in any state(open or closed).

If a SYN segment arrives at a port with no application listening, the recieving stack must respond with a RST.ACK immediatly and unconditionally any segment arriving at a port with or without SYN.ACK must be responded RST

i.e Silence from an open port, a RST from a closed one
Get OxSEEKER’s stories in your inbox

Join Medium for free to get updates from this writer.

Remember me for faster sign in

Common NMAP probes (since it’s the most widely used tool)

    Nmap -sT:uses the OS’s standard connect () socket call instead of crafting raw packet directly, completes the normal TCP handshake and immediatly ends the connection with RST or the target just responds with RST if the port is closed, one important caveat though is; Everything here is logged as the target system treats this as an active connection and it is much slower to propergate.
    TCP SYN Scan or Half-open Scan (Nmap -sS) : this is the most widely used scans because it is faster and more stealtlier. The scanner sends a SYN, like a normal handshake and if the port is opened the listener responds with SYN.ACK rather than completing the handshake it drops the connection directly. if it is closed the listener just respond with RST
    Since it never really completes the handshake many systems don’t log it as an event although all of this is still visible on the network telemetry
    TCP FIN, NULL and XMAS:
    TCP FIN sends a single segment with a FIN flag set
    TCP NULL sends a single segment with no flag at all
    TCP XMAS: sends a single segement with the FIN, PSH and URG flags allset simultaneously
    These flags when set out appear as malformed packets and most OS don’t know how to manage those packets so by inference this mean that an open port is shown by the absence of response while a closed port is confirmed by the explict presense of RST.
    This technique only works against UNIX based OS eg Linux and MAC,
    This technique won’t work against a Windsows machine because it does not follow the same silent-discard convention so it’s responds with RST regardless of the situation. This can say something about the attacker.
    TCP ACK scan: This sends a single segment with only the ACK flag set, thi technique is mostly used to find network filters or firewalls, if a RST comes back, the segement reached an unfiltered port if nothing comes back or ICMP unreachable that means there is a firewall on the target.

HOST DISCOVERY

This is a light weight sweep simple made to determine which addresses are active.
Types

    ICMP Echo sweeps: This sends requests in sequence across the a range of addresses with any ICMP echo reply confirming a live host at that address
    ARP-based Discovery: This work on a local network by sending an ARP discover request and seeing which device responds

Wireshark filters

    tcp.flags.syn == 1 && tcp.flags.ack ==0 -> connection -scan
    tcp.flags == 0x000 -> Null scan
    tcp.flags.fin ==1 && tcp.flags.syn == 0 ->FIN scan
    tcp.flags.fin ==1 && tcp.flags.psh ==1 && tcp.flags.urg ==1 -> xmas scan
    tcp.flags.ack ==1 && tcp.flags.syn == 0 && tcp.len ==0 -> ACK scan
    ICMP.type ==3 && ICMP.code ==3 unreachable
    ICMP.type ==8 ->ICMP host discovery

That’s all for todays learning, ciao ;)
