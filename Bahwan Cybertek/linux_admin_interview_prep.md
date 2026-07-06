# Linux Administration, Configuration & Performance Tuning — Interview Prep

> A zero-to-interview-ready guide. Part 1 builds the mental model everything else depends on — read it fully even if some of it feels obvious.

---

## How to use this guide

Each question has:
- **Why they ask this** — what's actually being tested
- **Simple explanation** — the plain-English mental model
- **Deeper detail** — the technical depth for follow-ups
- **What to say** — a spoken-style answer you can adapt

Troubleshooting scenarios add a **step-by-step approach**, **exact commands**, and a **mental model** you can reapply to any scenario you've never seen — because interviewers frequently improvise a new scenario specifically to test how you *think*, not what you memorized.

---

## PART 1: Foundations (the mental model everything else builds on)

### 1.1 What is Linux, actually?

**Linux** is the **kernel** — the core program that talks directly to hardware (CPU, memory, disks, network cards) and decides how to share those resources among running programs. What most people call "Linux" (Ubuntu, RHEL, CentOS, Debian, Amazon Linux) is technically a **distribution**: the kernel plus a bundle of tools, package managers, and default configuration built around it.

**Mental picture — think in layers:**

```
┌─────────────────────────────────────┐
│   Applications (nginx, mysql, etc.)  │   ← userspace: what you and users interact with
├─────────────────────────────────────┤
│   Shell & system tools (bash, systemd, coreutils) │
├─────────────────────────────────────┤
│   Kernel (process scheduling, memory management,  │
│   filesystem, networking stack, device drivers)   │   ← kernel space: talks to hardware
├─────────────────────────────────────┤
│   Hardware (CPU, RAM, Disk, NIC)     │
└─────────────────────────────────────┘
```

**Why this layering matters for troubleshooting:** almost every problem you'll be asked about lives at one of these layers, and the fix — and the tools you use — differ completely depending on which layer is actually broken. A "slow server" could be a hardware limit, a kernel-level resource exhaustion (memory, file descriptors), or an application misconfiguration. Good troubleshooting always starts by figuring out *which layer* you're actually dealing with — you'll see this pattern repeat throughout Part 4.

### 1.2 "Everything is a file" — the single most important Linux idea

In Linux, almost everything is represented as a file you can read, write, or list — not just documents. Devices (`/dev/sda` for a disk), running process info (`/proc/1234/`), kernel settings (`/proc/sys/...`, `/sys/...`), and even network sockets are exposed through the filesystem. This is *why* so much of Linux administration is just "read/write the right file" rather than needing a special API or GUI for every subsystem.

This matters practically: when you don't know a command for something, you can very often find the answer by looking directly at a file under `/proc` or `/sys`.

### 1.3 The Filesystem Hierarchy (know what lives where)

| Path | What's there | Why it matters |
|---|---|---|
| `/etc` | System-wide configuration files | Almost every "how do I configure X" question means "which file under /etc do I edit." |
| `/var` | Variable data: logs (`/var/log`), spool/queue data, caches | Where things *grow* over time — the first place to check when disk fills up. |
| `/proc` | A **virtual** filesystem — a live window into kernel and process state (not real files on disk) | Read `/proc/cpuinfo`, `/proc/meminfo`, `/proc/<pid>/status` directly to inspect live system state. |
| `/sys` | Another virtual filesystem exposing kernel/device/driver settings, often tunable | Used for hardware/kernel tuning parameters. |
| `/home` | User home directories | Personal files, per-user config. |
| `/usr` | Installed programs and their supporting files (the bulk of installed software) | `/usr/bin`, `/usr/lib`, etc. |
| `/bin`, `/sbin` | Essential command binaries (some distros symlink these into `/usr/bin`, `/usr/sbin`) | Core system commands need to exist even before `/usr` is mounted, historically. |
| `/tmp` | Temporary files, usually cleared on reboot | Common location for scratch space; sometimes mounted as `tmpfs` (RAM-backed). |
| `/root` | The root user's home directory (not to be confused with `/`, the top-level root of the whole filesystem) | Easy interview trip-up: `/` is not the same as `/root`. |
| `/boot` | Kernel image, bootloader config | Relevant when troubleshooting boot failures. |
| `/dev` | Device files representing hardware (disks, terminals, etc.) | `/dev/sda`, `/dev/null`, `/dev/zero` — hardware and virtual devices as files. |

### 1.4 Processes: the other core mental model

A **process** is a running instance of a program. Every process has:
- A **PID** (Process ID) — unique identifier.
- A **PPID** (Parent Process ID) — who started it. Every process except the very first (`PID 1`, usually `systemd`) has a parent — this forms a **process tree**.
- A **state** — Running (R), Sleeping (S, waiting for something like I/O or a timer — this is normal and common), Uninterruptible Sleep (D, waiting on I/O in a way that *can't* be interrupted — important for troubleshooting, see Part 4), Stopped (T), Zombie (Z, finished but not yet cleaned up by its parent).
- **Resource usage**: CPU time, memory (RSS = resident set size = actual RAM used; VSZ = virtual memory size, which can be much larger and less meaningful on its own).

**Why the process tree matters:** when a process dies unexpectedly or misbehaves, understanding "who is its parent, and what does the parent do when a child dies" (e.g., `systemd` will typically restart a managed service) is often the actual troubleshooting answer.

### 1.5 Users, permissions, and why they exist

Every file has an **owner** (a user) and a **group**, and three sets of permissions — for the owner, the group, and everyone else — each of which can allow **r**ead, **w**rite, and e**x**ecute.

```
-rwxr-xr--  1 alice devteam  1240 Jul  5 10:00 deploy.sh
 │└┬┘└┬┘└┬┘
 │ │  │  └─ others: r-- (read only)
 │ │  └──── group (devteam): r-x (read + execute)
 │ └─────── owner (alice): rwx (read + write + execute)
 └───────── file type (- = regular file, d = directory, l = symlink)
```

This exists so multiple users can safely share one machine — permissions are the enforcement mechanism preventing one user (or a compromised process running as that user) from reading/modifying another user's or the system's files. **Root** (UID 0) bypasses almost all permission checks — which is exactly why "least privilege" (not running things as root unless truly necessary) is a core security principle you should mention when relevant.

### 1.6 The init system: systemd (what starts everything, and keeps it running)

When a Linux machine boots, the kernel starts exactly one process: **PID 1**. On virtually every modern distribution, that's **systemd**. systemd's job is to start all other system services in the right order, keep them running (restarting them if they crash, if configured to), and manage dependencies between them (e.g., "don't start the web server until the network is up").

Services are defined as **unit files** (`.service`, `.socket`, `.timer`, `.mount`, etc.), and you interact with systemd primarily through `systemctl` (control services) and `journalctl` (read logs). This is foundational — a huge fraction of real-world Linux admin work is "a service won't start / keeps restarting / isn't enabled on boot," and it always routes through systemd.

---

## PART 2: Linux Administration & Configuration — Interview Q&A

### Q1. Explain Linux file permissions in depth — including SUID, SGID, and the sticky bit.

**Why they ask this:** Baseline check that you understand the permission model beyond just `chmod 755`.

**Simple explanation:** Permissions are read (r=4), write (w=2), execute (x=1) for three groups: owner, group, others. You add the numbers to get a mode like `755` = owner has 7 (rwx), group has 5 (r-x), others have 5 (r-x).

**Deeper detail — the special bits people forget:**
- **SUID (Set User ID, shown as `s` in the owner's execute position)**: when set on an executable, it runs with the *file owner's* privileges, not the privileges of whoever ran it. Classic example: `/usr/bin/passwd` is owned by root and has SUID set, so a regular user can run it to update `/etc/shadow` (which normal users can't write directly) — the program runs briefly "as root" just for that controlled task.
- **SGID (Set Group ID)**: similar, but for group. On a directory, SGID has a special extra meaning: any new file created inside inherits the *directory's* group, rather than the creating user's primary group — very useful for shared team directories.
- **Sticky bit** (shown as `t` in others' execute position): on a directory, it means only the file's owner (or root) can delete/rename files inside it, even if others have write permission on the directory. Classic example: `/tmp` has the sticky bit set — everyone can create files there, but you can't delete someone else's file.

**Commands:**
```bash
chmod 4755 script.sh      # sets SUID (the leading 4)
chmod 2755 /shared/dir    # sets SGID (the leading 2)
chmod 1777 /tmp           # sets sticky bit (the leading 1)
chmod u+s script.sh       # symbolic form for SUID
```

**What to say:** "Standard permissions cover read/write/execute for owner, group, others. SUID makes a program run as its owner regardless of who invokes it — used carefully, like `passwd`. SGID on a directory makes new files inherit the directory's group, which is great for shared team folders. The sticky bit on a directory like `/tmp` stops users from deleting each other's files even though everyone can write there."

---

### Q2. What's the difference between hard links and symbolic (soft) links?

**Simple explanation:** Both let one file be "reachable" from two different names/paths. A **symbolic link** is like a shortcut — it's a separate small file that just points to a path. A **hard link** is a second name for the exact same underlying data — there's no "original" and "copy," they're both equally real.

**Deeper detail:**
- Every file on disk is really an **inode** (a data structure holding metadata + pointers to actual data blocks) plus one or more directory entries (names) pointing to that inode.
- A **hard link** is just another directory entry pointing to the *same inode*. Deleting one name doesn't delete the data — the data is only actually freed when the *last* hard link (the inode's link count) is removed.
- A **symbolic link** is its own inode, containing a *path string* pointing to another file. If the target is deleted or moved, the symlink breaks ("dangling symlink").
- Hard links cannot cross filesystems/partitions (since inodes are only unique within one filesystem) and cannot link to directories (to avoid loops). Symlinks can do both.

**Commands:**
```bash
ln target.txt hardlink.txt          # hard link
ln -s target.txt symlink.txt        # symbolic link
ls -li file.txt                     # -i shows the inode number, to compare
```

**What to say:** "A hard link is another name for the same inode — same data, no concept of 'original.' A symlink is a separate tiny file containing a path to the target, similar to a shortcut, and it breaks if the target moves. I use symlinks for things like pointing `/etc/localtime` at a timezone file, and I've seen hard links used for space-efficient backups where unchanged files share the same inode across backup snapshots."

---

### Q3. Explain the Linux boot process at a high level.

**Simple explanation:** Power on → firmware finds a bootloader → bootloader loads the kernel → kernel initializes hardware and starts PID 1 (systemd) → systemd starts all configured services in dependency order → you get a login prompt.

**Deeper detail (the stages, in order):**
1. **BIOS/UEFI** — firmware does a hardware self-test (POST) and hands off to a bootloader found on a bootable disk.
2. **Bootloader (GRUB, typically)** — presents a menu (if configured), loads the selected kernel image and an **initramfs** (a small temporary root filesystem with just enough drivers to mount the real root filesystem — needed, for example, if the real root disk requires a driver or is on LVM/RAID/encrypted storage).
3. **Kernel initialization** — the kernel takes over, initializes detected hardware, mounts the real root filesystem, and starts **PID 1**.
4. **PID 1 (systemd)** — reads its configuration and starts units in dependency order, working toward a **target** (systemd's concept of a "boot state," e.g., `multi-user.target` for normal multi-user mode without a GUI, `graphical.target` if a desktop environment is expected).
5. **Login prompt / getty** — once required services are up, a login prompt (console or SSH) becomes available.

**What to say:** "Firmware hands off to a bootloader like GRUB, which loads the kernel and an initramfs. The kernel initializes hardware, mounts the real root filesystem, and starts systemd as PID 1. systemd then brings the system up to a target state — like multi-user or graphical — by starting units in the order their dependencies require."

---

### Q4. What is systemd, and how do you manage services with it?

**Simple explanation:** systemd is the first program the kernel starts (PID 1); it's responsible for starting, stopping, and supervising every other service on the system, and you control it with the `systemctl` command.

**Deeper detail — the commands you must know cold:**
```bash
systemctl status nginx        # is it running? recent log lines, PID, uptime
systemctl start nginx         # start now (until reboot, unless also enabled)
systemctl stop nginx          # stop now
systemctl restart nginx       # stop then start
systemctl reload nginx        # ask the service to re-read config without a full restart (if it supports it)
systemctl enable nginx        # make it start automatically on future boots
systemctl disable nginx       # remove it from auto-start on boot
systemctl enable --now nginx  # enable AND start in one command
systemctl is-enabled nginx    # check if it's set to start on boot
systemctl list-units --type=service --state=running   # see everything currently running
systemctl daemon-reload       # tell systemd to re-read unit files after you've edited one
```
- **Important distinction**: `start`/`stop` affect the *current* running state; `enable`/`disable` affect whether it starts *automatically on the next boot*. A service can be running-but-not-enabled (won't survive a reboot) or enabled-but-not-running (will start next boot, but isn't up right now) — interviewers love this distinction because it's a very real, very common source of "wait, why didn't it come back up after the reboot" incidents.
- A **unit file** (e.g., `/etc/systemd/system/myapp.service`) declares `[Unit]` (description, dependencies via `After=`/`Requires=`), `[Service]` (how to start it, restart policy via `Restart=on-failure`, the user to run as), and `[Install]` (what target it attaches to when enabled).

**What to say:** "systemd is PID 1 and supervises every other service. `start`/`stop`/`restart` control the current running state; `enable`/`disable` control whether it comes up automatically after a reboot — those are independent, and mixing them up is a classic cause of 'it worked, then the server rebooted and it didn't come back.' After editing a unit file directly, you need `daemon-reload` before systemd will notice the change."

---

### Q5. How do you read and manage logs on a modern Linux system?

**Simple explanation:** Modern systemd-based systems primarily use `journald`, a binary log store you query with `journalctl`. Many applications still also write plain-text logs under `/var/log/`.

**Deeper detail — key commands:**
```bash
journalctl -u nginx              # logs for just the nginx unit
journalctl -u nginx -f           # follow live, like `tail -f`
journalctl -u nginx --since "10 minutes ago"
journalctl -p err                # only priority "error" and above
journalctl -b                    # logs since the current boot only
journalctl -b -1                 # logs from the *previous* boot (great for post-crash investigation)
journalctl --disk-usage          # how much disk space the journal itself is consuming
```
- **Log rotation**: plain-text logs in `/var/log` are managed by `logrotate`, configured under `/etc/logrotate.d/`, which periodically compresses/archives/deletes old logs so they don't fill the disk forever. journald has its own size-capping config (`/etc/systemd/journald.conf`, e.g., `SystemMaxUse=`).
- **Centralized logging**: in production, logs are typically also shipped off-box (rsyslog forwarding, or agents like Fluentd/Filebeat into a central system like ELK/Splunk/CloudWatch) — because a server that's crashed or had its disk fill up is exactly the moment local-only logs become hardest to retrieve.

**What to say:** "For systemd-managed services I go straight to `journalctl -u <service>`, using `-f` to follow live and `-b -1` to look at the previous boot after a crash. Plain-text logs under `/var/log` are still common for many applications and are kept from growing unbounded by `logrotate`. In production I'd also expect logs to be shipped to a central system, since local logs are the least reliable exactly when a box is in trouble."

---

### Q6. How does Linux networking configuration work — interfaces, routing, DNS?

**Simple explanation:** A network interface (like `eth0` or `ens5`) needs an IP address to send/receive traffic; a routing table decides which interface/gateway to use for a given destination; DNS translates names to IP addresses.

**Deeper detail — commands and where config lives:**
```bash
ip addr show           # (modern) show interfaces and their IP addresses — replaces old `ifconfig`
ip route show           # show the routing table — which gateway handles which destination
ip link show             # show interface status (up/down)
ss -tulnp               # show listening ports and established connections — replaces old `netstat`
cat /etc/resolv.conf     # DNS resolver configuration (which DNS servers to query)
dig example.com          # query DNS directly, see the full resolution chain
```
- Persistent network configuration lives differently by distro: `netplan` (Ubuntu, YAML-based, generates NetworkManager or systemd-networkd config), `/etc/sysconfig/network-scripts/` (older RHEL/CentOS), or NetworkManager's own config for desktop-oriented systems.
- **Default gateway**: the router traffic is sent to when the destination isn't on the local subnet — shown as the `default via <ip>` line in `ip route show`.
- **DNS resolution order** is typically controlled by `/etc/nsswitch.conf` (the `hosts:` line) — commonly checking `/etc/hosts` first, then DNS.

**What to say:** "I use `ip addr` and `ip route` as the modern replacements for `ifconfig`/old `route`, `ss` instead of `netstat` for socket/port info, and `dig` for DNS debugging. Persistent config location depends on the distro — netplan on Ubuntu, network-scripts on older RHEL — but the live, in-kernel state is always inspectable through the `ip` command regardless of how it was configured."

---

### Q7. How do Linux firewalls work — iptables vs. firewalld vs. nftables?

**Simple explanation:** A firewall decides, per-packet, whether to allow, drop, or reject network traffic based on rules (source/destination IP, port, protocol). On Linux, this is enforced inside the kernel via a framework called **netfilter**; `iptables`, `nftables`, and `firewalld` are different tools/front-ends for configuring it.

**Deeper detail:**
- **iptables** — the classic tool; rules are organized into **chains** (`INPUT` for incoming traffic, `OUTPUT` for outgoing, `FORWARD` for traffic passing through as a router) within **tables** (`filter` is the default/most common; `nat` for address translation). Rules are evaluated top-to-bottom, first match wins (unless it's a non-terminating rule).
```bash
iptables -L -n -v          # list current rules, numeric (no DNS lookups), verbose (with counters)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # append a rule: allow inbound TCP on port 22
```
- **firewalld** — a higher-level, "zone"-based abstraction (e.g., `public`, `internal`) commonly used on RHEL/CentOS/Fedora; you manage services/ports per zone rather than raw rule chains, and changes can be applied without dropping existing connections.
```bash
firewall-cmd --list-all
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload
```
- **nftables** — the modern successor to iptables, meant to eventually replace it, with a cleaner rule syntax and better performance; many distros now translate `iptables` commands to nftables under the hood for compatibility.

**What to say:** "All of these ultimately configure netfilter in the kernel. iptables uses chains and tables evaluated in order, first match wins. firewalld is a friendlier zone-based abstraction commonly on RHEL-family systems, good for dynamic environments since you don't manage raw ordered rules directly. nftables is the modern replacement for iptables with better performance and syntax. Day to day, I'd check `iptables -L -n -v` or `firewall-cmd --list-all` depending on the distro to see what's actually being enforced."

---

### Q8. How does cron work, and what's the difference between a user crontab and system cron?

**Simple explanation:** `cron` is a daemon that runs commands automatically on a schedule you define (e.g., "every day at 2 AM"). Each user can have their own schedule (crontab), and there are also system-wide schedule locations.

**Deeper detail:**
```bash
crontab -l          # list current user's cron jobs
crontab -e          # edit current user's cron jobs
crontab -u alice -l # list another user's crontab (needs privilege)
```
Cron syntax: `minute hour day-of-month month day-of-week command`
```
0 2 * * *  /usr/local/bin/backup.sh     # every day at 02:00
*/15 * * * * /usr/local/bin/healthcheck.sh   # every 15 minutes
```
- **System-wide** cron jobs can also be dropped as files into `/etc/cron.d/`, or as scripts into `/etc/cron.daily/`, `/etc/cron.hourly/`, etc. (run by `run-parts`).
- **Common real-world gotcha**: cron jobs run with a *minimal environment* — not your interactive shell's `PATH` or environment variables. A script that works fine when you run it manually can fail silently under cron because it can't find a binary that was only in your interactive shell's `PATH`. Always use full absolute paths inside cron scripts, and explicitly set any needed environment variables at the top of the crontab or inside the script.
- **systemd timers** are the modern alternative to cron, offering better logging (via journald) and dependency management — increasingly preferred for anything beyond simple periodic tasks.

**What to say:** "cron reads per-user crontabs plus system-wide entries under `/etc/cron.d` and the `cron.daily/hourly` directories, run via `run-parts`. The most common real-world gotcha is that cron jobs run with a minimal environment, not your login shell's `PATH`, so scripts that work interactively can fail silently under cron — I always use absolute paths and explicitly set environment variables in cron scripts. For anything more complex I'd reach for a systemd timer instead, since it integrates with journald logging and unit dependencies."

---

### Q9. How does SSH work, and how do you set up key-based authentication?

**Simple explanation:** SSH lets you securely log into a remote machine over an encrypted connection. Key-based authentication replaces typing a password with a cryptographic key pair — a **private key** you keep secret on your machine, and a **public key** you place on the server; the server can verify you hold the matching private key without the key ever being transmitted.

**Deeper detail:**
```bash
ssh-keygen -t ed25519 -C "my-key"     # generates a private/public key pair (ed25519 is modern/preferred over older RSA)
ssh-copy-id user@server                # copies your public key to the server's ~/.ssh/authorized_keys
ssh -i ~/.ssh/id_ed25519 user@server   # connect using a specific private key
```
- The server checks the presented public key against entries in `~/.ssh/authorized_keys` for that user; successful auth proves you hold the corresponding private key via a cryptographic challenge, not by sending the key itself over the wire.
- Server-side config lives in `/etc/ssh/sshd_config` — common hardening settings interviewers may ask about: `PermitRootLogin no` (disallow direct root SSH login), `PasswordAuthentication no` (force key-only auth), `Port` (change from default 22, minor "security through obscurity" but reduces automated scan noise).
- After changing `sshd_config`, you must reload the service: `systemctl reload sshd` (reload, not restart, so you don't drop existing sessions while testing).
- Common permission gotcha: SSH refuses to use a private key or `authorized_keys` file if its permissions are too open (e.g., world-readable) — it should be `600` for the private key file and `700` for the `~/.ssh` directory.

**What to say:** "SSH key auth uses a public/private key pair — the private key never leaves your machine; the server holds only the public key and issues a cryptographic challenge only the matching private key can answer. Common hardening is disabling root login and password auth in `sshd_config`, and one very common real gotcha is SSH silently refusing to use keys or `authorized_keys` if permissions are too permissive — the private key needs to be `600` and `~/.ssh` needs to be `700`."

---

### Q10. What's the difference between a package manager like `apt`/`yum`/`dnf` and the underlying `dpkg`/`rpm`?

**Simple explanation:** `dpkg` (Debian/Ubuntu) and `rpm` (RHEL/CentOS/Fedora) are the low-level tools that actually install/remove individual package files (`.deb`/`.rpm`) — but they don't know how to fetch a package or resolve its dependencies from the internet. `apt`/`yum`/`dnf` are higher-level tools built on top that handle downloading, dependency resolution, and repository management.

**Deeper detail:**
```bash
# Debian/Ubuntu family
apt update                 # refresh the list of available packages from repositories
apt install nginx          # resolves dependencies, downloads, and installs via dpkg underneath
dpkg -l | grep nginx        # low-level: is it installed, what version
dpkg -i package.deb         # low-level: install a local .deb file directly (no dependency resolution)

# RHEL/CentOS/Fedora family
dnf install nginx           # modern equivalent of yum
rpm -qa | grep nginx        # low-level: query installed packages
rpm -ivh package.rpm        # low-level: install a local .rpm directly
```
**Why this distinction matters practically:** if `apt install` or `dnf install` fails due to a broken dependency chain or repository issue, understanding that the low-level tool (`dpkg`/`rpm`) is what actually tracks installed package state lets you diagnose issues like "half-installed" packages (`dpkg --configure -a` to resume a broken/interrupted install) that the high-level tool alone won't clearly explain.

**What to say:** "`dpkg`/`rpm` are the low-level package database tools — they install a specific package file and track what's installed, but don't fetch anything or resolve dependencies. `apt`/`yum`/`dnf` sit on top, handling repositories and dependency resolution, then calling down to `dpkg`/`rpm` to actually do the install. When I hit a weird 'broken package' state, I usually drop to the low-level tool — like `dpkg --configure -a` — to see and fix the actual installed-package state directly."

---

## PART 3: Performance Tuning — Concepts & Interview Q&A

### 3.1 The mental model for performance work: the USE Method

Before individual tools, learn this framework (widely credited to performance engineer Brendan Gregg) — interviewers who ask serious performance questions are often specifically checking if you know a *systematic* method rather than randomly running commands.

For every resource (CPU, memory, disk, network), check three things:
- **U**tilization — how busy is it? (e.g., % CPU busy, % of memory used)
- **S**aturation — is work queuing up waiting for it? (e.g., processes waiting to run on CPU, I/O wait queue)
- **E**rrors — are operations on this resource failing? (e.g., dropped network packets, disk I/O errors)

**Why this matters:** utilization alone can be misleading — a disk at 100% utilization might be totally fine if nothing is waiting on it, while a disk at 60% utilization with a long, growing queue (saturation) is already a real problem. Checking all three, for each resource, systematically, is how you avoid tunnel-visioning on the first suspicious number you see.

### 3.2 CPU performance concepts

**Load average** (from `uptime` or `top`) — the three numbers (1, 5, 15-minute averages) represent, roughly, the average number of processes that were either running on a CPU or waiting for a resource (CPU or, on Linux specifically, also uninterruptible I/O wait) over that time window.

**Key gotcha interviewers love**: on Linux, load average includes processes in **uninterruptible sleep (D state)** — meaning **high load average does not necessarily mean high CPU usage**. A load average of 20 on an 4-core box could mean CPU is maxed out, *or* it could mean 20 processes are stuck waiting on slow disk I/O while the CPU is nearly idle. You must correlate load average against actual CPU utilization (`top`, `mpstat`) to know which it is — this exact confusion is Scenario 1 in Part 4.

**`nice` / `renice`** — every process has a "niceness" value (-20 to 19) influencing CPU scheduling priority; lower niceness = higher priority (more eager to run). A background batch job can be started with a high nice value (low priority) so it doesn't compete with interactive/latency-sensitive work for CPU time.
```bash
nice -n 10 ./batch_job.sh          # start a new process with lower priority
renice -n 5 -p 1234                # change priority of an already-running process
```

### 3.3 Memory performance concepts

**Buffers/cache are not "used" memory in the way people assume** — this is probably the single most common Linux memory misconception, and interviewers ask about it constantly.

```bash
free -h
#               total   used   free  shared  buff/cache  available
# Mem:           16Gi   4.2Gi   1.1Gi   200Mi     10Gi       11Gi
```
Linux aggressively uses "free" RAM to cache recently-read disk data (page cache) and filesystem metadata (buffers), because RAM is much faster than disk and this cache can be reused to speed up future reads. **This memory is reclaimed instantly and automatically** the moment an application actually needs it — it is not memory that's unavailable. The `available` column in `free -h` is the number that actually matters: it's the kernel's own estimate of memory that can be given to a new process without swapping, *including* cache that would be reclaimed if needed. A server showing "only 1GB free" but 10GB in buff/cache is, in almost all cases, perfectly healthy — not "almost out of memory."

**Swap** — disk space used as overflow when physical RAM is full. Swapping is *vastly* slower than RAM (disk vs. RAM latency differs by orders of magnitude), so heavy, sustained swapping ("thrashing") is a serious performance problem, even though a small amount of swap usage in isolation isn't automatically alarming.

**The OOM killer** — when the kernel genuinely runs out of memory (including swap) and can't satisfy an allocation, it invokes the **OOM (Out Of Memory) killer**, which picks a process to kill (based on a calculated "badness" score, roughly weighing memory usage) to free up memory and keep the system alive. This is a kernel-level last resort, and finding OOM killer log entries (`dmesg` or `journalctl -k`) is a key diagnostic skill (Scenario in Part 4).

### 3.4 Disk I/O performance concepts

- **IOPS** (I/O operations per second) vs. **throughput** (MB/s) — small random reads/writes are limited by IOPS; large sequential transfers are limited by throughput. Different workloads (a database doing small random writes vs. a backup job doing large sequential reads) hit different limits.
- **I/O wait (`%wa` in `top`)** — the percentage of time the CPU was idle *specifically because* it was waiting on a disk I/O operation to complete. High I/O wait means disk, not CPU, is your bottleneck.
- **Disk schedulers** — the kernel can reorder/merge I/O requests for efficiency using different algorithms (e.g., `mq-deadline`, `none`/noop for very fast NVMe/SSDs where reordering overhead isn't worth it, `bfq` for fairness on desktop-like workloads). Viewable/tunable via `/sys/block/<device>/queue/scheduler`.

### 3.5 Network performance concepts

- **TCP tuning knobs** commonly adjusted via `sysctl` for high-throughput or high-connection-count servers: `net.core.somaxconn` (max queued incoming connections waiting to be accepted), `net.ipv4.tcp_max_syn_backlog`, TCP buffer sizes (`net.ipv4.tcp_rmem`/`tcp_wmem`) for high-latency/high-bandwidth links.
- **`TIME_WAIT`** — after a TCP connection closes, it lingers briefly in `TIME_WAIT` state (default around 60 seconds) to safely handle any delayed packets. Under very high connection churn (e.g., a server opening a huge number of short-lived outbound connections), this can exhaust available local ports — a real, specific, diagnosable problem (see Part 4).

### 3.6 Kernel-level tuning: `sysctl` and `ulimit`

- **`sysctl`** reads/writes live kernel parameters, backed by files under `/proc/sys/`. Changes made with `sysctl -w` are live but don't survive reboot unless also written to `/etc/sysctl.conf` or a file under `/etc/sysctl.d/`.
```bash
sysctl vm.swappiness                     # read current value
sysctl -w vm.swappiness=10               # change live (temporary)
echo "vm.swappiness=10" >> /etc/sysctl.d/99-custom.conf   # persist across reboots
sysctl -p                                # reload all sysctl.conf settings
```
  - `vm.swappiness` (0-100) controls how aggressively the kernel prefers to swap out memory vs. reclaim page cache — lower values make the kernel more reluctant to swap, generally preferred on servers where swapping under load is undesirable.
- **`ulimit`** controls per-process/per-user resource limits (max open file descriptors, max processes, max memory) — set in the current shell session, or persistently via `/etc/security/limits.conf` (for PAM-based logins) or, for systemd-managed services, `LimitNOFILE=` etc. directly in the unit file.
```bash
ulimit -n                # show current max open file descriptors for this shell
ulimit -n 65535          # raise it for this session
```

**What to say (general framing for any performance question):** "I use the USE method — utilization, saturation, and errors — for whichever resource is suspect, rather than jumping straight to a fix. And for memory specifically, I always check `available` in `free -h` rather than `free`, because page cache being high is normal and healthy, not a sign of memory pressure — real pressure shows up as swapping and, in the worst case, OOM killer log entries."

---

## PART 4: Troubleshooting — Mental Model, Commands & Scenario Questions

### 4.1 The universal troubleshooting framework

Combine two ideas from Part 1 and Part 3 into one repeatable process:

1. **Identify the symptom precisely.** "The server is slow" is not a starting point — slow *how*? Slow to respond over the network? High latency on every request? Slow only for one specific operation (e.g., disk writes)? Get the user/monitoring to be specific before touching a terminal.
2. **Pick the resource(s) most likely implicated** — CPU, memory, disk I/O, network, or "none of the above, it's actually application logic." Use the **USE method** (4.1 above) on each candidate resource rather than guessing.
3. **Work top-down or bottom-up depending on what you know:**
   - **Top-down** (start at the application, work down): good when you have a specific error message or specific failing request to chase.
   - **Bottom-up** (start at hardware/kernel, work up): good when the symptom is vague ("everything feels slow") — check CPU/memory/disk/network health first, since if a fundamental resource is exhausted, no amount of application-level debugging will help until that's fixed.
4. **Correlate multiple signals before concluding anything.** A single high number (e.g., high load average) is a clue, not a diagnosis — cross-check against a second, independent signal (e.g., actual CPU%, actual I/O wait%) before deciding what's actually wrong. This is the single biggest habit separating strong candidates from weak ones in scenario questions.
5. **Check logs at the layer you suspect** — kernel messages (`dmesg`, `journalctl -k`) for hardware/driver/OOM issues, systemd/journal for service-level issues, application logs for application-level issues.
6. **Form a hypothesis, then make the smallest possible change to test it.** Don't restart the whole service, reboot the box, or change five settings at once — you'll never know which one actually fixed it (or whether it fixed itself).
7. **Fix the root cause, not just the symptom** — e.g., don't just kill a process eating memory; understand *why* it grew unbounded (a leak? an unexpectedly large workload? a missing limit?) before deciding the incident is closed.
8. **After resolving, ask what would have caught this earlier** — a monitoring alert threshold, a resource limit, a log rotation policy, a capacity plan.

Say this framework's *shape* out loud in interviews — it demonstrates process, which is exactly what's being tested by an invented scenario.

### 4.2 Command cheat sheet — know what each is *for*

| Command | What it's for |
|---|---|
| `top` / `htop` | Live overview: CPU%, memory, per-process usage, load average. `htop` is a friendlier, colorized, scrollable version. First command to run for "something feels wrong." |
| `uptime` | Quick load average + how long the system has been up. |
| `ps aux` / `ps -ef` | Snapshot of all running processes (not live, unlike top) — good for scripting/grepping. |
| `ps -eo pid,ppid,stat,cmd` | Custom view showing process state (`STAT` column) — key for spotting `D` (uninterruptible I/O wait) or `Z` (zombie) processes. |
| `vmstat 1` | Repeating snapshot of CPU, memory, swap, and I/O activity every 1 second — great for watching trends live (e.g., is swap activity (`si`/`so` columns) actually happening right now). |
| `free -h` | Memory and swap usage, human-readable. Always check the `available` column, not `free`. |
| `iostat -xz 1` | Per-disk I/O statistics: utilization (`%util`), queue size, latency (`await`) — the core disk performance command. |
| `iotop` | Like `top` but for disk I/O — shows which *process* is generating disk activity (needs root, and the kernel's I/O accounting enabled). |
| `sar` | Historical performance data (CPU, memory, disk, network) if the `sysstat` package's collector is enabled — useful for "what happened last night at 2 AM" after the fact, not just live monitoring. |
| `ss -tulnp` | Show listening ports and established connections, with the owning process (`-p` needs root for full detail). Modern replacement for `netstat`. |
| `netstat -ant \| awk '{print $6}' \| sort \| uniq -c` | Classic one-liner to count connections by TCP state (e.g., how many in `TIME_WAIT`). |
| `dmesg -T` | Kernel ring buffer messages, human-readable timestamps (`-T`). First place to check for hardware errors, driver issues, and OOM killer activity. |
| `journalctl -k` | Same kernel messages, but via journald (on systemd systems), with better filtering (`--since`, `-p err`, etc.). |
| `df -h` | Disk space usage per mounted filesystem. |
| `df -i` | **Inode** usage per filesystem — a completely separate resource from disk space; a filesystem can be "full" on inodes while showing plenty of free space in `df -h`. |
| `du -sh <dir>` | Disk space used by a specific directory (walks the tree and sums file sizes) — used to find *what's* consuming space `df` says is used. |
| `lsof` | Lists open files (remember: "everything is a file," including network sockets and directories) — `lsof /path/to/file` shows what process has a file open; `lsof -p <pid>` shows everything a specific process has open. Critical for the "disk full but nothing found" scenario below. |
| `strace -p <pid>` | Traces the system calls a running process is making, live — the deepest level of "what is this process actually doing right now," useful when a process is hung and you don't know why. |
| `lsblk` | Shows block devices (disks, partitions) and how they're mounted, in tree form. |
| `mount` / `/etc/fstab` | `mount` shows currently mounted filesystems; `/etc/fstab` defines what should be mounted automatically at boot. |
| `systemctl status <service>` | Is it running, recent log tail, restart count — often the single fastest way to see "why did this service die." |
| `nc -zv host port` (netcat) | Test whether a specific TCP port on a host is reachable — fast way to isolate "is this a network/firewall problem or an application problem." |
| `traceroute` / `mtr` | Shows the network path (hop by hop) to a destination — `mtr` combines traceroute with continuous ping stats per hop, better for spotting *which* hop is dropping/slow. |
| `dig` / `nslookup` | Query DNS directly, bypassing application-level caching, to see exactly what a DNS server returns. |
| `curl -v` | Make an HTTP(S) request with verbose output showing the full request/response and TLS handshake — extremely useful for isolating network vs. application-layer HTTP issues. |


### 4.3 Scenario-Based Troubleshooting Questions

Try to answer each **before** reading the Approach — that's how the reflex actually forms.

---

#### Scenario 1: "The server has a load average of 25 on an 8-core machine, but users say pages load slowly, not that CPU is pegged." What do you check?

**Mental model:** This is the classic **load average vs. CPU utilization** trap from section 3.2. Linux load average counts processes waiting on CPU *and* processes stuck in uninterruptible I/O wait (`D` state). A high load number does not by itself tell you which one it is.

**Approach:**
1. Check actual CPU utilization: `top` or `mpstat 1` — if `%idle` is high (CPU is mostly idle) despite a high load average, CPU is *not* your bottleneck, despite the scary load number.
2. Look at process states directly: `ps -eo pid,ppid,stat,cmd | grep " D"` — processes stuck in `D` state are waiting on I/O (usually disk, sometimes NFS/network storage) in a way that can't even be interrupted by a signal.
3. If you find many `D`-state processes, pivot immediately to disk investigation: `iostat -xz 1` — look at `%util` (is the disk saturated) and `await` (average I/O latency — rising latency under load is a clear saturation signal).
4. Identify *which* process/file is driving the I/O: `iotop` (live, by-process disk activity).
5. Root causes to consider once you've confirmed it's disk-bound: a runaway process doing excessive logging/writes, a full or failing disk (check `dmesg` for disk errors), a noisy-neighbor workload on shared storage, or simply undersized IOPS for the workload.

**Why this matters:** This is one of the most-asked "gotcha" scenarios precisely because it punishes candidates who treat "high load average" as synonymous with "high CPU usage" — they're related but distinct signals, and conflating them leads you to investigate entirely the wrong resource.

---

#### Scenario 2: One process is consuming very high CPU — how do you find and confirm it, and what do you do next?

**Mental model:** First *confirm* utilization precisely (don't act on a hunch), then decide whether the fix is "let it be, it's expected" (a legitimate batch job) or "something is actually wrong" (an infinite loop, a stuck retry storm, a runaway thread).

**Approach:**
1. `top` (sorted by CPU% by default) or `htop` — identify the PID and process name.
2. Confirm it's sustained, not a brief spike: watch it for a bit, or check `ps -o %cpu,etime -p <pid>` (CPU% alongside elapsed run time).
3. Understand *what* it's doing before deciding to kill it: `strace -p <pid>` briefly (see what syscalls it's making — spinning on a particular syscall repeatedly is a strong clue), or for a multi-threaded process, check per-thread CPU with `top -H -p <pid>`.
4. If it's legitimate but shouldn't be hogging the whole box: `renice` it to lower priority rather than killing it, if the work still needs to complete.
5. If it's genuinely stuck (e.g., an infinite loop bug), gather diagnostic info *before* killing it if at all possible (a stack trace or core dump can be essential for actually fixing the underlying bug) — killing it first and asking questions later loses your only chance to diagnose the real defect.

**Why this matters:** Interviewers want to see you don't reflexively `kill -9` the first suspicious process — losing diagnostic evidence before understanding *why* something misbehaved just guarantees a repeat incident later.

---

#### Scenario 3: A process was unexpectedly killed, and you suspect the OOM killer. How do you confirm it, and what's your follow-up?

**Mental model:** The OOM killer is a kernel-level, last-resort action — it only fires when the system genuinely cannot satisfy a memory allocation. It always leaves a kernel log trail, so "did the OOM killer do this" is always directly confirmable, never a guess.

**Approach:**
1. Check kernel logs for an OOM event: `dmesg -T | grep -i "out of memory"` or `journalctl -k | grep -i oom`. You'll see the killed process's name, PID, and the scoring that led to its selection.
2. Correlate timing with `free -h` history if available (`sar -r` if `sysstat` was collecting) — was there a genuine memory spike, or a slow leak building up over time?
3. Identify *why* memory ran out:
   - A single process leaking or legitimately needing more memory than the box has (check its memory growth over time, e.g., via monitoring or repeated `ps`/`smem` snapshots).
   - Multiple processes each using a moderate amount, collectively exceeding capacity (a capacity/sizing issue, not a single culprit's bug).
   - No swap configured at all, meaning the system had zero cushion before hitting the OOM killer the moment physical RAM filled — worth checking `swapon --summary`.
4. Decide the fix at the right level: add memory/resize the instance, set memory limits on individual services (cgroups, or `MemoryMax=` in a systemd unit) so one runaway process can't take the whole box down, fix an actual application memory leak, or (as a safety net, not a real fix) ensure reasonable swap exists to absorb short spikes gracefully rather than triggering an immediate hard kill.

**Why this matters:** This tests whether you know the OOM killer always logs its actions (so "was it the OOM killer" should never be answered with a shrug), and whether you think about *systemic* prevention (limits, capacity) rather than just restarting the killed service and moving on.

---

#### Scenario 4: `df -h` shows plenty of free space, but you're still getting "No space left on device."

**Mental model:** Disk space and **inodes** are two completely separate, independently exhaustible resources. `df -h` only shows space; it does not show inode usage by default. A filesystem with millions of tiny files (a very common pattern: session files, cache files, mail queue files) can run out of inodes — the "slots" for file metadata — long before it runs out of raw bytes.

**Approach:**
1. Check inode usage specifically: `df -i` — if `IUse%` is at 100% while `df -h`'s space usage looks fine, you've confirmed inode exhaustion, not space exhaustion.
2. Find what's consuming inodes: look for directories with an enormous number of small files — common culprits are application session/cache directories, mail spools, or a logging system writing one file per event instead of appending.
3. Fix by deleting/archiving the excess small files, and address the root cause (rotate/cap the thing generating them, or switch it to fewer, larger files/log rotation instead of one-file-per-event).
4. Note: you generally cannot "add inodes" to an already-formatted filesystem without reformatting it with a different inode ratio at creation time — this is a good detail to mention, since it shows you understand it's a structural, not just an operational, limit.

**Why this matters:** This is one of the highest-value "sounds obvious once you know it" scenarios in Linux interviews — many candidates have literally never checked `df -i` and will spin their wheels checking `du` repeatedly on a space number that already looked fine.

---

#### Scenario 5: `df -h` shows a filesystem is full, but running `du -sh /*` doesn't add up to nearly that much space.

**Mental model:** On Linux, deleting a file only removes its *directory entry* (the name). If a running process still has that file **open**, the kernel keeps the actual data blocks allocated until that process closes the file handle — even though the file is invisible to `ls`/`du` because its name is gone. `df` reports real disk block usage; `du` walks visible directory entries — so they diverge exactly in this situation.

**Approach:**
1. Recognize the symptom pattern immediately: `df` and `du` disagreeing by a large margin is close to a unique fingerprint for this specific issue.
2. Find processes holding deleted-but-still-open files: `lsof +L1` (lists open files with a link count of 0 or more precisely, deleted files still open — the flag/approach varies slightly by `lsof` version, but the concept is "find open file handles pointing to unlinked files") or more simply: `lsof / | grep deleted`.
3. Identify the offending process (very often a long-running service like a database or log-writing daemon that opened a huge log file, which was then deleted externally — e.g., by a cron job or manual `rm` — while the service kept writing to its still-open handle).
4. Fix options: restart/reload the offending service (which closes and reopens its file handles, actually freeing the space) — this is usually the fastest real fix; or, if it supports it, signal the service to reopen its log file cleanly (many daemons support a `SIGHUP`-triggered log reopen specifically to avoid this).
5. Prevent recurrence: use proper log rotation (`logrotate`, or the application's built-in rotation) which handles this correctly (typically via a `copytruncate` strategy or an explicit reopen signal) instead of a naive external `rm` on files a service might still have open.

**Why this matters:** This scenario specifically tests your grasp of the inode/directory-entry model from section 1.2/1.5 ("everything is a file," and a directory entry is just a name pointing at an inode) — it's a favorite because the fix requires understanding *why* the space isn't freed, not just running a command by rote.

---

#### Scenario 6: A systemd service fails to start. Walk through how you'd diagnose it.

**Mental model:** systemd almost always tells you exactly why, if you look in the right two places — you rarely need to guess.

**Approach:**
1. `systemctl status <service>` first — this alone often shows the exact failure (a config syntax error, "port already in use," a permission denied, the exit code).
2. If the status output is truncated or you need more history: `journalctl -u <service> -n 100 --no-pager` (last 100 lines) or `journalctl -u <service> -b` (all logs from this boot).
3. Common root causes to check, roughly in order of frequency:
   - **Config syntax error** — many services validate config and refuse to start on a bad file; the log usually names the exact line/file.
   - **Port already in use** — another process is already bound to the port this service wants. Find it: `ss -tulnp | grep <port>`.
   - **Permission issue** — the service's configured user can't read a required file/directory, or can't bind to a low-numbered port (<1024) without elevated capability.
   - **Missing dependency** — a unit's `After=`/`Requires=` dependency (e.g., a database) isn't up yet or failed itself.
   - **Bad unit file** after a manual edit without `systemctl daemon-reload` — systemd is still running against the old definition.
4. Fix the identified root cause, then `systemctl daemon-reload` (if the unit file changed) and `systemctl restart <service>`, and confirm with `systemctl status` again that it's not just "started" but genuinely healthy (no immediate re-crash — check the restart count via `status` too).

**Why this matters:** This tests basic operational fluency — systemd is *designed* to make failures diagnosable, and interviewers want to see you actually look (status + journalctl) before speculating.

---

#### Scenario 7: A service fails to start with "Address already in use" — how do you resolve it?

**Mental model:** Only one process can bind to a given (IP, port, protocol) combination at a time. This error means something else already holds that port — possibly a previous instance of the same service that didn't fully stop, or a genuinely different application.

**Approach:**
1. Find what's currently holding the port: `ss -tulnp | grep :<port>` (needs root for full process detail) — this shows the PID and process name.
2. Decide: is that the *old* instance of the service you're trying to start (e.g., a previous process that didn't exit cleanly), or a genuinely different, unrelated service that happens to want the same port?
3. If it's a stuck old instance: stop it properly (`systemctl stop`, or if it's not managed by systemd and won't respond, `kill <pid>`, escalating to `kill -9` only if it doesn't terminate gracefully), then start the intended service.
4. If it's a different, legitimate service that needs that port, you have a genuine conflict to resolve by reconfiguring one of them to use a different port.

**Why this matters:** Confirms you know to *identify* the actual holder of a resource before acting (rather than immediately reaching for `kill -9` on a guess), and that you understand this is fundamentally a "one owner per port" kernel-level constraint, not an application bug.

---

#### Scenario 8: You can't connect to a remote service (e.g., a web app on another server) — walk through your diagnostic approach.

**Mental model:** Work through the network stack **layer by layer, from closest to you outward**, isolating exactly where the chain breaks, rather than guessing "firewall" or "DNS" first.

**Approach (in order):**
1. **DNS** — does the hostname even resolve? `dig example.com` or `nslookup example.com`. If this fails, nothing past this point matters yet — fix DNS first.
2. **Basic reachability** — `ping <ip>` (note: many environments block ICMP for security, so a failed ping alone isn't conclusive — don't over-read it).
3. **Path** — `traceroute <ip>` or `mtr <ip>` to see if the path is normal or dies/degrades at a specific hop (useful for spotting an intermediate network/firewall issue vs. the destination itself being down).
4. **Port-level reachability** — `nc -zv <ip> <port>` or `telnet <ip> <port>` — this tells you specifically whether the TCP port is open and accepting connections, independent of whether the application behind it is actually working correctly. This is the single most valuable step for separating "network problem" from "application problem."
5. **If the port is open but the app misbehaves** — pivot from network tools to application-level tools: `curl -v <url>` to see the actual HTTP request/response and TLS handshake in detail.
6. **If the port is closed/unreachable** — check, in order: is the target service actually running and listening there (`ss -tulnp` *on the target machine*, if you have access)? Is a firewall (security group, iptables, firewalld) blocking it, on either end? Is there a routing issue in between?

**Why this matters:** This is the single most common "walk me through your approach" scenario in Linux/networking interviews, precisely because it rewards a structured, layer-by-layer method over random guessing, and the "port open, but app broken" distinction (step 4→5) is exactly the kind of precision interviewers are listening for.

---

#### Scenario 9: DNS resolution is failing or returning unexpected results — how do you debug it?

**Mental model:** DNS resolution on a Linux client involves several potential layers (local hosts file → local cache/resolver → configured DNS servers → the actual authoritative chain) — you need to figure out *which layer* is giving the wrong answer.

**Approach:**
1. Check `/etc/hosts` first — a stale manual entry here silently overrides DNS entirely for that name, and is a very common "why is it resolving to the wrong IP" cause that has nothing to do with DNS servers at all.
2. Check `/etc/resolv.conf` to see which DNS servers are actually configured.
3. Query directly against a specific DNS server, bypassing any local cache, to see the "ground truth" answer: `dig @8.8.8.8 example.com` (or your actual configured DNS server's IP) vs. `dig example.com` (using the default resolver) — if these two disagree, the problem is in local caching/config, not the authoritative DNS data itself.
4. Check `/etc/nsswitch.conf`'s `hosts:` line to confirm the resolution order Linux is actually using (e.g., `files dns` means `/etc/hosts` is checked before DNS).
5. If using a local caching resolver (e.g., `systemd-resolved`), check its own status/cache: `resolvectl status` / `resolvectl statistics` on systems using it, since a stale local cache entry can also be the culprit even when `/etc/hosts` and upstream DNS both look correct.

**Why this matters:** Tests whether you know resolution isn't "just DNS" — there are multiple layers that can each independently produce a wrong or stale answer, and `dig @<specific-server>` is the key technique for isolating which layer is at fault.

---

#### Scenario 10: SSH connection is refused, or hangs, or asks for a password when you expect key-based auth to just work — how do you debug each variant?

**Mental model:** These are three distinct failure modes at different layers, and conflating them wastes time.

**Approach by symptom:**

- **"Connection refused"** — this means something actively rejected the TCP connection attempt (fast failure), most often: the SSH daemon isn't running on the target (`systemctl status sshd` on the target if you have another way in), or a firewall on the target is explicitly rejecting (vs. silently dropping) the port.
- **Connection hangs / times out** (no response at all) — usually a network path or firewall issue where packets are being silently dropped rather than rejected — check security groups/firewall rules on both ends, and confirm the target is actually reachable at all (see Scenario 8's approach).
- **Prompts for a password when you expected key auth** — this means the server never successfully validated your key, so it fell back to the next allowed auth method. Debug with verbose client output: `ssh -vvv user@host` — this shows exactly which keys were offered and why each was rejected (wrong key, permissions too open on the server's `authorized_keys` or `~/.ssh`, or `PubkeyAuthentication` disabled server-side). Also directly check permissions: private key should be `600`, `~/.ssh` directory `700`, `authorized_keys` `600` — SSH silently refuses to use overly-permissive files as a security precaution, without always making the reason obvious to the user.

**Why this matters:** Distinguishing "refused" vs. "timeout" vs. "auth fallback" as three different problems needing three different investigations is a sign of real hands-on experience versus surface-level familiarity.

---

#### Scenario 11: You notice a growing number of zombie (`Z` state) processes — what's happening and how do you fix it?

**Mental model:** A **zombie process** has already finished executing, but its exit status hasn't been "reaped" (collected) by its parent process yet — it's not consuming CPU or real memory, just a small slot in the process table, but a large accumulation signals a real bug.

**Approach:**
1. Confirm and identify: `ps aux | grep 'Z'` (state column shows `Z`) and note their **parent** PIDs.
2. Understand the mechanism: when a child process finishes, the kernel keeps a minimal record (exit code) until the parent calls `wait()` on it. A well-behaved parent reaps its children promptly; zombies pile up when the **parent** either has a bug where it never calls `wait()`, or is itself stuck/unresponsive.
3. The real target of investigation is the **parent process**, not the zombies themselves (zombies can't be killed with `kill` — there's no running process left to signal; they disappear only once reaped).
4. Fix: address the bug in the parent (or restart it, which causes the zombies to be reparented to `init`/`systemd`, PID 1, which reliably reaps orphans) — restarting the misbehaving parent process is usually the fastest practical remediation, followed by a real fix/upgrade if it's a known application bug.

**Why this matters:** Common trick question because candidates instinctively try to `kill -9` the zombie itself, which is impossible/meaningless — the correct target is always the parent.

---

#### Scenario 12: A high-throughput server is running out of available outbound ports / seeing connection failures under load, and `netstat`/`ss` shows a huge number of connections in `TIME_WAIT`.

**Mental model:** Every outbound TCP connection uses a local (source IP, source port) combination; after a connection closes, it lingers in `TIME_WAIT` for a period (default ~60s on Linux) before that port is reusable. Under very high connection churn (e.g., making a new short-lived outbound connection per request instead of reusing connections), you can exhaust the available local port range faster than `TIME_WAIT` entries expire.

**Approach:**
1. Confirm the pattern: `ss -tan state time-wait | wc -l` (count) alongside `ss -tan | awk '{print $1}' | sort | uniq -c` (breakdown by state) to see the scale.
2. Check the ephemeral port range available: `sysctl net.ipv4.ip_local_port_range` — a narrow range makes exhaustion more likely.
3. Real fixes, roughly best-to-worst:
   - **Best**: fix the application to reuse connections (connection pooling / keep-alive) instead of opening a new short-lived connection per request — this addresses the actual root cause (excessive connection churn), not just the symptom.
   - **Good**: widen the ephemeral port range via `sysctl -w net.ipv4.ip_local_port_range="1024 65535"` if it was unnecessarily narrow, giving more headroom.
   - **Situational**: enable `net.ipv4.tcp_tw_reuse` (allows reusing a `TIME_WAIT` socket for a new outgoing connection when safe to do so) — a legitimate kernel tuning knob for this exact scenario, though it doesn't replace fixing genuinely excessive connection churn at the application level.
   - **Avoid** unless you fully understand the implications: `tcp_tw_recycle` was a similar-sounding but far more dangerous setting (it caused real connection problems for clients behind NAT) and was removed entirely from modern kernels — a good detail to mention if this comes up, since confusing `tw_reuse` and the removed `tw_recycle` is a common mistake.

**Why this matters:** Tests whether you distinguish a genuine application-architecture problem (too many short-lived connections) from something you can just "sysctl your way out of" — the correct answer prioritizes fixing the application pattern, with kernel tuning as a secondary lever.

---

#### Scenario 13: An application logs "Too many open files" — how do you diagnose and fix it?

**Mental model:** Every process has a limit on how many file descriptors (open files, sockets, pipes — remember, "everything is a file") it can hold simultaneously. This error means the process hit that ceiling — either it's leaking file descriptors (opening things and never closing them) or the configured limit is simply too low for legitimate, expected load.

**Approach:**
1. Confirm the current limit for the affected process: `cat /proc/<pid>/limits | grep "Max open files"`, or for a user's shell session, `ulimit -n`.
2. Check how many file descriptors the process actually has open right now, and see if the number is climbing over time (a leak) vs. simply legitimately high (undersized limit): `ls /proc/<pid>/fd | wc -l`, sampled a few times a minute or two apart.
3. If it's climbing steadily without bound → suspect a genuine leak (the application opening files/sockets and not closing them) — use `lsof -p <pid>` to see *what kind* of descriptors are accumulating (regular files? sockets in a particular state? this narrows down where in the app's logic to look).
4. If it's high but stable, just above a too-low ceiling → raise the limit at the right layer:
   - For a systemd-managed service: set `LimitNOFILE=65535` in its unit file's `[Service]` section, then `daemon-reload` and restart.
   - For login-shell-based processes: `/etc/security/limits.conf` (`username soft/hard nofile 65535`), which requires a new login session to take effect.
5. Raising the limit is the correct fix for "legitimately needs more descriptors than the old default allowed" — but if the real cause is a leak, raising the limit only delays the same failure, so confirm which case you're in before declaring it fixed.

**Why this matters:** Tests whether you distinguish "config limit too low" from "application bug leaking descriptors" — treating every instance as "just raise the limit" is a red flag that you'd paper over real bugs.

---

#### Scenario 14: A cron job that works fine when you run it manually doesn't seem to run (or fails) under cron.

**Mental model:** Covered in Part 2, Q8 — cron executes with a **minimal environment**, not your interactive shell's environment. This is close to the single most common real-world cron issue.

**Approach:**
1. First, actually confirm cron attempted to run it at all: check `journalctl -u cron` (or `-u crond` depending on distro) or `/var/log/cron` / `/var/log/syslog` (varies by distro) around the scheduled time, and double check the crontab syntax/schedule itself is what you think it is (`crontab -l`).
2. If it ran but failed, capture what actually happened — redirect the script's output explicitly, since cron often silently discards or mails output that nobody reads:
   ```
   */15 * * * * /usr/local/bin/script.sh >> /var/log/script.log 2>&1
   ```
3. Compare the environment cron gives the script against your interactive shell — the most common differences are `PATH` (cron's is minimal, often just `/usr/bin:/bin`) and any environment variables your `.bashrc`/`.bash_profile` sets that a non-interactive cron shell never loads.
4. Fix by using absolute paths for every command/binary inside the script (don't rely on `PATH` at all), and explicitly setting any required environment variables either at the top of the crontab or inside the script itself, rather than assuming they'll be inherited.

**Why this matters:** This is asked constantly because it's genuinely one of the highest-frequency real production issues, and the fix pattern (redirect output, use absolute paths, don't assume environment) generalizes to basically all "works interactively, fails under automation" problems, including systemd timers and CI pipelines.

---

#### Scenario 15: A filesystem unexpectedly became read-only, and applications are failing to write.

**Mental model:** The Linux kernel will sometimes **remount a filesystem read-only automatically** as a protective measure when it detects a serious underlying error it can't safely recover from — this is the filesystem's own integrity protection kicking in, not something you did in configuration.

**Approach:**
1. Confirm the current mount state: `mount | grep <mountpoint>` — look for `ro` in the options where you expect `rw`.
2. Check kernel/filesystem logs for *why* — this is almost never silent: `dmesg -T | grep -iE "error|ext4|xfs|remount"` — you're looking for filesystem-level error messages (e.g., "EXT4-fs error," a journal error, or explicit I/O errors) that immediately preceded the remount.
3. The underlying trigger is very often an actual **disk hardware problem** (failing drive, bad sectors) — check `smartctl -a /dev/<device>` (from `smartmontools`) for the disk's own health/error counters, and check `dmesg` for I/O errors reported by the storage controller/driver.
4. Short-term remediation (only after understanding *why*, and only if you're confident it's safe): `mount -o remount,rw <mountpoint>` — but doing this without addressing the underlying cause on a genuinely failing disk risks further corruption; on real hardware failure, the correct path is data recovery/migration and replacing the disk, not just remounting.
5. Depending on filesystem type, a filesystem check (`fsck`) may be needed (must be run against an **unmounted** filesystem, so this typically requires taking the system/volume offline or booting from rescue media for the root filesystem).

**Why this matters:** Tests whether you recognize a read-only remount as a *symptom* (the kernel protecting data integrity after detecting corruption/errors) rather than a misconfiguration to just casually override — blindly remounting `rw` on a genuinely failing disk without investigating is exactly the kind of shortcut a senior interviewer is checking you won't take.

---

#### Scenario 16 (meta-question): "Walk me through how you'd approach a Linux problem you've genuinely never encountered before."

**What to say:** "I'd start by pinning down the symptom precisely rather than acting on a vague description — what exactly is failing, and how do we know? Then I'd decide whether to work top-down from the application or bottom-up from system resources, usually bottom-up if the symptom is vague, since if CPU, memory, disk, or network is fundamentally exhausted, nothing above that layer matters yet. I'd apply the USE method — utilization, saturation, errors — to whichever resource looks suspect, and specifically avoid concluding anything from a single number; I'd cross-check at least two independent signals, like load average against actual CPU%, before deciding what's really going on. I'd check logs at the layer I suspect — kernel messages for hardware/OOM issues, journalctl for service issues, application logs otherwise. Then I'd change one thing at a time and verify, rather than changing several things and hoping. And once it's fixed, I'd ask what monitoring, limit, or process change would have caught this earlier, so it doesn't just repeat."


---

## PART 5: Rapid-Fire Reference (last-minute review)

- **Layers mental model**: hardware → kernel → userspace/applications. Figure out *which layer* is actually broken before picking tools.
- **Everything is a file** — devices, process info (`/proc`), kernel settings (`/sys`), even sockets. When stuck, look for a relevant path under `/proc` or `/sys`.
- **`start`/`stop` ≠ `enable`/`disable`** — current state vs. boot-time behavior are independent in systemd. A very common real incident source.
- **Load average counts CPU-waiting *and* uninterruptible I/O-waiting (`D` state) processes** — high load ≠ high CPU. Always cross-check against actual `%CPU`/`%idle`.
- **`buff/cache` in `free -h` is not "used" memory** — it's reclaimable cache. Check the `available` column, not `free`.
- **The OOM killer always logs its actions** — `dmesg`/`journalctl -k` will confirm or rule it out definitively; never guess.
- **`df` = disk space, `df -i` = inodes** — two independent resources; "no space left" can mean either.
- **`df` vs `du` mismatch = a deleted-but-still-open file** — find it with `lsof | grep deleted`, fix by restarting/reloading the holding process.
- **Zombies are reaped by their parent, not killed directly** — the parent is the actual thing to investigate/fix.
- **Cron runs with a minimal environment** — always use absolute paths and explicitly set variables; don't assume `PATH` carries over from your shell.
- **SSH silently rejects overly-permissive key files** — private key `600`, `~/.ssh` `700`.
- **`nc -zv host port` isolates network-reachable vs. application-broken** — the single most useful step in "can't connect" scenarios.
- **A read-only remount is the kernel protecting itself from corruption**, not a config toggle to casually override — investigate `dmesg` before remounting `rw`.
- **The USE method** (Utilization, Saturation, Errors) applied per-resource is your systematic framework for any performance question.

---

## PART 6: How to carry yourself in the interview

- **Say your mental model out loud**, even when you instantly recognize the scenario — interviewers weight *process* heavily because it's what predicts how you'll handle a real incident they haven't scripted.
- **Cross-check signals before concluding** — explicitly mentioning "I wouldn't conclude X from this one number alone, I'd check Y too" is one of the strongest signals of real production experience you can give.
- If a command's exact flags escape you mid-interview, say what you'd do conceptually and that you'd confirm with `man <command>` or `--help` — that's honest and realistic; nobody has every flag memorized.
- If you have a real story (even a small one — a disk-full incident, a cron gotcha, a slow-service investigation), use it. A specific, concrete example beats a textbook-perfect answer every time.
- It's fine to ask **one clarifying question** on a scenario prompt (e.g., "is this a single box or part of a cluster behind a load balancer?") before diagnosing — it shows you gather context like a real engineer would, not uncertainty.
