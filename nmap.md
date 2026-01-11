# 1) Nmap

## 1.1 Host Discovery

**Scan network range:**

    sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.0/24` | Target network range. |
| `-sn` | Disables port scanning. |
| `-oA tnet` | Stores the results in all formats starting with the name 'tnet'. |

This scanning method works only if the firewalls of the hosts allow it. Otherwise, we can use other scanning techniques to find out if the hosts are active or not. We will take a closer look at these techniques in "`Firewall and IDS Evasion`".

* * *

**Scan IP list:**

with an IP list with the hosts we need to test.

    sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5

| **Scanning Options** | **Description** |
| --- | --- |
| `-sn` | Disables port scanning. |
| `-oA tnet` | Stores the results in all formats starting with the name 'tnet'. |
| `-iL` | Performs defined scans against targets in provided 'hosts.lst' list. |

In this example, we see that only 3 of 7 hosts are active. Remember, this may mean that the other hosts ignore the default **ICMP echo requests** because of their firewall configurations. Since `Nmap` does not receive a response, it marks those hosts as inactive.

* * *

**Scan Multiple IPs**:

It can also happen that we only need to scan a small part of a network. An alternative to the method we used last time is to specify multiple IP addresses.

    sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20| grep for | cut -d" " -f5

or

    sudo nmap -sn -oA tnet 10.129.2.18-20| grep for | cut -d" " -f5

* * *

**Scan Single IP:**

Before we scan a single host for open ports and its services, we first have to determine if it is alive or not. For this, we can use the same method as before.

    sudo nmap 10.129.2.18 -sn -oA host 

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.18` | Performs defined scans against the target. |
| `-sn` | Disables port scanning. |
| `-oA host` | Stores the results in all formats starting with the name 'host'. |

If we disable port scan (`-sn`), Nmap automatically ping scan with `ICMP Echo Requests` (`-PE`). Once such a request is sent, we usually expect an `ICMP reply` if the pinging host is alive. The more interesting fact is that our previous scans did not do that because before Nmap could send an ICMP echo request, it would send an `ARP ping` resulting in an `ARP reply`. We can confirm this with the "`--packet-trace`" option. To ensure that ICMP echo requests are sent, we also define the option (`-PE`) for this.

    sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace 

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.18` | Performs defined scans against the target. |
| `-sn` | Disables port scanning. |
| `-oA host` | Stores the results in all formats starting with the name 'host'. |
| `-PE` | Performs the ping scan by using 'ICMP Echo requests' against the target. |
| `--packet-trace` | Shows all packets sent and received |

Another way to determine why Nmap has our target marked as "alive" is with the "`--reason`" option.

    sudo nmap 10.129.2.18 -sn -oA host -PE --reason 

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.18` | Performs defined scans against the target. |
| `-sn` | Disables port scanning. |
| `-oA host` | Stores the results in all formats starting with the name 'host'. |
| `-PE` | Performs the ping scan by using 'ICMP Echo requests' against the target. |
| `--reason` | Displays the reason for specific result. |

We see here that `Nmap` does indeed detect whether the host is alive or not through the `ARP request` and `ARP reply` alone. To disable ARP requests and scan our target with the desired `ICMP echo requests`, we can disable ARP pings by setting the "`--disable-arp-ping`" option. Then we can scan our target again and look at the packets sent and received.

    sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping 

We have already mentioned in the "`Learning Process`," and at the beginning of this module, it is essential to pay attention to details. An `ICMP echo request` can help us determine if our target is alive and identify its system. More strategies about host discovery can be found at: https://nmap.org/book/host-discovery-strategies.html

* * *

## 1.2 Host and Port Scanning

| **State** | **Description** |
| --- | --- |
| `open` | This indicates that the connection to the scanned port has been established. These connections can be **TCP connections**, **UDP datagrams** as well as **SCTP associations**. |
| `closed` | When the port is shown as closed, the TCP protocol indicates that the packet we received back contains an `RST` flag. This scanning method can also be used to determine if our target is alive or not. |
| `filtered` | Nmap cannot correctly identify whether the scanned port is open or closed because either no response is returned from the target for the port or we get an error code from the target. |
| `unfiltered` | This state of a port only occurs during the **TCP-ACK** scan and means that the port is accessible, but it cannot be determined whether it is open or closed. |
| `open\\|filtered` | If we do not get a response for a specific port, `Nmap` will set it to that state. This indicates that a firewall or packet filter may protect the port. |
| `closed\\|filtered` | This state only occurs in the **IP ID idle** scans and indicates that it was impossible to determine if the scanned port is closed or filtered by a firewall. |

* * *

**Discovering Open TCP Ports**:

By default, `Nmap` scans the top 1000 TCP ports with the SYN scan (`-sS`). This SYN scan is set only to default when we run it as root because of the socket permissions required to create raw TCP packets. Otherwise, the TCP scan (`-sT`) is performed by default. This means that if we do not define ports and scanning methods, these parameters are set automatically.We can define the ports one by one (`-p 22,25,80,139,445`), by range (`-p 22-445`), by top ports (`--top-ports=10`) from the `Nmap` database that have been signed as most frequent, by scanning all ports (`-p-`) but also by defining a fast port scan, which contains top 100 ports (`-F`).

* * *

**Scanning top 10 TCP ports**:

    sudo nmap 10.129.2.28 --top-ports=10 

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `--top-ports=10` | Scans the specified top ports that have been defined as most frequent. |

We see that we only scanned the top 10 TCP ports of our target, and `Nmap` displays their state accordingly. If we trace the packets `Nmap` sends, we will see the `RST` flag on `TCP port 21` that our target sends back to us. To have a clear view of the SYN scan, we disable the ICMP echo requests (`-Pn`), DNS resolution (`-n`), and ARP ping scan (`--disable-arp-ping`).

* * *

**Nmap - Trace the Packets**:

    sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 21` | Scans only the specified port. |
| `--packet-trace` | Shows all packets sent and received. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |

* * *

**Connect Scan**:

The Nmap [TCP Connect Scan](https://nmap.org/book/scan-methods-connect-scan.html) (`-sT`) uses the TCP three-way handshake to determine if a specific port on a target host is open or closed. The scan sends an `SYN` packet to the target port and waits for a response. It is considered open if the target port responds with an `SYN-ACK` packet and closed if it responds with an `RST` packet.

The `Connect` scan (also known as a full TCP connect scan) is highly accurate because it completes the three-way TCP handshake, allowing us to determine the exact state of a port (open, closed, or filtered). However, it is not the most stealthy. In fact, the Connect scan is one of the least stealthy techniques, as it fully establishes a connection, which creates logs on most systems and is easily detected by modern IDS/IPS solutions. That said, the Connect scan can still be useful in certain situations, particularly when accuracy is a priority, and the goal is to map the network without causing significant disruption to services. Since the scan fully establishes a TCP connection, it interacts cleanly with services, making it less likely to cause service errors or instability compared to more intrusive scans. While it is not the most stealthy method, it is sometimes considered a more "polite" scan because it behaves like a normal client connection, thus having minimal impact on the target services.

Scans like the SYN scan (also known as a half-open scan) are generally considered more stealthy because they do not complete the full handshake, leaving the connection incomplete after sending the initial SYN packet. This minimizes the chance of triggering connection logs while still gathering port state information. Advanced IDS/IPS systems, however, have adapted to detect even these subtler techniques.

    sudo nmap 10.129.2.28 -p 443 --packet-trace --disable-arp-ping -Pn -n --reason -sT

* * *

**Filtered Ports:**

When a port is shown as filtered, it can have several reasons. In most cases, firewalls have certain rules set to handle specific connections. The packets can either be `dropped`, or `rejected`.When a packet gets dropped, `Nmap` receives no response from our target, and by default, the retry rate (`--max-retries`) is set to `10`. This means `Nmap` will resend the request to the target port to determine if the previous packet was accidentally mishandled or not.

Let us look at an example where the firewall `drops` the TCP packets we send for the port scan. Therefore we scan the TCP port **139**, which was already shown as filtered. To be able to track how our sent packets are handled, we deactivate the ICMP echo requests (`-Pn`), DNS resolution (`-n`), and ARP ping scan (`--disable-arp-ping`) again.

    sudo nmap 10.129.2.28 -p 139 --packet-trace -n --disable-arp-ping -Pn

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 139` | Scans only the specified port. |
| `--packet-trace` | Shows all packets sent and received. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `-Pn` | Disables ICMP Echo requests. |

We see in the last scan that `Nmap` sent two TCP packets with the SYN flag. By the duration (`2.06s`) of the scan, we can recognize that it took much longer than the previous ones (`~0.05s`).The case is different if the firewall rejects the packets. For this, we look at TCP port `445`, which is handled accordingly by such a rule of the firewall.

    sudo nmap 10.129.2.28 -p 445 --packet-trace -n --disable-arp-ping -Pn

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 445` | Scans only the specified port. |
| `--packet-trace` | Shows all packets sent and received. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `-Pn` | Disables ICMP Echo requests. |

As a response, we receive an `ICMP` reply with `type 3` and `error code 3`, which indicates that the desired port is unreachable. Nevertheless, if we know that the host is alive, we can strongly assume that the firewall on this port is rejecting the packets, and we will have to take a closer look at this port later.

* * *

**Discovering Open UDP Ports**:

Some system administrators sometimes forget to filter the UDP ports in addition to the TCP ones. Since `UDP` is a `stateless protocol` and does not require a three-way handshake like TCP. We do not receive any acknowledgment. Consequently, the timeout is much longer, making the whole `UDP scan` (`-sU`) much slower than the `TCP scan` (`-sS`).

Let's look at an example of what a UDP scan (`-sU`) can look like and what results it gives us.

    sudo nmap 10.129.2.28 -F -sU

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-F` | Scans top 100 ports. |
| `-sU` | Performs a UDP scan. |

Another disadvantage of this is that we often do not get a response back because `Nmap` sends empty datagrams to the scanned UDP ports, and we do not receive any response. So we cannot determine if the UDP packet has arrived at all or not. If the UDP port is `open`, we only get a response if the application is configured to do so.

    sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 137 --reason

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-sU` | Performs a UDP scan. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |
| `-p 137` | Scans only the specified port. |
| `--reason` | Displays the reason a port is in a particular state. |

If we get an ICMP response with `error code 3` (port unreachable), we know that the port is indeed `closed`.

    sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 100 --reason

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-sU` | Performs a UDP scan. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |
| `-p 100` | Scans only the specified port. |
| `--reason` | Displays the reason a port is in a particular state. |

For all other ICMP responses, the scanned ports are marked as (`open|filtered`).

    sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 138 --reason

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-sU` | Performs a UDP scan. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |
| `-p 138` | Scans only the specified port. |
| `--reason` | Displays the reason a port is in a particular state. |

Another handy method for scanning ports is the `-sV` option which is used to get additional available information from the open ports. This method can identify versions, service names, and details about our target.

    sudo nmap 10.129.2.28 -Pn -n --disable-arp-ping --packet-trace -p 445 --reason  -sV

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |
| `-p 445` | Scans only the specified port. |
| `--reason` | Displays the reason a port is in a particular state. |
| `-sV` | Performs a service scan. |

More information about port scanning techniques we can find at: [Port Scanning Techniques | Nmap Network Scanning](https://nmap.org/book/man-port-scanning-techniques.html)

* * *

Q1: Find all TCP ports on your target. Submit the total number of found TCP ports as the answer.

    nmap -Pn -n -p- -sS 10.129.2.49

Q2: Enumerate the hostname of your target and submit it as the answer. (case-sensitive)

    nmap -A -T4 -F 10.129.2.49

* * *

## 1.3 Saving the Results

**Different Formats:**

While we run various scans, we should always save the results. We can use these later to examine the differences between the different scanning methods we have used. `Nmap` can save the results in 3 different formats.

* Normal output (`-oN`) with the `.nmap` file extension
* Grepable output (`-oG`) with the `.gnmap` file extension
* XML output (`-oX`) with the `.xml` file extension

We can also specify the option (`-oA`) to save the results in all formats. The command could look like this:

    sudo nmap 10.129.2.28 -p- -oA target

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p-` | Scans all ports. |
| `-oA target` | Saves the results in all formats, starting the name of each file with 'target'. |

If no full path is given, the results will be stored in the directory we are currently in. Next, we look at the different formats `Nmap` has created for us.

* * *

**Stylesheets:**

With the XML output, we can easily create HTML reports that are easy to read, even for non-technical people. This is later very useful for documentation, as it presents our results in a detailed and clear way.To convert the stored results from XML format to HTML, we can use the tool `xsltproc`.

    xsltproc target.xml -o target.html

If we now open the HTML file in our browser, we see a clear and structured presentation of our results.

More information about the output formats can be found at: https://nmap.org/book/output.html

* * *

Q1: Perform a full TCP port scan on your target and create an HTML report. Submit the number of the highest port as the answer.

    nmap 10.129.64.71 -sS -p- -n -Pn -oA report
    
    xsltproc report.xml -o report.html
    
    ls
    cacert.der  Downloads  Public        report.nmap  Videos
    Desktop     Music      report.gnmap  report.xml
    Documents   Pictures   report.html   Templates

* * *

## 1.4 Service Enumeration

**Service Version Detection**:

It is recommended to perform a quick port scan first, which gives us a small overview of the available ports. This causes significantly less traffic, which is advantageous for us because otherwise we can be discovered and blocked by the security mechanisms. We can deal with these first and run a port scan in the background, which shows all open ports (`-p-`).We can use the version scan to scan the specific ports for services and their versions (`-sV`).

A full port scan takes quite a long time. To view the scan status, we can press the `[Space Bar]` during the scan, which will cause `Nmap` to show us the scan status.

    sudo nmap 10.129.2.28 -p- -sV

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p-` | Scans all ports. |
| `-sV` | Performs service version detection on specified ports. |

Another option (`--stats-every=5s`) that we can use is defining how periods of time the status should be shown. Here we can specify the number of seconds (`s`) or minutes (`m`), after which we want to get the status.

    sudo nmap 10.129.2.28 -p- -sV --stats-every=5s

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p-` | Scans all ports. |
| `-sV` | Performs service version detection on specified ports. |
| `--stats-every=5s` | Shows the progress of the scan every 5 seconds. |

We can also increase the `verbosity level` (`-v` / `-vv`), which will show us the open ports directly when `Nmap` detects them.

    sudo nmap 10.129.2.28 -p- -sV -v 

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p-` | Scans all ports. |
| `-sV` | Performs service version detection on specified ports. |
| `-v` | Increases the verbosity of the scan, which displays more detailed information. |

* * *

**Banner Grabbing:**

Once the scan is complete, we will see all TCP ports with the corresponding service and their versions that are active on the system.

    sudo nmap 10.129.2.28 -p- -sV

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p-` | Scans all ports. |
| `-sV` | Performs service version detection on specified ports. |

Primarily, `Nmap` looks at the banners of the scanned ports and prints them out. If it cannot identify versions through the banners, `Nmap` attempts to identify them through a signature-based matching system, but this significantly increases the scan's duration. One disadvantage to `Nmap`'s presented results is that the automatic scan can miss some information because sometimes `Nmap` does not know how to handle it. Let us look at an example of this.

    sudo nmap 10.129.2.28 -p- -sV -Pn -n --disable-arp-ping --packet-trace

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p-` | Scans all ports. |
| `-sV` | Performs service version detection on specified ports. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |

If we look at the results from `Nmap`, we can see the port's status, service name, and hostname. Nevertheless, let us look at this line here:

* `NSOCK INFO [0.4200s] nsock_trace_handler_callback(): Callback: READ SUCCESS for EID 18 [10.129.2.28:25] (35 bytes): 220 inlane ESMTP Postfix (Ubuntu)..`

Then we see that the SMTP server on our target gave us more information than `Nmap` showed us. Because here, we see that it is the Linux distribution `Ubuntu`. It happens because, after a successful three-way handshake, the server often sends a banner for identification. This serves to let the client know which service it is working with. At the network level, this happens with a `PSH` flag in the TCP header. However, it can happen that some services do not immediately provide such information. It is also possible to remove or manipulate the banners from the respective services. If we `manually` connect to the SMTP server using `nc`, grab the banner, and intercept the network traffic using `tcpdump`, we can see what `Nmap` did not show us.

* * *

_TCPDump:_

    NemBanTom@htb[/htb]$ sudo tcpdump -i eth0 host 10.10.14.2 and 10.129.2.28
    
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

_NC:_

    NemBanTom@htb[/htb]$  nc -nv 10.129.2.28 25
    
    Connection to 10.129.2.28 port 25 [tcp/*] succeeded!
    220 inlane ESMTP Postfix (Ubuntu)

_TCPDump - Intercepted traffic:_

    18:28:07.128564 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [S], seq 1798872233, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 331260178 ecr 0,sackOK,eol], length 0
    18:28:07.255151 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [S.], seq 1130574379, ack 1798872234, win 65160, options [mss 1460,sackOK,TS val 1800383922 ecr 331260178,nop,wscale 7], length 0
    18:28:07.255281 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], ack 1, win 2058, options [nop,nop,TS val 331260304 ecr 1800383922], length 0
    18:28:07.319306 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [P.], seq 1:36, ack 1, win 510, options [nop,nop,TS val 1800383985 ecr 331260304], length 35: SMTP: 220 inlane ESMTP Postfix (Ubuntu)
    18:28:07.319426 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], ack 36, win 2058, options [nop,nop,TS val 331260368 ecr 1800383985], length 0

The first three lines show us the three-way handshake.

|     |     |     |
| --- | --- | --- |
| 1.  | `[SYN]` | `18:28:07.128564 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [S], <SNIP>` |
| 2.  | `[SYN-ACK]` | `18:28:07.255151 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [S.], <SNIP>` |
| 3.  | `[ACK]` | `18:28:07.255281 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], <SNIP>` |

After that, the target SMTP server sends us a TCP packet with the `PSH` and `ACK` flags, where `PSH` states that the target server is sending data to us and with `ACK` simultaneously informs us that all required data has been sent.

|     |     |     |
| --- | --- | --- |
| 4.  | `[PSH-ACK]` | `18:28:07.319306 IP 10.129.2.28.smtp > 10.10.14.2.59618: Flags [P.], <SNIP>` |

The last TCP packet that we sent confirms the receipt of the data with an `ACK`.

|     |     |     |
| --- | --- | --- |
| 5.  | `[ACK]` | `18:28:07.319426 IP 10.10.14.2.59618 > 10.129.2.28.smtp: Flags [.], <SNIP>` |

* * *

Q1: Enumerate all ports and their services. One of the services contains the flag you have to submit as the answer.

    ┌─[eu-academy-4]─[10.10.14.184]─[htb-ac-2122384@htb-gysanbupsf]─[~]
    └──╼ [★]$ nmap 10.129.64.71 -sV -p-
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-21 14:11 CST
    Nmap scan report for 10.129.64.71
    Host is up (0.046s latency).
    Not shown: 65528 closed tcp ports (reset)
    PORT      STATE SERVICE     VERSION
    22/tcp    open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    80/tcp    open  http        Apache httpd 2.4.29 ((Ubuntu))
    110/tcp   open  pop3        Dovecot pop3d
    139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
    143/tcp   open  imap        Dovecot imapd (Ubuntu)
    445/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
    31337/tcp open  ftp         ProFTPD
    Service Info: Host: NIX-NMAP-DEFAULT; OS: Linux; CPE: cpe:/o:linux:linux_kernel
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 69.71 seconds
    
    
    ┌─[eu-academy-4]─[10.10.14.184]─[htb-ac-2122384@htb-gysanbupsf]─[~]
    └──╼ [★]$ nc 10.129.64.71 31337
    220 HTB{pr0F7pDv3r510nb4nn3r}

* * *

## 1.5 Nmap Scripting Engine

| **Category** | **Description** |
| --- | --- |
| `auth` | Determination of authentication credentials. |
| `broadcast` | Scripts, which are used for host discovery by broadcasting and the discovered hosts, can be automatically added to the remaining scans. |
| `brute` | Executes scripts that try to log in to the respective service by brute-forcing with credentials. |
| `default` | Default scripts executed by using the `-sC` option. |
| `discovery` | Evaluation of accessible services. |
| `dos` | These scripts are used to check services for denial of service vulnerabilities and are used less as it harms the services. |
| `exploit` | This category of scripts tries to exploit known vulnerabilities for the scanned port. |
| `external` | Scripts that use external services for further processing. |
| `fuzzer` | This uses scripts to identify vulnerabilities and unexpected packet handling by sending different fields, which can take much time. |
| `intrusive` | Intrusive scripts that could negatively affect the target system. |
| `malware` | Checks if some malware infects the target system. |
| `safe` | Defensive scripts that do not perform intrusive and destructive access. |
| `version` | Extension for service detection. |
| `vuln` | Identification of specific vulnerabilities |

* * *

**Default Scripts:**

    sudo nmap <target> -sC

**Specific Scripts Category:**

    sudo nmap <target> --script <category>

**Defined Scripts:**

    sudo nmap <target> --script <script-name>,<script-name>,...

For example, let us keep working with the target SMTP port and see the results we get with two defined scripts.

**Nmap - Specifying Scripts**:

    sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 25` | Scans only the specified port. |
| `--script banner,smtp-commands` | Uses specified NSE scripts. |

We see that we can recognize the **Ubuntu** distribution of Linux by using the' banner' script. The `smtp-commands` script shows us which commands we can use by interacting with the target SMTP server. In this example, such information may help us to find out existing users on the target. `Nmap` also gives us the ability to scan our target with the aggressive option (`-A`). This scans the target with multiple options as service detection (`-sV`), OS detection (`-O`), traceroute (`--traceroute`), and with the default NSE scripts (`-sC`).

**Nmap - Aggressive Scan:**

    sudo nmap 10.129.2.28 -p 80 -A

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 80` | Scans only the specified port. |
| `-A` | Performs service detection, OS detection, traceroute and uses defaults scripts to scan the target. |

**Vulnerability Assessment:**

    sudo nmap 10.129.2.28 -p 80 -sV --script vuln 

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 80` | Scans only the specified port. |
| `-sV` | Performs service version detection on specified ports. |
| `--script vuln` | Uses all related scripts from specified category. |

The scripts used for the last scan interact with the webserver and its web application to find out more information about their versions and check various databases to see if there are known vulnerabilities. More information about NSE scripts and the corresponding categories we can find at: [NSEDoc Reference Portal &mdash; Nmap Scripting Engine documentation](https://nmap.org/nsedoc/index.html)

Q1: Use NSE and its scripts to find the flag that one of the services contain and submit it as the answer.

    ┌─[eu-academy-4]─[10.10.14.34]─[htb-ac-2122384@htb-5qfwkp9hgu]─[~]
    └──╼ [★]$ nmap -sV -sC -p- -n 10.129.104.96
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-22 13:20 CST
    Stats: 0:01:04 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
    Service scan Timing: About 85.71% done; ETC: 13:22 (0:00:04 remaining)
    Stats: 0:01:38 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
    NSE Timing: About 99.28% done; ETC: 13:22 (0:00:00 remaining)
    Stats: 0:02:04 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
    NSE Timing: About 85.71% done; ETC: 13:23 (0:00:03 remaining)
    Nmap scan report for 10.129.104.96
    Host is up (0.052s latency).
    Not shown: 65528 closed tcp ports (reset)
    PORT      STATE SERVICE     VERSION
    22/tcp    open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    | ssh-hostkey: 
    |   2048 71:c1:89:90:7f:fd:4f:60:e0:54:f3:85:e6:35:6c:2b (RSA)
    |   256 e1:8e:53:18:42:af:2a:de:c0:12:1e:2e:54:06:4f:70 (ECDSA)
    |_  256 1a:cc:ac:d4:94:5c:d6:1d:71:e7:39:de:14:27:3c:3c (ED25519)
    80/tcp    open  http        Apache httpd 2.4.29 ((Ubuntu))
    |_http-server-header: Apache/2.4.29 (Ubuntu)
    |_http-title: Apache2 Ubuntu Default Page: It works
    110/tcp   open  pop3        Dovecot pop3d
    |_pop3-capabilities: CAPA SASL UIDL AUTH-RESP-CODE RESP-CODES PIPELINING TOP
    139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
    143/tcp   open  imap        Dovecot imapd (Ubuntu)
    |_imap-capabilities: IDLE IMAP4rev1 more have post-login Pre-login ENABLE LOGIN-REFERRALS OK listed SASL-IR LOGINDISABLEDA0001 capabilities LITERAL+ ID
    445/tcp   open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
    31337/tcp open  ftp         ProFTPD
    Service Info: Host: NIX-NMAP-DEFAULT; OS: Linux; CPE: cpe:/o:linux:linux_kernel
    
    Host script results:
    | smb-security-mode: 
    |   account_used: guest
    |   authentication_level: user
    |   challenge_response: supported
    |_  message_signing: disabled (dangerous, but default)
    | smb-os-discovery: 
    |   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
    |   Computer name: nix-nmap-default
    |   NetBIOS computer name: NIX-NMAP-DEFAULT\x00
    |   Domain name: \x00
    |   FQDN: nix-nmap-default
    |_  System time: 2025-12-22T20:22:32+01:00
    | smb2-security-mode: 
    |   3:1:1: 
    |_    Message signing enabled but not required
    |_nbstat: NetBIOS name: NIX-NMAP-DEFAUL, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
    | smb2-time: 
    |   date: 2025-12-22T19:22:32
    |_  start_date: N/A
    |_clock-skew: mean: -20m00s, deviation: 34m38s, median: 0s
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 127.68 seconds

A hint a webserverre mondja a taxot, szóval:

    ┌─[eu-academy-4]─[10.10.14.34]─[htb-ac-2122384@htb-5qfwkp9hgu]─[~]
    └──╼ [★]$ nmap -p 80 --script=vuln 10.129.104.96
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-22 13:55 CST
    Nmap scan report for 10.129.104.96
    Host is up (0.050s latency).
    
    PORT   STATE SERVICE
    80/tcp open  http
    |_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
    |_http-csrf: Couldn't find any CSRF vulnerabilities.
    | http-enum: 
    |_  /robots.txt: Robots file
    |_http-dombased-xss: Couldn't find any DOM based XSS.
    
    Nmap done: 1 IP address (1 host up) scanned in 31.27 seconds

A robots.txt-ben ott a flag.

* * *

## 1.6 Performance

Scanning performance plays a significant role when we need to scan an extensive network or are dealing with low network bandwidth. We can use various options to tell `Nmap` how fast (`-T <0-5>`), with which frequency (`--min-parallelism <number>`), which timeouts (`--max-rtt-timeout <time>`) the test packets should have, how many packets should be sent simultaneously (`--min-rate <number>`), and with the number of retries (`--max-retries <number>`) for the scanned ports the targets should be scanned.

**Timeouts:**

When Nmap sends a packet, it takes some time (`Round-Trip-Time` - `RTT`) to receive a response from the scanned port. Generally, `Nmap` starts with a high timeout (`--min-RTT-timeout`) of 100ms. Let us look at an example by scanning the whole network with 256 hosts, including the top 100 ports.

    sudo nmap 10.129.2.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.0/24` | Scans the specified target network. |
| `-F` | Scans top 100 ports. |
| `--initial-rtt-timeout 50ms` | Sets the specified time value as initial RTT timeout. |
| `--max-rtt-timeout 100ms` | Sets the specified time value as maximum RTT timeout. |

When comparing the two scans, we can see that we found two hosts less with the optimized scan, but the scan took only a quarter of the time. From this, we can conclude that setting the initial RTT timeout (`--initial-rtt-timeout`) to too short a time period may cause us to overlook hosts.

* * *

**Max Retries:**

Another way to increase scan speed is by specifying the retry rate of sent packets (`--max-retries`). The default value is `10`, but we can reduce it to `0`. This means if Nmap does not receive a response for a port, it won't send any more packets to that port and will skip it.

    sudo nmap 10.129.2.0/24 -F --max-retries 0 | grep "/tcp" | wc -l

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.0/24` | Scans the specified target network. |
| `-F` | Scans top 100 ports. |
| `--max-retries 0` | Sets the number of retries that will be performed during the scan. |

Again, we recognize that accelerating can also have a negative effect on our results, which means we can overlook important information.

* * *

**Rates:**

During a white-box penetration test, we may get whitelisted for the security systems to check the systems in the network for vulnerabilities and not only test the protection measures. If we know the network bandwidth, we can work with the rate of packets sent, which significantly speeds up our scans with `Nmap`. When setting the minimum rate (`--min-rate <number>`) for sending packets, we tell `Nmap` to simultaneously send the specified number of packets. It will attempt to maintain the rate accordingly.

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.default
    
    <SNIP>
    Nmap done: 256 IP addresses (10 hosts up) scanned in 29.83 seconds

optimized scan:

    sudo nmap 10.129.2.0/24 -F -oN tnet.minrate300 --min-rate 300

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.0/24` | Scans the specified target network. |
| `-F` | Scans top 100 ports. |
| `-oN tnet.minrate300` | Saves the results in normal formats, starting the specified file name. |
| `--min-rate 300` | Sets the minimum number of packets to be sent per second. |

* * *

**Timing:**

Because such settings cannot always be optimized manually, as in a black-box penetration test, `Nmap` offers six different timing templates (`-T <0-5>`) for us to use. These values (`0-5`) determine the aggressiveness of our scans. This can also have negative effects if the scan is too aggressive, and security systems may block us due to the produced network traffic. The default timing template used when we have defined nothing else is the normal (`-T 3`).

* `-T 0` / `-T paranoid`
* `-T 1` / `-T sneaky`
* `-T 2` / `-T polite`
* `-T 3` / `-T normal`
* `-T 4` / `-T aggressive`
* `-T 5` / `-T insane`

These templates contain options that we can also set manually, and have seen some of them already. The developers determined the values set for these templates according to their best results, making it easier for us to adapt our scans to the corresponding network environment. The exact used options with their values we can find here: https://nmap.org/book/performance-timing-templates.html

Default scan:

    sudo nmap 10.129.2.0/24 -F -oN tnet.default 

Insane scan:

    sudo nmap 10.129.2.0/24 -F -oN tnet.T5 -T 5

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.0/24` | Scans the specified target network. |
| `-F` | Scans top 100 ports. |
| `-oN tnet.T5` | Saves the results in normal formats, starting the specified file name. |
| `-T 5` | Specifies the insane timing template. |

More information about scan performance we can find at https://nmap.org/book/man-performance.html

* * *

## 1.7 Firewall and IDS/IPS Evasion

**Determine Firewalls and Their Rules:**

We already know that when a port is shown as filtered, it can have several reasons. In most cases, firewalls have certain rules set to handle specific connections. The packets can either be `dropped`, or `rejected`. The `dropped` packets are ignored, and no response is returned from the host.

This is different for rejected packets, which elicit an explicit response. TCP packets are returned with an RST flag, while ICMP can contain different types of error codes.

Such errors include:

* Net Unreachable
* Net Prohibited
* Host Unreachable
* Host Prohibited
* Port Unreachable
* Proto Unreachable

Nmap's TCP ACK scan (`-sA`) method is much harder to filter for firewalls and IDS/IPS systems than regular SYN (`-sS`) or Connect scans (`sT`) because they only send a TCP packet with only the `ACK` flag. When a port is closed or open, the host must respond with an `RST` flag.Unlike outgoing connections, all connection attempts (with the `SYN` flag) from external networks are usually blocked by firewalls. However, the packets with the `ACK` flag are often passed by the firewall because the firewall cannot determine whether the connection was first established from the external network or the internal network.

If we look at these scans, we will see how the results differ.

**SYN Scan:**

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace
    
    Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 14:56 CEST
    SENT (0.0278s) TCP 10.10.14.2:57347 > 10.129.2.28:22 S ttl=53 id=22412 iplen=44  seq=4092255222 win=1024 <mss 1460>
    SENT (0.0278s) TCP 10.10.14.2:57347 > 10.129.2.28:25 S ttl=50 id=62291 iplen=44  seq=4092255222 win=1024 <mss 1460>
    SENT (0.0278s) TCP 10.10.14.2:57347 > 10.129.2.28:21 S ttl=58 id=38696 iplen=44  seq=4092255222 win=1024 <mss 1460>
    RCVD (0.0329s) ICMP [10.129.2.28 > 10.10.14.2 Port 21 unreachable (type=3/code=3) ] IP [ttl=64 id=40884 iplen=72 ]
    RCVD (0.0341s) TCP 10.129.2.28:22 > 10.10.14.2:57347 SA ttl=64 id=0 iplen=44  seq=1153454414 win=64240 <mss 1460>
    RCVD (1.0386s) TCP 10.129.2.28:22 > 10.10.14.2:57347 SA ttl=64 id=0 iplen=44  seq=1153454414 win=64240 <mss 1460>
    SENT (1.1366s) TCP 10.10.14.2:57348 > 10.129.2.28:25 S ttl=44 id=6796 iplen=44  seq=4092320759 win=1024 <mss 1460>
    Nmap scan report for 10.129.2.28
    Host is up (0.0053s latency).
    
    PORT   STATE    SERVICE
    21/tcp filtered ftp
    22/tcp open     ssh
    25/tcp filtered smtp
    MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
    
    Nmap done: 1 IP address (1 host up) scanned in 0.07 seconds

**ACK Scan:**

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
    
    Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 14:57 CEST
    SENT (0.0422s) TCP 10.10.14.2:49343 > 10.129.2.28:21 A ttl=49 id=12381 iplen=40  seq=0 win=1024
    SENT (0.0423s) TCP 10.10.14.2:49343 > 10.129.2.28:22 A ttl=41 id=5146 iplen=40  seq=0 win=1024
    SENT (0.0423s) TCP 10.10.14.2:49343 > 10.129.2.28:25 A ttl=49 id=5800 iplen=40  seq=0 win=1024
    RCVD (0.1252s) ICMP [10.129.2.28 > 10.10.14.2 Port 21 unreachable (type=3/code=3) ] IP [ttl=64 id=55628 iplen=68 ]
    RCVD (0.1268s) TCP 10.129.2.28:22 > 10.10.14.2:49343 R ttl=64 id=0 iplen=40  seq=1660784500 win=0
    SENT (1.3837s) TCP 10.10.14.2:49344 > 10.129.2.28:25 A ttl=59 id=21915 iplen=40  seq=0 win=1024
    Nmap scan report for 10.129.2.28
    Host is up (0.083s latency).
    
    PORT   STATE      SERVICE
    21/tcp filtered   ftp
    22/tcp unfiltered ssh
    25/tcp filtered   smtp
    MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
    
    Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 21,22,25` | Scans only the specified ports. |
| `-sS` | Performs SYN scan on specified ports. |
| `-sA` | Performs ACK scan on specified ports. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |

Please pay attention to the RCVD packets and its set flag we receive from our target. With the SYN scan (`-sS`) our target tries to establish the TCP connection by sending a packet back with the SYN-ACK (`SA`) flags set and with the ACK scan (`-sA`) we get the `RST` flag because TCP port 22 is open. For the TCP port 25, we do not receive any packets back, which indicates that the packets will be dropped.

**Detect IDS/IPS:**

Unlike firewalls and their rules, the detection of IDS/IPS systems is much more difficult because these are passive traffic monitoring systems. `IDS systems` examine all connections between hosts. If the IDS finds packets containing the defined contents or specifications, the administrator is notified and takes appropriate action in the worst case.

`IPS systems` take measures configured by the administrator independently to prevent potential attacks automatically. It is essential to know that IDS and IPS are different applications and that IPS serves as a complement to IDS.

Several virtual private servers (`VPS`) with different IP addresses are recommended to determine whether such systems are on the target network during a penetration test. If the administrator detects such a potential attack on the target network, the first step is to block the IP address from which the potential attack comes.As a result, we will no longer be able to access the network using that IP address, and our Internet Service Provider (`ISP`) will be contacted and blocked from all access to the Internet.

* `IDS systems` alone are usually there to help administrators detect potential attacks on their network. They can then decide how to handle such connections. We can trigger certain security measures from an administrator, for example, by aggressively scanning a single port and its service. Based on whether specific security measures are taken, we can detect if the network has some monitoring applications or not.
  
* One method to determine whether such `IPS system` is present in the target network is to scan from a single host (`VPS`). If at any time this host is blocked and has no access to the target network, we know that the administrator has taken some security measures. Accordingly, we can continue our penetration test with another `VPS`.
  

Consequently, we know that we need to be quieter with our scans and, in the best case, disguise all interactions with the target network and its services.

**Decoys:**

There are cases in which administrators block specific subnets from different regions in principle. This prevents any access to the target network. Another example is when IPS should block us. For this reason, the Decoy scanning method (`-D`) is the right choice. With this method, Nmap generates various random IP addresses inserted into the IP header to disguise the origin of the packet sent.With this method, we can generate random (`RND`) a specific number (for example: `5`) of IP addresses separated by a colon (`:`). Our real IP address is then randomly placed between the generated IP addresses. In the next example, our real IP address is therefore placed in the second position. Another critical point is that the decoys must be alive. Otherwise, the service on the target may be unreachable due to SYN-flooding security mechanisms.

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5
    
    Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 16:14 CEST
    SENT (0.0378s) TCP 102.52.161.59:59289 > 10.129.2.28:80 S ttl=42 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
    SENT (0.0378s) TCP 10.10.14.2:59289 > 10.129.2.28:80 S ttl=59 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
    SENT (0.0379s) TCP 210.120.38.29:59289 > 10.129.2.28:80 S ttl=37 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
    SENT (0.0379s) TCP 191.6.64.171:59289 > 10.129.2.28:80 S ttl=38 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
    SENT (0.0379s) TCP 184.178.194.209:59289 > 10.129.2.28:80 S ttl=39 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
    SENT (0.0379s) TCP 43.21.121.33:59289 > 10.129.2.28:80 S ttl=55 id=29822 iplen=44  seq=3687542010 win=1024 <mss 1460>
    RCVD (0.1370s) TCP 10.129.2.28:80 > 10.10.14.2:59289 SA ttl=64 id=0 iplen=44  seq=4056111701 win=64240 <mss 1460>
    Nmap scan report for 10.129.2.28
    Host is up (0.099s latency).
    
    PORT   STATE SERVICE
    80/tcp open  http
    MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
    
    Nmap done: 1 IP address (1 host up) scanned in 0.15 seconds

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 80` | Scans only the specified ports. |
| `-sS` | Performs SYN scan on specified ports. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |
| `-D RND:5` | Generates five random IP addresses that indicates the source IP the connection comes from. |

The spoofed packets are often filtered out by ISPs and routers, even though they come from the same network range. Therefore, we can also specify our VPS servers' IP addresses and use them in combination with "`IP ID`" manipulation in the IP headers to scan the target.

Another scenario would be that only individual subnets would not have access to the server's specific services. So we can also manually specify the source IP address (`-S`) to test if we get better results with this one. Decoys can be used for SYN, ACK, ICMP scans, and OS detection scans. So let us look at such an example and determine which operating system it is most likely to be.

**Testing Firewall Rule:**

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -n -Pn -p445 -O
    
    Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-22 01:23 CEST
    Nmap scan report for 10.129.2.28
    Host is up (0.032s latency).
    
    PORT    STATE    SERVICE
    445/tcp filtered microsoft-ds
    MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
    Too many fingerprints match this host to give specific OS details
    Network Distance: 1 hop
    
    OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 3.14 seconds

**Scan by Using Different Source IP:**

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -n -Pn -p 445 -O -S 10.129.2.200 -e tun0
    
    Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-22 01:16 CEST
    Nmap scan report for 10.129.2.28
    Host is up (0.010s latency).
    
    PORT    STATE SERVICE
    445/tcp open  microsoft-ds
    MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
    Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
    Aggressive OS guesses: Linux 2.6.32 (96%), Linux 3.2 - 4.9 (96%), Linux 2.6.32 - 3.10 (96%), Linux 3.4 - 3.10 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (94%), Synology DiskStation Manager 5.2-5644 (94%), Linux 2.6.32 - 2.6.35 (94%), Linux 2.6.32 - 3.5 (94%)
    No exact OS matches for host (test conditions non-ideal).
    Network Distance: 1 hop
    
    OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 4.11 seconds

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-n` | Disables DNS resolution. |
| `-Pn` | Disables ICMP Echo requests. |
| `-p 445` | Scans only the specified ports. |
| `-O` | Performs operation system detection scan. |
| `-S` | Scans the target by using different source IP address. |
| `10.129.2.200` | Specifies the source IP address. |
| `-e tun0` | Sends all requests through the specified interface. |

**DNS Proxying:**

By default, `Nmap` performs a reverse DNS resolution unless otherwise specified to find more important information about our target. These DNS queries are also passed in most cases because the given web server is supposed to be found and visited. The DNS queries are made over the `UDP port 53`. The `TCP port 53` was previously only used for the so-called "`Zone transfers`" between the DNS servers or data transfer larger than 512 bytes. More and more, this is changing due to IPv6 and DNSSEC expansions. These changes cause many DNS requests to be made via TCP port 53.

However, `Nmap` still gives us a way to specify DNS servers ourselves (`--dns-server <ns>,<ns>`). This method could be fundamental to us if we are in a demilitarized zone (`DMZ`). The company's DNS servers are usually more trusted than those from the Internet. So, for example, we could use them to interact with the hosts of the internal network. As another example, we can use `TCP port 53` as a source port (`--source-port`) for our scans. If the administrator uses the firewall to control this port and does not filter IDS/IPS properly, our TCP packets will be trusted and passed through.

**SYN-Scan of a Filtered Port:**

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace
    
    Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-21 22:50 CEST
    SENT (0.0417s) TCP 10.10.14.2:33436 > 10.129.2.28:50000 S ttl=41 id=21939 iplen=44  seq=736533153 win=1024 <mss 1460>
    SENT (1.0481s) TCP 10.10.14.2:33437 > 10.129.2.28:50000 S ttl=46 id=6446 iplen=44  seq=736598688 win=1024 <mss 1460>
    Nmap scan report for 10.129.2.28
    Host is up.
    
    PORT      STATE    SERVICE
    50000/tcp filtered ibm-db2
    
    Nmap done: 1 IP address (1 host up) scanned in 2.06 seconds

**SYN-Scan From DNS Port:**

    NemBanTom@htb[/htb]$ sudo nmap 10.129.2.28 -p50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53
    
    SENT (0.0482s) TCP 10.10.14.2:53 > 10.129.2.28:50000 S ttl=58 id=27470 iplen=44  seq=4003923435 win=1024 <mss 1460>
    RCVD (0.0608s) TCP 10.129.2.28:50000 > 10.10.14.2:53 SA ttl=64 id=0 iplen=44  seq=540635485 win=64240 <mss 1460>
    Nmap scan report for 10.129.2.28
    Host is up (0.013s latency).
    
    PORT      STATE SERVICE
    50000/tcp open  ibm-db2
    MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)
    
    Nmap done: 1 IP address (1 host up) scanned in 0.08 seconds

| **Scanning Options** | **Description** |
| --- | --- |
| `10.129.2.28` | Scans the specified target. |
| `-p 50000` | Scans only the specified ports. |
| `-sS` | Performs SYN scan on specified ports. |
| `-Pn` | Disables ICMP Echo requests. |
| `-n` | Disables DNS resolution. |
| `--disable-arp-ping` | Disables ARP ping. |
| `--packet-trace` | Shows all packets sent and received. |
| `--source-port 53` | Performs the scans from specified source port. |

Now that we have found out that the firewall accepts `TCP port 53`, it is very likely that IDS/IPS filters might also be configured much weaker than others. We can test this by trying to connect to this port by using `Netcat`.

    NemBanTom@htb[/htb]$ ncat -nv --source-port 53 10.129.2.28 50000
    
    Ncat: Version 7.80 ( https://nmap.org/ncat )
    Ncat: Connected to 10.129.2.28:50000.
    220 ProFTPd

* * *

**Firewall and IDS/IPS Evasion Labs:**

In the next three sections, we get different scenarios to practice where we have to scan our target. Firewall rules and IDS/IPS protect the systems, so we need to use the techniques shown to bypass the firewall rules and do this as quiet as possible. Otherwise, we will be blocked by IPS.

**Easly Lab:**

Now let's get practical. A company hired us to test their IT security defenses, including their `IDS` and `IPS` systems. Our client wants to increase their IT security and will, therefore, make specific improvements to their `IDS/IPS` systems after each successful test. We do not know, however, according to which guidelines these changes will be made. Our goal is to find out specific information from the given situations.

We are only ever provided with a machine protected by `IDS/IPS` systems and can be tested. For learning purposes and to get a feel for how `IDS/IPS` can behave, we have access to a status web page at:

Our client wants to know if we can identify which operating system their provided machine is running on. Submit the OS name as the answer.

Method1:

    ┌─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ nmap -sS -O -T3 --disable-arp-ping -Pn 10.129.159.60
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-25 04:06 CST
    Nmap scan report for 10.129.159.60
    Host is up (0.050s latency).
    Not shown: 869 closed tcp ports (reset), 128 filtered tcp ports (no-response)
    PORT      STATE SERVICE
    22/tcp    open  ssh
    80/tcp    open  http
    10001/tcp open  scp-config
    No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
    TCP/IP fingerprint:
    OS:SCAN(V=7.94SVN%E=4%D=12/25%OT=22%CT=1%CU=43228%PV=Y%DS=2%DC=I%G=Y%TM=694
    OS:D0C9B%P=x86_64-pc-linux-gnu)SEQ(SP=104%GCD=1%ISR=10E%TI=Z%CI=Z%II=I%TS=A
    OS:)SEQ(SP=104%GCD=3%ISR=10E%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552
    OS:ST11NW7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1
    OS:=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O
    OS:=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N
    OS:)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=
    OS:S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=N)U1
    OS:(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI
    OS:=N%T=40%CD=S)
    
    Network Distance: 2 hops
    
    OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 13.95 seconds

method 2(ttl):

    ┌─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ nmap -sn -PE --disable-arp-ping --packet-trace 10.129.159.60
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-25 04:09 CST
    SENT (0.0340s) ICMP [10.10.14.97 > 10.129.159.60 Echo request (type=8/code=0) id=62710 seq=0] IP [ttl=57 id=58443 iplen=28 ]
    RCVD (0.0843s) ICMP [10.129.159.60 > 10.10.14.97 Echo reply (type=0/code=0) id=62710 seq=0] IP [ttl=63 id=45209 iplen=28 ]
    NSOCK INFO [0.1270s] nsock_iod_new2(): nsock_iod_new (IOD #1)
    NSOCK INFO [0.1270s] nsock_connect_udp(): UDP connection requested to 8.8.8.8:53 (IOD #1) EID 8
    NSOCK INFO [0.1270s] nsock_read(): Read request from IOD #1 [8.8.8.8:53] (timeout: -1ms) EID 18
    NSOCK INFO [0.1270s] nsock_iod_new2(): nsock_iod_new (IOD #2)
    NSOCK INFO [0.1270s] nsock_connect_udp(): UDP connection requested to 1.1.1.1:53 (IOD #2) EID 24
    NSOCK INFO [0.1270s] nsock_read(): Read request from IOD #2 [1.1.1.1:53] (timeout: -1ms) EID 34
    NSOCK INFO [0.1270s] nsock_write(): Write request for 44 bytes to IOD #1 EID 43 [8.8.8.8:53]
    NSOCK INFO [0.1270s] nsock_trace_handler_callback(): Callback: CONNECT SUCCESS for EID 8 [8.8.8.8:53]
    NSOCK INFO [0.1270s] nsock_trace_handler_callback(): Callback: WRITE SUCCESS for EID 43 [8.8.8.8:53]
    NSOCK INFO [0.1270s] nsock_trace_handler_callback(): Callback: CONNECT SUCCESS for EID 24 [1.1.1.1:53]
    NSOCK INFO [0.1280s] nsock_trace_handler_callback(): Callback: READ SUCCESS for EID 18 [8.8.8.8:53] (44 bytes): .............60.159.129.10.in-addr.arpa.....
    NSOCK INFO [0.1280s] nsock_read(): Read request from IOD #1 [8.8.8.8:53] (timeout: -1ms) EID 50
    NSOCK INFO [0.1280s] nsock_iod_delete(): nsock_iod_delete (IOD #1)
    NSOCK INFO [0.1280s] nevent_delete(): nevent_delete on event #50 (type READ)
    NSOCK INFO [0.1280s] nsock_iod_delete(): nsock_iod_delete (IOD #2)
    NSOCK INFO [0.1280s] nevent_delete(): nevent_delete on event #34 (type READ)
    Nmap scan report for 10.129.159.60
    Host is up (0.050s latency).
    Nmap done: 1 IP address (1 host up) scanned in 0.13 seconds

64==linux

Method 3 (version scan):

    ┌─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ nmap -p22,80,10001 -sV --disable-arp-ping -Pn 10.129.159.60
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-25 04:12 CST
    Stats: 0:02:05 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
    Service scan Timing: About 66.67% done; ETC: 04:15 (0:01:03 remaining)
    Nmap scan report for 10.129.159.60
    Host is up (0.050s latency).
    
    PORT      STATE SERVICE     VERSION
    22/tcp    open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    80/tcp    open  http        Apache httpd 2.4.29 ((Ubuntu))
    10001/tcp open  scp-config?
    1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
    SF-Port10001-TCP:V=7.94SVN%I=7%D=12/25%Time=694D0E16%P=x86_64-pc-linux-gnu
    SF:%r(NULL,1F,"220\x20HTB{pr0F7pDv3r510nb4nn3r}\r\n")%r(GetRequest,1F,"220
    SF:\x20HTB{pr0F7pDv3r510nb4nn3r}\r\n")%r(SIPOptions,1F,"220\x20HTB{pr0F7pD
    SF:v3r510nb4nn3r}\r\n");
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 167.42 seconds

Még ez is egy jó megoldás:

    sudo nmap -sV 10.129.22.245 -Pn -p80

sudo = default -sS scan

* * *

**Medium Lab**

After we conducted the first test and submitted our results to our client, the administrators made some changes and improvements to the `IDS/IPS` and firewall. We could hear that the administrators were not satisfied with their previous configurations during the meeting, and they could see that the network traffic could be filtered more strictly.

After the configurations are transferred to the system, our client wants to know if it is possible to find out our target's DNS server version. Submit the DNS server version of the target as the answer.

    ┌─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ sudo nmap -p53 -sUV -n --disable-arp-ping -Pn 10.129.244.182
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-25 04:40 CST
    Nmap scan report for 10.129.244.182
    Host is up (0.051s latency).
    
    PORT   STATE SERVICE VERSION
    53/udp open  domain  (unknown banner: HTB{GoTtgUnyze9Psw4vGjcuMpHRp})
    1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
    SF-Port53-UDP:V=7.94SVN%I=7%D=12/25%Time=694D14C0%P=x86_64-pc-linux-gnu%r(
    SF:DNSVersionBindReq,57,"\0\x06\x85\0\0\x01\0\x01\0\x01\0\0\x07version\x04
    SF:bind\0\0\x10\0\x03\xc0\x0c\0\x10\0\x03\0\0\0\0\0\x1f\x1eHTB{GoTtgUnyze9
    SF:Psw4vGjcuMpHRp}\xc0\x0c\0\x02\0\x03\0\0\0\0\0\x02\xc0\x0c")%r(DNSStatus
    SF:Request,C,"\0\0\x90\x04\0\0\0\0\0\0\0\0")%r(NBTStat,105,"\x80\xf0\x80\x
    SF:90\0\x01\0\0\0\r\0\0\x20CKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\0\0!\0\x01\0\0
    SF:\x02\0\x01\x006\xee\x80\0\x14\x01B\x0cROOT-SERVERS\x03NET\0\0\0\x02\0\x
    SF:01\x006\xee\x80\0\x04\x01C\xc0\?\0\0\x02\0\x01\x006\xee\x80\0\x04\x01F\
    SF:xc0\?\0\0\x02\0\x01\x006\xee\x80\0\x04\x01D\xc0\?\0\0\x02\0\x01\x006\xe
    SF:e\x80\0\x04\x01A\xc0\?\0\0\x02\0\x01\x006\xee\x80\0\x04\x01E\xc0\?\0\0\
    SF:x02\0\x01\x006\xee\x80\0\x04\x01G\xc0\?\0\0\x02\0\x01\x006\xee\x80\0\x0
    SF:4\x01K\xc0\?\0\0\x02\0\x01\x006\xee\x80\0\x04\x01I\xc0\?\0\0\x02\0\x01\
    SF:x006\xee\x80\0\x04\x01H\xc0\?\0\0\x02\0\x01\x006\xee\x80\0\x04\x01J\xc0
    SF:\?\0\0\x02\0\x01\x006\xee\x80\0\x04\x01L\xc0\?\0\0\x02\0\x01\x006\xee\x
    SF:80\0\x04\x01M\xc0\?");
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 15.29 seconds

* * *

**Hard Lab:**

With our second test's help, our client was able to gain new insights and sent one of its administrators to a `training course` for `IDS/IPS` systems. As our client told us, the training would last `one week`. Now the administrator has taken all the necessary precautions and wants us to test this again because specific services must be changed, and the communication for the provided software had to be modified.

Now our client wants to know if it is possible to find out the version of the running services. Identify the version of service our client was talking about and submit the flag as the answer.

    ┌─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ sudo nmap -sV -Pn --source-port 53 --disable-arp-ping 10.129.2.47
    Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-25 04:53 CST
    Nmap scan report for 10.129.2.47
    Host is up (0.050s latency).
    Not shown: 869 closed tcp ports (reset), 128 filtered tcp ports (no-response)
    PORT      STATE SERVICE    VERSION
    22/tcp    open  ssh        OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
    80/tcp    open  http       Apache httpd 2.4.29 ((Ubuntu))
    50000/tcp open  tcpwrapped
    Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
    
    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 38.75 seconds

netcat:

    ┌─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ sudo netcat -nv -p 53 10.129.2.47 50000
    (UNKNOWN) [10.129.2.47] 50000 (?) open
    220 HTB{kjnsdf2n982n1827eh76238s98di1w6}

vagy ncat:

    ─[eu-academy-4]─[10.10.14.97]─[htb-ac-2122384@htb-obeondqgy8]─[~]
    └──╼ [★]$ sudo ncat -nv --source-port 53 10.129.2.47 50000
    Ncat: Version 7.94SVN ( https://nmap.org/ncat )
    Ncat: Connected to 10.129.2.47:50000.
    220 HTB{kjnsdf2n982n1827eh76238s98di1w6}

---END---
