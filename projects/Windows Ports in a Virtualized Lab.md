
# Lab Report – Nmap, Wireshark and Windows Ports in a Virtualized Lab

## Environment

 - Hypervisor: virtualized lab with two VMs
    - Ubuntu Linux (sniffer, running Wireshark)
    - Windows (target)  
 - Network: all two machines in the same host-only / internal network
    - Example IPs:
      - Ubuntu: 192.168.56.10
      - Windows: 192.168.56.12
        
## Goal

The goal of this lab was to understand why, when scanning the Windows VM from Ubuntu using nmap, only the outbound traffic from the scanner was visible in Wireshark, and why responses from Windows were often not seen. The lab also aimed to clarify the difference between:
  - A port listening locally on Windows (visible in netstat / Get-NetTCPConnection), and
  - A port being reachable and clearly “open” from another host, as seen by nmap and Wireshark.

## Initial Observations

1. **ICMP (ping)**
  - Windows responded to ICMP echo requests when pinged from other machines.
  - This confirmed basic IP connectivity and that the Windows host was reachable.
    
2. **Nmap scans**
  - When scanning Windows from Kali/Ubuntu, Wireshark running on Ubuntu initially showed only packets coming from the scanner (e.g., SYN), but not necessarily responses from Windows for some ports.
  - For some ports, especially “sensitive” ones like TCP 139 (NetBIOS-SSN), nmap reported the port as filtered/host down even though netstat on Windows showed it as LISTENING.
    
3. **Wireshark capture location**
  - Wireshark was capturing on the Ubuntu VM, not on Windows.
  - Because the virtual network switch behaves like a real switch, the sniffer on a third VM may not see all traffic between two other endpoints unless the switch forwards or mirrors it.
  - For correctly configured interfaces, Ubuntu could still see traffic for some flows (e.g., for the HTTP test on port 8080 described later).
4. **Inspecting Windows Ports**
  - On the Windows VM, the following commands were used to inspect listening ports:
```powershell
Get-NetTCPConnection -State Listen
Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -eq 5357 }
Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -eq 139 }```
Typical findings:

Port 5357 listening on :: (IPv6 any).

Port 139 listening on 192.168.56.12 (host-only interface).

This confirmed that Windows had local listeners on these ports. However, that alone does not guarantee that the ports are reachable from other hosts or that Windows will respond to arbitrary scans from the lab network.

Key Concept: Listening vs. Reachable
A critical takeaway from the lab:

LISTENING in netstat / Get-NetTCPConnection only tells us that some process has bound to that port on some local IP address.

For a remote host to see the port as open, all of the following must be true:

The service is bound to an IP address that is reachable from the remote host (e.g., not only 127.0.0.1).

The firewall and network policies allow incoming connections from that remote IP and network profile.

The OS actually responds to incoming SYN packets (with SYN,ACK or RST) instead of silently dropping them.

In this lab, this distinction was particularly important for ports like 139, where local listening did not imply a clear response to nmap from another VM.

Controlled Test with Python HTTP Server (Port 8080)
To have a clean, predictable service, a simple HTTP server was started on the Windows VM:

powershell
cd C:\path\to\some\directory
python -m http.server 8080
This command starts a basic HTTP server:

It listens on TCP port 8080.

It usually binds to 0.0.0.0, meaning it accepts connections on all interfaces.

It serves files from the current directory over HTTP.

From Ubuntu, connectivity was tested using:

bash
curl http://192.168.56.12:8080
and port scanning using:

bash
nmap -sS -p 8080 -Pn 192.168.56.12
Wireshark Capture on Ubuntu
A Wireshark capture taken on Ubuntu for the curl request to port 8080 showed a textbook TCP + HTTP exchange:

TCP three-way handshake

192.168.56.10 → 192.168.56.12, TCP 50008 → 8080 [SYN]

192.168.56.12 → 192.168.56.10, TCP 8080 → 50008 [SYN, ACK]

192.168.56.10 → 192.168.56.12, TCP 50008 → 8080 [ACK]

This confirms that:

Ubuntu’s client initiated the connection with SYN.

Windows’ HTTP server responded with SYN,ACK.

Ubuntu acknowledged with ACK.

In other words: port 8080 is open and reachable.

HTTP request and response

Client (Ubuntu) sent: GET / HTTP/1.1, Host: 192.168.56.12:8080, User-Agent: curl/..., Accept: */*.

Server (Windows) responded with:

HTTP/1.0 200 OK

Server: SimpleHTTP/0.6 Python/3.x

Content-type: text/html; charset=utf-8

Content-Length: ...

HTML content in the payload.

This proves that:

The service is not only listening but actually handling application-level HTTP requests correctly.

Ubuntu can see both the request and the response in Wireshark when the service is reachable and allowed.

TCP connection teardown

Server and client exchanged FIN, ACK packets to close the connection cleanly.

This capture is an ideal reference of “good” behavior: full handshake, request, response, and teardown, with traffic visible in both directions.

Behavior of Port 139
In contrast, when scanning port 139 (NetBIOS-SSN):

Windows showed the port as LISTENING locally on 192.168.56.12:139.

However, from Ubuntu:

nmap -sS -p 139 -Pn 192.168.56.12 only showed outgoing SYN packets in Wireshark.

There were no SYN,ACK or RST responses from Windows for these attempts.

Interpretation:

The OS and/or firewall on Windows is configured to silently drop incoming connection attempts to this port from the lab machine, or more generally, from that network/profile.

From the perspective of nmap, “SYN → no response” leads to results such as filtered or “host down”, depending on the scan type and configuration.

This explains why nmap did not report port 139 as clearly open, even though netstat showed it as listening.

In other words:

Port 8080 (Python HTTP server):

Listening, reachable, responds with SYN,ACK and HTTP 200 → nmap and Wireshark see it as open and active.

Port 139 (NetBIOS-SSN):

Listening locally, but Windows does not respond to those SYN packets from the lab host → from the outside it looks filtered/stealth, not clearly open.

This demonstrates the difference between:

“service bound to a port”,

“port allowed and visible on the network”,

“behavior seen by nmap and captured by Wireshark.”

Lessons Learned
Sniffer placement matters

In a virtual environment, a third VM running Wireshark is not automatically a “hub”; it still depends on how the virtual switch forwards traffic.

In this lab, Ubuntu was correctly capturing the HTTP flow on port 8080 but still did not see responses for port 139, because Windows simply never sent them.

Listening ≠ Open for everyone

A port can be in LISTEN state locally while still being effectively invisible or filtered from remote hosts because of firewall rules, network profiles, or OS security policies.

“SYN → silence” is meaningful

When nmap sends a SYN and never sees SYN,ACK or RST, it interprets that as filtered or host down, not as cleanly open or closed.

This pattern appeared clearly on port 139.

Using a simple HTTP server is an excellent teaching tool

python -m http.server 8080 plus curl and Wireshark gives a clean, easy-to-read example of the TCP handshake, HTTP request/response, and connection close.

This makes it much easier to compare “good” traffic (8080) with “stealthy/filtered” traffic (139).

Summary
The lab started from the question: “Why do I only see traffic from the scanner and not from Windows in Wireshark when using nmap?”

By:

Validating connectivity (ICMP),

Checking local listeners on Windows,

Starting a controlled HTTP server on port 8080,

Using curl and nmap from Ubuntu,

Capturing the traffic in Wireshark,

it became clear that:

For an open, reachable port (8080), Wireshark shows a complete SYN → SYN,ACK → ACK handshake and full HTTP exchange.

For a protected port like 139, Windows may silently drop SYN packets from the lab network, causing nmap to see the port as filtered/host down, and Wireshark to show only outgoing SYN from the scanner.

This demonstrates the practical difference between a port that is locally listening and one that is remotely observable as open, and shows how to use Wireshark, curl, and nmap together to reason about that behavior.
