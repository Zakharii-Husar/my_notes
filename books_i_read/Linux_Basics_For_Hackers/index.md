# LINUX BASICS FOR HACKERS

## Getting Started with Networking, Scripting, and Security in Kali Linux

*By OccupyTheWeb*



# Book Notes — Linux Basics for Hackers

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

---

## Chapter 4: Adding and Removing Software

Linux distributions use **package managers** to install, update, and remove software. Debian-based distros (Kali, Ubuntu) use `apt`.

### Core Commands

| Command               | What It Does                                                                                                                                                                 |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `apt search keyword`  | Searches the package index for packages matching the keyword.                                                                                                                |
| `apt install package` | Downloads and installs a package (plus its dependencies).                                                                                                                    |
| `apt remove package`  | Removes the package binary but **keeps** its configuration files. Useful if you plan to reinstall later and want your settings intact.                                       |
| `apt purge package`   | Removes the package **and** its configuration files. A clean wipe.                                                                                                           |
| `apt update`          | Refreshes the local list of available packages from all configured repositories. Does **not** install anything — just updates the catalog. Always run this before `upgrade`. |
| `apt upgrade`         | Downloads and installs newer versions of all currently installed packages (based on the catalog from `update`).                                                              |
| `apt autoremove`      | Removes packages that were installed as dependencies but are no longer needed by anything.                                                                                   |

### Repositories and `/etc/apt/sources.list`

- **Repositories** are remote servers that host `.deb` packages for a given distro.
- Different distros maintain their own repos — they overlap heavily but aren't identical.
- The file `/etc/apt/sources.list` lists every repo your system will search when you run `apt update`.
- It's worth **adding backup repos** (e.g. Ubuntu's) so your system has a fallback if Kali's repo doesn't carry a particular package.
- When you request a package, the system searches all repos in `sources.list` and picks the most recent version it finds.

### Repository Categories (Debian/Ubuntu)

Many distros split packages into categories by licensing and support level:

| Category       | What's In It                                                                              |
| -------------- | ----------------------------------------------------------------------------------------- |
| **main**       | Free, open-source software officially supported by the distro.                            |
| **universe**   | Community-maintained free software (not officially supported).                            |
| **multiverse** | Software that is *not* free/open-source (proprietary drivers, codecs, etc.).              |
| **restricted** | Proprietary software that the distro considers essential (e.g. certain hardware drivers). |
| **backports**  | Newer versions of software back-ported to an older stable release.                        |

---

## Chapter 5: Controlling File and Directory Permissions

Every file/directory in Linux has three permission sets: **owner (u)**, **group (g)**, and **others (o)**. Each set can have **read (r)**, **write (w)**, and **execute (x)**.

### Reading Permissions

`ls -l` output looks like: `-rwxr-xr--`. Break it down:

```javascript
-  rwx  r-x  r--
│   │    │    └── others: read only
│   │    └─────── group:  read + execute
│   └──────────── owner:  read + write + execute
└──────────────── file type (- = file, d = directory)
```

### Numeric (Octal) Method — `chmod 755`

Each permission has a numeric value: **r=4, w=2, x=1**. Sum them per set:

| Digit | Permissions | Meaning                |
| ----- | ----------- | ---------------------- |
| 7     | rwx         | read + write + execute |
| 6     | rw-         | read + write           |
| 5     | r-x         | read + execute         |
| 4     | r--         | read only              |
| 0     | ---         | no permissions         |

Examples:

- `chmod 755 file` → owner rwx, group r-x, others r-x (typical for scripts/executables).
- `chmod 644 file` → owner rw-, group r--, others r-- (typical for regular files).
- `chmod 777 file` → everyone can do everything (almost never a good idea — security risk).

### Symbolic (UGO) Method — `chmod u-w file`

Uses letters to add/remove permissions:

- `chmod u+x file` — give the owner execute permission.
- `chmod g-w file` — remove write permission from the group.
- `chmod o=r file` — set others to read only (overwriting whatever was there).
- `chmod a+r file` — give read to **a**ll (user + group + others).

### Default Permissions with `umask`

When you create a file or directory, Linux doesn't give it `777` by default — it subtracts the **umask** value.

- Default base: **666** for files, **777** for directories (files don't get execute by default for safety).
- `umask` shows the current mask (commonly `022`).
- The actual permissions = base − umask.
  - File: 666 − 022 = **644** (rw-r--r--)
  - Directory: 777 − 022 = **755** (rwxr-xr-x)
- `umask 077` — would make new files 600 (owner only) and new directories 700. Useful for security-sensitive environments.

**Why it's useful:** `umask` lets you control the *default* security posture of every new file/directory without remembering to `chmod` each one.

### Special Permissions

Beyond the standard rwx, Linux has three special permission bits:

| Permission              | Numeric | On Files                                                                                                                                             | On Directories                                                                                                             |
| ----------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **SUID** (Set User ID)  | 4xxx    | File executes as its **owner**, not as the user who ran it. Example: `/usr/bin/passwd` runs as root so regular users can change their own passwords. | No meaningful effect.                                                                                                      |
| **SGID** (Set Group ID) | 2xxx    | File executes with the **group's** permissions.                                                                                                      | New files created inside inherit the **directory's group** (useful for shared project folders).                            |
| **Sticky Bit**          | 1xxx    | No meaningful effect.                                                                                                                                | Only the file's owner (or root) can delete/rename files inside, even if others have write access. Classic example: `/tmp`. |

- Set SUID: `chmod 4755 file`
- Set SGID: `chmod 2755 directory`
- Set Sticky: `chmod 1755 directory`

### Hacker Relevance

Misconfigured SUID/SGID binaries are a classic **privilege escalation** vector. If a binary owned by root has the SUID bit set and contains a vulnerability, an attacker can exploit it to gain root-level access.

---

## Chapter 6: Process Management

### Viewing Processes

- **ps** — snapshot of current processes.
  - `ps aux` — shows processes for **all users** with detailed info (user, PID, CPU%, MEM%, command).
- **top** — live, auto-refreshing view of the most resource-hungry processes. Press `q` to quit, `k` to kill a process interactively.

### Adjusting Priority with `nice` and `renice`

Every process has a **niceness** value from **-20** (highest priority) to **19** (lowest priority). Default is 0.

- **nice -n 10 command** — start a command with a lower priority (be "nicer" to other processes).
- **renice 5 -p PID** — change the priority of an already-running process.
- Only root can set negative niceness (higher priority).

### Killing Processes

- **kill PID** — sends a signal to a process. Default signal is **SIGTERM (15)** — politely asks the process to shut down.
- **kill -9 PID** — sends **SIGKILL** — forces immediate termination (the process cannot catch or ignore it). Use as a last resort.
- **killall name** — kills all processes matching a name (e.g. `killall apache2`).

Common kill signals:

| Signal  | Number | Behavior                                                                 |
| ------- | ------ | ------------------------------------------------------------------------ |
| SIGHUP  | 1      | Hang up — often used to tell a daemon to reload its config.              |
| SIGINT  | 2      | Interrupt — same as pressing Ctrl+C.                                     |
| SIGKILL | 9      | Forced kill — cannot be caught or ignored.                               |
| SIGTERM | 15     | Graceful termination — the default. Process can clean up before exiting. |
| SIGSTOP | 19     | Pause the process (cannot be caught).                                    |

### Background and Foreground

- **command &** — runs a command in the background so you get your shell back.
- **fg** — brings the most recent background process back to the foreground.
- **bg** — resumes a suspended process (Ctrl+Z) in the background.
- **jobs** — lists all background/suspended jobs in the current shell.

### Scheduling with `at` and `crond`

- **at** — schedule a command to run **once** at a specific time.
  - Example: `echo "backup.sh" | at 2:00 AM` — runs `backup.sh` at 2 AM tonight.
- **crond** (cron daemon) — schedules **recurring** tasks (daily, weekly, monthly, etc.).
  - Edit the cron table with `crontab -e`.
  - Cron format: `minute hour day_of_month month day_of_week command`
  - Example: `0 3 * * 1 /root/backup.sh` — runs every Monday at 3:00 AM.

---

## Chapter 7: Managing User Environment Variables

### Environment Variables vs. Shell Variables

|              | Environment Variables                                                   | Shell Variables                                  |
| ------------ | ----------------------------------------------------------------------- | ------------------------------------------------ |
| **Scope**    | Available to the current shell **and all child processes** (inherited). | Available only in the **current shell session**. |
| **Set with** | `export VAR=value`                                                      | `VAR=value` (no `export`)                        |
| **View all** | `env` or `printenv`                                                     | `set` (shows both shell + env vars)              |

- **echo $VAR** — print the value of a variable.
- **export VAR=value** — promote a shell variable to an environment variable (or create one directly).
- **unset VAR** — delete a variable.

### Key Environment Variables

| Variable   | Purpose                                                                                                                                   |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `PATH`     | Colon-separated list of directories the shell searches for executables. When you type a command, the shell walks this list left to right. |
| `HOME`     | Current user's home directory (e.g. `/root`, `/home/kali`).                                                                               |
| `USER`     | Current username.                                                                                                                         |
| `SHELL`    | Path to the user's default shell (e.g. `/bin/bash`).                                                                                      |
| `PS1`      | Defines the shell prompt appearance (e.g. `\u@\h:\w\$` → `kali@kali:~$`).                                                                 |
| `HISTSIZE` | Number of commands stored in shell history.                                                                                               |

### Making Changes Persistent

Variables set in a shell session are lost when the shell closes. To make them permanent:

- **Per-user:** Add `export VAR=value` to `~/.bashrc` (runs on every new interactive shell) or `~/.profile` (runs on login).
- **System-wide:** Add to `/etc/environment` or `/etc/profile`.

### Modifying `PATH`

- `export PATH=$PATH:/new/directory` — appends a directory to PATH.
- Putting a directory **first** in PATH means its executables take precedence (relevant for hacking: you can trick a SUID binary into running your version of a command).

---

## Chapter 8: Bash Scripting

### Basics

- A bash script is a plain text file containing a sequence of commands.
- First line should be the **shebang**: `#!/bin/bash` — tells the OS which interpreter to use.
- Make it executable: `chmod +x script.sh`, then run with `./script.sh`.

### Variables and User Input

```bash
#!/bin/bash
name="world"
echo "Hello, $name"

read -p "Enter your name: " user_name
echo "Hi, $user_name"
```

- `$variable` or `${variable}` — reference a variable's value.
- `read` — pauses and reads input from the user.

### Conditionals

```bash
if [ "$1" == "admin" ]; then
    echo "Welcome, admin"
elif [ "$1" == "guest" ]; then
    echo "Limited access"
else
    echo "Unknown user"
fi
```

- `$1`, `$2`, … — positional arguments passed to the script.
- `$#` — number of arguments. `$@` — all arguments.
- Common test operators: `-eq`, `-ne`, `-gt`, `-lt` (numeric); `==`, `!=` (string); `-f` (file exists), `-d` (directory exists).

### Loops

```bash
# For loop
for ip in $(seq 1 254); do
    ping -c 1 192.168.1.$ip
done

# While loop
count=0
while [ $count -lt 10 ]; do
    echo $count
    ((count++))
done
```

### Functions

```bash
scan_port() {
    local host=$1
    local port=$2
    (echo >/dev/tcp/$host/$port) 2>/dev/null && echo "Port $port open" || echo "Port $port closed"
}

scan_port 192.168.1.1 80
```

### Practical Patterns for Hacking

- **Piping and chaining:** `cat wordlist.txt | while read word; do ... done`
- **Command substitution:** `result=$(whoami)`
- **Exit codes:** `$?` holds the exit code of the last command (0 = success).

---

## Chapter 9: Compressing and Archiving

### Compression Tools

| Tool      | Extension | Notes                                                                         |
| --------- | --------- | ----------------------------------------------------------------------------- |
| **gzip**  | `.gz`     | Fast, good compression. Replaces the original file. Decompress with `gunzip`. |
| **bzip2** | `.bz2`    | Slower but better compression ratio than gzip. Decompress with `bunzip2`.     |
| **xz**    | `.xz`     | Best compression ratio, slowest. Decompress with `unxz`.                      |

- These compress **single files**. To compress a group of files, first create an archive with `tar`.

### Archiving with `tar`

`tar` bundles multiple files/directories into a single file (a "tarball"), optionally compressing it:

| Command                          | What It Does                                                    |
| -------------------------------- | --------------------------------------------------------------- |
| `tar -cvf archive.tar dir/`      | **C**reate a tar archive with **v**erbose output to a **f**ile. |
| `tar -xvf archive.tar`           | E**x**tract an archive.                                         |
| `tar -czvf archive.tar.gz dir/`  | Create + compress with g**z**ip.                                |
| `tar -cjvf archive.tar.bz2 dir/` | Create + compress with b**z**ip2 (j flag).                      |
| `tar -tf archive.tar`            | List contents without extracting.                               |

### Bit-for-Bit Copying with `dd`

`dd` (sometimes called "disk destroyer" — be careful) copies data at the raw block level. It doesn't care about file systems — it reads and writes raw bytes.

```javascript
dd if=/dev/sda of=/backup/sda.img bs=4M status=progress
```

| Parameter         | Meaning                                                                                                 |
| ----------------- | ------------------------------------------------------------------------------------------------------- |
| `if=`             | **I**nput **f**ile (source) — can be a device, partition, or file.                                      |
| `of=`             | **O**utput **f**ile (destination).                                                                      |
| `bs=`             | **B**lock **s**ize — how much data to read/write at a time (e.g. `4M`). Larger = faster for big copies. |
| `count=`          | Number of blocks to copy (omit to copy everything).                                                     |
| `status=progress` | Shows a live progress indicator.                                                                        |

**Why dd matters:**

- **Forensic imaging:** Creates an exact bit-for-bit copy of a drive, including deleted files, slack space, and hidden partitions. This is the standard for evidence acquisition.
- **Bootable USB creation:** `dd if=kali.iso of=/dev/sdb bs=4M` writes an ISO directly to a USB drive.
- **Disk wiping:** `dd if=/dev/zero of=/dev/sda bs=4M` overwrites an entire disk with zeros.
- **Backup/restore partitions:** Copy a partition to an image file and restore it later.

**Caution:** `dd` has no confirmation prompt. If you swap `if` and `of`, you overwrite your source. Always double-check the device names with `lsblk` before running.

---

## Chapter 10: Filesystem and Storage Device Management

### The Linux Filesystem Hierarchy

Linux organizes everything under a single root `/`. Key directories:

| Directory        | Purpose                                                                       |
| ---------------- | ----------------------------------------------------------------------------- |
| `/root`          | Home directory of the root user.                                              |
| `/home`          | Home directories for regular users.                                           |
| `/etc`           | System-wide configuration files.                                              |
| `/bin`, `/sbin`  | Essential binaries (`bin` for all users, `sbin` for system/admin).            |
| `/usr`           | User programs, libraries, documentation.                                      |
| `/var`           | Variable data — logs (`/var/log`), mail, spool files.                         |
| `/tmp`           | Temporary files (often cleared on reboot).                                    |
| `/dev`           | Device files — Linux represents hardware as files here.                       |
| `/mnt`, `/media` | Mount points for manually or automatically mounted filesystems.               |
| `/proc`          | Virtual filesystem exposing kernel and process info (not real files on disk). |

### Device Files and Naming

Linux represents storage devices as files in `/dev`:

| Device      | Meaning                            |
| ----------- | ---------------------------------- |
| `/dev/sda`  | First SATA/SCSI/USB disk.          |
| `/dev/sdb`  | Second disk, and so on.            |
| `/dev/sda1` | First partition on the first disk. |
| `/dev/sda2` | Second partition, etc.             |

### Mounting and Unmounting

A filesystem must be **mounted** to a directory before you can access it.

- **mount /dev/sdb1 /mnt/usb** — makes the partition's contents accessible at `/mnt/usb`.
- **umount /mnt/usb** — unmounts the filesystem (always unmount before physically removing a drive to avoid corruption).
- **lsblk** — lists all block devices and their mount points in a tree view.
- **df -h** — shows disk space usage for all mounted filesystems in human-readable format.
- **/etc/fstab** — configuration file that defines which filesystems to mount automatically at boot.

### Checking and Repairing Filesystems

- **fsck /dev/sdb1** — filesystem check and repair. Must be run on **unmounted** filesystems.

### Creating Filesystems

- **mkfs** — make filesystem. Formats a partition with a chosen filesystem type.
  - `mkfs.ext4 /dev/sdb1` — formats with ext4 (the most common Linux filesystem).
  - `mkfs.vfat /dev/sdb1` — formats with FAT (for USB drives that need to work with Windows/Mac).
  - `mkfs.ntfs /dev/sdb1` — formats with NTFS.
