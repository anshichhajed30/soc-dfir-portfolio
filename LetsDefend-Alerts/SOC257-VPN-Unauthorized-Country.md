# SOC257 — VPN Connection Detected from Unauthorized Country

Platform: LetsDefend  
Date Completed: April 2026  
Difficulty: Easy  
Category: Unauthorized Access Attempt  
Verdict: True Positive — False Alarm (No Breach)

## Alert Details

| Field           | Value                                             |
|-----------------|---------------------------------------------------|
| Event ID        | 225                                               |
| Alert Rule      | SOC257 - VPN Connection from Unauthorized Country |
| Source IP       | 113.161.158.12                                    |
| Source Location | Bien Hoa, Vietnam                                 |
| Target User     | monica@letsdefend.io                              |
| Destination IP  | 33.33.33.33                                       |
| Port            | 443                                               |
| Timestamp       | Feb 13, 2024 — 02:01 AM                           | 

## Investigation Summary

### What Happened
Three consecutive VPN login attempts were made from Vietnam using Monica's
credentials. Each attempt triggered an OTP via email MFA. All three MFA 
attempts failed — no session was established.

### Why the Alert Triggered
Geographic anomaly — VPN connection originated from Vietnam, outside 
authorized locations. Rule SOC257 flagged the unauthorized country.

### How the Attack Was Conducted
Attacker likely used previously obtained credentials (credential stuffing 
or password spraying). The MFA requirement blocked all three attempts.

### Threat Intelligence
External IP: 113.161.158.12
Target account: monica@letsdefend.io

## Impact Assessment

- No successful authentication
- No VPN session established
- No data accessed or exfiltrated based on current logs
- MFA functioned as intended

## Actions Taken / Recommended

1. Confirm with Monica whether she was traveling or using a VPN legitimately
2. Reset Monica's password as a precaution (credentials may be compromised)
3. Block source IP: 113.161.158.12
4. Monitor Monica's account for further anomalous activity
5. Review if other accounts received similar attempts (lateral spray check)

## MITRE ATT&CK Techniques

T1078 – Valid Accounts
T1621 – Multi-Factor Authentication Request Generation

## Lessons Learned

- MFA is an effective control against credential stuffing
- Geographic-based alert rules catch anomalies that signature rules miss
- Even failed attempts indicate credential exposure — password hygiene matters

## Tools Used
- LetsDefend SIEM (built-in log management)
- VirusTotal
- AbuseIPDB
