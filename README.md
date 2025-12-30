# Snort-IDS-Wazuh-SIEM-Integration-Project


A complete Security Operations Center (SOC) project demonstrating network intrusion detection with Snort IDS integrated into Wazuh SIEM for centralized monitoring, alerting, and MITRE ATT&CK framework mapping.

## Table of Contents
- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Prerequisites](#prerequisites)
- [Installation & Configuration](#installation--configuration)
- [Custom Rules & Detection](#custom-rules--detection)
- [Viewing Alerts](#viewing-alerts)
- [Troubleshooting](#troubleshooting)
- [Project Outcomes](#project-outcomes)

---

## Project Overview

This project integrates **Snort IDS** (Intrusion Detection System) with **Wazuh SIEM** to create a functional security monitoring environment. Snort detects network-based attacks, and Wazuh centralizes, analyzes, and visualizes these alerts with custom rules and MITRE ATT&CK mapping.

**Use Cases:**
- Network reconnaissance detection (SNMP probes, port scans)
- Nmap scan detection (XMAS, SYN, NULL scans)
- Real-time threat intelligence and alerting
- Security event correlation and analysis

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Windows Host Machine                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │          Wazuh Manager (Docker Container)             │  │
│  │  - Collects logs from agents                          │  │
│  │  - Processes custom rules                             │  │
│  │  - Indexes alerts in OpenSearch                       │  │
│  │  - Wazuh Dashboard (Web UI)                           │  │
│  └───────────────────────────────────────────────────────┘  │
│                           ▲                                  │
│                           │ Log forwarding (Agent 002)       │
│                           │                                  │
└───────────────────────────┼──────────────────────────────────┘
                            │
┌───────────────────────────┼──────────────────────────────────┐
│                 Linux Mint VM (VirtualBox)                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Snort IDS                                          │    │
│  │  - Monitors network traffic                         │    │
│  │  - Generates alerts → /var/log/snort/snort.alert    │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Wazuh Agent                                        │    │
│  │  - Reads Snort logs                                 │    │
│  │  - Forwards to Wazuh Manager                        │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│                                                             │
└──────────────────────────────────────────────────────────────┘
```

---

## Technologies Used

| Component | Version | Purpose |
|-----------|---------|---------|
| **Wazuh Manager** | 4.14.0 | SIEM platform for log aggregation and analysis |
| **Wazuh Agent** | 4.14.0 | Log collector and forwarder |
| **Snort** | 2.9.x | Network-based intrusion detection system |
| **Docker** | Latest | Container runtime for Wazuh Manager |
| **Linux Mint** | 21.x | Host OS for Snort IDS and Wazuh Agent |
| **Windows 10/11** | - | Host OS for Wazuh Manager (Docker) |

---

## Prerequisites

### Windows Host (Wazuh Manager)
- Docker Desktop installed and running
- Minimum 4GB RAM allocated to Docker
- Network connectivity between host and VM

### Linux Mint VM (Snort + Agent)
- VirtualBox or VMware
- Bridged network adapter (for communication with Windows host)
- Sudo privileges

---

## Installation & Configuration

### Step 1: Deploy Wazuh Manager (Windows Host)

1. **Download Wazuh Docker deployment:**
   ```powershell
   git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.0
   cd wazuh-docker\single-node
   ```

2. **Start Wazuh stack:**
   ```powershell
   docker-compose up -d
   ```

3. **Access Wazuh Dashboard:**
   - URL: `https://<Windows-Host-IP>` (e.g., `https://192.168.56.1`)
   - Default credentials: `admin / SecretPassword` (change in `docker-compose.yml`)

4. **Verify containers are running:**
   ```powershell
   docker ps
   ```

---

### Step 2: Install Snort IDS (Linux Mint VM)

1. **Update system:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Install Snort:**
   ```bash
   sudo apt install snort -y
   ```

3. **Configure Snort network interface:**
   - During installation, specify your network interface (e.g., `enp0s3`)
   - Set HOME_NET to your VM's subnet (e.g., `192.168.43.0/24`)

4. **Edit Snort configuration:**
   ```bash
   sudo nano /etc/snort/snort.conf
   ```

   **Key settings:**
   ```bash
   # Set your network
   var HOME_NET 192.168.43.0/24
   
   # Output alert to fast format
   output alert_fast: /var/log/snort/snort.alert.fast
   ```

5. **Create log directory:**
   ```bash
   sudo mkdir -p /var/log/snort
   sudo chmod 755 /var/log/snort
   ```

6. **Run Snort in detection mode:**
   ```bash
   sudo snort -A fast -q -u snort -g snort -c /etc/snort/snort.conf -i enp0s3 -D
   ```

7. **Verify Snort is running:**
   ```bash
   ps aux | grep snort
   ```

---

### Step 3: Install Wazuh Agent (Linux Mint VM)

1. **Download and install agent:**
   ```bash
   wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.0-1_amd64.deb
   sudo dpkg -i wazuh-agent_4.14.0-1_amd64.deb
   ```

2. **Configure agent to connect to manager:**
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```

   **Update the manager IP:**
   ```xml
   <client>
     <server>
       <address>192.168.56.1</address>  <!-- Your Windows host IP -->
       <port>1514</port>
       <protocol>tcp</protocol>
     </server>
   </client>
   ```

3. **Configure log collection for Snort:**
   ```bash
   sudo nano /var/ossec/etc/ossec.conf
   ```

   **Add this inside `<ossec_config>`:**
   ```xml
   <localfile>
     <log_format>snort-full</log_format>
     <location>/var/log/snort/snort.alert.fast</location>
   </localfile>
   ```

4. **Start Wazuh agent:**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable wazuh-agent
   sudo systemctl start wazuh-agent
   ```

5. **Verify agent status:**
   ```bash
   sudo systemctl status wazuh-agent
   ```

---

### Step 4: Register Agent in Wazuh Manager

**From Windows PowerShell:**

```powershell
docker exec -it single-node-wazuh.manager-1 bash
```

**Inside the container:**

```bash
# Register agent
/var/ossec/bin/manage_agents

# Select option: A (Add agent)
# Agent name: mint
# Agent IP: 192.168.43.134
# Agent ID: 002 (or auto-assigned)
```

**Extract the agent key and add it to the Linux VM:**

On the VM:
```bash
sudo /var/ossec/bin/manage_agents
# Select option: I (Import key)
# Paste the key from manager
```

**Restart agent:**
```bash
sudo systemctl restart wazuh-agent
```

**Verify connection in Wazuh Dashboard:**
- Go to **Agents** → You should see agent `mint (002)` with status **Active**

---

## Custom Rules & Detection

### Step 5: Create Custom Snort Rules for Wazuh

**Access Wazuh Manager container:**

```powershell
docker exec -it single-node-wazuh.manager-1 bash
```

**Create custom rule file:**

```bash
cat << 'EOF' > /var/ossec/etc/rules/snort_rules.xml
<group name="snort,ids,network">
  
  <rule id="100200" level="10">
    <if_sid>20100,20101</if_sid>
    <match>SNMP</match>
    <description>Snort: SNMP reconnaissance attempt detected from $(data.srcip)</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>

  <rule id="100201" level="11">
    <if_sid>20100,20101</if_sid>
    <match>SCAN nmap</match>
    <description>Snort: Nmap scan detected from $(data.srcip)</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>

  <rule id="100202" level="12">
    <if_sid>20100,20101</if_sid>
    <match>XMAS</match>
    <description>Snort: XMAS scan detected from $(data.srcip)</description>
    <mitre>
      <id>T1046</id>
    </mitre>
  </rule>

</group>
EOF
```

**Restart Wazuh Manager:**

```bash
/var/ossec/bin/wazuh-control restart
exit
```

**Test custom rules:**

```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/wazuh-logtest
```

**Paste a sample Snort log:**
```
12/29-10:49:47.303905  [**] [1:1228:7] SCAN nmap XMAS [**] [Classification: Attempted Information Leak] [Priority: 2] {TCP} 192.168.43.1:49119 -> 192.168.43.134:1
```

**Expected output:**
```
id: '100202'
level: '12'
description: 'Snort: XMAS scan detected from 192.168.43.1'
```

---

## Viewing Alerts

### Generate Test Alerts

**From attacker machine (Windows host or another VM):**

```bash
# XMAS scan
nmap -sX 192.168.43.134

# SNMP probe
nmap -sU -p 161 192.168.43.134
```

### View Alerts in Wazuh Dashboard

#### Option 1: Threat Hunting (Main View)

1. Navigate to **Threat Intelligence → Threat Hunting**
2. Time range: **Last 15 minutes**
3. Search query:
   ```
   rule.groups:snort
   ```
   or
   ```
   rule.id:(100200 OR 100201 OR 100202)
   ```

#### Option 2: Discover (Raw Logs)

1. Navigate to **Explore → Discover**
2. Select data view: `wazuh-alerts-*`
3. Search:
   ```
   rule.groups:snort
   ```

**Key fields to review:**
- `rule.description` - Alert description
- `data.srcip` - Source IP address
- `data.dstip` - Destination IP address
- `rule.level` - Severity (10, 11, 12)
- `rule.mitre.id` - MITRE ATT&CK technique

#### Option 3: MITRE ATT&CK Dashboard

1. Navigate to **Threat Intelligence → MITRE ATT&CK**
2. Look for **T1046 - Network Service Discovery**
3. Click to see all related Snort detections

---

## Troubleshooting

### Issue: Agent not showing as "Active"

**Solution:**
```bash
# On Linux VM
sudo systemctl status wazuh-agent
sudo tail -f /var/ossec/logs/ossec.log

# Check connectivity
telnet 192.168.56.1 1514
```

### Issue: No Snort alerts appearing in Wazuh

**Solution:**

1. **Verify Snort is generating alerts:**
   ```bash
   sudo tail -f /var/log/snort/snort.alert.fast
   ```

2. **Check Wazuh agent is reading the file:**
   ```bash
   sudo tail -f /var/ossec/logs/ossec.log | grep snort
   ```

3. **Verify alerts are reaching the manager:**
   ```powershell
   docker exec -it single-node-wazuh.manager-1 tail -f /var/ossec/logs/alerts/alerts.json
   ```

### Issue: Custom rules not firing (still seeing rule 20101)

**Solution:**

1. **Verify rule file syntax:**
   ```bash
   docker exec -it single-node-wazuh.manager-1 cat /var/ossec/etc/rules/snort_rules.xml
   ```

2. **Test rules with logtest:**
   ```bash
   docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/wazuh-logtest
   ```

3. **Check for rule loading errors:**
   ```bash
   docker logs single-node-wazuh.manager-1 | grep -i error
   ```

### Issue: Threat Hunting shows error "Error fetching data"

**Solution:**

1. **Refresh index pattern:**
   - Go to **Stack Management → Data Views**
   - Click refresh icon on `wazuh-alerts-*`

2. **Use Discover instead:**
   - Navigate to **Explore → Discover**
   - This is more reliable for raw log viewing

---

## Project Outcomes

### Skills Demonstrated

✅ **IDS Deployment** - Installed and configured Snort for network monitoring  
✅ **SIEM Integration** - Connected Snort logs to centralized Wazuh SIEM  
✅ **Log Forwarding** - Configured Wazuh agent for cross-host log collection  
✅ **Custom Rule Development** - Created detection rules with severity mapping  
✅ **MITRE ATT&CK Mapping** - Aligned detections with industry-standard framework  
✅ **Docker Operations** - Deployed and managed containerized SIEM infrastructure  
✅ **Cross-Platform Integration** - Linux IDS → Windows SIEM via Docker  
✅ **Troubleshooting** - Diagnosed and resolved configuration issues

### Detection Capabilities

| Attack Type | Snort Signature | Custom Rule ID | Severity | MITRE Technique |
|-------------|-----------------|----------------|----------|-----------------|
| SNMP Reconnaissance | `1:1418:11`, `1:1421:11` | 100200 | 10 | T1046 |
| Nmap Port Scan | `1:1228:7` | 100201 | 11 | T1046 |
| XMAS Scan | `1:1228:7` | 100202 | 12 | T1046 |

---

## Future Enhancements

- Add email alerting for critical detections
- Create custom Wazuh dashboards for executive reporting
- Integrate additional threat intelligence feeds
- Implement automated response actions (e.g., firewall blocking)
- Expand detection coverage with custom Snort rules

---

## References

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Snort Documentation](https://www.snort.org/documents)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Wazuh Docker GitHub](https://github.com/wazuh/wazuh-docker)

---

## License

This project is for educational and demonstration purposes.

---

## Author

**Your Name**  
SOC Analyst | Security Engineering  
[GitHub](https://github.com/yourusername) | [LinkedIn](https://linkedin.com/in/yourprofile)
