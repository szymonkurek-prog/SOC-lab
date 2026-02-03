# SOC-lab
Building own SOC lab for different projects to start gaining experience as a SOC analyst.

A small lab for practicing SOC‑analyst work.  
I use:  
- Laptop with Windows 11 + VirtualBox (VMs: Windows 11, Kali, Ubuntu).  
- Lenovo with Ubuntu Server + Splunk as SIEM.  

## What I have already done

- [x] Created a SOC‑lab account on the laptop.  
- [ ] Installed VirtualBox and created a Windows 11 VM.  
- [ ] Installed a Kali Linux VM.  
- [x] Installed an Ubuntu Desktop VM.  
- [ ] Configured Splunk on Lenovo.  
- [ ] Connected Windows 11 to Splunk (Sysmon + Universal Forwarder).  

## How to run this lab

1. On the laptop, start VirtualBox.  
2. Start the Windows 11, Kali, and Ubuntu VMs.  
3. On Lenovo, start Splunk (`sudo /opt/splunk/bin/splunk start`).  
4. In Splunk, check whether you see logs from Windows 11.  

## What I plan next

- Add a honeypot on a third computer.  


