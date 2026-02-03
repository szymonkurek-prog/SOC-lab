# SOC Homelab

A small home lab for practicing SOC‑analyst skills.  
This environment is designed to simulate a basic SOC with endpoints, an attacker machine, and a central SIEM.

## Devices and roles

- **Laptop (Windows 11, i5‑11300H, 16 GB RAM)**  
  - Host for VirtualBox.  
  - Runs VMs: Windows 11 (endpoint), Kali Linux (attacker), Ubuntu Desktop (analyst / extra endpoint).  
  - Main workstation and lab documentation.

- **Lenovo (i5, 8 GB RAM)**  
  - Ubuntu Server 22.04 LTS.  
  - Runs **Splunk Enterprise** as SIEM to collect logs from endpoints.

## What I have already done

- [x] Created a dedicated SOC‑lab account on the laptop (local, admin, no bloatware).  
- [x] Installed VirtualBox on Windows 11.  
- [x] Created and installed VMs on the laptop:  
  - Windows 11 (endpoint, will run Sysmon + Splunk UF).  
  - Kali Linux (attacker).  
  - Ubuntu Desktop (analyst / extra endpoint).  
- [ ] Installed and configured Splunk on Lenovo.  
- [ ] Connected Windows 11 VM to the SIEM (Splunk UF or Wazuh agent).  
- [ ] Connected Ubuntu Desktop VM to the SIEM.  
- [ ] Will add a physical honeypot on a third computer and connect it to the SIEM.

## How to run this lab

1. On the laptop:  
   - Log in to the **SOC‑lab** account.  
   - Start VirtualBox.  
   - Start the VMs: **Windows 11**, **Kali Linux**, and **Ubuntu Desktop**.  

2. On Lenovo:  
   - Log in to Ubuntu Server.  
   - Start Splunk:  
     ```bash
     sudo /opt/splunk/bin/splunk start
     ```  
   - Open Splunk in a browser: `https://<Lenovo_IP>:8000`.  

3. In Splunk (or Wazuh):  
   - Verify that logs from Windows 11 and Ubuntu Desktop are arriving.  
   - Run example searches (SPL) to see events from the endpoint and attacker VMs.

## What I plan next

- [ ] Install and configure **Sysmon** on the Windows 11 VM.  
- [ ] Install **Splunk Universal Forwarder** on Windows 11 and Ubuntu Desktop.  
- [ ] Write basic detection rules and dashboards in Splunk for:  
  - SSH brute‑force attempts.  
  - RDP logins.  
  - Network scans from Kali.  
- [ ] Add a **physical honeypot** (third computer) and connect it to the SIEM.  
- [ ] Document TryHackMe SOC Level 1 exercises in `/docs/thm-soc/` and link them to real lab events.
