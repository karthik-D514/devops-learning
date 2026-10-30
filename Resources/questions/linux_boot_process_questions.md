# Linux Boot Process & Rescue - Real-Time Interview Questions

---

## Section: Linux Boot Sequence & Architecture

### Q1.1
Explain the end-to-end Linux boot process in 5 clear chronological stages, starting from the physical power button being pressed to the final login prompt. What are the key artifacts or binaries involved in each stage?

### Q1.2
What is the difference between legacy BIOS and modern UEFI firmware? Explain their differences regarding disk partitioning schemes (MBR vs GPT), partition size limits, and where the bootloader binaries reside on disk.

### Q1.3
Why can't the Linux kernel directly mount the real root filesystem (`/`) from a hard drive on a modern enterprise system (e.g., RHEL with LVM or Software RAID)? What role does the `initramfs` play in solving this chicken-and-egg problem?

### Q1.4
What is `switch_root`? Why does the system use `switch_root` instead of standard `chroot` when transitioning from `initramfs` to the real root filesystem?

### Q1.5
What is the difference between `vmlinuz` and `initramfs`? Where are these files stored on disk, and who loads them into physical memory (RAM)?

### Q1.6
What is the purpose of the **Boot Loader Specification (BLS)** in modern enterprise distributions like RHEL 8/9 and Rocky Linux? Where are kernel boot entries stored instead of editing `/boot/grub2/grub.cfg` directly?

---

## Section: Boot Troubleshooting & SRE Incidents

### Q2.1
A Linux server fails to boot and drops into an emergency rescue shell with the message:
`Welcome to emergency mode! After logging in, type "journalctl -xb" to view system logs...`
The logs indicate that a filesystem could not be mounted. What is the most common root cause, and what exact steps do you take to fix it and allow the server to boot normally?

### Q2.2
You are locked out of the root account on a production RHEL/Rocky Linux virtual machine and nobody knows the password. You have console access via the hypervisor. Explain the exact step-by-step procedure to reset the root password using `rd.break` in GRUB2, including the critical SELinux step.

### Q2.3
During boot, a server displays the error:
`Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)`
What does this error signify, and how would you resolve it from the GRUB menu or rescue media?

### Q2.4
An application team complains that their newly launched RHEL server is taking 7 minutes to boot up before becoming accessible via SSH. What commands do you run to identify the exact service or dependency chain causing the boot delay?

### Q2.5
What is the difference between passing `systemd.unit=rescue.target` and passing `rd.break` as a kernel boot parameter in GRUB2? When would you use one over the other?
