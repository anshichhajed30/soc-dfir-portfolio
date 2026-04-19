# OpenWire — CyberDefenders Lab Writeup

**Platform:** CyberDefenders  
**Category:** Network Forensics  
**Difficulty:** Medium  
**Status:** ✅ Completed  
**Date:** April 2026  

---

## Scenario

As a Tier-2 SOC analyst, I received an escalation regarding a 
public-facing server making outbound connections to multiple suspicious 
IPs. Following standard IR protocol, the server was isolated from the 
network and a PCAP was obtained from the NSM utility for analysis.

**Objective:** Analyze the PCAP, identify malicious activity, trace 
the attack vector, and extract all IOCs.

---

## Tools Used

- **Wireshark** — PCAP analysis, conversation statistics, TCP stream 
  reconstruction

---

## Vulnerability

**CVE-2023-46604 — Apache ActiveMQ Remote Code Execution**

A critical unauthenticated RCE vulnerability (CVSS 10.0) in Apache 
ActiveMQ's OpenWire protocol. The flaw exists in the 
`BaseDataStreamMarshaller.createThrowable` method, which can be abused 
by sending a specially crafted ClassInfo packet to load a remote Java 
class and execute arbitrary commands — without any authentication.

Actively exploited in the wild by ransomware groups including 
HelloKitty and TellYouThePass following public disclosure in 
October 2023.

---

## Investigation Process

### 1. Identifying the Primary C2 Server

Opened the PCAP in Wireshark and navigated to:

```
Statistics → Conversations → IPv4
```

Reviewed all external connections to the server (`134.209.197.3`). 
Identified one IP with a disproportionately high packet count — 
approximately 4867 packets (~5 MB) — indicating sustained C2 
communication.

**Primary C2 IP: `146.190.21.92`**

---

### 2. Identifying the Exploited Port and Service

Examined packet 5 in the PCAP. The destination port used by the 
attacker was **61616** — the default port for Apache ActiveMQ's 
OpenWire protocol. This confirmed the attack vector immediately from 
the challenge name and traffic pattern.

**Port:** `61616`  
**Vulnerable Service:** `Apache ActiveMQ`

---

### 3. Finding the Second C2 Server

Returned to `Statistics → Conversations → IPv4`. Identified two 
additional external IPs communicating with the server:
- `128.199.52.72`
- `84.239.49.16`

Filtered traffic for `128.199.52.72`:

```
ip.addr == 128.199.52.72
```

Followed TCP Stream on packet 34. Observed **ELF magic bytes** in the 
response — confirming this server was serving a Linux executable 
(payload) to the victim host.

**Second C2 IP: `128.199.52.72`**

---

### 4. Identifying the Dropped Reverse Shell

Navigated to packet 11 and followed the TCP stream. The stream 
revealed the attacker dropping a reverse shell executable into the 
`/tmp/` directory on the compromised server.

**Reverse Shell Filename:** `docker`  
**Drop Location:** `/tmp/docker`

---

### 5. Java Class Used in the Exploit

From the same TCP stream (packet 11), the XML exploit payload 
invoked the following Java class to achieve code execution:

**Java Class:** `java.lang.ProcessBuilder`

---

### 6. Root Cause — Vulnerable Method

After researching CVE-2023-46604 in depth, the specific vulnerable 
Java method that allows an attacker to run arbitrary code is:

**`BaseDataStreamMarshaller.createThrowable`**

This method improperly deserializes a ClassInfo command from the 
OpenWire protocol, allowing a remote attacker to supply a URL 
pointing to a malicious Java class which gets loaded and executed 
on the broker.

---

## IOC Summary

| Type | Value |
|---|---|
| Primary C2 IP | `146.190.21.92` |
| Second C2 IP | `128.199.52.72` |
| Victim Server IP | `134.209.197.3` |
| Exploited Port | `61616` |
| Vulnerable Service | Apache ActiveMQ |
| CVE | CVE-2023-46604 |
| Dropped Payload | `/tmp/docker` (ELF reverse shell) |
| Java Class (exploit) | `java.lang.ProcessBuilder` |
| Vulnerable Method | `BaseDataStreamMarshaller.createThrowable` |

---

## Wireshark Techniques Used

```
Statistics → Conversations → IPv4   # Identify top talkers
ip.addr == 128.199.52.72            # Filter specific IP traffic
Follow TCP Stream                   # Reconstruct attacker sessions
                                    # and identify ELF magic bytes
```

---

## What I Learned

- How to use Wireshark's Conversation Statistics to quickly identify 
  C2 servers by volume anomaly
- How CVE-2023-46604 abuses Java class loading via the OpenWire 
  protocol to achieve unauthenticated RCE
- How to identify ELF executables inside TCP streams (magic bytes)
- How attackers stage payloads in `/tmp/` to avoid detection
- The role of `BaseDataStreamMarshaller.createThrowable` in the 
  exploit chain — not just the CVE number but the actual code path

---

## Remediation / Blue Team Takeaways

- Patch Apache ActiveMQ to version 5.15.16, 5.16.7, 5.17.6, 
  or 5.18.3+
- Block port 61616 from all untrusted/external networks via firewall
- Monitor for outbound ELF downloads from server hosts
- Alert on unexpected connections from broker/middleware servers
- Restrict ActiveMQ to internal network segments only — never 
  internet-facing
- Implement egress filtering to detect reverse shell callbacks

---

