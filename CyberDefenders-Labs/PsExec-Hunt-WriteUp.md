# PsExec Hunt — CyberDefenders Lab Writeup

**Platform:** CyberDefenders  
**Category:** Network Forensics / Endpoint Forensics  
**Difficulty:** Easy  
**Status:** ✅ Completed  
**Date:** April 2026  

---

## Scenario

A threat was detected on the corporate network involving suspicious 
lateral movement activity. A PCAP was provided for analysis. The goal 
was to identify the source of the attack, the target machine, the 
user account involved, and the tools used to move laterally across 
the network.

---

## Tools Used

- **Wireshark** — PCAP analysis, conversation statistics, 
  protocol filtering, TCP stream reconstruction

---

## Background — What is PsExec?

PsExec is a legitimate Microsoft Sysinternals CLI tool designed for 
remote administration — used by sysadmins to execute processes on 
remote systems without manually installing client software. However, 
it is heavily abused by attackers for lateral movement because it:

- Requires no additional software on the target
- Runs with SYSTEM-level privileges if the attacker has admin access
- Uses SMB (port 445) which is often allowed internally
- Leaves minimal traces compared to other remote execution methods

When PsExec runs, it drops a service binary called `psexesvc.exe` 
on the target machine, executes it, then automatically deletes it 
upon completion — making detection harder.

---

## Key Protocol — SMB (Server Message Block)

SMB is a Windows network file-sharing protocol that allows systems 
to read, write, and request services from remote servers. It is the 
backbone of PsExec lateral movement. Key SMB shares involved in a 
PsExec attack:

| Share | Purpose |
|---|---|
| `IPC$` | Inter-Process Communication — enables remote administration and service control |
| `ADMIN$` | Hidden admin share mapping to `C:\Windows` — used to drop `psexesvc.exe` |

---

## Investigation Process

### 1. Protocol Hierarchy Check

First step when analyzing any PCAP — check what protocols are present:

```
Statistics → Protocol Hierarchy
```

Confirmed heavy SMB traffic — consistent with PsExec or other 
SMB-based lateral movement.

---

### 2. Identifying Key Hosts

Navigated to:

```
Statistics → Conversations → IPv4
```

Three IPs stood out with significant traffic volume. Applied a 
display filter to focus on the two most active:

```
ip.addr == 10.0.0.130 or ip.addr == 10.0.0.131
```

Identified all three hosts:

| IP | Hostname | Notes |
|---|---|---|
| `10.0.0.130` | HR-PC | Attacker machine |
| `10.0.0.131` | MARKETING-PC | Present in network |
| `10.0.0.133` | SALES-PC | Victim machine |

---

### 3. Tracing the Attack Path

Analyzed SMB traffic between `10.0.0.130` (HR-PC) and 
`10.0.0.133` (SALES-PC). Observed the following connection 
sequence — a textbook PsExec lateral movement pattern:

**Step 1 — IPC$ Connection**  
HR-PC connected to SALES-PC via `IPC$` share. This is the 
initial handshake used to enumerate and communicate with 
remote services before executing anything.

**Step 2 — ADMIN$ Connection**  
HR-PC then connected to `ADMIN$` on SALES-PC. The `ADMIN$` 
share maps directly to `C:\Windows` — this is how PsExec 
copies its service binary to the remote machine.

**Step 3 — psexesvc.exe Execution Request**  
HR-PC sent a connection request to execute `psexesvc.exe` on 
SALES-PC. This is the PsExec service binary that runs on the 
target to handle remote command execution. It is automatically 
deleted after the session ends to reduce forensic footprint.

---

### 4. User Account Identified

The username associated with the attack activity from HR-PC 
was identified as:

**Username: `ssales`**

This suggests either a compromised account or an insider 
threat using credentials from the sales department to 
move laterally.

---

## Attack Summary

| Field | Detail |
|---|---|
| Attack Type | Lateral Movement via PsExec |
| Protocol Abused | SMB (Port 445) |
| Attacker Machine | `10.0.0.130` — HR-PC |
| Victim Machine | `10.0.0.133` — SALES-PC |
| Username Used | `ssales` |
| Shares Accessed | `IPC$`, `ADMIN$` |
| Malicious Binary | `psexesvc.exe` (auto-deleted post-execution) |

---

## Wireshark Filters Used

```
Statistics → Conversations → IPv4        # Identify active hosts
ip.addr == 10.0.0.130 or 
ip.addr == 10.0.0.131                    # Focus on suspicious IPs
smb2                                     # Filter SMB traffic
```

---

## What I Learned

- How to identify lateral movement patterns in a PCAP using 
  SMB conversation analysis
- The role of `IPC$` and `ADMIN$` shares in a PsExec attack chain
- Why `psexesvc.exe` is significant — it is a reliable IOC for 
  PsExec-based lateral movement even though it self-deletes
- How attackers abuse legitimate admin tools (Living off the Land) 
  to blend in with normal traffic
- How to map IP addresses to hostnames and usernames through 
  SMB session data in Wireshark

---

## Blue Team Takeaways

- Monitor and alert on `ADMIN$` and `IPC$` share access from 
  non-admin workstations
- Restrict lateral SMB traffic between workstations using 
  host-based firewall rules (block port 445 peer-to-peer)
- Enable Windows Event ID `7045` — detects new service 
  installation (psexesvc.exe triggers this)
- Enable Sysmon Event ID `1` — process creation, catches 
  psexesvc.exe execution
- Audit and rotate credentials — `ssales` account was used 
  from HR-PC, suggesting credential compromise
- Implement least privilege — standard users should not have 
  admin rights on remote machines

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Lateral Movement | Remote Services: SMB/Windows Admin Shares | T1021.002 |
| Execution | System Services: Service Execution | T1569.002 |
| Defense Evasion | File Deletion (psexesvc auto-delete) | T1070.004 |

---

## References

- [Microsoft Sysinternals — PsExec](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec)
- [MITRE ATT&CK — T1021.002](https://attack.mitre.org/techniques/T1021/002/)
- [CyberDefenders PsExec Hunt Lab](https://cyberdefenders.org)
