# SOC Home Lab Threat Emulation & Detection Verification Guide

---

## 1. Network Perimeter Telemetry (pfSense Firewall via Syslog)

### Overview & Methodology
Evaluated perimeter visibility by configuring centralized Syslog forwarding from a pfSense 2.7.2 firewall to the Wazuh Manager over UDP port 514. Inbound port scan traffic was executed to validate stateful inspection logging and rule triggering on denied packets.

### Execution Vector
From the external assessment instance (`<ATTACKER_HOST>`), a non-ping TCP SYN port scan was launched targeting common service ports on the firewall's external interface (`<PFSENSE_WAN_IP>`):

```bash
nmap -Pn -p 23,80,443,8080 <PFSENSE_WAN_IP>
```

*Port Scan Results:*
```text
PORT     STATE    SERVICE
23/tcp   filtered telnet
80/tcp   filtered http
443/tcp  filtered https
8080/tcp filtered http-proxy
```

### Forensic Evidence & SIEM Ingestion
The firewall's default drop policy intercepted the unauthorized ingress attempts, passing the raw CSV packet metadata (`filterlog`) to the SIEM.

| Telemetry Field | Normalized Value | Technical Assessment |
| :--- | :--- | :--- |
| `rule.id` | 100114 | Wazuh custom firewall detection rule. |
| `rule.level` | 7 | Medium-high severity indicating actively blocked ingress traffic event. |
| `rule.description` | pfSense: Blocked traffic | Automated alert triggered on firewall discard events. |
| `rule.groups` | `["pfsense", "firewall"]` | Correlation taxonomies for SIEM alerting. |
| `decoder.name` | pfsense-custom | Custom regex/CSV decoder parsing FreeBSD filterlog structures. |
| `full_log` | `filterlog[...]: ...,match,block,in,4,...` | Validates directional packet block (in), Layer 4 protocol, and destination port telemetry. |

#### Triggered Events Sample
| rule.description | rule.level | rule.id |
| :--- | :---: | :---: |
| pfSense: Blocked traffic | 7 | 100114 |
| pfSense: Blocked traffic | 7 | 100114 |
| pfSense: Blocked traffic | 7 | 100114 |
| pfSense: Blocked traffic | 7 | 100114 |

---

## 2. Brute Force Authentication Attack (Hydra vs. SSH)

### Overview & Methodology
Simulated an automated credential stuffing / dictionary attack targeting the OpenSSH daemon deployed on a Windows 11 endpoint to verify local logon auditing and threshold-based brute force alert generation.

### Execution Vector
Using THC-Hydra, an automated password-guessing assault was launched using a structured dictionary against an administrative username:

```bash
hydra -l Administrator -P passwords.txt ssh://<TARGET_IP>
```

*Password list (`passwords.txt`) used during simulation:*
```text
$ cat passwords.txt
12345678
password
123456789
qwerty
admin
111111
guest
welcome
password123
iloveyou
```

*Execution completion output:*
```text
1 of 1 target completed, 0 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-09-22 18:42:07
```

### Forensic Evidence & SIEM Ingestion
Multiple sequential authentication rejections were recorded by the Windows Security Log (Event ID 4625), mapped via Sysmon/Security decoders, and correlated into alert streams:

* **Initial Rejections (`rule.id`: 60122, Level 5):** `Logon Failure - Unknown user or bad password` triggered for each invalid credential attempt.
* **Session Teardown (`rule.id`: 67023 / 60137, Level 3):** Recorded non-service account disconnection and logon failure logoffs.
* **Post-Authentication / Success Tracking (`rule.id`: 60106 / 67028):** Monitored successful logon and special privilege assignment events during baseline comparisons.

| rule.description | rule.level | rule.id |
| :--- | :---: | :---: |
| Non service account logged off. | 3 | 67023 |
| Non service account logged off. | 3 | 67023 |
| Non service account logged off. | 3 | 67023 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Logon Failure - Unknown user or bad password | 5 | 60122 |
| Windows User Logoff | 3 | 60137 |
| Special privileges assigned to new logon. | 3 | 67028 |
| Windows Logon Success | 3 | 60106 |

---

## 3. Threat Intelligence & Endpoint Correlation (VirusTotal, AbuseIPDB, AlienVault OTX)

### 3.1. File Integrity Monitoring (FIM) & VirusTotal API Integration
* **Mechanism:** Tested real-time file creation monitoring using Wazuh Syscheck correlated with automated VirusTotal v3 API hash lookups.
* **Payload Placed:** EICAR Anti-Virus Test File placed inside a monitored path (`C:\test\malware_test.txt` / `c:\virustotaltest\malware_test.txt`).
* **Detection (`rule.id`: 87105, Level 12):**
  * **Status:** `data.virustotal.malicious: 1`
  * **Detection Ratio:** 66 security vendors/engines flagged the SHA-256 hash as malicious (`data.virustotal.positives: 66`).
  * **Permalink Reference:** Public VirusTotal analysis confirmed known signatures across industry AV engines.

| rule.description | rule.level | rule.id |
| :--- | :---: | :---: |
| VirusTotal: Alert - c:\virustotaltest\malware_test.txt - 66 engines detected this file | 12 | 87105 |
| VirusTotal: Alert - c:\virustotaltest\malware_test.txt - 66 engines detected this file | 12 | 87105 |

---

### 3.2. IP Reputation Correlation (AbuseIPDB Integration)
* **Mechanism:** Integrated external IP abuse verification triggered by failed authentication logs and anomalous external addresses.
* **Detection (`rule.id`: 100060, Level 7):**
  * **Alert Description:** `AbuseIPDB: Suspicious IP detected - Score: 7% (2 reports).`
  * **Enrichment:** Automated geolocation resolution identifying source ASN, country, and abuse confidence score prior to analyst review.

| rule.description | rule.level | rule.id |
| :--- | :---: | :---: |
| AbuseIPDB: IP sospechosa detectada (118.25.6.39) - Score: 7% (2 reportes) | 7 | 100060 |

---

### 3.3. Threat Feed Validation (AlienVault OTX Integration)
* **Mechanism:** Queried AlienVault Open Threat Exchange pulses for known malicious scanners and reconnaissance nodes.
* **Detection (`rule.id`: 100070, Level 10):**
  * **Alert Description:** `AlienVault OTX: Malicious IP detected - Associated with 22 pulse(s).`
  * **Intelligence Detail:** Identified correlated threat actor campaigns, auto-generated community pulses, and malicious scanner metadata.

| rule.description | rule.level | rule.id |
| :--- | :---: | :---: |
| AlienVault OTX: IP maliciosa detectada (89.248.165.176) - Asociada a 22 pulso(s): jan2,2025 clone Auto-generate... | 10 | 100070 |

---

## 4. Network & Host-Based Intrusion Detection (Suricata & Sysmon)

* **Suricata NIDS (`rule.id`: 86601, Level 3):** Detected suspicious ICMP ping and transit sweep signatures across the monitored virtual switch interface.
* **Sysmon Process Monitoring (Event ID 1 / `rule.id`: 100051):** Flagged suspicious subshell and script interpreter spawning (Windows PowerShell) during administrative tasks.
* **Sysmon File Creation (Event ID 11 / `rule.id`: 100052):** Monitored script and binary drops in temporal directories (`AppData\Local\Temp`).

| rule.description | rule.level | rule.id |
| :--- | :---: | :---: |
| Suricata: Alert - TEST: Ping detectado en la red | 3 | 86601 |
| Suricata: Alert - TEST: Ping detectado en la red | 3 | 86601 |
| Suricata: Alert - TEST: Ping detectado en la red | 3 | 86601 |
| Sysmon (ID 11): Creación de script o ejecutable en disco (C:\\Users\\rodri\\AppData\\Local\\Temp\\_PSScriptPo... | 6 | 100052 |
| Sysmon (ID 1): Ejecución de proceso sospechoso/consola (C:\\Windows\\SysWOW64\\Windows PowerShell\\v1.0.. | 6 | 100051 |

---

## 5. MITRE ATT&CK Framework Mapping

| MITRE ATT&CK Technique | ID | Detection Layer | Relevant Lab Event |
| :--- | :--- | :--- | :--- |
| **Brute Force: Password Guessing** | T1110.001 | Host SIEM (Windows Security) | Hydra SSH attack generating Rule 60122. |
| **Command and Scripting Interpreter: PowerShell** | T1059.001 | Endpoint Telemetry (Sysmon) | Sysmon Event 1 (`rule.id`: 100051) console invocation. |
| **User Execution: Malicious File** | T1204.002 | FIM + Threat Intelligence | Real-time file drop flagged by VirusTotal (Rule 87105). |
| **Network Service Discovery** | T1046 | Perimeter Firewall (pfSense) | Inbound Nmap TCP SYN scan blocked by Rule 100114. |
| **Adversary Infrastructure: Network Traffic** | T1590 | External Threat Intel | AbuseIPDB & AlienVault OTX alerts (100060, 100070). |

---

## 6. Dashboard Telemetry Comparison

### Before (Pre-Attack Baseline)
The SIEM overview establishes a clean baseline with only minimal background operational noise (23 low/medium events) and zero high or critical security alerts.

* **Critical severity:** 0 (Rule level 15 or higher)
* **High severity:** 0 (Rule level 12 to 14)
* **Medium severity:** 2 (Rule level 7 to 11)
* **Low severity:** 21 (Rule level 0 to 6)
* **Active Agents:** 1

### After (Post-Attack Telemetry)
Following the attack simulations and threat intel integrations, total alerts surged past 1,540, registering 2 high-severity threat detections and 219 medium-severity authentication and firewall incidents.

* **Critical severity:** 0 (Rule level 15 or higher)
* **High severity:** 2 (Rule level 12 to 14)
* **Medium severity:** 219 (Rule level 7 to 11)
* **Low severity:** 1,320 (Rule level 0 to 6)
* **Active Agents:** 1