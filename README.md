# 🔒 Domain-Integrated Cybersecurity Lab with SIEM Monitoring

---

## Overview

In this project I engineered an entire simulated a real-world enterprise network infastructre, combining Windows Active Directory domain services with centralized security monitoring via Splunk. Built entirely on a singular macOS host machine using only VirtualBox virtual machines, the lab environment is designed to allow hands-on experience with system engineering/administration, network troubleshooting, SIEM data ingestion, and cyberattack simulation/remediation.

---

## 📂 Lab Environment

| Component | Details |
|:---|:---|
| **Host System** | macOS (Apple Silicon) |
| **Hypervisors** | VirtualBox (VMs) |
| **Domain Controller/ AD** | Windows Server 2022 (Server222, Hostname: DC1) |
| **Client Machine** | Windows 10 (FinancePC1),(HRPC1), (ITPC1) |
| **SIEM Server** | Splunk integrated w/ Windows 10 (SplunkVM1) |
| **Attacker Machine** | Kali Linux (Virtualbox VM) |
| **Domain Name** | corp.local |
| **IP Addressing** | Static IPs within 10.0.0.X subnet |
| **DNS Server** | 10.0.0.98 (DC1) |

---

## 🛠️ Setup Phases

### Phase 1: Windows Server 2022 Domain Controller (DC1) (PowerShell-Only Setup)

1. **Installed Windows Server 2022** on Server222 VM (no GUI environment).
2. **Networking Configuration via PowerShell:**
    ```powershell
    New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.98 -PrefixLength 24 -DefaultGateway 10.0.0.1
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
    ```
3. **Installed Active Directory Domain Services Role:**
    ```powershell
    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
    ```
4. **Promoted Server to a Domain Controller for `corp.local`:**
    ```powershell
    Import-Module ADDSDeployment
    Install-ADDSForest -DomainName "corp.local" -SafeModeAdministratorPassword (ConvertTo-SecureString -AsPlainText "YourDSRMpassword" -Force) -Force
    ```
    - *(Note: The server automatically rebooted after promotion.)*
5. **Validation (Post-Reboot):**
    - Confirmed domain services installation:
      ```powershell
      Get-WindowsFeature AD-Domain-Services
      ```
    - Checked Active Directory status:
      ```powershell
      Get-ADDomain
      ```

📸 *(Screenshot suggestion: Output of `Get-ADDomain` confirming domain creation.)*

---

### Phase 2: Configure Client VM's (FinancePC1,HRPC1,ITPC1)

1. **Install Windows 10** on new VirtualBox VM. (Do this 2 more times for HRPC1 and ITPC1)
2. **Networking Settings:**
    - Static IP: `10.0.0.100`
    - Subnet Mask: `255.255.255.0`
    - Default Gateway: `10.0.0.1`
    - DNS Server: `10.0.0.98`
3. **Domain Join:**
    - Open **PowerShell as Administrator**.
    - Execute:
      ```powershell
      Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.98
      Add-Computer -DomainName corp.local -Credential corp\Administrator -Restart
      ```
4. **Troubleshooting Steps:**
    - Verify DNS settings.
    - Ping DC1 to ensure connectivity.
    - If domain join fails, use `Test-ComputerSecureChannel`.
5. **Validation:**
    - Log into FinancePC1 with `corp\Administrator` credentials.
    - Confirm domain join via System Properties.
  
    - (insert picture of domain join)

---

### Phase 3: SIEM Setup (Splunk Deployment)

#### Step 1: Install and Configure Splunk Enterprise (SplunkVM1)
- Created a new Windows 10 VM (`SplunkVM1`) inside VirtualBox.
- Downloaded and installed **Splunk Enterprise Free** edition.
- During installation:
  - Accepted the Free License Agreement (non-production use).
  - Set up initial admin credentials.
- Post-install:
  - Enabled **Splunk Web** (listens on port 8000).
  - Configured **TCP Data Inputs**:
    - Navigated to **Settings → Data Inputs → TCP → Add New**.
    - Created a new TCP Listener on **Port 9997** (default for forwarders).

📸 *(Screenshot here: Splunk Data Inputs page showing port 9997 active)*

#### Step 2: Install and Configure Splunk Universal Forwarder (FinancePC1)
- Installed **Splunk Universal Forwarder** on **FinancePC1**.
- During setup:
  - Configured it to **send Security and Application Event Logs**.
  - Set the destination IP address to **SplunkVM1’s static IP** (on TCP 9997).
- Verified **Windows Firewall** was allowing outbound traffic to 9997.

📸 *(Screenshot here: Forwarder settings or install wizard showing connection to SplunkVM1)*

#### Step 3: Validate Log Forwarding
- Logged into **Splunk Web** (`https://<SplunkVM1-IP>:8000`).
- Ran a Splunk Search:
  ```spl
  index=* host=FinancePC1
  ```
- Confirmed Security and Application logs were actively being received.

📸 *(Screenshot here: Splunk search results showing real-time logs from FinancePC1)*

---

### Phase 4: Simulate Attacks (Preparation)

- **Deploy Kali Linux via UTM.**
- **Simulate Attacks:**
    - Network scanning via Nmap:
      ```bash
      nmap -sS 10.0.0.0/24
      ```
    - Brute-force RDP (hydra or rdesktop).
    - Malicious PowerShell scripts.
- **Monitor Splunk for:**
    - Event ID 4625 (Failed login attempts).
    - Network scans.
    - Suspicious PowerShell execution (Event ID 4104).

---

### Phase 5: Attack Detection and Findings in Splunk

After conducting attack simulations using Kali Linux (KaliVM), key security events were successfully detected and captured in Splunk.

#### 1. Detected Events

| Attack Type | Event Detected | Event ID / Evidence |
|:---|:---|:---|
| Network Scanning (Nmap) | Spike in inbound TCP SYN packets | Logged in Windows Firewall logs |
| Brute-force RDP attempts | Multiple failed RDP login attempts | Windows Event ID `4625` |
| Malicious PowerShell Activity | Suspicious script execution | Windows Event ID `4104` |

#### 2. Example Splunk Searches

- **Failed RDP Login Attempts:**

    ```spl
    index=windows EventCode=4625
    ```

- **Suspicious PowerShell Executions:**

    ```spl
    index=windows EventCode=4104
    ```

- **Detect Possible Port Scanning (Firewall Logs):**

    ```spl
    index=windows sourcetype="WinEventLog:Security" action=blocked
    ```

#### 3. Screenshots

📸 *(Screenshot suggestion: Show Splunk search results highlighting detected attacks — Event ID 4625 failures, suspicious PowerShell logs, network scan detections.)*

#### 4. Findings Summary

- **FinancePC1** generated multiple failed login events during brute-force RDP simulation.
- **Suspicious PowerShell activity** was logged when running test scripts from KaliVM.
- **Network scans** triggered multiple blocked connections recorded in Firewall logs.
- All relevant events were successfully forwarded to and detected in **SplunkVM1**.

---

## 📈 Planned Diagrams

### 1. Network Topology

```
                   +---------------+
                   |  macOS Host    |
                   +-------+-------+
                           |
                    (VirtualBox / UTM)
                           |
   +-----------+------------+-----------+------------+
   |           |                        |            |
+--+--+    +---+---+                +----+---+    +---+---+
| DC1 |    | FinancePC1 |          | SplunkVM1 |  | KaliVM |
+-----+    +-----------+          +----------+  +--------+
 (AD)         (Client)               (SIEM)      (Attacker)
```

📸 *(Screenshot here: A simple network topology diagram made with draw.io or Lucidchart)*

### 2. Splunk Data Flow

```
FinancePC1 (Forwarder) ---> SplunkVM1 (Indexer/Search Head)
```

📸 *(Screenshot here: Splunk showing logs forwarded from FinancePC1)*

---

## 🚀 Future Enhancements

- Create Organizational Units (OUs) for HR, Finance, IT.
- Apply Group Policy Objects (GPOs) for security baselines.
- Build custom Splunk Dashboards (Login Trends, Attack Alerts).
- Configure real-time Splunk Alerting (emails or notifications).
- Simulate Advanced Persistent Threat (APT)-style attack chains.
- (Optional Stretch Goals):
  - Create a phishing simulation (send a fake phishing email to a dummy mailbox).
  - Write a mini incident report for one of the simulated attacks.

---

## 📈 Project Status

| Task | Status |
|:---|:---|
| Domain Setup | ✅ Complete |
| Client Join to Domain | ✅ Complete |
| SIEM Setup | ✅ Complete |
| Log Forwarding | ✅ Verified |
| Attack Simulation | ✅ Complete |

---

## 📚 Summary

This lab environment replicates a small-to-medium enterprise network structure, integrating core IT and cybersecurity practices. It demonstrates a technical foundation in Windows Server management, Active Directory administration, endpoint deployment, Splunk SIEM operations, and attack monitoring.

This project is ideal for inclusion in a professional GitHub portfolio or LinkedIn showcase to highlight real-world applicable cybersecurity and system administration skills.

---

# ✨ End of Writeup

---

