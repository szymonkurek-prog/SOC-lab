# Lab: Nmap Host Discovery with Windows and Linux

## 1. Goal of the exercise

The goal of this exercise was to build a simple lab environment with three virtual machines (Windows 10, Ubuntu/Linux, Kali) and understand:

- how host discovery in nmap works (ICMP, TCP SYN, TCP ACK),
- how the Windows firewall affects ping and nmap,
- how to configure IP addressing and firewall rules using both GUI and PowerShell.

---

## 2. Test environment

- 3 virtual machines in the same subnet:
  - **Windows 10 Pro** – target (scanned host)
  - **Ubuntu / other Linux** – host running nmap
  - **Kali Linux** – additional test host (offensive tools)
- Static IP addresses configured on Windows (GUI + PowerShell) and on the Linux machines.
- All VMs in the same virtual network (`192.168.56.0/24`, host‑only / internal mode).

---

## 3. Network configuration on Windows 10 (IP, mask, gateway, DNS)

### 3.1. Configuration via GUI

A static IP address was configured on Windows 10 using:

`Settings → Network & Internet → Change adapter options → Properties → Internet Protocol Version 4 (TCP/IPv4)`

Values set:

- IP address (e.g. `192.168.56.12`)
- Subnet mask `/24` (e.g. `255.255.255.0`)
- Default gateway (e.g. `192.168.56.1`)
- DNS server (e.g. `8.8.8.8` or router IP)

### 3.2. Configuration via PowerShell

```powershell
Get-NetAdapter

New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.56.12 `
  -PrefixLength 24 `
  -DefaultGateway 192.168.56.1

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
  -ServerAddresses 8.8.8.8
```
---

## 4. Windows firewall and ICMP (ping)
### 4.1. Initial problem
#### Symptom:

- Windows could ping Linux/Kali.
- Linux/Kali could not ping Windows.

#### Cause:

- The Windows firewall was blocking incoming ICMP Echo Request (ping).

### 4.2. Checking firewall status
```powershell
Get-NetFirewallProfile | Format-Table Name, Enabled

Get-NetFirewallProfile -Profile Private | Format-List *
```
This shows which profiles (Domain/Private/Public) are enabled and their default policies.

### 4.3. Checking ICMP (Echo Request) rules
```powershell
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" |
  Where-Object DisplayName -like "*Echo Request*" |
  Format-Table DisplayName, Enabled, Direction
```
Explanation:

- DisplayGroup "File and Printer Sharing" selects all rules in the File and Printer Sharing group.
- like "*Echo Request*" uses wildcards to match rules that contain Echo Request in their display name.
  
The table shows whether these rules are enabled and in which direction they apply.

### 4.4. Enabling ping replies
Enable all Echo Request rules:

```powershell
Get-NetFirewallRule -DisplayGroup "File and Printer Sharing" |
  Where-Object DisplayName -like "*Echo Request*" |
  Enable-NetFirewallRule
```
Effect:

- Windows started replying to ping from Kali/Ubuntu.

ping ```192.168.56.12``` worked in both directions.

---

## 5. Host discovery in nmap – tests from Ubuntu
### 5.1. Role of sudo
sudo runs commands with administrator (root) privileges. For nmap this is important because it allows:

sending raw ICMP packets,

sending custom TCP SYN/ACK probes,

using full host discovery capabilities.

### 5.2. Commands tested (single host – Windows)
ICMP Echo ping:

bash
sudo nmap -sn -PE <IP_Windows>
TCP SYN ping on ports 80 and 443:

bash
sudo nmap -sn -PS80,443 <IP_Windows>
TCP ACK ping on ports 80 and 443:

bash
sudo nmap -sn -PA80,443 <IP_Windows>
All three methods detected the host as up.

Method summary:

-PE – ICMP Echo Request (similar to classic ping).

-PS – sends TCP SYN; a SYN/ACK or RST response indicates the host is alive.

-PA – sends TCP ACK; an RST response indicates the host is alive (often useful when SYN packets are filtered more strictly).

### 5.3. Conclusions from host discovery tests
After enabling ICMPv4 in the Windows firewall, -sn -PE was sufficient as the default host discovery method in this lab.

-PS and -PA serve as backup / alternative methods when:

ICMP is disabled or filtered,

you want to observe how a firewall handles SYN vs ACK packets.

---

## 6. Host discovery vs port scan
Host discovery (-sn, -PE, -PS, -PA, etc.):

Answers the question: “Is this host alive?”

Uses a small number of probes (ICMP or TCP) to determine whether a host responds at all.

Does not perform a full port scan.

Port scan (-sS, -sT, -sU, etc.):

Answers the question: “Which ports are open/closed/filtered on this host?”

Systematically tests multiple ports and reports their state.

Typical nmap workflow:

Host discovery → get a list of live IP addresses.

Port scan → scan selected hosts for open ports and services.

---

## 7. Proposed next steps
As a continuation of this lab, the following steps are recommended:

Run a basic TCP port scan on the Windows host:

bash
sudo nmap -sS <IP_Windows>
sudo nmap -sV <IP_Windows>
Compare which ports are open and what services nmap identifies.

Modify Windows firewall rules (block/unblock specific ports) and observe how the nmap results change.

---

## 8. Suggested screenshots for the README
Recommended screenshots to include in the GitHub README or wiki:

Lab topology

Hypervisor view showing the 3 VMs and their network adapters / subnet.

Windows IP configuration (GUI)

IPv4 properties window with IP address, subnet mask, default gateway, and DNS.

PowerShell – IP configuration

Output of Get-NetAdapter and Get-NetIPAddress displaying the configured IP, prefix length and gateway.

Optionally: console after running New-NetIPAddress.

Firewall status

Output of Get-NetFirewallProfile | Format-Table Name, Enabled.

Echo Request rules

Output of the command with *Echo Request* showing rule names, enabled status and direction.

Second screenshot after Enable-NetFirewallRule, showing the rules enabled.

Ping from Ubuntu/Kali to Windows

Terminal with successful ping <IP_Windows> after firewall changes.

nmap host discovery – single host

Output of:

sudo nmap -sn -PE <IP_Windows>

sudo nmap -sn -PS80,443 <IP_Windows>

sudo nmap -sn -PA80,443 <IP_Windows>

Each showing “Host is up”.

nmap host discovery – entire subnet

Output of sudo nmap -sn -PE 192.168.X.0/24 listing all discovered hosts (Ubuntu, Kali, Windows).

Each screenshot can be briefly captioned in the README, for example:

Fig. 3 – Windows firewall after enabling Echo Request rules; the host starts replying to ping from Linux.

---

## 9. Example “Conclusions” section (optional)
You can add something like this at the end of the README:

Learned how to configure static IP addresses on Windows using both GUI and PowerShell.

Observed how Windows Defender Firewall can block ping and affect nmap host discovery.

Practiced different nmap host discovery methods (-PE, -PS, -PA) and saw how they behave against the same host.

Understood the difference between host discovery and a full port scan.

Prepared a lab environment that can be extended with more advanced nmap scans and firewall experiments.
