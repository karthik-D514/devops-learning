# Linux Boot Process & Rescue - Real-Time Interview Answers

---

## Section: Linux Boot Sequence & Architecture

### Q1.1 Answer
The Linux boot process consists of 5 chronological stages:
1. **Hardware & Firmware (POST & BIOS/UEFI):**
   - The PSU stabilizes voltage and signals `PWR_OK`.
   - The CPU executes code from the **Reset Vector** (`0xFFFFFFF0`) mapped to ROM.
   - **POST** tests CPU, calibrates RAM using Cache-As-RAM and SPD timings, and polls PCI buses for controllers.
   - Firmware locates the configured boot device (reads MBR Sector 0 on BIOS or reads `/boot/efi` ESP on UEFI).
2. **Bootloader (GRUB2):**
   - GRUB2 displays the boot menu and parses configuration (`grub.cfg` / BLS entries).
   - Loads the selected compressed kernel image (`vmlinuz`) and initial RAM disk (`initramfs`) into RAM.
   - Passes kernel command-line arguments (e.g., `root=UUID=... ro quiet rhgb`) and transfers CPU execution to the kernel.
3. **Kernel Initialization:**
   - `vmlinuz` self-extracts, switches CPU to 64-bit Long Mode, and initializes virtual memory paging and interrupt descriptor tables.
   - Detects CPUs, initializes SMP (Symmetric Multiprocessing), and mounts the temporary root filesystem (`initramfs`) in RAM.
4. **Initramfs (Temporary Root):**
   - Contains a minimal Linux environment and storage driver modules (LVM, software RAID, SCSI/NVMe, multipath, LUKS).
   - Loads required drivers, detects the physical storage volume, and mounts the real root disk partition to `/sysroot` in read-only (`ro`) mode.
   - Executes `switch_root` to pivot from the RAM disk to the real root on `/sysroot` and executes `/sbin/init`.
5. **Systemd Initialization (PID 1):**
   - `systemd` (PID 1) takes over system management.
   - Parses `/etc/fstab`, validates filesystems with `fsck`, and remounts root `/` as read-write (`rw`).
   - Concurrently starts units to reach the default target (e.g., `multi-user.target` or `graphical.target`).
   - Launches user-space login services (`getty@tty1.service` or display manager).

---

### Q1.2 Answer
| Feature | Legacy BIOS | Modern UEFI |
| :--- | :--- | :--- |
| **Architecture** | 16-bit processor mode; limited access to 1MB memory during boot. | 32-bit or 64-bit mode; full memory and hardware access. |
| **Partition Scheme** | **MBR (Master Boot Record)**. | **GPT (GUID Partition Table)**. |
| **Disk & Partition Limits** | Max 2 TB disk size; max 4 primary partitions. | Disks up to 9.4 ZB ($10^{21}$ bytes); up to 128 primary partitions. |
| **Bootloader Location** | Sector 0 (first 512 bytes) of the physical disk. | Dedicated FAT32 **EFI System Partition (ESP)** mounted at `/boot/efi`. |
| **Security** | No verification of boot code. | **Secure Boot** cryptographically validates digital signatures of bootloader and kernel binaries. |

---

### Q1.3 Answer
- **The Chicken-and-Egg Problem:** The real root filesystem (`/`) is frequently hosted on complex storage abstractions like **LVM Logical Volumes**, **Software RAID (mdadm)**, **iSCSI/SAN multipath**, or **LUKS encryption**.
- The Linux kernel binary cannot bundle every storage driver in existence because the kernel image would be hundreds of megabytes. Instead, storage drivers are compiled as loadable kernel modules (`.ko` files) stored on disk inside `/lib/modules/`.
- However, the kernel cannot read `/lib/modules/` without first mounting the root disk—which requires the storage drivers!
- **Role of `initramfs`:** It breaks this cycle. The bootloader loads `initramfs` directly into RAM alongside the kernel. `initramfs` contains a tiny root filesystem with only the exact drivers and tools needed (LVM, RAID, device mappers) to assemble the storage stack, locate the true root filesystem, and mount it to `/sysroot`.

---

### Q1.4 Answer
- **Why not `chroot`:** A standard `chroot` only changes the apparent root directory for the current running process and its children. It **leaves the old `initramfs` root filesystem mounted in RAM**, permanently consuming memory (typically 30MB–100MB of RAM) and polluting the mount namespace.
- **Why `switch_root`:** `switch_root` explicitly:
  1. Recursively removes all files and directories from the temporary RAM filesystem (reclaiming physical RAM).
  2. Overmounts `/sysroot` onto `/`.
  3. Re-parents all processes to the new root and executes `/sbin/init` as PID 1 via `execve`.

---

### Q1.5 Answer
- **`vmlinuz`**: The bootable, compressed Linux operating system kernel binary. Contains the core architecture logic, CPU scheduler, virtual memory management, and built-in drivers.
- **`initramfs`**: A compressed cpio archive of a minimal user-space filesystem containing device drivers (`.ko` files), udev rules, and scripts needed to activate the storage volume where `/` resides.
- **Storage Location**: Both files are stored in the **`/boot`** filesystem (e.g. `/boot/vmlinuz-<kernel_version>` and `/boot/initramfs-<kernel_version>.img`).
- **Who loads them**: The **bootloader (GRUB2)** reads both files from disk and loads them into physical RAM before transferring control to the kernel.

---

### Q1.6 Answer
- **Boot Loader Specification (BLS):** A modern standard adopted in RHEL 8/9, CentOS Stream, Fedora, and Rocky Linux that separates boot menu entries from the main GRUB configuration.
- In legacy systems, installing or removing a kernel required running `grub2-mkconfig` to completely rewrite `/boot/grub2/grub.cfg`.
- With BLS, each installed kernel has its own self-contained configuration snippet stored in:
  ```text
  /boot/loader/entries/<machine-id>-<kernel-version>.conf
  ```
- When a new kernel RPM is installed or removed, only that specific `.conf` file is added or deleted. `/boot/grub2/grub.cfg` remains clean and static, dynamically populating the boot menu at runtime.

---

## Section: Boot Troubleshooting & SRE Incidents

### Q2.1 Answer
- **Most Common Root Cause:** A syntax error, typo, or invalid entry in **`/etc/fstab`** (e.g. referencing a disk UUID or NFS mount that no longer exists, or a typo in mount options). Systemd fails during local filesystem mounting and drops to emergency mode.
- **Troubleshooting Steps:**
  1. Log into emergency mode with the root password.
  2. Inspect the failure log:
     ```bash
     journalctl -xb | grep -i "failed to mount"
     ```
  3. Remount the root filesystem as read-write (in emergency mode, it is often mounted read-only):
     ```bash
     mount -o remount,rw /
     ```
  4. Open `/etc/fstab` in an editor:
     ```bash
     vi /etc/fstab
     ```
  5. Identify the broken mount line. Comment it out or correct the typo/UUID.
  6. Test mounting all filesystems defined in `/etc/fstab` without rebooting:
     ```bash
     mount -a
     ```
     If `mount -a` returns cleanly with no errors, type `exit` or `systemctl reboot` to resume normal boot.

---

### Q2.2 Answer (The RHCSA Root Password Reset Procedure)
1. **Reboot the VM** and access the GRUB2 boot menu.
2. Highlight the desired kernel and press **`e`** to edit the boot parameters.
3. Locate the line starting with **`linux`** (or `linux16` / `linuxefi`).
4. Move to the end of this line and append:
   ```text
   rd.break
   ```
5. Press **`Ctrl + X`** (or `F10`) to boot.
6. The system boots into the `initramfs` emergency shell before the real root is pivoted. The real root disk is mounted at `/sysroot` in read-only (`ro`) mode.
7. Remount `/sysroot` with write permissions:
   ```bash
   mount -o remount,rw /sysroot
   ```
8. Chroot into the real system environment:
   ```bash
   chroot /sysroot
   ```
9. Change the root password:
   ```bash
   passwd root
   ```
10. **Critical SELinux Step:** Because SELinux was not running during the emergency shell, `/etc/shadow` will be saved without valid SELinux security contexts. To prevent SELinux from blocking all root logins upon reboot, force an automatic filesystem relabel on the next boot:
    ```bash
    touch /.autorelabel
    ```
11. Exit chroot and exit the initramfs shell:
    ```bash
    exit
    exit
    ```
12. The system automatically relabels SELinux contexts and reboots with the new root password active.

---

### Q2.3 Answer
- **Meaning:** The kernel successfully initialized itself into RAM, but **could not find or mount the root filesystem partition**.
- **Root Causes:**
  1. The `initramfs` image is missing or corrupted.
  2. The `root=` kernel parameter in GRUB points to an invalid disk, incorrect UUID, or non-existent LVM path.
  3. The storage driver (e.g. VirtIO, NVMe, megaraid) required to read the disk is missing from the `initramfs`.
- **Resolution:**
  1. In the GRUB menu, press `e` and verify the `root=` parameter matches the correct disk UUID (`ls -l /dev/disk/by-uuid` via rescue media).
  2. Boot an older kernel entry listed in the GRUB menu to see if the problem is specific to a newly installed kernel.
  3. If `initramfs` is corrupted, boot via RHEL/Rocky Installation ISO in **Rescue Mode**, chroot into the system (`chroot /mnt/sysimage`), and rebuild the `initramfs` using:
     ```bash
     dracut -f /boot/initramfs-$(uname -r).img $(uname -r)
     ```

---

### Q2.4 Answer
To diagnose slow boot times, use the **`systemd-analyze`** suite:
1. **Overall Boot Breakdown:**
   ```bash
   systemd-analyze
   ```
   *Displays the time spent in Firmware, Loader, Kernel, Initrd, and Userspace.*
2. **Identify Slowest Services:**
   ```bash
   systemd-analyze blame
   ```
   *Prints a list of all running units sorted by initialization time.*
3. **Inspect the Critical Dependency Path:**
   ```bash
   systemd-analyze critical-chain
   ```
   *Outputs a dependency tree highlighting the chain of services that directly delayed the final target.*

---

### Q2.5 Answer
- **`systemd.unit=rescue.target`**:
  - The boot process executes **fully through the `initramfs`**, pivots to the real root filesystem, starts PID 1 (`systemd`), mounts all standard filesystems from `/etc/fstab`, and drops to a single-user root shell.
  - **Requires the root password** to log in.
  - Used for maintaining system configurations when the root password is known.
- **`rd.break`**:
  - Interrupts the boot process **early inside the `initramfs`** before `switch_root` occurs and before `systemd` on the disk is executed.
  - The root disk is mounted read-only at `/sysroot`, and no userspace services have started.
  - **Does NOT require a password**.
  - Used primarily for disaster recovery, root password resets, and resolving storage driver/LVM assembly failures.
