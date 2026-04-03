# Chapters 1–3: Linux Fundamentals & Networking

## Chapter 1: Getting Started with the Basics

### Finding Things

- **locate** — searches a pre-built database for files by name. Fast, but the database only updates once a day automatically (see `updatedb`).
- **updatedb** — manually rebuilds the `locate` database. Run this if you need `locate` to find recently created files.
- **whereis** — locates binaries, source files, and man pages for a command.
- **which** — returns the path to the binary that would be executed, searching only directories listed in `$PATH`.
- **find** — the most powerful search tool; walks the directory tree in real time.
  - Usage: `find directory options expression`
  - Example: `find / -type f -name apache2`
- **grep** — filters input or file contents by keyword/pattern (often piped with other commands).

### Creating & Appending Files

- **cat > new\_file** — opens a text prompt; what you type becomes the content of `new_file` (Ctrl+D to save).
- **cat >> existing\_file** — appends typed text to an existing file.
- **touch filename** — creates a new empty file (or updates the timestamp of an existing one).

---

## Chapter 2: Text Manipulation

- **head** — displays the first *n* lines of a file (default 10).
- **tail** — displays the last *n* lines of a file (default 10).
- **nl** — displays file content with line numbers.
- **sed** (Stream EDitor) — find and replace text in a stream or file.
  - Example: `sed s/old/new/g filename`
- **more** — like `cat`, but paginates output — press Enter to advance one line, Space for one page.
- **less** — like `more`, but allows backward scrolling and searching within the file (`/pattern`).

---

## Chapter 3: Analyzing and Managing Networks

### Key Networking Concepts

| Term                  | What It Is                                                                                                                                                                                               | Why We Use It                                                                                                                                                                                       |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **IP address**        | A unique numerical label assigned to every device on a network (e.g. `192.168.1.10`).                                                                                                                    | So devices can find and talk to each other on a network.                                                                                                                                            |
| **MAC (HWaddr)**      | A hardware address burned into every network interface card (e.g. `00:11:22:33:44:55`). Works at the data-link layer (Layer 2).                                                                          | Identifies a physical device on a local network segment. Unlike IP, it doesn't change with network configuration (unless spoofed).                                                                  |
| **Netmask**           | A bitmask that splits an IP address into a *network* portion and a *host* portion (e.g. `255.255.255.0`).                                                                                                | Tells the OS which part of the IP identifies the local network vs. individual hosts. Two devices with the same network portion (after applying the mask) can communicate directly without a router. |
| **Broadcast (Bcast)** | A special address that targets *all* devices on a subnet (e.g. `192.168.1.255`).                                                                                                                         | Used for protocols that need to reach everyone at once — like ARP ("who has this IP?") and DHCP discovery.                                                                                          |
| **DHCP**              | Dynamic Host Configuration Protocol — a service that automatically assigns IP addresses, netmasks, gateways, and DNS servers to devices when they join a network.                                        | Without DHCP you'd have to configure every device's network settings manually. Your home router almost certainly runs a DHCP server.                                                                |
| **Daemon**            | A background process that runs continuously and provides a service (e.g. `dhcpd` is the DHCP daemon, `sshd` is the SSH daemon). Named after Greek daemons — helpful spirits that work behind the scenes. | Daemons handle requests silently so services like networking, web serving, and logging "just work" without user interaction.                                                                        |
| **DNS**               | Domain Name System — translates human-readable names (`google.com`) into IP addresses (`142.250.x.x`).                                                                                                   | So you don't have to memorize IP addresses. Your system checks `/etc/resolv.conf` to know which DNS server to ask.                                                                                  |
| **Loopback (lo)**     | A virtual network interface that routes traffic back to the same machine (`127.0.0.1` / `localhost`).                                                                                                    | Lets programs on the same machine communicate over the network stack without hitting any physical network. Useful for local development and testing.                                                |

### Querying Network Interfaces

- **ifconfig** — displays or configures network interfaces.
  - **eth0** — first wired Ethernet connection (additional ones: `eth1`, `eth2`, …).
  - **wlan0** — first wireless interface (has its own MAC address).
  - **lo** — loopback interface.
- **iwconfig** — like `ifconfig`, but specifically for wireless interfaces (shows SSID, frequency, signal strength, etc.).

### Monitoring Connections

- **netstat** — displays network connections, routing tables, and interface statistics.
  - `netstat -a` — shows all active connections (listening + established).
- **ss** — modern replacement for `netstat`; faster and more detailed output.

### Changing Network Information

- **Change IP:**
- **Change netmask and broadcast:**
- **Spoof MAC address** (take interface down, change, bring back up):
- **Request a new IP from DHCP:**&#x54;his tells the DHCP daemon on the network to assign a fresh IP to the interface. Useful after spoofing a MAC or when troubleshooting connectivity.

### Manipulating DNS

- **dig** — queries DNS servers for detailed info about a domain (records, TTL, authoritative servers, etc.).
- **Changing your DNS server:** edit `/etc/resolv.conf` and set the `nameserver` line (e.g. `nameserver 8.8.8.8`).
- **/etc/hosts** — a local file that maps hostnames to IPs *before* DNS is consulted. Useful for overrides and local development.
