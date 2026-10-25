# 🗺️ Complete Linux Master Architecture & SRE Incident Map

> 🤖 **Note:** *This document is AI-generated as an overarching reference and architectural blueprint*

---

## 1. The Complete Linux Operating System Stack (End-to-End)

```mermaid
flowchart TD
    subgraph STACK1["1. Boot & Hardware Layer"]
        HW["Hardware / Firmware<br>(BIOS / UEFI / CPU / RAM / Disks)"]
        GRUB["Bootloader: GRUB2<br>(grub.cfg, kernel selection, rd.break)"]
        KERNEL["Linux Kernel (vmlinuz)<br>(Hardware drivers, memory paging, syscalls, cgroups)"]
        INITRD["Initramfs (initrd.img)<br>(Temporary root filesystem in RAM with disk drivers)"]
        HW --> GRUB --> KERNEL --> INITRD
    end

    subgraph STACK2["2. Storage & Filesystem Layer"]
        RAW["Raw Disks / Block Devices<br>(/dev/sda, /dev/nvme0n1, MBR / GPT)"]
        LVM["LVM Layer<br>(PV -> VG -> LV | Online resizing)"]
        FS["Filesystem Layer<br>(ext4, xfs, Inodes, /etc/fstab, Mount points)"]
        RAW --> LVM --> FS
    end

    subgraph STACK3["3. Init & Supervision Layer (PID 1)"]
        SYSTEMD["systemd (PID 1)<br>(Grand Process Supervisor & Service Orchestrator)"]
        TARGETS["Targets / Runlevels<br>(multi-user.target, graphical.target, rescue.target)"]
        UNITS["Unit Files<br>(.service, .socket, .timer, .mount)"]
        INITRD -->|"Pivot root to /"| SYSTEMD
        SYSTEMD --> TARGETS
        SYSTEMD --> UNITS
    end

    subgraph STACK4["4. Security & Access Control Layer"]
        USERS["Identity Subsystem<br>(/etc/passwd, /etc/shadow, /etc/group)"]
        PERM["Permissions & ACLs<br>(Octal 755/640, SUID, SGID, Sticky Bit)"]
        SUDO["Privilege Escalation<br>(sudo, visudo, /etc/sudoers.d/)"]
        SELINUX["Mandatory Access Control (MAC)<br>(SELinux: Enforcing/Permissive, contexts, booleans)"]
        USERS --> PERM --> SUDO
        PERM --> SELINUX
    end

    subgraph STACK5["5. Process Execution & Memory Layer"]
        PROC["Processes in RAM (PIDs)<br>(States: R, S, D, Z, T | CFS Scheduler & nice)"]
        SIGNALS["Signal Subsystem<br>(SIGTERM 15, SIGKILL 9, SIGHUP 1, SIGSTOP 19)"]
        THREADS["Threads (LWP)<br>(Share process memory space)"]
        UNITS -->|"ExecStart"| PROC
        PROC --- THREADS
        SIGNALS -->|"Controls"| PROC
    end

    subgraph STACK6["6. Networking Subsystem"]
        NETDEV["Network Interfaces<br>(eth0, ens3, bonding, NetworkManager / nmcli)"]
        TCP["TCP/IP Protocol Stack<br>(IP Addressing, Subnets, Routing table, Sockets)"]
        DNS["DNS Resolution Chain<br>(/etc/hosts -> /etc/resolv.conf -> DNS servers)"]
        FIREWALL["Packet Filtering<br>(firewalld zones & rich rules, netfilter / iptables)"]
        NETDEV --> TCP --> DNS
        TCP --> FIREWALL
    end

    subgraph STACK7["7. Observability, Logging & Telemetry"]
        JOURNAL["systemd-journald (journalctl)<br>(Service stdout/stderr, boot logs)"]
        RSYSLOG["rsyslog & logrotate<br>(/var/log/secure, /var/log/messages)"]
        DMESG["Kernel Ring Buffer (dmesg)<br>(Hardware I/O timeouts, OOM killer)"]
        PROC_FS["Virtual Filesystem (/proc & /sys)<br>(Real-time kernel & process state)"]
    end

    FS --> STACK3
    STACK3 --> STACK4
    STACK4 --> STACK5
    STACK5 --> STACK6
    STACK5 --> STACK7
    STACK6 --> STACK7
```

---

## 2. Cross-Subsystem Incident Triage Matrix

When a production incident strikes, use this system-level guide to pinpoint **which subsystem failed** and **where to look**:

| Failure Category | Real-World Symptoms | Root Cause Behind the Scenes | Primary Diagnostic Commands | Primary Log / Config Files |
| :--- | :--- | :--- | :--- | :--- |
| **Boot Failure** | Server hangs during boot; drops into Emergency Mode shell. | Corrupted `/etc/fstab` (typo or missing UUID), or corrupted Initramfs. | `journalctl -xb`<br>`systemctl default` | `/etc/fstab`<br>`/boot/grub2/grub.cfg`<br>`/etc/default/grub` |
| **Forgotten Root Password** | Locked out of root; sudoers access lost. | Administrator credentials lost or compromised. | Reboot into GRUB2 → append `rd.break` to kernel line → remount `/sysroot` as `rw` → `chroot /sysroot` → `passwd` → `touch /.autorelabel`. | Initramfs environment |
| **Storage / Disk Full** | Apps crash with *"No space left on device"*; database refuses writes. | **Case A:** Disk blocks full (large logs).<br>**Case B:** Inode table exhausted (too many tiny files).<br>**Case C:** Deleted files still held open by running processes. | `df -h` (blocks)<br>`df -i` (inodes)<br>`lsof +L1` (deleted files held by PIDs)<br>`find / -type f -size +200M` | `/var/log/`<br>`/tmp`<br>`/var/lib/` |
| **I/O Hang / Uninterruptible Sleep** | Load Average is high (e.g. 40), but CPU utilization is only 2-5%. Apps time out. | Processes stuck in **State `D`** waiting for unresponsive storage, dead SAN volume, or hung NFS mount. | `ps -eo pid,stat,wchan,cmd \| grep ' D '`<br>`cat /proc/<PID>/wchan`<br>`cat /proc/<PID>/stack` | `dmesg -T \| grep -iE 'nfs\|error\|scsi'` |
| **Service Crash Loop** | `systemctl status` shows `failed (Result: start-limit-hit)` or `code=exited, status=203/EXEC`. | ExecStart path wrong, binary missing `+x`, or app crashing repeatedly exceeding `StartLimitBurst`. | `journalctl -u <svc> -xe`<br>`systemctl reset-failed <svc>`<br>`systemctl daemon-reload` | `/etc/systemd/system/<svc>.service` |
| **Process Exhaustion / Fork Bomb** | Cannot run commands; bash errors: *"fork: retry: Resource temporarily unavailable"*. | System reached `pid_max` or user reached `ulimit -u` due to runaway scripts or leaked zombies. | `ps -e \| wc -l`<br>`cat /proc/sys/kernel/pid_max`<br>`ulimit -u` | `/etc/security/limits.conf`<br>`/proc/sys/kernel/pid_max` |
| **Network Blackout** | Server cannot reach gateway or external APIs; SSH connects very slowly. | Network interface DOWN, missing default gateway, or MTU mismatch. | `ip addr show`<br>`ip route show`<br>`ping -c 3 <gateway>`<br>`nmcli device status` | `/etc/NetworkManager/`<br>`/etc/sysconfig/network-scripts/` |
| **DNS Resolution Failure** | *"Could not resolve host"*; curl/APIs fail by domain name but work by IP address. | `/etc/resolv.conf` empty/misconfigured, nameserver unreachable, or search domain broken. | `dig +trace api.github.com`<br>`nslookup google.com`<br>`cat /etc/resolv.conf` | `/etc/resolv.conf`<br>`/etc/hosts`<br>`/etc/nsswitch.conf` |
| **Firewall Dropping Traffic** | App is running on port 8080 (`ss -tulnp` shows LISTEN), but external clients get *"Connection timed out"*. | Port is blocked by `firewalld` or iptables rules. | `firewall-cmd --list-all`<br>`firewall-cmd --add-port=8080/tcp --permanent`<br>`firewall-cmd --reload` | `/etc/firewalld/zones/` |
| **SELinux Denials (AVC)** | App fails with *"Permission Denied"* even though standard permissions are `777` (`rwxrwxrwx`). | SELinux label on file/port doesn't match service domain policy. | `sestatus`<br>`ls -Z /path/to/file`<br>`ausearch -m avc -ts recent`<br>`audit2why` | `/var/log/audit/audit.log`<br>`/etc/selinux/config` |
| **Sudo Misconfiguration** | Sudo fails with *"no valid sudoers sources found"* or rules in drop-in are ignored. | Filename has dot (`.conf`), permissions not `0440`, or syntax error in `/etc/sudoers.d/`. | `visudo -c` (check syntax)<br>`ls -l /etc/sudoers.d/`<br>`chmod 0440 /etc/sudoers.d/<file>` | `/etc/sudoers`<br>`/etc/sudoers.d/` |
| **Out-Of-Memory (OOM) Kill** | Application suddenly terminates without warning or exit code. | Linux kernel ran out of physical RAM + Swap and invoked the OOM Killer to save the OS. | `dmesg -T \| grep -i -E 'killed process\|oom'`<br>`journalctl -k \| grep -i oom`<br>`free -h` | `/var/log/messages`<br>`dmesg` buffer |

---

## 3. The Enterprise SRE 6-Pillar Diagnostic Toolchain

Whenever logging into an unknown or degraded Linux machine, run these commands in order:

```bash
# 1. System Load & Uptime (Is it CPU-bound or I/O-bound?)
uptime
top -b -n 1 | head -n 20

# 2. Memory & Swap Pressure (Are we close to an OOM kill?)
free -h
vmstat 1 5

# 3. Storage & Inode Capacity (Is disk space or inode count full?)
df -h
df -i

# 4. Kernel & Hardware Health (Are there disk timeouts or crashes?)
dmesg -T | tail -50

# 5. Network Sockets & Port Listeners (Is the service actually listening?)
ss -tulnp
ip route

# 6. Service & Application Logs (What did the application say before dying?)
journalctl -p err -b
```

---
