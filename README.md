# 🛡️ Azure SOC Honeypot Lab — RDP Brute-Force Detection & Threat Hunting

> **A hands-on SOC project using Microsoft Azure, Microsoft Sentinel, and KQL to detect and hunt real-world RDP brute-force attacks against a deliberately exposed honeypot VM.**

---

## 📋 Project Overview

A Windows Virtual Machine (`labtestvm`) was deployed on Azure with **RDP port 3389 intentionally exposed** to the public internet as a honeypot. Within hours, the VM attracted thousands of real brute-force login attempts from automated scanners worldwide.

Microsoft Sentinel was configured to:
- Ingest Windows Security Event logs
- Detect successful RDP sign-ins with a custom analytics rule
- Alert via Microsoft Defender for Cloud
- Enable KQL-based threat hunting to surface top attacking IPs

---

## 🏗️ Architecture

```
Internet (Attackers)
        │
        ▼ RDP :3389
┌──────────────────────┐
│   Azure NSG          │  ← Inbound rule: Allow/Deny port 3389
│   labtestvm-nsg      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│   labtestvm          │  Windows Server · Standard D2s v3
│   Public IP:         │  South Africa North (Zone 1)
│   4.222.185.235      │
└──────────┬───────────┘
           │ Security Events
           ▼
┌──────────────────────┐
│  Log Analytics       │  Windows Security Events → Sentinel
│  Workspace (labtest) │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐     ┌─────────────────────────────┐
│  Microsoft Sentinel  │────▶│  Microsoft Defender Alerts  │
│  Analytics Rules     │     │  "Successful local sign-ins" │
│  KQL Hunting Queries │     │  Severity: High             │
└──────────────────────┘     └─────────────────────────────┘
```

---

## ⚙️ Prerequisites

- Azure subscription (free tier works)
- Resource group in your preferred region
- Microsoft Sentinel workspace connected to a Log Analytics workspace
- Windows VM with the **Microsoft Monitoring Agent (MMA)** or **Azure Monitor Agent (AMA)** installed
- Data connector: **Windows Security Events** enabled in Sentinel

---

## 🚀 Lab Walkthrough

### Step 1 — Deploy the VM
1. Create a Windows Server VM in Azure (Standard D2s v3 or similar)
2. Assign a public IP
3. Note the VM's public IP address

### Step 2 — Open RDP (Honeypot Mode)
1. Go to **VM → Networking → Network settings**
2. Add an inbound NSG rule:
   - **Priority:** 300
   - **Port:** 3389
   - **Protocol:** TCP
   - **Source/Destination:** Any
   - **Action:** Allow
3. Start the VM

### Step 3- Configure Microsoft Sentinel
1. Create a Sentinel workspace
2. Connect the **Windows Security Events** data connector
3. Create an **Analytics Rule** for EventID 4624 (successful logon, logon type 10)

### Step 4- Trigger the Alert
1. RDP into the VM from your machine using `mstsc`
2. Check **Microsoft Defender → Alerts** for the triggered rule

### Step 5- Harden the VM
1. Return to **NSG → Inbound rules**
2. Change the RDP rule Action from **Allow** → **Deny**

### Step 6- Threat Hunt with KQL
Run the following in **Sentinel → Logs** to find real brute-force attackers:

```kql
SecurityEvent
| where TimeGenerated > ago(1d)
| where EventID == 4625
| summarize Count=count() by IpAddress, Computer
| where Count >= 15
| order by Count desc
```

---

## 📊 Key Findings

Within **a few hours** of exposure, the honeypot attracted **real automated attack traffic**:

| IP Address       | Failed Logins (24h) |
|-----------------|---------------------|
| 185.156.73.74   | 10,214              |
| 185.156.73.169  | 10,133              |
| 92.63.197.69    | 10,010              |
| 185.156.73.59   | 10,003              |
| 92.63.197.9     | 9,957               |
| 185.156.73.69   | 9,942               |

The `185.156.73.x` and `92.63.197.x` subnets showed coordinated scanning consistent with automated brute-force tooling.

---

## 🔍 KQL Queries

See [`/kql/`](./kql/) for:
- `brute_force_hunt.kql` - Top attacking IPs by failed login count
- `rdp_success_detection.kql` - Sentinel analytics rule for successful RDP logins

---

## 📸 Screenshots

All evidence screenshots are in [`/screenshots/`](./Screenshots/):

| File | Description |
|------|-------------|
| `resourse-group_snapshot.png` | Azure resource group setup |
| `vm-overview_snapshot.png` | VM overview with public IP |
| `rdp-open_snapshot.png` | NSG rule — RDP allowed (honeypot) |
| `rdplogin-success_snapshot.png` | Defender alert triggered |
| `rdp-closed_snapshot.png` | NSG rule — RDP denied (hardened) |
| `brute-force-hunt_snapshot.png` | KQL hunt results in Sentinel |

---

## 💡 Lessons Learned

- **Exposed RDP is an immediate target** - real attacks started within hours
- **Sentinel end-to-end detection works** - from log ingestion to Defender alert
- **KQL is essential** for SOC analysts - a 6-line query surfaced the full threat picture
- **NSG rules are the fastest containment tool** for network-level response

---

## 🔒 Recommendations

- ❌ Never expose RDP publicly in production - use **Azure Bastion** or a VPN
- ✅ Enable **Just-in-Time (JIT) VM access** in Defender for Cloud
- ✅ Enforce **MFA** for all VM logins
- ✅ Set up Sentinel analytics rules for **failed login thresholds**
- ✅ Regularly review Sentinel workbooks for lateral movement indicators

---

## 📄 Full Documentation

See [`docs/Azure_SOC_Honeypot_Project.pdf`](./Azure_SOC_Honeypot_Project.pdf) for the complete project writeup with all screenshots.

---

## 🪪 License

MIT License - feel free to fork and adapt for your own SOC lab.

---

*All attack data is real, captured from internet-facing honeypot infrastructure on Azure. No production systems were affected.*
