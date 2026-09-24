# SOC Home Lab: Deployment & Integration Guide

A streamlined setup manual to build an integrated Security Operations Center (SOC) home lab on a single Windows 11 host using Oracle VirtualBox, host-based telemetry agents, and external threat intelligence APIs.

---

## 1. Network & Virtual Machine Baseline

Deploy three core virtual machines in VirtualBox using **Bridged Networking** attached to your primary physical Wi-Fi/Ethernet adapter to allow bidirectional L3 routing across components.

* **pfSense Firewall:** Edge firewall and network gateway. Set Promiscuous Mode to *Allow All* on the WAN interface.
* **Wazuh Server (SIEM / Manager):** All-in-one Wazuh architecture (Manager, Indexer, and Dashboard).
* **Kali Linux:** Simulation attacker platform set up with standard testing toolkits (Nmap, Hydra).

---

## 2. Endpoint & Telemetry Setup (Windows 11 Host)

### A. Microsoft Sysmon (EDR Telemetry)

1. Download Microsoft Sysmon alongside an updated configuration file (e.g., `sysmonconfig.xml`).
2. Open PowerShell as Administrator and run the installer:
   ```powershell
   .\Sysmon64.exe -accepteula -i sysmonconfig.xml
   ```
3. Confirm the service status:
   ```powershell
   Get-Service -Name Sysmon64
   ```

### B. Suricata NIDS (Network Inspection)

1. Install Npcap with **"WinPcap API-compatible mode"** enabled.
2. Install Suricata to default path (`C:\Program Files\Suricata`).
3. Identify your active network interface GUID via PowerShell:
   ```powershell
   Get-NetAdapter | Select-Object Name, InterfaceGuid
   ```
4. Open `suricata.yaml` and configure the capture interface under the `pcap` block:
   ```yaml
   pcap:
     - interface: \Device\NPF_{YOUR-INTERFACE-GUID}
       checksum-checks: auto
   ```
5. Launch Suricata in live capture mode:
   ```powershell
   cd "C:\Program Files\Suricata"
   .\suricata.exe -c "suricata.yaml" --pcap
   ```

---

### C. Wazuh Agent Deployment

1. Download and install the Wazuh Windows Agent.
2. Edit `C:\Program Files (x86)\ossec-agent\ossec.conf` to declare your Wazuh Server IP address:
   ```xml
   <client>
     <server>
       <address>YOUR_WAZUH_SERVER_IP</address>
       <port>1514</port>
       <protocol>tcp</protocol>
     </server>
   </client>
   ```
3. Append log harvesting blocks for Sysmon and Suricata `eve.json`:
   ```xml
   <!-- Sysmon Telemetry -->
   <localfile>
     <location>Microsoft-Windows-Sysmon/Operational</location>
     <log_format>eventchannel</log_format>
   </localfile>

   <!-- Suricata Alerts -->
   <localfile>
     <location>C:\Program Files\Suricata\log\eve.json</location>
     <log_format>json</log_format>
   </localfile>
   ```
4. Start the agent service:
   ```powershell
   Restart-Service -Name WazuhSvc
   ```

---

## 3. Perimeter Telemetry: pfSense Syslog Forwarding

1. Log in to the pfSense webConfigurator (**Status > System Logs > Settings**).
2. Enable **Send log messages to remote syslog server**.
3. Set the remote server to `YOUR_WAZUH_SERVER_IP:514` using the **UDP** protocol.
4. Enable logging for **Firewall Events** and **Authentication processes**.
5. On the Wazuh Server, ensure `wazuh-remoted` is listening on port 514:
   ```bash
   sudo ss -ulnp | grep 514
   ```

---

## 4. Threat Intelligence Integrations (VirusTotal, AbuseIPDB & AlienVault OTX)

Wazuh automatically enriches alerts and suspicious telemetry by querying external Cyber Threat Intelligence (CTI) APIs through the `wazuh-integratord` daemon.

### A. Obtain API Keys

* **VirusTotal:** Register at VirusTotal and copy your free Public API Key (used for automated file hash inspection via FIM / Sysmon).
* **AbuseIPDB:** Create an account at AbuseIPDB and generate a Key under the API tab (used for IP reputation scoring).
* **AlienVault OTX:** Sign up at AlienVault OTX and retrieve your OTX Key (used for pulse and IOC correlation).

### B. Configure ossec.conf on Wazuh Server

Access your Wazuh Server via SSH and edit `/var/ossec/etc/ossec.conf`. Add the integration blocks inside the main `<ossec_config>` section:

```xml
<!-- Virus Total Hash Lookup Integration -->
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>
  <group>syscheck,sysmon_event1</group>
  <alert_format>json</alert_format>
</integration>

<!-- AbuseIPDB IP Reputation Integration -->
<integration>
  <name>custom-abuseipdb</name>
  <hook_url>https://api.abuseipdb.com/api/v2/check</hook_url>
  <api_key>YOUR_ABUSEIPDB_API_KEY</api_key>
  <rule_id>100114,100115</rule_id>
  <alert_format>json</alert_format>
</integration>

<!-- AlienVault OTX Threat Intelligence Integration -->
<integration>
  <name>alienvault-otx</name>
  <api_key>YOUR_ALIENVAULT_OTX_API_KEY</api_key>
  <group>firewall_drop,authentication_failed</group>
  <alert_format>json</alert_format>
</integration>
```

### C. Apply Changes & Validate Integration Service

1. Restart the Wazuh Manager to apply the new API hooks:
   ```bash
   sudo systemctl restart wazuh-manager
   ```
2. Verify that the integrator daemon spawned correctly:
   ```bash
   sudo /var/ossec/bin/wazuh-control status | grep integratord
   ```
   *Expected output:* `wazuh-integratord is running...`

3. Monitor integration requests and API rate-limit logs in real time:
   ```bash
   sudo tail -f /var/ossec/logs/integrations.log
   ```