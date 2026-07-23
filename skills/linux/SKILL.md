---
name: linux
slug: linux
version: 1.0.1
description: >-
  Avoids Linux traps — permission denials, disk-full mysteries, OOM kills,
  unkillable processes, systemd and cron failures, SSH lockouts. Use when
  operating, debugging, or hardening Linux systems.
homepage: https://clawic.com/skills/linux
changelog: Deeper administration and troubleshooting guidance
metadata:
  clawdbot:
    emoji: 🐧
    os:
    - linux
    - darwin
    displayName: Linux
---

## When To Use

- Diagnosing "permission denied", disk full, OOM kills, unkillable processes, or services that fail only at boot
- Running or reviewing operations that touch permissions, signals, systemd units, cron jobs, or firewalls
- Changing config on a remote host over SSH without locking yourself out
- Reading system tools whose output misleads: `free`, `df`, `top`, load average
- Not for shell scripting syntax (`bash`) or container build/runtime internals (`docker`)

## Quick Reference

| Symptom | First move |
|---------|-----------|
| "Permission denied" though perms look right | `namei -l <path>` — first failing component is the bug; then ACL (`+` in `ls -l`), SELinux (`ls -Z`), immutable (`lsattr`) |
| `df` says full, `du` can't find it | `lsof +L1` for deleted-but-open files; then the bind-mount check (→ Disk And Filesystem) |
| "No space left on device" but `df -h` shows free | `df -i` — inode exhaustion from many small files |
| `kill -9` doesn't kill it | `ps -o pid,stat,wchan <pid>` — D state waits on I/O; no signal helps (→ Processes And Signals) |
| Service fails at boot, starts fine by hand | Ordering and env: `network-online.target`, absolute paths, drop-in via `systemctl edit` (→ Systemd And Cron) |
| Job runs in your shell, fails in cron | Minimal PATH, no profile, `%` is special (→ Systemd And Cron) |
| SSH key suddenly rejected, no error | Perms: dir 700, key 600, home dir not group-writable; `ssh -vvv` (→ Networking And SSH) |
| Process or container exits with code 137 | 128+9 = SIGKILL, almost always OOM — `dmesg -T \| grep -i oom` |
| Load average high, CPU mostly idle | I/O wait and D-state tasks inflate load — storage problem, not compute (→ Commands That Lie) |
| Anything else | Core Rules below, then the matching section |

## Core Rules

1. Never `chmod 777` — it removes the audit trail and usually still fails (ACL mask, SELinux, mount options). Diagnose instead: `namei -l <path>` prints every path component; directories need `x` to traverse, the file needs the right bit for the uid/gid that actually runs.
2. Signal ladder: SIGTERM → wait → SIGKILL only if ignored. systemd itself waits 90s by default (`TimeoutStopSec`) before escalating. `kill -9` first skips cleanup handlers; on a database that means crash recovery on next start.
3. Disk-full triage in this order: `df -h` → `df -i` → `lsof +L1` → `mount --bind / /mnt && du -xh --max-depth=1 /mnt`. Each step catches a class the previous one cannot: space, inodes, deleted-but-open, files shadowed under mount points.
4. Remote-change safety: keep the current SSH session open, apply, verify from a NEW session before closing the old one. Run the validator when one exists: `sshd -t`, `visudo -c`, `nginx -t`, `mount -a` (tests fstab without rebooting).
5. Live change ≠ persistent change — pair every runtime command with its persistence mechanism: `sysctl -w` with a file in `/etc/sysctl.d/`, `iptables` with `iptables-save`/`netfilter-persistent`, `systemctl start` with `enable`. "Works now" is untested until it survives a reboot.
6. Capacity is relative, not absolute: alarm on load1/nproc > 1 sustained (load 8 on 4 cores = 2x oversubscribed), and on low `available` in `free` — never on low "free", cache is doing its job.
7. Never edit unit files under `/usr/lib/systemd/` — package upgrades silently overwrite them. `systemctl edit <unit>` writes a drop-in under `/etc/` that survives; any manual edit needs `daemon-reload` to take effect.
8. Root is not omnipotent: `chattr +i` blocks writes even for root, SELinux denies root by policy, file capabilities replace root entirely. When root gets "permission denied", check `lsattr` and `ls -Z` before doubting the filesystem.
9. Guard destructive paths against empty variables: `rm -rf "${DIR:?}/"` aborts if DIR is unset — `rm -rf $DIR/` with unset DIR expands to `rm -rf /`.

## Permissions And Ownership

- Shared team directory: `chmod 2775 dir` (setgid) + `setfacl -d -m g::rwx dir` — new files inherit the directory's group and perms. Without setgid every file lands in the creator's primary group and collaboration breaks one file at a time.
- Mode-string suffix in `ls -l`: `+` means ACLs present (`getfacl`; the `mask` line caps effective perms — an ACL grant can be silently neutered by the mask), `.` means SELinux context.
- `mv` preserves SELinux context and ownership; `cp` inherits the destination's defaults. A file moved into `/var/www` keeps its `user_home_t` context and the web server is denied; `restorecon -v <file>` fixes it.
- File capabilities (`setcap cap_net_bind_service=+ep`) live in xattrs: lost on package upgrade and on plain `cp`. Re-apply after upgrading any binary you granted caps to.
- Recursive perms without breaking directories: `chmod -R u=rwX,go=rX` — capital `X` sets execute only on dirs and files already executable. `chmod -R 600` strips directory `x` and locks you out of the tree.
- Setuid is ignored on scripts (kernel security policy) — it only works on binaries; use `sudo` rules or capabilities.
- `chown -R` follows symlinks pointing outside the target — use `--no-dereference` (`-h`).
- Default umask 022 makes every new file world-readable; set 077 on multi-user or sensitive hosts.

## Processes And Signals

- D state (uninterruptible sleep) is immune to every signal including SIGKILL — almost always a dead NFS mount or failing disk. `wchan` shows what it waits on; fix the I/O (`umount -l` the dead mount) and the process exits on its own.
- Zombies are already dead — they cost one process-table slot, no memory. Killing them is impossible; kill or fix the PARENT, and init adopts and reaps them.
- Closing the terminal SIGHUPs background jobs: `disown -h %1` rescues an already-running job; `nohup`/`setsid` protect one you're about to start.
- `pkill -f` matches the FULL command line: `pkill -f python` kills editors, agents, and daemons that merely mention python. Always preview with `pgrep -af <pattern>`.
- Exit-code decode: 126 = found but not executable, 127 = command not found, 128+N = killed by signal N (137 SIGKILL/OOM, 139 SIGSEGV, 143 SIGTERM).
- "Too many open files": soft limit is commonly 1024 (`ulimit -n`). systemd services ignore `/etc/security/limits.conf` (that's PAM, login sessions only) — set `LimitNOFILE=` in the unit.

## Disk And Filesystem

- Huge open logfile: `rm` frees nothing while a process holds it. `: > /path/file` truncates in place and frees space immediately — safe for append-mode writers (normal loggers); a non-append writer keeps its offset and leaves a sparse file.
- Files written to a mount point before the disk mounted are shadowed underneath and invisible to `du`: `mount --bind / /mnt/root` exposes the underlying tree.
- ext4 reserves 5% for root by default — 50GB "missing" on a 1TB volume. `tune2fs -m 1` on data volumes; keep the 5% on the root fs so root can still log and recover when users fill it.
- `du -xh --max-depth=1 /` stays on one filesystem — finds what's eating the root disk without descending into mounted volumes.
- Atomic replace requires a same-filesystem rename: `mv` across filesystems is copy+delete, and readers can catch the half-written file. Write the temp file into the destination directory, then `mv`.
- `/tmp` is cleared on reboot and often tmpfs (RAM-backed) — a big file there eats memory. Data that must survive a reboot but stay temp-like goes in `/var/tmp`.
- Space hogs beyond files: `journalctl --vacuum-size=500M`, `docker system prune -a`, and LVM/ZFS/cloud snapshots that hold deleted data alive.
- `du` reports blocks, `ls -l` reports apparent size — sparse files diverge wildly; compare with `du --apparent-size`.

## Memory And OOM

- Read `available` in `free`, never "free" — cache is reclaimable on demand. Low "free" with high `available` is a healthy system.
- OOM forensics: `dmesg -T | grep -iE "out of memory|oom"` — the killer picks the highest `oom_score` (roughly: biggest memory user), often not the process that triggered the shortage.
- Protect critical daemons with `OOMScoreAdjust=-900` in the unit (range -1000..1000). Reserve -1000 (full exemption) for sshd only — an exempt memory hog forces the kernel to kill everything else around it.
- Containers die at their cgroup limit long before host OOM: exit 137 with plenty of free host memory means the container's limit, not the host, is the ceiling.
- `vm.swappiness` defaults to 60; set 1-10 for databases and latency-sensitive services so the kernel prefers dropping cache over swapping anonymous pages.
- Thrash detection: `vmstat 1` — `si`/`so` sustained nonzero is swap thrashing, which is worse than an OOM kill: everything crawls instead of one process dying cleanly.
- Summing RSS across processes overcounts shared memory (each process counts the same shared pages). PSS (`smem`, or `/proc/<pid>/smaps_rollup`) divides shared pages fairly — use it before claiming "these workers use 40GB".

## Networking And SSH

- `localhost` is dual-stack: the resolver may return `::1` first while the service listens only on `127.0.0.1` — "connection refused" on a running service. Test both literals; bind `0.0.0.0` or `::` when in doubt.
- `ss -tlnp` replaces netstat for "what's listening and which pid owns it".
- Ports below 1024 don't need root: `setcap cap_net_bind_service=+ep <binary>` (re-apply after upgrades, → Permissions And Ownership).
- Outbound connection math: ephemeral range 32768-60999 ≈ 28k ports, TIME_WAIT holds each for 60s → ceiling ≈ 28232/60 ≈ 470 new connections/s to one destination. Fix with keep-alive/pooling and `net.ipv4.tcp_tw_reuse=1` (safe for outbound; `tcp_tw_recycle` broke NAT clients and was removed in kernel 4.12).
- "nf_conntrack: table full, dropping packet" in dmesg = silent drops under load — raise `net.netfilter.nf_conntrack_max`; each entry is cheap, the drops are not.
- MTU black hole: small requests work, large transfers hang. Probe with `ping -M do -s 1472 <host>` (1472 + 28 headers = 1500); fix with a lower interface MTU or `net.ipv4.tcp_mtu_probing=1`.
- Raw `iptables` rules vanish on reboot — persist with `iptables-save`/`netfilter-persistent`, or use firewalld/ufw which persist by design.
- SSH auth fails silently on permissions: `~/.ssh` 700, keys 600, and the HOME DIRECTORY must not be group/world-writable — StrictModes rejects without a client-side error, and the home-dir check is the one nobody looks at.
- Reaching internal hosts: `ssh -J bastion host` (ProxyJump), not agent forwarding — a root admin on the bastion can use your forwarded agent; ProxyJump never exposes keys.
- Debug from both ends at once: client `ssh -vvv`, server `journalctl -u sshd -f` during the attempt.
- Host key changed after a rebuild: `ssh-keygen -R <host>`. Idle drops: `ServerAliveInterval 60` in `~/.ssh/config`. Host blocks: first match wins — specific hosts above wildcards.

## Systemd And Cron

- `systemctl enable` only wires boot; `start` only affects now — `enable --now` for both.
- `After=` orders, it never pulls in: pair `Wants=network-online.target` WITH `After=network-online.target`. `After=network.target` alone means "the network stack exists", not "the network is up" — the classic boot-only failure.
- `Restart=on-failure` alone trips the start limit: defaults are 5 starts per 10s window, then "start request repeated too quickly" and systemd gives up. Add `RestartSec=5`: 5 retries then span ≥20s, which never exhausts the 10s window.
- Persistent journal without touching config: default `Storage=auto` persists only if `/var/log/journal` exists — `mkdir -p /var/log/journal && systemctl restart systemd-journald`.
- `disable` removes boot start but dependencies can still activate the unit; `mask` blocks every activation path.
- systemd timers vs cron: `Persistent=true` runs a missed job after downtime; cron silently skips it (anacron excepted).
- Cron runs with minimal PATH (`/usr/bin:/bin`) and no shell profile — reproduce failures with `env -i /bin/sh -c 'your command'` before blaming the job.
- `%` in a crontab line means newline (rest becomes stdin) — escape it: `date +\%F`.
- Overlap guard for any job that can outlast its interval: `flock -n /var/lock/job.lock <cmd>` — a slow run plus the next tick = two instances corrupting shared state.
- Capture both streams: `cmd >> /var/log/job.log 2>&1` — default output goes to local mail nobody reads.
- `@reboot` fires on cron daemon restart too, not only on boot. And keep `crontab -l > ~/crontab.bak` current: `-r` sits next to `-e` and wipes everything with no undo.

## Commands That Lie

- `top` %CPU is per-core: 400% = 4 cores saturated, not a bug.
- Load average counts D-state (disk-wait) tasks: load 30 with idle CPU is a storage incident, not a compute one.
- `df` vs `du` disagreement is data: deleted-but-open files or shadowed mounts (→ Disk And Filesystem).
- `which` misses aliases, functions, and builtins — `type -a <cmd>` shows what the shell will actually run.
- `du -sh *` skips dotfiles — `du -sh .` for the true directory total.
- `ping` success proves ICMP reachability only, never that a port is open — `nc -zv host port` or `curl` the actual service.
- `ps aux` %MEM can sum past 100% — shared pages counted once per process (→ PSS in Memory And OOM).
- `df` shows the filesystem, not the physical disk — thin-provisioned and overlay storage can run out underneath a "half-empty" filesystem.

## Output Gates

Before running a destructive or remote-risky command, verify:

- Variables in destructive paths expanded and echoed first? `rm -rf` targets use `"${VAR:?}"`.
- Fallback session open before touching sshd, sudoers, firewall, or network config on a remote host?
- SIGTERM sent and waited before any `-9`?
- Persistence step included, or is this change gone on reboot?
- Match blast radius previewed? `pgrep -af` before `pkill -f`; `find ... -print` before `-delete`; `lsblk -f` immediately before `mkfs`/`dd`.

## Traps

| Trap | Why it fails | Do instead |
|------|-------------|------------|
| `sudo echo x > /etc/file` | The redirect runs in YOUR shell before sudo starts | `echo x \| sudo tee /etc/file` |
| Editing sudoers or fstab bare | One typo = no sudo at all / unbootable box | `visudo`; edit fstab then `mount -a` to test before reboot |
| Fixing web perms with `chmod -R 777` | Masks the real cause (ACL mask, SELinux, wrong owner) and makes every file writable by any local process | Diagnose: `namei -l`, `getfacl`, `ls -Z` |
| Testing a cron job in your login shell | Your shell has PATH and env that cron lacks — it proves nothing | `env -i /bin/sh -c 'cmd'` |
| Storing state in `/tmp` | Cleared on reboot, often RAM-backed tmpfs | `/var/tmp` for reboot-surviving temp data |
| `dd`/`mkfs` on a device name from memory | `sda` vs `sdb` is one keystroke and device names reorder across boots | `lsblk -f` immediately before; address disks by UUID or /dev/disk/by-id |

## Related Skills

More Clawic skills, get them at https://clawic.com/skills/linux (install if the user confirms):
- `bash` — shell scripting syntax and safety, beyond OS behavior
- `sysadmin` — routine server administration: users, storage, maintenance
- `docker` — container builds, images, and runtime debugging
- `debugging` — general fault-isolation strategy when the bug isn't OS-level
- `vps` — provisioning and securing rented servers end to end

## Feedback

- If useful, star it: https://clawic.com/skills/linux
- Latest version: https://clawic.com/skills/linux

Part of [Clawic](https://clawic.com), the verified skill library. Get this skill: https://clawic.com/skills/linux.
