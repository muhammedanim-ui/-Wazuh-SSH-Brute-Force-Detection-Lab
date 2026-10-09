# -Wazuh-SSH-Brute-Force-Detection-Lab
 # Wazuh SSH Brute-Force Detection Lab

A home-lab SIEM project: deploy **Wazuh 4.14.8** on Kali Linux, ingest SSH logs from `systemd-journald`, simulate failed logins, and confirm the detections in the Wazuh dashboard with MITRE ATT&CK mapping.

## Objectives
- Stand up a single-node Wazuh manager + dashboard
- Collect SSH authentication logs on a journald-only system
- Generate failed logins for a non-existent user
- Verify alerts via CLI (`alerts.log`) and the dashboard (Discover)

## Lab Environment
| Item | Value |
|---|---|
| OS | Kali Linux (VirtualBox VM) |
| Host IP | 10.0.2.15 |
| Wazuh version | 4.14.8 (API on :55000) |
| Log source | `ssh.service` via journald |
| Test user | `fakeuser` (does not exist) |

## Architecture
```
ssh client ──► sshd (10.0.2.15:22) ──► systemd-journald
                                             │ localfile (journald)
                                             ▼
                                     wazuh-logcollector
                                             ▼
                     wazuh-analysisd (rules 5710, 2502) ──► alerts.log
                                             ▼
                                 Wazuh Dashboard (Discover)
```

## Detections Observed
| Rule ID | Level | Description | MITRE |
|---|---|---|---|
| 5710 | 5 | sshd: Attempt to login using a non-existent user | T1110 Brute Force |
| 2502 | 10 | syslog: User missed the password more than one time | T1110 Brute Force |

## Steps
1. **Start Wazuh manager**
   `sudo systemctl start wazuh-manager`
  

2. **Start SSH service**
   `sudo systemctl start ssh`
   

3. **Simulate failed logins** (wrong password 3x)
   `ssh fakeuser@10.0.2.15`
 

4. **Confirm logs in journald**
   `sudo journalctl -u ssh --no-pager -n 30`
  

5. **Configure Wazuh to read the journal** (Kali has no `/var/log/auth.log`)
   Add to `/var/ossec/etc/ossec.conf`, then restart the manager:
```xml
   <localfile>
     <log_format>journald</log_format>
     <location>ssh.service</location>
   </localfile>
```
   
6. **Verify alerts on CLI**
   `sudo tail -n 20 /var/ossec/logs/alerts/alerts.log`
 
7. **Log in to the dashboard**
   

8. **Confirm API connection is online**
  

9. **Overview dashboard: alerts by severity**


10. **Run a second test to generate fresh alerts**
    

11. **Discover: rule 2502 hits**
    Index `wazuh-alerts-*`, filters `manager.name: kali` and `rule.level: 7 to 11`
   
## Lessons Learned
- Kali has no `auth.log` by default, so a `journald` localfile block is required.
- The ssh service is inactive by default and must be started first.
- Alert timestamps mix UTC and local time (EDT); normalise when correlating.
- With no agents registered, the manager itself (agent 000) is the monitored endpoint.

## Next Steps
- Active response to block repeat offenders (`firewall-drop`)
- Register a second VM as a Wazuh agent
- Custom rule to escalate 5+ failures in 60s

> All testing was done against my own isolated VM.
