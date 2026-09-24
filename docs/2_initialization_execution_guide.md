# SOC Home Lab Initialization & Execution Guide

This document outlines the systematic operational sequence to boot, verify, and validate all defensive components (pfSense, Suricata, Wazuh SIEM, and Sysmon) alongside the attacking machine (Kali Linux) prior to executing threat simulation scenarios.

---

## 1. Perimeter Security: pfSense Firewall (VirtualBox)

### Overview
Boot the pfSense virtual appliance to handle routing and edge firewall logging. The WAN interface is set to **Bridged Adapter** to communicate with the local subnet, forwarding network telemetry to the Wazuh Server via UDP 514.

### Operational Check
* Ensure pfSense successfully acquires an IP address on its WAN interface matching your local network range (`192.x.x.x`).
* Keep the VirtualBox session running in the background.

```text
0) Logout (SSH only)                  9) pfTop
1) Assign Interfaces                 10) Filter Logs
2) Set interface(s) IP address       11) Restart webConfigurator
3) Reset webConfigurator password    12) PHP shell
4) Reset to factory defaults         13) Update from console
5) Reboot system                     14) Enable Secure Shell (sshd)
6) Halt system                       15) Restore recent configuration
7) Ping host                         16) Restart PHP-FPM
8) Shell

Enter an option:
```

---

## 2. SIEM Core: Wazuh Server Initialization (VirtualBox)

### Overview
Power on the Wazuh Server virtual machine. Once boot sequences finalize at the login prompt (`[wazuh-user@wazuh-server ~]$`), all core orchestration daemons (`wazuh-modulesd`, `wazuh-analysisd`, `wazuh-remoted`, `wazuh-indexer`) will initialize automatically.

```text
WAZUH Open Source Security Platform
https://wazuh.com
[wazuh-user@wazuh-server ~]$
```

### Daemon Health & Recovery Commands
If the Wazuh Web UI fails to load or daemons require a synchronization restart, connect via SSH or the local terminal and run:

```bash
# Restart core Wazuh components in sequential order
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart wazuh-dashboard

# Verify active listening state on UDP port 514 for pfSense syslog reception
sudo ss -ulnp | grep 514

# Check Wazuh daemon status
sudo /var/ossec/bin/wazuh-control status
```

---

## 3. SIEM UI: Wazuh Dashboard Access (Web Browser)

### Overview
Open a web browser on the host machine and access the web management portal at your Wazuh VM static IP (`https://192.x.x.x`).

### Operational Check
* Authenticate using administrative credentials.
* Navigate to **Overview**: Validate that **Active Agents** displays at least `1` (representing the monitored Windows 11 endpoint).
* Confirm real-time alert ingestion across severity levels (Low, Medium, High, Critical).

---

## 4. Endpoint Defense: Wazuh Agent & Sysmon (Windows 11)

### Overview
The Windows 11 endpoint hosts the Wazuh Agent and Microsoft Sysmon. Both services are registered to start automatically to maintain process execution logs, integrity validation, and telemetry forwarding to the SIEM manager.

### Service Verification Commands
Open PowerShell as Administrator on Windows 11 and run:

```powershell
# Verify that Wazuh and Sysmon services are running under Automatic start
Get-Service | Where-Object { $_.Name -match "wazuh|sysmon" -or $_.DisplayName -match "wazuh|sysmon" } | Select-Object Name, DisplayName, Status, StartType
```

*Expected Terminal Output:*
```text
PS C:\WINDOWS\system32> Get-Service | Where-Object { $_.Name -match "wazuh|sysmon|suricata" -or $_.DisplayName -match "wazuh|sysmon|suricata" } | Select-Object Name, DisplayName, Status, StartType

Name       DisplayName   Status    StartType
----       -----------   ------    ---------
Sysmon64   Sysmon64      Running   Automatic
WazuhSvc   Wazuh         Running   Automatic
```

---

## 5. NIDS: Suricata Live Capture (Windows 11)

### Overview
Suricata operates as a host-based Network Intrusion Detection System (NIDS) coupled with Npcap. With the physical Wi-Fi interface GUID pre-configured in `suricata.yaml` under the `pcap` capture section, execute Suricata directly with the pcap capture flag.

### Launch Execution
Run the following commands in PowerShell as Administrator:

```powershell
cd "C:\Program Files\Suricata"
.\suricata.exe -c "suricata.yaml" --pcap
```

### Expected Console Output
```text
Info: win32-service: Running as service: no
i: suricata: This is Suricata version 8.0.0 RELEASE running in SYSTEM mode
i: threads: Threads created -> RX: 1 W: 16 FM: 1 FR: 1 Engine started.
```

---

## 6. Threat Actor: Kali Linux (VirtualBox)

### Overview
Start the Kali Linux virtual machine. The primary network adapter is configured as **Bridged Adapter** (Promiscuous Mode: *Allow All*) connected to the host's physical Wi-Fi adapter.

### Network Verification (Kali Terminal)
```bash
# Verify assigned IP address
ip a

# Test direct Layer 3 connectivity to the Windows 11 target host
ping -c 4 <WINDOWS_11_IP>
```

---

## Verdict
With pfSense filtering the boundary, Suricata inspecting live packet captures, Sysmon auditing local endpoint execution, and Wazuh indexing events, the environment is ready for adversarial simulation runs (Nmap discovery sweeps, brute-force testing, and exploitation analysis).