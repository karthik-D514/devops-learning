# Linux Real-Time Interview Questions

## Section 1: Filesystem Hierarchy Standard (FHS) & Directory Structure

### Q1.1
Explain the fundamental architectural difference between `/bin`, `/sbin`, `/usr/bin`, and `/usr/sbin`. Why does `/sbin` exist separately if modern distributions symlink them?

### Q1.2
A third-party monitoring agent needs to be deployed manually via a `.tar.gz` archive (not using `dnf` or `apt`). According to Linux FHS conventions, should it be placed in `/usr/local` or `/opt`? Explain the distinction.

### Q1.3
Explain the difference between `/tmp` and `/var/tmp`. If both store temporary files, why are both directories necessary in an enterprise Linux system?

### Q1.4
A developer complains that their application cannot write temporary files to `/tmp` because the disk is full. When you check `df -h /tmp`, it shows 100% utilization, but the root filesystem `/` still has 50GB free. What causes this partition layout behavior?

---

## Section 2: Inodes, Hard Links & Symbolic Links

### Q2.1
An application crashes with the error: `No space left on device`. However, when you run `df -h`, it shows 45GB of free disk space available on the volume. What is the root cause, what command confirms it, and how do you resolve it?

### Q2.2
A production database log file `/var/log/app.log` grew to 30GB, filling up the disk. An engineer deleted the file using `rm -f /var/log/app.log`. `ls` confirms the file is gone, but `df -h` still reports the disk is 100% full. Why did the space not free up, and what command releases the trapped disk space without restarting the server?

### Q2.3
Why does creating a hard link across different disk partitions or mount points fail with `Invalid cross-device link`, whereas a symbolic link succeeds? Explain what happens at the inode layer.

### Q2.4
Why does the Linux kernel strictly forbid users and administrators from creating hard links to directories, whereas symbolic links to directories are permitted?

### Q2.5
What command do you use to find all files on a filesystem that point to the exact same physical data blocks (same inode number) as `/etc/hosts`?

---

## Section 3: Standard Permissions, Ownership & chown

### Q3.1
A user owns a file with permissions `rw-r--r--` (`644`), but when they try to save edits to it, they receive `Permission Denied`. Assuming the filesystem is read-write and no immutable attributes are set, what directory-level permission is causing this?

### Q3.2
Explain why execute (`x`) permission on a directory has a completely different meaning than execute permission on a regular file. What happens if a directory has read (`r`) permission but no execute (`x`) permission?

### Q3.3
What is the exact octal calculation for permissions `rwxr-x---`? Explain the numeric weight of each bit.

### Q3.4
An administrator attempts to change ownership of a directory tree using `chown /var/www :devops`. The command fails with an error. What is wrong with the syntax, and what is the correct command to recursively change the owner to `deploy` and the group to `devops`?

### Q3.5
What is `umask`? If the system default umask is `027`, what will be the default permissions on a newly created regular file and a newly created directory? Show the math.

---

## Section 4: Special Permissions (SUID, SGID & Sticky Bit)

### Q4.1
When inspecting `/etc/passwd` and `/usr/bin/passwd`, which one has the SUID bit set, and why? What catastrophic security hole would open if SUID were set on `/etc/passwd` directly?

### Q4.2
When running `ls -l`, you notice a binary displays `-rwSr-xr-x` with an uppercase `S` instead of a lowercase `s`. What does the uppercase `S` signify, and how do you fix it?

### Q4.3
You have a collaborative directory `/data/engineering` shared among 20 engineers in the `developers` group. When Engineer A creates a file, Engineer B cannot edit it because it belongs to Engineer A's primary personal group. What command and special permission fixes this for all future files?

### Q4.4
Why does `/tmp` have the Sticky Bit (`1000`) enabled? What security and operational chaos would occur in a multi-tenant Linux server if the Sticky Bit were removed from `/tmp`?

### Q4.5
How do you write a single `find` command to scan the entire system for unauthorized SUID files owned by root?

---

## Section 5: User Management & Privilege Escalation (su, sudo, visudo)

### Q5.1
An administrator uses `su` (without the hyphen) to switch from `azureuser` to `root` and installs packages via `pip install`. Afterwards, `azureuser` experiences `Permission Denied` errors running basic daily commands in their own home directory. Explain the exact mechanism of this environment pollution.

### Q5.2
What is the technical and architectural difference between `su -`, `sudo -i`, and `sudo -s`?

### Q5.3
An administrator runs `usermod -G devops alex` to add user `alex` to the `devops` group. Immediately after, `alex` complains they lost `sudo` (`wheel`) and `docker` privileges. What mistake did the administrator make, and what was the correct command?

### Q5.4
A junior admin creates a drop-in sudoers file at `/etc/sudoers.d/custom.conf` with valid syntax. However, `sudo` completely ignores the rules inside. What are the two strict rules of `/etc/sudoers.d/` that were violated?

### Q5.5
Why must `/etc/sudoers` always be edited using `visudo` rather than standard text editors like `vim` or `nano`? What safety mechanisms does `visudo` enforce?

### Q5.6
Write the exact sudoers configuration line to allow user `deployer` to restart and check the status of service `nginx` with root privileges without prompting for a password, while forbidding all other root commands.

---

## Section 6: Process Management, Signals & Load Average

### Q6.1
Your monitoring alert triggers: `Load Average is 35, but CPU utilization is only 4%`.
1. What does Linux Load Average measure beyond CPU-bound processes?
2. What process state are the majority of processes in?
3. What is the root cause on the server?

### Q6.2
A developer notices a process in `ps aux` with state `Z` (`[app] <defunct>`). They run `kill -9 <PID>` repeatedly, but the process does not disappear.
1. Why does `kill -9` fail to remove a zombie process?
2. How do you properly clear the zombie without rebooting the server?
3. When does a zombie process transition from harmless to a critical incident?

### Q6.3
Explain the difference between an Orphan process and a Zombie process. What does PID 1 (`systemd`) do when a parent process crashes before its children?

### Q6.4
What is the difference between signal `15` (`SIGTERM`) and signal `9` (`SIGKILL`)? Why must automated deployment scripts and orchestrators send `SIGTERM` first before `SIGKILL`?

### Q6.5
You update the SSL certificate configuration file for an Nginx web server. You want Nginx to load the new certificate immediately without dropping any active client connections. What signal number or command do you send, and how does the daemon handle it?

### Q6.6
You start a 2-hour database export script `./dump.sh` in an active SSH session. You must shut your laptop and leave immediately. What exact sequence of terminal job control commands do you run to send the script to the background and protect it from terminating when SSH disconnects?

### Q6.7
A user wants to run a batch job with high CPU priority. They attempt `nice -n -10 ./job.sh` and receive `Permission denied`. However, `nice -n 10 ./job.sh` works. Why does the Linux kernel enforce this asymmetric restriction?

### Q6.8
What is `wchan` (Wait Channel)? How does an engineer use `/proc/<PID>/wchan` or `/proc/<PID>/stack` during a production freeze?

---

## Section 7: systemd & Service Supervision

### Q7.1
A newly deployed `.service` unit fails immediately upon `systemctl start`. Running `systemctl status` reveals:
`Process: 4120 ExecStart=/usr/local/bin/worker (code=exited, status=203/EXEC)`
What are the two most likely causes of error `203/EXEC`?

### Q7.2
An engineer edits `/etc/systemd/system/myapp.service` to update a port number, and runs `systemctl restart myapp`. However, the application continues to run on the old port. What mandatory step did the engineer miss, and why?

### Q7.3
A backend application crashes multiple times due to a bad configuration. Running `systemctl start myapp` fails with:
`Failed to start myapp.service: Unit is in a failed state.`
`systemctl status` reports: `failed (Result: start-limit-hit)`.
What caused this state, and what exact two commands are required to allow the service to start again?

### Q7.4
In a `systemd` unit file, what is the exact difference between `After=`, `Wants=`, and `Requires=`? What happens if `redis.service` is listed under `Wants=` and fails to start?

### Q7.5
What does `systemctl enable` do under the hood at the filesystem level? Does running `systemctl enable` start the service in memory?

### Q7.6
What is the difference between `systemctl disable <svc>` and `systemctl mask <svc>`? How does `systemd` implement masking on disk?

### Q7.7
What `journalctl` command flags do you run to inspect live real-time logs for a specific failing unit with error descriptions included?

---

## Section 8: The find Command & Advanced File Searching

### Q8.1
You are troubleshooting a sudden 100% disk utilization alert on `/var`. What exact `find` command do you run to locate all regular files in `/var` larger than 500MB?

### Q8.2
What is the difference between `find /var/log -type f -mtime +30` and `find /var/log -type f -mtime -30`? Explain how the plus and minus prefixes change the evaluation logic.

### Q8.3
Why does `find /usr/bin -perm 4000` return almost zero results, whereas `find /usr/bin -perm -4000` lists all SUID binaries on the system?

### Q8.4
Explain the syntax and components of the `-exec` action in the `find` command:
`find /opt/logs -type f -name "*.log" -mtime +14 -exec gzip {} \;`
What do `{}` and `\;` represent?

### Q8.5
Why must wildcards be quoted when using `-name` (e.g. `-name "*.log"` vs `-name *.log`)? What unexpected behavior occurs if quotes are omitted in a directory containing matching files?
