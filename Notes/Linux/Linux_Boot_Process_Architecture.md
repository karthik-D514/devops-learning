# 🚀 Linux Boot Process - Architecture & Stage Verification Guide

## 1. High-Level Architecture Diagram

```
  +------------------------------------------------------------------------+
  | 1. HARDWARE & FIRMWARE (POST & UEFI / BIOS)                            |
  |    Power On ──> POST (CPU/RAM Check) ──> Firmware finds Boot Device    |
  |    • UEFI: Reads EFI System Partition (ESP) at /boot/efi (FAT32)       |
  |    • BIOS: Reads Master Boot Record (MBR Sector 0)                     |
  +------------------------------------------------------------------------+
                                       │
                                       ▼
  +------------------------------------------------------------------------+
  | 2. BOOTLOADER (GRUB2)                                                  |
  |    Reads /boot/grub2/grub.cfg (or BLS entries) ──> Shows Kernel Menu   |
  |    Loads 2 critical files from /boot into RAM:                         |
  |      • vmlinuz   (Compressed Linux Kernel Binary)                      |
  |      • initramfs (Temporary Storage Driver Archive in RAM)             |
  +------------------------------------------------------------------------+
                                       │
                                       ▼
  +------------------------------------------------------------------------+
  | 3. KERNEL INITIALIZATION                                               |
  |    • Decompresses vmlinuz in memory                                    |
  |    • Switches CPU to 64-bit Long Mode; sets up Virtual Memory & IDT    |
  |    • Detects CPUs (SMP) and built-in hardware controllers              |
  |    • Mounts initramfs as temporary root in RAM                         |
  +------------------------------------------------------------------------+
                                       │
                                       ▼
  +------------------------------------------------------------------------+
  | 4. INITRAMFS & ROOT PIVOT                                              |
  |    • Loads storage modules (LVM, Software RAID, NVMe, SCSI, Multipath) |
  |    • Detects real disk volume & mounts it at /sysroot (Read-Only)      |
  |    • Executes: switch_root /sysroot /sbin/init                         |
  |      (Discards RAM disk, frees memory, pivots /sysroot to real /)      |
  +------------------------------------------------------------------------+
                                       │
                                       ▼
  +------------------------------------------------------------------------+
  | 5. SYSTEM INITIALIZATION (systemd - PID 1)                             |
  |    • systemd starts as PID 1                                           |
  |    • Parses /etc/fstab & remounts root / as Read-Write (rw)            |
  |    • Mounts remaining filesystems (/home, /var, swap, network mounts)  |
  |    • Starts system services concurrently via dependency graph          |
  |    • Reaches multi-user.target ──> Spawns login prompts (getty/SSH)    |
  +------------------------------------------------------------------------+
```

---

## 2. The 5 Boot Stages Explained

The Linux boot process consists of **5 sequential stages**, moving from hardware up to running user-space services:

### 1. Hardware & Firmware (POST & UEFI/BIOS)
- When power hits the machine, the motherboard runs **POST** (Power-On Self-Test) to verify physical components like the CPU, memory (RAM), and storage controllers.
- Once POST passes, firmware takes control:
  - **Modern UEFI:** Reads the EFI System Partition (**ESP**) formatted as FAT32 at `/boot/efi` to locate boot binaries.
  - **Legacy BIOS:** Reads the first 512 bytes (**MBR Sector 0**) of the boot disk.
- The firmware hands execution over to the bootloader.

### 2. Bootloader (GRUB2)
- GRUB2 reads configuration files from `/boot/grub2/grub.cfg` (or `/boot/loader/entries/` on RHEL/Rocky Linux).
- It displays the kernel selection menu and countdown.
- GRUB2's primary job is to load two critical files from the `/boot` partition into physical RAM:
  1. **`vmlinuz`**: The compressed Linux kernel binary.
  2. **`initramfs`**: The driver helper RAM archive.
- It passes kernel boot parameters (like `root=... ro quiet rhgb`) and transfers CPU execution to the kernel.

### 3. Kernel Initialization
- The kernel self-extracts in physical RAM.
- It switches the CPU to 64-bit Long Mode, sets up memory management (virtual memory page tables), and initializes built-in hardware drivers.
- **The Catch:** Modern root filesystems (`/`) usually live on complex storage (LVM, software RAID, multipath, or encrypted disks). The kernel does not bundle these drivers directly in its binary to avoid bloat.
- Therefore, the kernel cannot yet read the physical disk. It mounts the `initramfs` image in RAM as a temporary root filesystem.

### 4. Initramfs & `switch_root`
- Inside `initramfs`, required storage driver modules (LVM, RAID, NVMe, SCSI) are loaded.
- It locates the real hard drive partition and mounts it at a temporary directory: **`/sysroot`** in **Read-Only (`ro`)** mode.
- Once mounted, it executes **`switch_root`**:
  - Cleans up and deletes the temporary RAM disk to reclaim memory.
  - Pivots `/sysroot` to become the true permanent root directory (`/`).
  - Executes **`/sbin/init`** from the real disk.

### 5. System Initialization (`systemd` - PID 1)
- `/sbin/init` is a symlink to `systemd`, which becomes **PID 1**.
- `systemd` parses **`/etc/fstab`**, runs filesystem checks (`fsck`), and remounts `/` as **Read-Write (`rw`)**.
- It mounts other partitions (`/home`, `/var`, swap, NFS mounts).
- It starts background daemons in parallel based on unit dependencies (`network.target`, `sshd.service`, `firewalld.service`).
- It reaches the default target (**`multi-user.target`** for server CLI, or **`graphical.target`** for desktop GUI) and spawns terminal login prompts.

---

## 3. How to Verify Success or Failure at Every Stage

| Stage | Success State | Failure State (Symptoms) | How to Verify / Command |
| :--- | :--- | :--- | :--- |
| **Stage 1: POST & Firmware** | Hardware beeps once/silent, OEM logo displays, firmware boots storage device. | Black screen, power loop, motherboard beep codes, or *"No bootable device found"*. | **Hardware Level:** Check Motherboard Port `0x80` debug LED, IPMI / iLO / iDRAC hardware console event logs.<br>**OS Level:** `[ -d /sys/firmware/efi ] && echo UEFI \|\| echo BIOS` |
| **Stage 2: GRUB2** | GRUB menu displays kernel choices, or automatically starts loading the default kernel. | System hangs at `grub>` rescue prompt, or displays `error: file '/boot/grub2/grub.cfg' not found`. | **Live Verification:**<br>`cat /proc/cmdline` (shows kernel options GRUB passed).<br>`ls -l /boot/grub2/grub.cfg` or `ls /boot/loader/entries/` |
| **Stage 3: Kernel** | Screen shows Linux kernel boot logs, text scrolls rapidly initializing devices. | **Kernel Panic:** Screen freezes with `Kernel panic - not syncing: VFS: Unable to mount root fs` or CPU lockup. | **Live Verification:**<br>`uname -r` (verifies running kernel version).<br>`dmesg \| head -n 30` (shows earliest kernel memory & CPU initialization messages). |
| **Stage 4: Initramfs & `switch_root`** | Initramfs detects storage controllers, mounts `/sysroot`, and pivots seamlessly to disk. | Drops into `dracut:/#` emergency shell. Error: `/dev/mapper/root does not exist` or storage timeout. | **Live Verification:**<br>`lsinitrd /boot/initramfs-$(uname -r).img \| grep -i lvm` (verifies required drivers exist inside the image).<br>`journalctl -b \| grep -i switch_root` |
| **Stage 5: systemd (PID 1)** | System displays `[ OK ]` green checkmarks for services and brings up the `login:` prompt. | Drops into `emergency.target` shell: *"Welcome to emergency mode!"* (typically caused by `/etc/fstab` typo). | **Live Verification:**<br>`systemctl is-system-running` (returns `running`, `degraded`, or `maintenance`).<br>`systemd-analyze` (breakdown of time per stage).<br>`journalctl -xb` (inspects failed mount or unit). |

---

## 4. Deep-Dive Questions & Core Concepts

### Why does `initramfs` exist? Why can't the kernel mount the disk directly?
The real root disk is usually on an LVM, RAID, or encrypted volume. The drivers to read those storage layers are compiled as modules (`.ko` files) that live inside `/lib/modules/` on the hard drive itself. The kernel cannot read the drive without the drivers, and cannot get the drivers without reading the drive. `initramfs` breaks this cycle by providing a tiny copy of those drivers in RAM first.

### Why is `switch_root` used instead of `chroot`?
A regular `chroot` only changes the root directory for a process, leaving the temporary RAM disk mounted in physical memory. `switch_root` frees up the physical RAM (reclaims 30–80MB), overmounts the real root `/`, and executes `/sbin/init` as true PID 1.

### Step-by-Step Triage: Fixing Emergency Mode on Boot
1. Enter the root password to access the emergency shell.
2. Run `journalctl -xb | grep -i mount` to identify which partition or disk failed to mount.
3. Remount the root filesystem as read-write:
   ```bash
   mount -o remount,rw /
   ```
4. Edit `/etc/fstab` using an editor (`vi /etc/fstab`) to comment out or fix the bad UUID, unreachable network share, or invalid mount option.
5. Test the fix without rebooting:
   ```bash
   mount -a
   ```
6. If `mount -a` completes cleanly with no errors, type `exit` or `systemctl reboot` to return to normal boot.
