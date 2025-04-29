# 🔒 Domain-Integrated Cybersecurity Lab with SIEM Monitoring

---

## Overview

In this project I engineered an entire simulated real-world enterprise network infastructure, combining Windows Active Directory domain services with centralized security monitoring via Splunk. Built entirely on a singular macOS host machine using only VirtualBox virtual machines, the lab environment is designed to allow hands-on experience with system engineering/administration, network troubleshooting, SIEM data ingestion, and cyberattack simulation/remediation.

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

<img width="526" alt="Screenshot 2025-04-27 at 6 16 21 PM" src="https://github.com/user-attachments/assets/79328d00-eb42-4150-bcf7-0b43aa2c3b1b" />
<img width="521" alt="Screenshot 2025-04-27 at 6 18 47 PM" src="https://github.com/user-attachments/assets/f7193e32-0b2c-48d7-abc3-581d3cc15d9c" />


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
   
  
   <img width="524" alt="Screenshot 2025-04-27 at 6 24 05 PM" src="https://github.com/user-attachments/assets/f9dfff61-f73a-411e-9241-abd19ae797c0" />
* *Notice Domain is corp.local, IP is 10.0.0.X* 

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
<img width="516" alt="Screenshot 2025-04-27 at 6 58 42 PM" src="https://github.com/user-attachments/assets/2d7840b9-1962-496d-9e05-3e28e331543b" />

<img width="507" alt="Screenshot 2025-04-27 at 6 42 23 PM" src="https://github.com/user-attachments/assets/a75b9cb2-7228-44a1-aee3-c22a29e58d66" />

* *On Splunk Enterprise App. port 9997 verification is funky, but in screenshot you see port 9997 is listening* 


#### Step 2: Install and Configure Splunk Universal Forwarder (FinancePC1)
- Installed **Splunk Universal Forwarder** on **FinancePC1**.
- During setup:
  - Configured it to **send Security and Application Event Logs**.
  - Set the destination IP address to **SplunkVM1’s static IP** (on TCP 9997).
- Verified **Windows Firewall** was allowing outbound traffic to 9997.


#### Step 3: Validate Log Forwarding
- Logged into **Splunk Web** (`https://<SplunkVM1-IP>:8000`).
- Ran a Splunk Search:
  ```spl
  index=* host=FinancePC1
  ```
- Confirmed Security and Application logs were actively being received.

<img width="522" alt="Screenshot 2025-04-27 at 6 30 36 PM" src="https://github.com/user-attachments/assets/43667b03-0e28-4a38-97b5-42184132425e" />

* *This verifies Logs are being sent from Finance PC1 to Splunk Enterprise* 


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

<img width="516" alt="Screenshot 2025-04-27 at 7 13 29 PM" src="https://github.com/user-attachments/assets/e60b28dd-3a66-4f4e-af09-94602ab7b2f8" />

* Using Splunk, we successfully ingested Windows security event logs from client machines on the "corp.local" domain.
In this phase, we performed a search and identified multiple EventCode 5379 events, which are related to Credential Manager credentials being read.
This confirmed that the Splunk server was correctly receiving and indexing logs from FinancePC1, and could detect key security events for monitoring and analysis.

#### 4. Findings Summary
- FinancePC1 successfully generated Windows security events, including EventCode 5379, related to Credential Manager activity.
- SplunkVM1 received and indexed all relevant log data from FinancePC1, confirming successful log forwarding and detection.
- The system demonstrated the ability to collect, centralize, and search security events across the domain environment.
- These findings validate the functionality of the SIEM setup and its readiness for detecting key security activities.

---

## 📈 Diagrams

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


### 2. Splunk Data Flow

```
FinancePC1 (Forwarder) ---> SplunkVM1 (Indexer/Search Head)
```



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



---

# ✨ End of Writeup

---

