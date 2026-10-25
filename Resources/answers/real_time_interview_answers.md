# Linux Real-Time Interview Answers

---

## Section 1: Filesystem Hierarchy Standard (FHS) & Directory Structure

### Q1.1 Answer
- **/bin**: Essential user command binaries required for single-user recovery mode and system booting (e.g., `ls`, `cp`, `cat`). Available to all users.
- **/sbin**: Essential system administration binaries required for booting, restoring, or repairing the system (e.g., `fdisk`, `reboot`, `iptables`). Intended for root.
- **/usr/bin** & **/usr/sbin**: Non-essential user and admin binaries installed by the OS package manager after the system is booted (e.g., `curl`, `python3`, `nginx`).
- **Why `/sbin` exists separately**: Historically, `/sbin` was on the root partition so sysadmins could repair an unbootable machine before `/usr` was mounted. In modern RHEL/CentOS systems, `/bin` and `/sbin` are symlinks pointing into `/usr/bin` and `/usr/sbin`, but the directory distinction remains in `$PATH` to separate root administration commands from regular user tools.

### Q1.2 Answer
- **`/opt`** is the correct FHS location for third-party, proprietary, or self-contained application packages that are installed manually outside of the native package manager (e.g. `/opt/datadog-agent`, `/opt/google/chrome`). All binaries, libraries, and configs live bundled in that single directory.
- **`/usr/local`** is intended for software compiled and built locally from source on that specific host (e.g., running `./configure && make && make install`), which distributes its files into `/usr/local/bin`, `/usr/local/lib`, etc.

### Q1.3 Answer
- **/tmp**: Temporary files designed for short-lived data. In modern systems, `/tmp` is often mounted as `tmpfs` (RAM-backed storage) or cleaned up automatically on system reboot or after 10 days of inactivity by `systemd-tmpfiles`.
- **/var/tmp**: Temporary files designed to survive a system reboot. Cleaned up only after 30 days of inactivity. Used by backup jobs, large downloads, or software installations that need persistent temporary cache across reboots.

### Q1.4 Answer
- `/tmp` is mounted on its own dedicated partition or a `tmpfs` RAM mount rather than sharing space with the root filesystem `/`.
- Run `df -h /tmp` to inspect the dedicated mount. If `/tmp` is full, inspect large temporary files with `find /tmp -type f -size +50M` and clear them, or resize the partition/tmpfs allocation.

---

## Section 2: Inodes, Hard Links & Symbolic Links

### Q2.1 Answer
- **Root Cause:** **Inode Exhaustion**. Even if gigabytes of block storage remain, if all allocated inodes are consumed (often by millions of tiny session files or mail queues), the kernel cannot create any new file metadata.
- **Command to Confirm:**
  ```bash
  df -i
  ```
  Look for a filesystem showing `100%` under `IUse%`.
- **Resolution:**
  1. Find directories containing abnormal file counts:
     ```bash
     find / -xdev -printf '%h\n' | sort | uniq -c | sort -k 1 -n | tail -20
     ```
  2. Purge old small files using `find /var/spool/clientmqueue -type f -delete`.

### Q2.2 Answer
- **Root Cause:** The file was unlinked from the directory tree by `rm`, but an active running process still has an **open file descriptor** pointing to that inode. The Linux kernel will not free physical disk blocks until both the directory link count is 0 **AND** all open file descriptors are closed.
- **Identify the Process:**
  ```bash
  lsof +L1
  # Or: lsof | grep deleted
  ```
- **Release the Space without Rebooting:**
  Truncate the open file descriptor through the `/proc` filesystem:
  ```bash
  > /proc/<PID>/fd/<FD_NUMBER>
  ```
  This immediately zeros out the trapped disk blocks while keeping the application process alive and running.

### Q2.3 Answer
- **Hard Links:** A hard link is merely another directory entry pointing directly to an existing **inode number** on the same physical filesystem table. Because each disk partition has its own independent inode table, an inode number (e.g. `12345`) only exists on its specific filesystem. Crossing partitions would cause inode collisions.
- **Symbolic Links:** A symlink is an independent new file with its own inode whose data block literally stores the string path (e.g. `/mnt/disk2/file.txt`) of the target file. The kernel resolves the string path across any mount point.

### Q2.4 Answer
- **Root Cause:** Creating hard links to directories would introduce **infinite directory cycles and loops** (e.g. `/dir/a/b/c` pointing back to `/dir/a`).
- **Consequences:** File system traversal algorithms (`find`, `du`, `tar`, backup engines, and filesystem tree checkers `fsck`) would enter infinite recursive loops, leading to memory exhaustion and kernel crashes.
- Symbolic links to directories are allowed because `fsck` and utilities can easily detect the symlink file type (`l`) and avoid circular traversal.

### Q2.5 Answer
1. Check the inode number of `/etc/hosts`:
   ```bash
   ls -i /etc/hosts
   ```
2. Search that specific filesystem for that inode:
   ```bash
   find / -inum <INODE_NUMBER> 2>/dev/null
   # Or directly:
   find / -samefile /etc/hosts 2>/dev/null
   ```

---

## Section 3: Standard Permissions, Ownership & chown

### Q3.1 Answer
- **Root Cause:** To save edits to a file, text editors create a temporary file and replace the original. This requires **Write (`w`) and Execute (`x`) permissions on the parent directory** where the file resides.
- If the user has `rw-` on the file but only `r--` or `r-x` (no write) on the directory, they cannot save the modified file.

### Q3.2 Answer
- **On a Regular File:**
  - `r` = Read content.
  - `w` = Modify content.
  - `x` = Execute as binary or script.
- **On a Directory:**
  - `r` = List filenames inside the directory (`ls`).
  - `w` = Create, delete, or rename files inside the directory (provided `x` is set).
  - `x` = Enter / traverse the directory (`cd`) and access file metadata (inodes) inside.
- **If a directory has `r` but no `x`:** You can see the list of file names (`ls`), but you cannot `cd` into it, cannot view file sizes/permissions (`ls -l` shows `???`), and cannot read file contents inside it.

### Q3.3 Answer
- `r = 4`, `w = 2`, `x = 1`, `- = 0`
- Owner: `rwx` = $4 + 2 + 1 = 7$
- Group: `r-x` = $4 + 0 + 1 = 5$
- Others: `---` = $0 + 0 + 0 = 0$
- **Total Octal:** **`750`**

### Q3.4 Answer
- **Syntax Error:** The target path must always come **LAST**. The user attempted `chown /path :group`.
- **Correct Recursive Command:**
  ```bash
  chown -R deploy:devops /var/www
  ```

### Q3.5 Answer
- **`umask`** (User Mask) is a 3-digit or 4-digit octal filter subtracted from default initial creation permissions.
  - Maximum default base for files: `666` (`rw-rw-rw-`, execution is never granted by default).
  - Maximum default base for directories: `777` (`rwxrwxrwx`).
- **Calculation for umask `027`:**
  - File: `666 - 027` (applying bitwise AND NOT) $\rightarrow$ `640` (`rw-r-----`).
  - Directory: `777 - 027` $\rightarrow$ `750` (`rwxr-x---`).

---

## Section 4: Special Permissions (SUID, SGID & Sticky Bit)

### Q4.1 Answer
- **/usr/bin/passwd** has the SUID bit set (`-rwsr-xr-x`). It needs SUID because standard users must update their own password hash stored in `/etc/shadow`, which is strictly owned by `root:root` with mode `000` or `600`.
- **/etc/passwd** is a plain text configuration file (`-rw-r--r--`). Setting SUID on a plain text configuration file is meaningless because text files cannot be executed by the kernel. Furthermore, if `/etc/passwd` were made writable by non-root users, anyone could edit UID 0 and gain instant root access.

### Q4.2 Answer
- **Uppercase `S`:** Signifies that the SUID bit is active, **BUT the underlying execute bit (`x`) is missing** for the owner. The binary cannot be executed.
- **Lowercase `s`:** Signifies SUID is set **AND** execute (`x`) permission is present.
- **Fix:** Add execute permission to the owner:
  ```bash
  chmod u+x /path/to/binary
  ```

### Q4.3 Answer
- **Solution:** Apply the **SGID bit (`2000`)** on the directory:
  ```bash
  chown :developers /data/engineering
  chmod 2775 /data/engineering
  ```
- **How it works:** Any new file or sub-directory created inside `/data/engineering` will automatically inherit group ownership of `developers` rather than the primary group of the individual creator.

### Q4.4 Answer
- **Purpose:** `/tmp` is world-writable (`777`) so any process can write temporary files. The Sticky Bit (`1000` / `+t`) ensures that **ONLY the file owner (or `root`) can delete or rename their own files**.
- **Without Sticky Bit:** Any user could delete other users' lock files, databases, or application sockets, causing immediate Denial of Service (DoS) and security compromise across the server.

### Q4.5 Answer
```bash
find / -type f -perm -4000 -user root 2>/dev/null
```

---

## Section 5: User Management & Privilege Escalation (su, sudo, visudo)

### Q5.1 Answer
- Running `su` (without dash) retains the original user's environment variables (`$HOME=/home/azureuser`, `$PWD`, etc.) while switching UID to 0 (root).
- When `pip` runs, it writes cache files, download locks, and directories inside `/home/azureuser/.cache/` **owned by `root:root` with mode `700`**.
- When `azureuser` logs in later, their tools cannot access their own home directory cache, crashing with `Permission Denied`.
- **Resolution:** Run `chown -R azureuser:azureuser /home/azureuser/` and always use `su -` or `sudo -i`.

### Q5.2 Answer
- **`su -`**: Switches to root as a full login shell, requiring the **root password**. Changes working directory to `/root` and reinitializes environment.
- **`sudo -i`**: Starts an interactive login shell as root using **your own password**. Sets `$HOME=/root` and reads root's login profile files cleanly.
- **`sudo -s`**: Starts a non-login shell as root. It grants root privileges but keeps your current directory and preserves parts of your existing environment.

### Q5.3 Answer
- **The Mistake:** The `-G` flag without `-a` (append) **replaces** all existing supplementary groups with only the groups specified. `alex` was removed from `wheel` and `docker`.
- **The Correct Command:**
  ```bash
  usermod -aG devops alex
  ```

### Q5.4 Answer
1. **Filename Rule:** Filenames in `/etc/sudoers.d/` **must not contain a period (`.`)**. The file was named `custom.conf`, so `sudo` completely ignores it.
2. **Permissions Rule:** Drop-in files must have strict permissions **`0440`** (or `0400`) owned by `root:root`. If writable by group or world, `sudo` rejects all sudoers files.

### Q5.5 Answer
- **Syntax Validation:** Upon save, `visudo` verifies grammar and syntax in memory. If an error exists, it prevents saving, protecting against accidental lockouts.
- **Concurrency Locking:** Locks `/etc/sudoers` against simultaneous conflicting edits by multiple admins or automation scripts.

### Q5.6 Answer
```text
deployer ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

---

## Section 6: Process Management, Signals & Load Average

### Q6.1 Answer
1. **Load Average Formula:** $\text{Load} = (\text{Processes in } R) + (\text{Processes in } D)$. In Linux, it includes both CPU-bound tasks and tasks in **Uninterruptible Sleep (`D` state)**.
2. **Process State:** State **`D`** (`TASK_UNINTERRUPTIBLE`).
3. **Root Cause:** I/O bottleneck—processes are blocked waiting on slow/failing local disk blocks, SAN storage, or an unresponsive NFS mount.

### Q6.2 Answer
1. **Why `kill -9` fails:** A zombie process is already dead (0 CPU, 0 RAM, no code executing). It is merely an uncollected entry in the kernel's process table waiting for its parent to call `wait()`. You cannot kill a dead process.
2. **Proper Cleanup:**
   - Send `kill -CHLD <PPID>` to nudge the parent to read the exit status.
   - If parent is hung, kill the parent process: `kill -15 <PPID>` (or `kill -9 <PPID>`). Once the parent dies, PID 1 adopts the zombie and immediately cleans it up.
3. **When it becomes Critical:** When thousands of zombies accumulate, exhausting the OS Process Table (`/proc/sys/kernel/pid_max`), preventing the kernel from spawning any new processes.

### Q6.3 Answer
- **Orphan Process:** A process whose parent died while the child was **still alive and executing**. The Linux kernel automatically re-parents the orphan to **PID 1 (`systemd`)**, which supervises it until it finishes.
- **Zombie Process:** A child process that has **terminated execution**, but its parent process is still alive and has not reaped its exit code.

### Q6.4 Answer
- **`SIGTERM (15)`**: Graceful termination request. The application can catch or intercept the signal, complete ongoing transactions, flush buffers to disk, close database connections, and exit cleanly.
- **`SIGKILL (9)`**: Forceful kill executed directly by the Linux kernel. Cannot be caught, blocked, or ignored by the process. Does not allow clean-up and can cause database/file corruption. Orchestrators send `SIGTERM`, wait a grace period (e.g. 30s), and only use `SIGKILL` as a last resort.

### Q6.5 Answer
- **Signal:** **Signal 1 (`SIGHUP`)** or command `systemctl reload nginx` (or `kill -HUP <PID>`).
- **Handling:** Nginx catches `SIGHUP`, parses the new configuration, spawns new worker processes with the new SSL certs, and gracefully drains existing connections on the old workers before terminating them. Zero downtime.

### Q6.6 Answer
1. Pause execution: Press **`Ctrl + Z`** (sends `SIGTSTP`).
2. Resume in background: Type **`bg %1`** (sends `SIGCONT`).
3. Disown from shell: Type **`disown -h %1`** (removes job from shell's SIGHUP table so it survives terminal logout).

### Q6.7 Answer
- **Nice Scale:** `-20` (highest priority) to `+19` (lowest priority).
- **Security Reason:** If unprivileged users could assign negative nice values (`-20`), a rogue user could launch a CPU-intensive script that starves system daemons and other users, causing a Denial of Service (DoS). Only `root` can increase priority.

### Q6.8 Answer
- **`wchan` (Wait Channel):** Displays the exact kernel function name where a sleeping or blocked process is currently parked.
- **Usage:**
  ```bash
  cat /proc/<PID>/wchan
  # Or:
  cat /proc/<PID>/stack
  ```
  If it shows `nfs_wait_bit`, the process is stuck on NFS. If `io_schedule`, it is waiting on local disk I/O.

---

## Section 7: systemd & Service Supervision

### Q7.1 Answer
1. The executable file path specified in `ExecStart=` does not exist.
2. The file exists, but lacks execute permissions (`+x`) for the configured service `User=`.

### Q7.2 Answer
- **Missed Step:**
  ```bash
  systemctl daemon-reload
  ```
- **Why:** `systemd` caches unit files in memory during boot. When you edit a unit file on disk, `systemd` continues using its in-memory copy until `daemon-reload` forces it to re-parse `/etc/systemd/system/`.

### Q7.3 Answer
- **Root Cause:** The service crashed repeatedly in a short time frame, exceeding `StartLimitBurst` (default 5 times in 10s). `systemd` locks the unit to prevent 100% CPU crash loops.
- **Two Recovery Commands:**
  ```bash
  systemctl reset-failed myapp
  systemctl start myapp
  ```

### Q7.4 Answer
- **`After=`**: Controls **ordering only**. Start this unit after the specified service has started. Does not create a dependency.
- **`Wants=`**: **Soft dependency**. Tries to start the service, but if it fails, this unit starts anyway.
- **`Requires=`**: **Hard dependency**. If the required service fails to start or crashes later, this unit will fail or be killed.
- **If `redis.service` fails under `Wants=`:** Your service will still successfully start and run.

### Q7.5 Answer
- **Filesystem Action:** It creates a **symbolic link** from `/etc/systemd/system/multi-user.target.wants/<service>.service` pointing to the unit file.
- **Execution:** Running `systemctl enable` **does NOT** start the service in memory. It only ensures it starts on subsequent boots. (Use `enable --now` to do both).

### Q7.6 Answer
- **`disable`**: Deletes the symlink from `multi-user.target.wants/`. The service will not start automatically on boot, but can still be started manually with `systemctl start`.
- **`mask`**: Symlinks the unit file in `/etc/systemd/system/<svc>.service` to **`/dev/null`**. The service **cannot be started under any circumstances** (not manually, not at boot, and not as a dependency).

### Q7.7 Answer
```bash
journalctl -u <service> -f -xe
```
- `-u`: Unit name.
- `-f`: Follow logs in real time.
- `-x`: Add explanatory help text from catalog.
- `-e`: Jump immediately to end of pager.

---

## Section 8: The find Command & Advanced File Searching

### Q8.1 Answer
```bash
find /var -type f -size +500M
```

### Q8.2 Answer
- **`+30`**: Modified **GREATER than 30 days ago** (older files for archiving/deletion).
- **`-30`**: Modified **LESS than 30 days ago** (recent files modified within the last month).

### Q8.3 Answer
- **`-perm 4000`** (no minus): Exact match. Searches for a file whose **exact and only permissions** are `---S------` (no read/write/execute for anyone), which rarely exists.
- **`-perm -4000`** (with minus): Bitmask match. Searches for files that have **AT LEAST the SUID bit set** regardless of user/group read/write/execute permissions (e.g. `4755` matches).

### Q8.4 Answer
- **`{}`**: Placeholder that `find` automatically replaces with the current matching file's path.
- **`\;`**: Escaped semicolon indicating the end of the command to execute for each file.

### Q8.5 Answer
- **Why quote:** Wildcard characters (`*`, `?`) must be passed literally to `find`.
- **If unquoted:** The shell expands `*.log` into all matching filenames in the **current working directory** before passing them to `find`. If the current directory has `a.log` and `b.log`, the shell runs `find /var/log -name a.log b.log`, causing syntax errors or incorrect searches.
