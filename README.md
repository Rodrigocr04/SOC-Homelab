# SOC Homelab: Multi-Layer Threat Detection & Incident Response Pipeline

A hands-on simulation of a Security Operations Center (SOC) built on a single computer. This project collects and connects activity from Windows endpoints (Sysmon), suspicious network traffic alerts (Suricata), and firewall blocks (pfSense) into one centralized monitoring dashboard (Wazuh). It also automatically checks external threat databases (like VirusTotal and AbuseIPDB) to instantly flag dangerous files and malicious IP addresses.

> **Disclaimer:**  
> This project is created strictly for educational and security research purposes. All simulation activities (such as port scanning and brute-force testing) must be performed exclusively within an isolated, controlled laboratory environment (the virtual machines detailed in this documentation) or on systems where you possess explicit, documented authorization. Never deploy or execute these techniques against third-party, public, or production environments.

---

## Architecture & Data Flow

```
[Attacker: Kali Linux] 
       │ (Inbound Attacks: Hydra, Nmap)
       ▼
 [pfSense 2.7.2 Firewall] ──(Syslog UDP 514: Block Events)──┐
       │                                                   │
       ├─► [Suricata NIDS] ────(eve.json)──────────┐       │
       │                                           ▼       ▼
       └─► [Windows 11 Target] ──(Agent TCP 1514)─► [Wazuh SIEM Manager]
           • Microsoft Sysmon                              │
           • FIM (File Integrity)                          ├─► VirusTotal API
                                                           ├─► AbuseIPDB API
                                                           └─► AlienVault OTX API
```

---

## Lab Documentation Guides

The laboratory implementation is structured into three comprehensive operational guides available in the [`docs/`](docs/) directory:

1. **[Guide 1: Deployment & Integration](docs/1-SOC%20Home%20Lab%20Deployment%20%26%20Integration.pdf)**
   * Network segmentation and Bridged Virtual Machine deployment (VirtualBox).
   * Host endpoint configuration: Sysmon installation and Suricata live PCAP capture binding.
   * Wazuh Agent setup, log harvesting channels, and pfSense Syslog forwarder activation.
   * Threat Intelligence daemon integration via `ossec.conf` (VirusTotal, AbuseIPDB, AlienVault OTX).

2. **[Guide 2: Initialization & Execution](docs/2-SOC%20Home%20Lab%20Initialization%20%26%20Execution.pdf)**
   * Deterministic startup sequence across hypervisor instances.
   * Health verification commands for Wazuh daemons, indexer status, and UDP 514 listeners.
   * Windows endpoint telemetry validation (`Sysmon64`, `WazuhSvc`) and live NIDS packet processing.

3. **[Guide 3: Threat Emulation & Detection Verification](docs/3-SOC%20Home%20Lab%20Threat%20Emulation%20%26%20Detection%20Verification%20Guide.pdf)**
   * External reconnaissance and firewall deny auditing via pfSense `filterlog`.
   * SSH credential stuffing emulation with THC-Hydra vs. Windows Security Event ID 4625.
   * Real-time malware detection via FIM hash queries against 66 VirusTotal engines.
   * IP reputation scoring (AbuseIPDB) and community indicator correlation (AlienVault OTX).
   * MITRE ATT&CK technique mapping and forensic alert verification.

---

## Summary of Simulated Attack Vectors & Detections

| Layer | Attack / Simulation | Detection Mechanism | SIEM Rule ID | Rule Level |
| :--- | :--- | :--- | :--- | :--- |
| **Perimeter** | Nmap TCP SYN Port Scan (`-Pn`) | pfSense Syslog Ingestion (`filterlog`) | `100114` | Level 7 |
| **Endpoint** | SSH Dictionary Brute Force (Hydra) | Windows Security Log (Logon Failure) | `60122` | Level 5 |
| **Host EDR** | PowerShell / Script Invocation | Microsoft Sysmon (Event ID 1 & 11) | `100051` / `100052` | Level 6 |
| **Network** | ICMP Host Discovery / Sweeps | Suricata NIDS Live Capture | `86601` | Level 3 |
| **CTI / FIM** | EICAR Test File Ingestion | Wazuh Syscheck + VirusTotal v3 API | `87105` | Level 12 |
| **CTI / IP** | Malicious Scanner IP Resolution | AbuseIPDB Integration Hook | `100060` | Level 7 |
| **CTI / Threat**| Adversary Infrastructure Validation | AlienVault OTX Pulse Correlation | `100070` | Level 10 |

---

## Key Takeaways & Core SOC Competencies

This project proves that effective, enterprise-ready security monitoring doesn't require expensive commercial tools. A solid detection and response setup can be built entirely with free, open-source software (Wazuh, pfSense, Suricata, Sysmon, and public threat intelligence feeds).

Beyond setting up the lab, this project provided practical experience with core, everyday analyst tasks:

* **Log Investigation:** Reading system activity and network logs to reconstruct the step-by-step timeline of an attack.
* **Alert Tuning:** Writing and adjusting detection rules to catch real threats while filtering out background noise and false alarms.
* **Threat Intelligence Integration:** Automatically verifying suspicious files and IP addresses against global threat databases to speed up investigations.
* **Incident Reporting:** Writing clear, professional reports detailing what happened and how attacks unfolded, mapped to industry standards like MITRE ATT&CK.

---

## Acknowledgments & Credits

This project and its implementation methodology were inspired by and based on the foundational architecture demonstrated in the [Wazuh-SOC-Lab repository by marxgoo](https://github.com/marxgoo/Wazuh-SOC-Lab). Special thanks for providing an outstanding reference for open-source SIEM and SOC engineering practices.