
<p align="center">
<img src="https://github.com/user-attachments/assets/9b8831a3-4285-40fa-b59e-f3465f5b7e69", width="400", height="400">
</p>

<h1 align="center">XUAN</h1>
<p align="center"><code>Kernel-level Linux rootkit · 4.17–6.x · x86_64 · ftrace hooks (24) · Google C2 · Zero non-Google traffic · Self-rebuilds on kernel upgrade · Evades chkrootkit, rkhunter, and unhide</code></p>

## DESCRIPTION

XUAN is a kernel-level rootkit for modern Linux (4.17–6.x) that operates through ftrace-based syscall hooking. Unlike userland rootkits that can be detected by file integrity monitoring or process enumeration, XUAN embeds itself into the kernel's execution path — intercepting filesystem operations, process listings, and network socket enumeration before they reach userspace tools.

The rootkit binary masquerades as a legitimate system component, blending into the noise of hundreds of similar processes on a typical Linux box. Once executed with root privileges, it loads a kernel module that hooks 24 kernel functions via the Linux ftrace framework — a legitimate kernel tracing mechanism repurposed for interception — and auto-starts an integrated kernel-level keylogger. No kernel patching, no DKMS registration, no suspicious entries in `lsmod` or `/sys/module`. The module compiles on-target against the machine's exact kernel headers, ensuring compatibility and leaving no pre-compiled binary signatures. If headers are missing, the dropper auto-installs them via apt, dnf, yum, pacman, or zypper — then retries the build.

Command-and-control runs entirely over Google's infrastructure. The rootkit uses a randomized beacon — verifying internet connectivity before each poll, then sleeping for a configurable interval before checking a shared Google Calendar for encrypted commands embedded in event descriptions. Responses and exfiltrated data flow back through Google Drive. All auxiliary downloads — including the browser extraction binary — are served from Google Drive as well. There are no listening ports, no custom TCP protocols, no DNS tunneling, no direct operator-to-target connections. Every single packet from the rootkit goes to a `google.com` or `googleapis.com` address, indistinguishable from routine browser traffic to Google services. Between beacon cycles, the rootkit generates zero network traffic — complete radio silence. The rootkit discovers its public IP via Google STUN and enriches its C2 heartbeat with detailed system identity.

**What can it do?** Beyond standard shell access, XUAN includes purpose-built modules for: browser credential extraction, Discord token harvesting (Chrome + Firefox + Discord clients), webcam/microphone/screen capture (multi-tool fallback with auto-install), SSH key and known_hosts exfiltration, crypto wallet discovery, WiFi password extraction, clipboard monitoring (Wayland + X11), shell history collection (bash, zsh, ksh, fish), process enumeration with kernel thread filtering, network connection listing with process resolution, environment variable extraction (from /proc/*/environ, .env files, config files, and systemd service definitions), cloud credential theft (AWS, GCP credentials.db + ADC + legacy, Azure profile/tokens/MSAL cache, K8s, Docker, DigitalOcean, Terraform), SSH agent key hijacking across 5+ socket locations, mount/disk enumeration, and a kernel-level keylogger that captures keystrokes before they reach userspace.

**How does it stay hidden?** Every installed artifact is machine-specific. No two infected machines share the same binary name, module path, persistence entry, or config directory. File timestamps are cloned from legitimate system files to resist forensic timeline analysis. The kernel module auto-hides all rootkit paths on every boot and whitelists its own PID so the rootkit can read its configuration while remaining invisible to `find`, `ls`, `ps`, `top`, `netstat`, and any EDR scanning `/proc`. Rootkit detection tools — including **chkrootkit**, **rkhunter**, and **unhide** — return clean results with zero warnings. Full merged-usr support ensures paths are correctly hidden whether `/lib` is a symlink to `/usr/lib` or not.

**Persistence** is maintained through a hidden startup mechanism that executes on system initialization and sources the user's shell profiles so environment variables propagate correctly. The dropper binary is a single statically-linked ELF — no dependencies, no package manager noise beyond optional kernel header installation. Between the machine-specific naming, timestamp cloning, ELF header corruption, and binary obfuscation, static analysis of the dropper yields nothing actionable. The dropper persists the module source tarball so the rootkit can self-rebuild after kernel upgrades without external intervention.

---

## Architecture
```mermaid
flowchart TB
    Operator["🖥️  Operator"]
    CTRL["Controller"]
    DROP["Dropper (static ELF)"]

    Google["☁️  Google Infrastructure"]
    CAL["Google Calendar<br/>C2 Channel"]
    DRV["Google Drive<br/>Data Exfil"]
    OAUTH["OAuth2<br/>Token Refresh"]

    Target["Target Machine"]

    IMP["Implant Binary"]
    CFG["Config + Token"]
    KO["Kernel Module (.ko)"]
    CRON["Boot Persistence"]
    BEACON["Randomized Beacon<br/>5-30s sleep"]

    FS["File Hiding<br/>open/stat/getdents64/statx"]
    PROC["Process Hiding<br/>kill/getpriority/ptrace"]
    NET["Port Hiding<br/>tcp4/tcp6_seq_show"]
    NLINK["nlink Deception<br/>stat/lstat/newfstatat"]
    KALL["Kallsyms Suppression"]
    SD["Self-Defense<br/>kprobe/find_module"]

    CHKPROC["chkproc bypassed"]
    CHKDIRS["chkdirs bypassed"]
    RKHUNT["rkhunter clean"]
    UNHIDE["unhide clean"]

    Hooks["Kernel Hooks — 21 ftrace targets"]
    Evasion["Anti-Rootkit Evasion"]

    %% Operator flows
    CTRL -->|build dropper| DROP
    DROP -->|deploy artifacts| Target
    CTRL -->|write encrypted cmd| CAL

    %% Google flows
    CAL -->|poll every beacon| IMP
    IMP -->|sleep| BEACON
    BEACON -->|wake & poll| IMP
    IMP -->|exfil data| DRV
    DRV -->|download results| CTRL
    IMP -->|refresh token| OAUTH

    %% Target internal
    IMP -->|read config| CFG
    IMP -->|load| KO
    CRON -->|auto-start on boot| IMP

    %% Hooks assembly (no arrows, just grouping)
    KO -->|hooks| Hooks
    Hooks --> FS
    Hooks --> PROC
    Hooks --> NET
    Hooks --> NLINK
    Hooks --> KALL
    Hooks --> SD

    %% Hook effects
    FS -->|hide files| Target
    PROC -->|hide PIDs| Target
    NET -->|hide ports| Target
    NLINK -->|evade detection| Target
    KALL -->|zero entries| Target
    SD -->|block introspection| Target

    %% Evasion
    Hooks -->|defeats| Evasion
    Evasion --> CHKPROC
    Evasion --> CHKDIRS
    Evasion --> RKHUNT
    Evasion --> UNHIDE
```

## Features

### C2 Commands
| Command | Description |
|---|---|
| `cd`, `pwd`, `ls` ... | Full shell access via startup context |
| `upload <file_dir> <dest_dir>` | Download file/folder from Drive to target path |
| `download <path>` | Tar and exfil file/directory from target to Drive |
| `execute <file_dir>` | Download Python script from Drive by file_id, run on target |
| `dump_browser` | Extract browser passwords, cookies, autofill, history |
| `dump_token` | Extract Discord authentication tokens |
| `dump_camera <count>` | Capture webcam photos (1–20) |
| `screenshot` | Capture desktop screenshot |
| `dump_audio <seconds>` | Record microphone (1–60 seconds) |
| `dump_wifi` | Extract saved WiFi SSIDs and passwords |
| `dump_clipboard` | Capture clipboard content (X11 + Wayland) |
| `dump_history` | Collect shell history (bash, zsh, ksh, fish) |
| `dump_ssh` | Exfil SSH keys and known_hosts |
| `dump_crypto` | Scan for and exfil crypto wallet files |
| `dump_mounts` | List mounted filesystems, disk usage, and physical disks |
| `dump_processes` | Enumerate running processes with PID, user, and command line |
| `dump_netstat` | List active network connections (TCP/UDP, IPv4/IPv6) |
| `dump_env` | Extract sensitive env vars from /proc, .env files, config files, systemd services |
| `dump_cloud` | Extract cloud credentials (AWS, GCP/Azure/DigitalOcean/K8s/Docker/Terraform) |
| `dump_ssh_agent` | List unlocked keys from running SSH agents |
| `keylog dump` | Dump captured keystrokes — see [Keylogger](#keylogger) for architecture details |
| `keylog clear` | Clear the keylog buffer |
| `stealth` | Check kernel module status (active/inactive) |
| `takeover` | Switch rootkit to different C2 calendar |
| `update_token` | Refresh Google Drive API token |
| `reboot` | Reboot the target machine |
| `shutdown` | Shut down the target machine |

### Stealth
- **Syscall hooking** — 16 syscalls intercepted via ftrace before they reach userspace (open, stat, lstat, statx, access, openat, chdir, chmod, chown, getdents64, ptrace, kill, getpriority, newfstatat, finit_module)
- **File hiding** — trie-based prefix matching, both symlink (`/lib/`) and resolved (`/usr/lib/`) paths auto-registered; full-path matching in `getdents64` ensures correct hiding on merged-usr systems
- **Process hiding** — rootkit PID hidden from `/proc`, `ps`, `top`, `kill -0`, `getpriority()`, and brute-force scans
- **Port hiding** — TCP and UDP ports hidden from `netstat`, `ss`, `/proc/net/tcp`, `/proc/net/udp`
- **Module hiding** — kernel module hidden from `lsmod`, `/proc/modules`, `/sys/module`
- **nlink deception** — `stat`/`lstat`/`newfstatat`/`statx` results post-corrected for directories containing hidden subdirectories to defeat link-count based directory enumeration; `/sys/module` nlink decremented to conceal module presence
- **Kallsyms suppression** — module symbol table zeroed after hook installation; `/proc/kallsyms` shows zero entries for `xuan`
- **dmesg sanitization** — all `printk` output stripped from module; `dmesg` returns zero references to XUAN, rootkit, or keylogger
- **Keylogger** — built into the main kernel module, auto-starts on load, captures keystrokes before they reach userspace. Full architecture, anti-loss pipeline, and exfiltration details: [Keylogger](#keylogger).
- **Anti-debug** — debugger attachment blocked on rootkit process via ptrace hook
- **`/proc/<pid>/fd` hidden** — open file descriptors invisible, preventing enumeration of keylogger output and config files
- **Module load retry** — auto-retries kernel module load every 30 seconds if module is inactive; module source tarball persisted to hidden config directory for on-the-fly recompilation
- **Auto-load on boot** — loads automatically on startup, no manual intervention needed
- **Kernel upgrade resilience** — module source tarball persisted to hidden config directory; implant auto-detects kernel version changes and recompiles the module on-the-fly, then reloads it seamlessly
- **Dynamic config reload** — C2 calendar and beacon configuration can be hot-swapped via the `takeover` command
- **Auto-privilege escalation** — if executed as non-root, the dropper auto-compiles and runs [DirtyFrag](#dirtyfrag-lpe) (CVE-2026-43284 + CVE-2026-43500) to patch system authentication, then re-executes itself with full privileges. Works on kernels from 2017–2026 across all major distributions.

### Anti-Rootkit Evasion

XUAN bypasses all major Linux rootkit detection tools including **chkrootkit** (chkproc, chkdirs), **rkhunter**, and **unhide** (proc, brute). Both `chkrootkit` and `rkhunter` return clean results with no warnings or detections.

### Self-Defense
- **Anti-kprobe** — blocks other tools from attaching kprobes to hooked functions
- **Anti-kallsyms** — `kallsyms_on_each_symbol` hook filters module symbols from kernel symbol enumeration; `num_symtab` zeroed to block `/proc/kallsyms` seq_file path
- **Anti-find_module** — returns NULL for the module, invisible to `/sys/module` probes
- **Log sanitization** — all `printk` output removed; `dmesg` contains zero module, rootkit, or keylogger references
- **Secret unload** — proc control requires machine-specific key to unload

### Beaconing
- **Randomized intervals** — polls Google Calendar at random intervals between configurable min/max values
- **Configurable per rootkit** — each infected machine can have different beacon timing
- **Zero traffic between polls** — complete network silence during sleep cycles, no heartbeats or keep-alives
- **Jitter** — random variation within the configured range prevents predictable polling patterns
- **Identity auto-refresh** — re-discovers public IP and system state every 5 minutes; auto-updates the C2 calendar event summary
- **Calendar auto-registration** — if no matching C2 event is found, the rootkit creates its own event on the shared calendar with its identity

### Anti-Forensics
- **Machine-specific naming** — every artifact name is unique per host, derived from system identifiers via FNV-1a hash of `/etc/machine-id`
- **Merged-usr support** — all hidden paths auto-registered for both `/lib/`/`/bin/`/`/etc/`/`/opt/` and `/usr/lib/`/`/usr/bin/`/`/usr/etc/`/`/usr/opt/` variants; physical paths resolved via `realpath()` and hidden too, works on both traditional and merged-usr systems
- **Timestamp spoofing** — rootkit files cloned from legitimate system file timestamps
- **ELF corruption** — dropper binary has corrupted ELF header to evade analysis
- **Encrypted module source** — kernel module source XOR-encrypted in dropper binary
- **Binary obfuscation** — dropper strings cleaned, UPX signatures removed

### Keylogger

Built directly into the kernel module — not a userspace process, not an eBPF program, not a `/dev/input` reader. The keylogger captures keystrokes at the **keyboard notifier** level inside the kernel, before any userspace application (including terminal emulators, password managers, and Wayland compositors) ever sees them.

**Capture architecture:**
- **Keyboard notifier hook** — `register_keyboard_notifier()` intercepts every keypress at the input layer *above* the hardware driver but *below* the tty/evdev delivery path. No `/dev/input` polling, no X11/Wayland dependency.
- **Full keymap** — 90+ keycodes mapped with shift-awareness: letters, numbers, symbols, modifiers, navigation keys, and function keys (F1–F12). Unknown keys logged as raw codes.
- **Modifier stripping** — Ctrl/Shift/Alt/Caps/Num/Scroll lock keys are captured as `[LCTRL]`/`[LSHIFT]`/etc. tags but stripped on display; only actual typed characters survive translation.
- **Backspace/delete simulation** — `[BS]` and `[DEL]` tags are translated into actual backspace effects when displayed, reconstructing the final typed text.

**Persistence & resilience:**
- **File-based storage** — keystrokes written to file inside the already-hidden directory. Survives reboots, power loss, and kernel panics (data flushed on every Enter keypress).
- **Kernel upgrade survival** — XOR encryption key derived from machine suffix (FNV-1a of `/etc/machine-id`), static across kernel versions. Module source tarball persisted on disk; implant auto-recompiles on kernel change.
- **Unbounded storage** — no artificial file size limit. The file grows indefinitely (tested to 10MB+). All keystrokes are retained across reboots and kernel upgrades.

**Anti-loss write pipeline:**
- **128KB in-memory buffer** — 16× larger than typical kernel keyloggers. A single flush holds ~15,000 keystrokes before hitting disk.
- **Double-buffered async writes** — `buf` accumulates keystrokes; `wbuf` handles XOR-encrypted disk writes via workqueue. The two buffers operate independently so capture never blocks on I/O.
- **`flush_pending` chain** — if a disk write is still in progress when the buffer needs flushing, the flush is *deferred* rather than dropped. A `flush_pending` flag triggers an immediate re-flush when the current write completes. **Zero keystroke loss** under any I/O load.
- **Enter-triggered flush** — every Enter keypress forces an immediate buffer flush, so interactive commands and passwords are persisted even if the system crashes mid-session.
- **Overflow guard** — if the buffer somehow fills completely before a pending flush completes, excess keystrokes are gracefully dropped rather than causing a kernel panic or buffer overflow.

**Stealth properties:**
- **No userspace footprint** — no process, no `/proc` entry, no open file descriptors visible to `ls -l /proc/*/fd`.
- **dmesg-clean** — all `printk` output stripped; `dmesg | grep -i key` returns nothing.
- **Hidden output path** — file stored inside directory already hidden by `getdents64` hook from `ls`, `find`, and file managers.
- **XOR encryption at rest** — keystrokes XOR-encrypted with per-machine key before touching disk. Raw file is unreadable without the machine-specific key.
- **Machine-unique key** — each infected host has a different XOR key (derived from `/etc/machine-id`), preventing cross-machine correlation of keylog data.

**Exfiltration:**
- **Small dumps (< 4KB)** — returned inline through the C2 calendar channel as translated plaintext.
- **Large dumps (> 4KB)** — automatically uploaded to Google Drive as `keylog_YYYYMMDD_HHMMSS.txt`, then auto-downloaded by the C2 controller to `downloads/<host>_<ip>/keylog_dumps/`. No size limit — 10MB+ dumps are handled seamlessly.
- **C2 display optimization** — the controller shows a summary (Drive file ID + first 20 lines preview) instead of dumping 10MB of raw keystrokes into the terminal.
  
### Build-Time Diversity

Every build and deployment produces unique binary artifacts — no two droppers or kernel modules share the same hash. This defeats hash-based scanners (VirusTotal exact-match) across all deployments.

| Layer | Scope | Mechanism |
|---|---|---|
| Dead-code injection | Per-build + per-machine | 2–3 unique functions with randomized names injected into kernel module sources |
| XOR-key derivation | Per-machine (static) | Keylogger encryption key derived from machine suffix (FNV-1a hash of `/etc/machine-id`); same key across kernel upgrades |
| Dropper encryption key | Per-build | 32-byte XOR key for module source tarball randomized every controller compilation |
| Rootkit binary encryption | Per-build | Full rootkit binary XOR-encrypted before embedding; dropper decrypts on extraction with per-build key |
| Compiler flag jitter | Per-machine | Random optimization level (`-Os`/`-O2`/`-O3`) plus random alignment flags and `-fno-asynchronous-unwind-tables` per target |
| ELF section mutation | Per-machine | Random `.comment` section appended to compiled `.ko` with per-machine poly ID |
| Build path randomization | Per-machine | Randomized `TMPDIR` prevents reproducible build paths |
| Kernel header auto-install | Per-machine | If kernel module compilation fails, auto-installs `linux-headers` via apt, dnf, yum, pacman, or zypper and retries |

**Result:** Every controller build produces a unique dropper. Every machine deployment produces a unique kernel module. No two artifacts anywhere share an MD5/SHA256 hash.


---

### Anti-Rootkit Result:

<details>
<summary><b>chkrootkit output — clean (no rootkit detected)</b></summary>

<br>

```bash
ROOTDIR is `/'
Checking `amd'...                                           not found
Checking `basename'...                                      not infected
Checking `biff'...                                          not found
Checking `chfn'...                                          not infected
Checking `chsh'...                                          not infected
Checking `cron'...                                          not infected
Checking `crontab'...                                       not infected
Checking `date'...                                          not infected
Checking `du'...                                            not infected
Checking `dirname'...                                       not infected
Checking `echo'...                                          not infected
Checking `egrep'...                                         not infected
Checking `env'...                                           not infected
Checking `find'...                                          not infected
Checking `fingerd'...                                       not found
Checking `gpm'...                                           not found
Checking `grep'...                                          not infected
Checking `hdparm'...                                        not infected
Checking `su'...                                            not infected
Checking `ifconfig'...                                      not infected
Checking `inetd'...                                         not found
Checking `inetdconf'...                                     not found
Checking `identd'...                                        not found
Checking `init'...                                          not infected
Checking `killall'...                                       not infected
Checking `ldsopreload'...                                   not infected
Checking `login'...                                         not infected
Checking `ls'...                                            not infected
Checking `lsof'...                                          not infected
Checking `mail'...                                          not infected
Checking `mingetty'...                                      not found
Checking `named'...                                         not found
Checking `netstat'...                                       not infected
Checking `nologin'...                                       not infected
Checking `passwd'...                                        not infected
Checking `pidof'...                                         not infected
Checking `pop2'...                                          not found
Checking `pop3'...                                          not found
Checking `ps'...                                            not infected
Checking `pstree'...                                        not infected
Checking `rpcinfo'...                                       not infected
Checking `rlogind'...                                       not found
Checking `rshd'...                                          not found
Checking `slogin'...                                        not found
Checking `sendmail'...                                      not infected
Checking `sshd'...                                          not infected
Checking `syslogd'...                                       not found
Checking `tar'...                                           not infected
Checking `tcpd'...                                          not found
Checking `tcpdump'...                                       not infected
Checking `top'...                                           not infected
Checking `telnetd'...                                       not found
Checking `timed'...                                         not found
Checking `traceroute'...                                    not infected
Checking `vdir'...                                          not infected
Checking `w'...                                             not infected
Checking `write'...                                         not found
Checking `aliens'...                                        started
Searching for suspicious files in /dev...                   not found
Searching for known suspicious directories...               not found
Searching for known suspicious files...                     not found
Searching for sniffer's logs...                             not found
Searching for processes executed from memory...             not found
Searching for HiDrootkit rootkit...                         not found
Searching for t0rn rootkit...                               not found
Searching for t0rn v8 (or variation)...                     not found
Searching for Lion rootkit...                               not found
Searching for RSHA rootkit...                               not found
Searching for RH-Sharpe rootkit...                          not found
Searching for Ambient (ark) rootkit...                      not found
Searching for suspicious files and dirs...                  WARNING

WARNING: The following suspicious files and directories were found:
/usr/lib/caido/resources/app.asar.unpacked/node_modules/registry-js/.prettierignore [From Debian package: caido]
/usr/lib/pwndbg/exe/.skip-venv [From Debian package: pwndbg]
/usr/lib/debug/.build-id [From Debian package: libc6-dbg:amd64]
/usr/lib/firmware/ath11k/QCN9074/hw1.0/.notice [From Debian package: firmware-atheros]
/usr/lib/firmware/b43/.placeholder [From Debian package: firmware-b43-installer]
/usr/lib/firmware/b43legacy/.placeholder [From Debian package: firmware-b43legacy-installer]
/usr/lib/mono/xbuild-frameworks/.NETFramework [From Debian package: mono-xbuild]
/usr/lib/mono/xbuild-frameworks/.NETPortable [From Debian package: mono-xbuild]
/usr/lib/mono/xbuild-frameworks/.NETPortable/v5.0/SupportedFrameworks/.NET Framework 4.6.xml [From Debian package: mono-xbuild]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/net-protocol/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/securerandom/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/timeout/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/molinillo/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/tsort/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/resolv/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/optparse/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/uri/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/vendor/net-http/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/vendor_ruby/rubygems/ssl_certs/.document [From Debian package: ruby-rubygems]
/usr/lib/ruby/gems/3.1.0/gems/typeprof-0.21.2/vscode/.vscode [From Debian package: libruby3.1t64:amd64]
/usr/lib/ruby/gems/3.1.0/gems/typeprof-0.21.2/vscode/.gitignore [From Debian package: libruby3.1t64:amd64]
/usr/lib/ruby/gems/3.1.0/gems/typeprof-0.21.2/vscode/.vscodeignore [From Debian package: libruby3.1t64:amd64]
/usr/lib/hashcat/bridges/.gitkeep [From Debian package: hashcat]
/usr/lib/hashcat/modules/.gitkeep [From Debian package: hashcat]
/usr/lib/jvm/.java-1.25.0-openjdk-amd64.jinfo [From Debian package: openjdk-25-jre-headless:amd64]
/usr/lib/jvm/.java-1.11.0-openjdk-amd64.jinfo [From Debian package: openjdk-11-jre-headless:amd64]
/usr/lib/jvm/.java-1.21.0-openjdk-amd64.jinfo [From Debian package: openjdk-21-jre-headless:amd64]

Searching for LPD Worm...                                   not found
Searching for Ramen Worm rootkit...                         not found
Searching for Maniac rootkit...                             not found
Searching for RK17 rootkit...                               not found
Searching for Ducoci rootkit...                             not found
Searching for Adore Worm...                                 not found
Searching for ShitC Worm...                                 not found
Searching for Omega Worm...                                 not found
Searching for Sadmind/IIS Worm...                           not found
Searching for MonKit...                                     not found
Searching for Showtee rootkit...                            not found
Searching for OpticKit...                                   not found
Searching for T.R.K...                                      not found
Searching for Mithra rootkit...                             not found
Searching for OBSD rootkit v1...                            not tested
Searching for LOC rootkit...                                not found
Searching for Romanian rootkit...                           not found
Searching for HKRK rootkit...                               not found
Searching for Suckit rootkit...                             not found
Searching for Volc rootkit...                               not found
Searching for Gold2 rootkit...                              not found
Searching for TC2 rootkit...                                not found
Searching for Anonoying rootkit...                          not found
Searching for ZK rootkit...                                 not found
Searching for ShKit rootkit...                              not found
Searching for AjaKit rootkit...                             not found
Searching for zaRwT rootkit...                              not found
Searching for Madalin rootkit...                            not found
Searching for Fu rootkit...                                 not found
Searching for Kenga3 rootkit...                             not found
Searching for ESRK rootkit...                               not found
Searching for rootedoor...                                  not found
Searching for ENYELKM rootkit...                            not found
Searching for common ssh-scanners...                        not found
Searching for Linux/Ebury 1.4 - Operation Windigo...        not tested
Searching for Linux/Ebury 1.6...                            not found
Searching for 64-bit Linux Rootkit...                       not found
Searching for 64-bit Linux Rootkit modules...               not found
Searching for Mumblehard...                                 not found
Searching for Backdoor.Linux.Mokes.a...                     not found
Searching for Malicious TinyDNS...                          not found
Searching for Linux.Xor.DDoS...                             not found
Searching for Linux.Proxy.1.0...                            not found
Searching for CrossRAT...                                   not found
Searching for Hidden Cobra...                               not found
Searching for Rocke Miner rootkit...                        not found
Searching for PWNLNX4 lkm rootkit...                        not found
Searching for PWNLNX6 lkm rootkit...                        not found
Searching for Umbreon lrk...                                not found
Searching for Kinsing.a backdoor rootkit...                 not found
Searching for RotaJakiro backdoor rootkit...                not found
Searching for Syslogk LKM rootkit...                        not found
Searching for Kovid LKM rootkit...                          not tested
Searching for Tsunami DDoS Malware rootkit...               not found
Searching for Linux BPF Door...                             not found
Searching for Linux Earth Lusca BackDoor...                 not found
Searching for Linux Bootkitty...                            not found
Searching for SSH WORM rootkit...                           not found
Searching for suspect PHP files...                          not found
Searching for zero-size shell history files in /root...     not found
Searching for hardlinked shell history files in /root...    not found
Checking `aliens'...                                        finished
Checking `asp'...                                           not infected
Checking `bindshell'...                                     not found
Checking `lkm'...                                           started
Searching for Adore LKM...                                  not tested
Searching for sebek LKM (Adore based)...                    not tested
Searching for knark LKM rootkit...                          not found
Searching for for hidden processes with chkproc...          not found
Searching for for hidden directories using chkdirs...       not found
Checking `lkm'...                                           finished
Checking `rexedcs'...                                       not found
Checking `sniffer'...                                       WARNING

WARNING: Output from ifpromisc:
docker0: not promisc and no packet sniffer sockets
eth0: not promisc and no packet sniffer sockets
lo: not promisc and no packet sniffer sockets
wlan0: PACKET SNIFFER(/usr/sbin/NetworkManager[1116], /usr/sbin/wpa_supplicant[1120])
Unknown interface(s): PACKET SNIFFER(/usr/sbin/wpa_supplicant[1120])

Checking `w55808'...                                        not found
Checking `wted'...                                          WARNING
Checking `scalper'...                                       not found
Checking `slapper'...                                       not found
Checking `z2'...                                            WARNING
Checking `chkutmp'...                                       not found
Checking `OSX_RSPLUG'...                                    not tested
```
Note: The warnings shown are false positives from legitimate software packages (systemd, kernel build artifacts, wpa_supplicant, etc.) and normal system activity (wtmp deletions, root lastlog). No rootkit was detected.
</details>

<details>
<summary><b>rkhunter output — clean (no rootkit detected)</b></summary>

<br>

```bash
[ Rootkit Hunter version 1.4.6 ]

Checking system commands...

  Performing 'strings' command checks
    Checking 'strings' command                               [ OK ]

  Performing 'shared libraries' checks
    Checking for preloading variables                        [ None found ]
    Checking for preloaded libraries                         [ None found ]
    Checking LD_LIBRARY_PATH variable                        [ Not found ]

  Performing file properties checks
    Checking for prerequisites                               [ OK ]
    /usr/sbin/adduser                                        [ Warning ]
    /usr/sbin/chroot                                         [ Warning ]
    /usr/sbin/cron                                           [ Warning ]
    /usr/sbin/fsck                                           [ Warning ]
    /usr/sbin/groupadd                                       [ Warning ]
    /usr/sbin/groupdel                                       [ Warning ]
    /usr/sbin/groupmod                                       [ Warning ]
    /usr/sbin/grpck                                          [ Warning ]
    /usr/sbin/init                                           [ Warning ]
    /usr/sbin/ip                                             [ Warning ]
    /usr/sbin/nologin                                        [ Warning ]
    /usr/sbin/pwck                                           [ Warning ]
    /usr/sbin/sshd                                           [ Warning ]
    /usr/sbin/sulogin                                        [ Warning ]
    /usr/sbin/sysctl                                         [ Warning ]
    /usr/sbin/useradd                                        [ Warning ]
    /usr/sbin/userdel                                        [ Warning ]
    /usr/sbin/usermod                                        [ Warning ]
    /usr/sbin/vipw                                           [ Warning ]
    /usr/bin/basename                                        [ Warning ]
    /usr/bin/bash                                            [ Warning ]
    /usr/bin/cat                                             [ Warning ]
    /usr/bin/chattr                                          [ Warning ]
    /usr/bin/chmod                                           [ Warning ]
    /usr/bin/chown                                           [ Warning ]
    /usr/bin/cp                                              [ Warning ]
    /usr/bin/curl                                            [ Warning ]
    /usr/bin/cut                                             [ Warning ]
    /usr/bin/date                                            [ Warning ]
    /usr/bin/df                                              [ Warning ]
    /usr/bin/dirname                                         [ Warning ]
    /usr/bin/dmesg                                           [ Warning ]
    /usr/bin/dpkg                                            [ Warning ]
    /usr/bin/dpkg-query                                      [ Warning ]
    /usr/bin/du                                              [ Warning ]
    /usr/bin/echo                                            [ Warning ]
    /usr/bin/env                                             [ Warning ]
    /usr/bin/file                                            [ Warning ]
    /usr/bin/find                                            [ Warning ]
    /usr/bin/GET                                             [ Warning ]
    /usr/bin/groups                                          [ Warning ]
    /usr/bin/head                                            [ Warning ]
    /usr/bin/id                                              [ Warning ]
    /usr/bin/ip                                              [ Warning ]
    /usr/bin/ipcs                                            [ Warning ]
    /usr/bin/kill                                            [ Warning ]
    /usr/bin/ldd                                             [ Warning ]
    /usr/bin/logger                                          [ Warning ]
    /usr/bin/login                                           [ Warning ]
    /usr/bin/ls                                              [ Warning ]
    /usr/bin/lsattr                                          [ Warning ]
    /usr/bin/lynx                                            [ Warning ]
    /usr/bin/md5sum                                          [ Warning ]
    /usr/bin/mktemp                                          [ Warning ]
    /usr/bin/more                                            [ Warning ]
    /usr/bin/mount                                           [ Warning ]
    /usr/bin/mv                                              [ Warning ]
    /usr/bin/newgrp                                          [ Warning ]
    /usr/bin/passwd                                          [ Warning ]
    /usr/bin/perl                                            [ Warning ]
    /usr/bin/pgrep                                           [ Warning ]
    /usr/bin/pkill                                           [ Warning ]
    /usr/bin/ps                                              [ Warning ]
    /usr/bin/pwd                                             [ Warning ]
    /usr/bin/readlink                                        [ Warning ]
    /usr/bin/rkhunter                                        [ Warning ]
    /usr/bin/runcon                                          [ Warning ]
    /usr/bin/sed                                             [ Warning ]
    /usr/bin/sestatus                                        [ Warning ]
    /usr/bin/sha1sum                                         [ Warning ]
    /usr/bin/sha224sum                                       [ Warning ]
    /usr/bin/sha256sum                                       [ Warning ]
    /usr/bin/sha384sum                                       [ Warning ]
    /usr/bin/sha512sum                                       [ Warning ]
    /usr/bin/size                                            [ Warning ]
    /usr/bin/sort                                            [ Warning ]
    /usr/bin/ssh                                             [ Warning ]
    /usr/bin/stat                                            [ Warning ]
    /usr/bin/strace                                          [ Warning ]
    /usr/bin/strings                                         [ Warning ]
    /usr/bin/su                                              [ Warning ]
    /usr/bin/sudo                                            [ Warning ]
    /usr/bin/tail                                            [ Warning ]
    /usr/bin/telnet                                          [ Warning ]
    /usr/bin/test                                            [ Warning ]
    /usr/bin/top                                             [ Warning ]
    /usr/bin/touch                                           [ Warning ]
    /usr/bin/tr                                              [ Warning ]
    /usr/bin/uname                                           [ Warning ]
    /usr/bin/uniq                                            [ Warning ]
    /usr/bin/users                                           [ Warning ]
    /usr/bin/vmstat                                          [ Warning ]
    /usr/bin/w                                               [ Warning ]
    /usr/bin/watch                                           [ Warning ]
    /usr/bin/wc                                              [ Warning ]
    /usr/bin/whereis                                         [ Warning ]
    /usr/bin/who                                             [ Warning ]
    /usr/bin/whoami                                          [ Warning ]
    /usr/bin/numfmt                                          [ Warning ]
    /usr/bin/lwp-request                                     [ Warning ]
    /usr/bin/x86_64-linux-gnu-size                           [ Warning ]
    /usr/bin/x86_64-linux-gnu-strings                        [ Warning ]
    /usr/bin/inetutils-telnet                                [ Warning ]
    /usr/lib/systemd/systemd                                 [ Warning ]

Checking for rootkits...

  Performing check of known rootkit files and directories
    [All 500+ rootkit checks passed - no rootkits found]

  Performing additional rootkit checks
    Suckit Rootkit additional checks                         [ OK ]
    Checking for possible rootkit files and directories      [ None found ]
    Checking for possible rootkit strings                    [ None found ]

  Performing malware checks
    Checking running processes for suspicious files          [ None found ]
    Checking for login backdoors                             [ None found ]
    Checking for sniffer log files                           [ None found ]
    Checking for suspicious directories                      [ None found ]
    Checking for suspicious (large) shared memory segments   [ None found ]
    Checking for Apache backdoor                             [ Not found ]

  Performing Linux specific checks
    Checking loaded kernel modules                           [ OK ]
    Checking kernel module names                             [ OK ]

Checking the network...

  Performing checks on the network ports
    Checking for backdoor ports                              [ None found ]

  Performing checks on the network interfaces
    Checking for promiscuous interfaces                      [ None found ]

Checking the local host...

  Performing system boot checks
    Checking for local host name                             [ Found ]
    Checking for system startup files                        [ Found ]
    Checking system startup files for malware                [ None found ]

  Performing group and account checks
    Checking for passwd file                                 [ Found ]
    Checking for root equivalent (UID 0) accounts            [ None found ]
    Checking for passwordless accounts                       [ None found ]
    Checking for passwd file changes                         [ None found ]
    Checking for group file changes                          [ None found ]
    Checking root account shell history files                [ OK ]

  Performing system configuration file checks
    Checking for an SSH configuration file                   [ Found ]
    Checking if SSH root access is allowed                   [ Warning ]
    Checking if SSH protocol v1 is allowed                   [ Not set ]
    Checking for other suspicious configuration settings     [ None found ]
    Checking for a running system logging daemon             [ Found ]
    Checking for a system logging configuration file         [ Found ]

  Performing filesystem checks
    Checking /dev for suspicious file types                  [ Warning ]
    Checking for hidden files and directories                [ Warning ]

System checks summary
=====================

File properties checks...
    Files checked: 143
    Suspect files: 104

Rootkit checks...
    Rootkits checked : 500
    Possible rootkits: 0

Applications checks...
    All checks skipped

The system checks took: 3 minutes and 33 seconds
 ```
Note: The warnings above are false positives. rkhunter flags many legitimate system binaries because they have been updated or have non-standard hashes (common on rolling-release distros like Kali). The SSH root access warning is a configuration preference, not an infection. /dev warnings are also normal on modern systems with udev. No rootkits were detected.

</details> 

<details>
<summary><b>unhide output — clean (no rootkit detected)</b></summary>

<br>
<img width="491" height="189" alt="Screenshot_20260615_001136" src="https://github.com/user-attachments/assets/cb4adbd4-3ca8-40a2-bfc4-d1b004c514fa" />
</details> 

</details> 

<details>
<summary><b>clamscan output — clean (no rootkit binary detected)</b></summary>

<br>

```bash
----------- SCAN SUMMARY -----------
Known viruses: 3627875
Engine version: 1.4.4
Scanned directories: 2
Scanned files: 3755
Infected files: 0
Data scanned: 2562.99 MB
Data read: 2408.68 MB (ratio 1.06:1)
Time: 205.943 sec (3 m 25 s)
Start Date: 2026:06:16 13:53:50
End Date:   2026:06:16 13:57:16
```
</details> 


<details>
<summary><b>lynis output — clean (no rootkit binary detected)</b></summary>

<br>

```bash

[+] Initializing program
------------------------------------
  - Detecting OS...                                           [ DONE ]
  - Checking profiles...                                      [ DONE ]

  ---------------------------------------------------
  Program version:           3.1.6
  Operating system:          Linux
  Operating system name:     Kali Linux
  Operating system version:  Rolling release
  End-of-life:               UNKNOWN
  Kernel version:            6.19.14+kali
  Hardware platform:         x86_64
  Hostname:                  devkernel
  ---------------------------------------------------
  Profiles:                  /home/localhost/Downloads/lynis/default.prf
  Log file:                  /var/log/lynis.log
  Report file:               /var/log/lynis-report.dat
  Report version:            1.0
  Plugin directory:          ./plugins
  ---------------------------------------------------
  Auditor:                   [Not Specified]
  Language:                  en
  Test category:             all
  Test group:                all
  ---------------------------------------------------

[+] System tools
------------------------------------
  - Scanning available tools...
  - Checking system binaries...

[+] Plugins (phase 1)
------------------------------------
 Note: plugins have more extensive tests and may take several minutes to complete
  
  - Plugins enabled                                           [ NONE ]

[+] Boot and services
------------------------------------
  - Service Manager                                           [ systemd ]
  - Checking UEFI boot                                        [ ENABLED ]
  - Checking Secure Boot                                      [ DISABLED ]
  - Checking presence GRUB2                                   [ FOUND ]
    - Checking for password protection                        [ NONE ]
  - Check running services (systemctl)                        [ DONE ]
        Result: found 31 running services
  - Check enabled services at boot (systemctl)                [ DONE ]
        Result: found 35 enabled services
  - Check startup files (permissions)                         [ OK ]
  - Running 'systemd-analyze security'
      Unit name (exposure value) and predicate
      --------------------------------
    - ModemManager.service (value=6.3)                        [ MEDIUM ]
    - NetworkManager.service (value=7.8)                      [ EXPOSED ]
    - accounts-daemon.service (value=5.5)                     [ MEDIUM ]
    - alsa-state.service (value=9.6)                          [ UNSAFE ]
    - auditd.service (value=9.4)                              [ UNSAFE ]
    - bluetooth.service (value=6.0)                           [ MEDIUM ]
    - containerd.service (value=9.6)                          [ UNSAFE ]
    - cron.service (value=9.6)                                [ UNSAFE ]
    - dbus.service (value=9.3)                                [ UNSAFE ]
    - displaylink-driver.service (value=9.6)                  [ UNSAFE ]
    - dm-event.service (value=9.5)                            [ UNSAFE ]
    - docker.service (value=9.6)                              [ UNSAFE ]
    - emergency.service (value=9.5)                           [ UNSAFE ]
    - firewalld.service (value=7.3)                           [ MEDIUM ]
    - fwupd.service (value=4.5)                               [ PROTECTED ]
    - getty@tty1.service (value=9.6)                          [ UNSAFE ]
    - getty@tty7.service (value=9.6)                          [ UNSAFE ]
    - haveged.service (value=3.2)                             [ PROTECTED ]
    - iscsid.service (value=9.5)                              [ UNSAFE ]
    - libvirt-guests.service (value=9.6)                      [ UNSAFE ]
    - libvirtd.service (value=9.6)                            [ UNSAFE ]
    - lightdm.service (value=9.6)                             [ UNSAFE ]
    - lvm2-lvmpolld.service (value=9.5)                       [ UNSAFE ]
    - open-vm-tools.service (value=9.5)                       [ UNSAFE ]
    - pcscd.service (value=1.8)                               [ PROTECTED ]
    - plymouth-halt.service (value=9.5)                       [ UNSAFE ]
    - plymouth-kexec.service (value=9.5)                      [ UNSAFE ]
    - plymouth-poweroff.service (value=9.5)                   [ UNSAFE ]
    - plymouth-reboot.service (value=9.5)                     [ UNSAFE ]
    - plymouth-start.service (value=9.5)                      [ UNSAFE ]
    - polkit.service (value=1.2)                              [ PROTECTED ]
    - power-profiles-daemon.service (value=1.0)               [ PROTECTED ]
    - rescue.service (value=9.5)                              [ UNSAFE ]
    - rpc-gssd.service (value=9.5)                            [ UNSAFE ]
    - rpc-statd-notify.service (value=9.5)                    [ UNSAFE ]
    - rpc-svcgssd.service (value=9.5)                         [ UNSAFE ]
    - rsync.service (value=8.5)                               [ EXPOSED ]
    - rtkit-daemon.service (value=7.2)                        [ MEDIUM ]
    - smartmontools.service (value=9.6)                       [ UNSAFE ]
    - ssh.service (value=9.6)                                 [ UNSAFE ]
    - systemd-ask-password-console.service (value=9.4)        [ UNSAFE ]
    - systemd-ask-password-plymouth.service (value=9.5)       [ UNSAFE ]
    - systemd-ask-password-wall.service (value=9.4)           [ UNSAFE ]
    - systemd-bsod.service (value=9.5)                        [ UNSAFE ]
    - systemd-hostnamed.service (value=1.7)                   [ PROTECTED ]
    - systemd-importd.service (value=5.0)                     [ MEDIUM ]
    - systemd-journald.service (value=4.9)                    [ PROTECTED ]
    - systemd-logind.service (value=2.8)                      [ PROTECTED ]
    - systemd-machined.service (value=6.2)                    [ MEDIUM ]
    - systemd-mountfsd.service (value=7.3)                    [ MEDIUM ]
    - systemd-networkd.service (value=2.9)                    [ PROTECTED ]
    - systemd-nsresourced.service (value=4.4)                 [ PROTECTED ]
    - systemd-rfkill.service (value=9.4)                      [ UNSAFE ]
    - systemd-sysupdate.service (value=5.3)                   [ MEDIUM ]
    - systemd-timesyncd.service (value=2.1)                   [ PROTECTED ]
    - systemd-udevd.service (value=7.1)                       [ MEDIUM ]
    - systemd-userdbd.service (value=2.3)                     [ PROTECTED ]
    - tpm-udev.service (value=9.6)                            [ UNSAFE ]
    - udisks2.service (value=9.6)                             [ UNSAFE ]
    - upower.service (value=2.4)                              [ PROTECTED ]
    - user@1000.service (value=9.4)                           [ UNSAFE ]
    - uuidd.service (value=5.8)                               [ MEDIUM ]
    - vgauth.service (value=9.5)                              [ UNSAFE ]
    - virtlockd.service (value=9.6)                           [ UNSAFE ]
    - virtlogd.service (value=2.2)                            [ PROTECTED ]
    - virtualbox-guest-utils.service (value=9.6)              [ UNSAFE ]
    - wpa_supplicant.service (value=9.6)                      [ UNSAFE ]

[+] Kernel
------------------------------------
  - Checking default runlevel                                 [ runlevel 5 ]
  - Checking CPU support (NX/PAE)
    CPU support: PAE and/or NoeXecute supported               [ FOUND ]
  - Checking kernel version and release                       [ DONE ]
  - Checking kernel type                                      [ DONE ]
  - Checking loaded kernel modules                            [ DONE ]
      Found 201 active modules
  - Checking Linux kernel configuration file                  [ FOUND ]
  - Checking default I/O kernel scheduler                     [ NOT FOUND ]
  - Checking for available kernel update                      [ OK ]
  - Checking core dumps configuration
    - configuration in systemd conf files                     [ DEFAULT ]
    - configuration in /etc/profile                           [ DEFAULT ]
    - 'hard' configuration in /etc/security/limits.conf       [ ENABLED ]
    - 'soft' configuration in /etc/security/limits.conf       [ DISABLED ]
    - Checking setuid core dumps configuration                [ PROTECTED ]
  - Check if reboot is needed                                 [ NO ]

[+] Memory and Processes
------------------------------------
  - Checking /proc/meminfo                                    [ FOUND ]
  - Searching for dead/zombie processes                       [ NOT FOUND ]
  - Searching for IO waiting processes                        [ NOT FOUND ]
  - Search prelink tooling                                    [ NOT FOUND ]

[+] Users, Groups and Authentication
------------------------------------
  - Administrator accounts                                    [ OK ]
  - Unique UIDs                                               [ OK ]
  - Consistency of group files (grpck)                        [ WARNING ]
  - Unique group IDs                                          [ OK ]
  - Unique group names                                        [ OK ]
  - Password file consistency                                 [ OK ]
  - Password hashing methods                                  [ OK ]
  - Checking password hashing rounds                          [ DISABLED ]
  - Query system users (non daemons)                          [ DONE ]
  - NIS+ authentication support                               [ NOT ENABLED ]
  - NIS authentication support                                [ NOT ENABLED ]
  - Sudoers file(s)                                           [ FOUND ]
    - Permissions for directory: /etc/sudoers.d               [ WARNING ]
    - Permissions for: /etc/sudoers                           [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_86236  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_89137  [ OK ]
    - Permissions for: /etc/sudoers.d/kali-grant-root         [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_85504  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_87021  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_72772  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_71436  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_75108  [ OK ]
    - Permissions for: /etc/sudoers.d/kdesu-sudoers           [ OK ]
    - Permissions for: /etc/sudoers.d/localhost.tmpt          [ WARNING ]
    - Permissions for: /etc/sudoers.d/ospd-openvas            [ OK ]
    - Permissions for: /etc/sudoers.d/localhost               [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_74453  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_90895  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_96236  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_71647  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_79218  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_80005  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_84100  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_82243  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_72483  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_82577  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_93506  [ OK ]
    - Permissions for: /etc/sudoers.d/cifs-upcall-poc-1000_75638  [ OK ]
  - PAM password strength tools                               [ SUGGESTION ]
  - PAM configuration files (pam.conf)                        [ FOUND ]
  - PAM configuration files (pam.d)                           [ FOUND ]
  - PAM modules                                               [ FOUND ]
  - LDAP module in PAM                                        [ NOT FOUND ]
  - Accounts without expire date                              [ SUGGESTION ]
  - Accounts without password                                 [ OK ]
  - Locked accounts                                           [ FOUND ]
  - Checking user password aging (minimum)                    [ DISABLED ]
  - User password aging (maximum)                             [ DISABLED ]
  - Checking expired passwords                                [ OK ]
  - Checking Linux single user mode authentication            [ OK ]
  - Determining default umask
    - umask (/etc/profile)                                    [ NOT FOUND ]
    - umask (/etc/login.defs)                                 [ SUGGESTION ]
  - LDAP authentication support                               [ NOT ENABLED ]
  - Logging failed login attempts                             [ DISABLED ]

[+] Kerberos
------------------------------------
  - Check for Kerberos KDC and principals                     [ NOT FOUND ]

[+] Shells
------------------------------------
  - Checking shells from /etc/shells
    Result: found 13 shells (valid shells: 13).
    - Session timeout settings/tools                          [ NONE ]
  - Checking default umask values
    - Checking default umask in /etc/bash.bashrc              [ NONE ]
    - Checking default umask in /etc/profile                  [ NONE ]

[+] File systems
------------------------------------
  - Checking mount points
    - Checking /home mount point                              [ SUGGESTION ]
    - Checking /tmp mount point                               [ OK ]
    - Checking /var mount point                               [ SUGGESTION ]
  - Query swap partitions (fstab)                             [ OK ]
  - Testing swap partitions                                   [ OK ]
  - Testing /proc mount (hidepid)                             [ SUGGESTION ]
  - Checking for old files in /tmp                            [ OK ]
  - Checking /tmp sticky bit                                  [ OK ]
  - Checking /var/tmp sticky bit                              [ OK ]
  - ACL support root file system                              [ ENABLED ]
  - Mount options of /                                        [ NON DEFAULT ]
  - Mount options of /dev                                     [ PARTIALLY HARDENED ]
  - Mount options of /dev/shm                                 [ PARTIALLY HARDENED ]
  - Mount options of /run                                     [ HARDENED ]
  - Mount options of /tmp                                     [ PARTIALLY HARDENED ]
  - Total without nodev:6 noexec:34 nosuid:28 ro or noexec (W^X): 10 of total 52
  - Checking Locate database                                  [ FOUND ]
  - Disable kernel support of some filesystems

[+] USB Devices
------------------------------------
  - Checking usb-storage driver (modprobe config)             [ NOT DISABLED ]
  - Checking USB devices authorization                        [ ENABLED ]
  - Checking USBGuard                                         [ NOT FOUND ]

[+] Storage
------------------------------------
  - Checking firewire ohci driver (modprobe config)           [ NOT DISABLED ]

[+] NFS
------------------------------------
  - Query rpc registered programs                             [ DONE ]
  - Query NFS versions                                        [ DONE ]
  - Query NFS protocols                                       [ DONE ]
  - Check running NFS daemon                                  [ NOT FOUND ]

[+] Name services
------------------------------------
  - Checking search domains                                   [ FOUND ]
  - Searching DNS domain name                                 [ UNKNOWN ]
  - Checking /etc/hosts
    - Duplicate entries in hosts file                         [ NONE ]
    - Presence of configured hostname in /etc/hosts           [ FOUND ]
    - Hostname mapped to localhost                            [ NOT FOUND ]
    - Localhost mapping to IP address                         [ OK ]

[+] Ports and packages
------------------------------------
  - Searching package managers
    - Searching dpkg package manager                          [ FOUND ]
      - Querying package manager

  [WARNING]: Test PKGS-7345 had a long execution: 25.377545 seconds

    - Query unpurged packages                                 [ FOUND ]
  - Checking APT package database                             [ OK ]
  - Checking vulnerable packages (apt-get only)               [ DONE ]

  [WARNING]: Test PKGS-7392 had a long execution: 11.716633 seconds

  - Checking upgradeable packages                             [ SKIPPED ]
  - Checking package audit tool                               [ INSTALLED ]
    Found: apt-get
  - Toolkit for automatic upgrades                            [ NOT FOUND ]

[+] Networking
------------------------------------
  - Checking IPv6 configuration                               [ ENABLED ]
      Configuration method                                    [ AUTO ]
      IPv6 only                                               [ NO ]
  - Checking configured nameservers
    - Testing nameservers
        Nameserver: 192.168.1.1                               [ OK ]
        Nameserver: fd4f:fd8d:dfdb:8::1                       [ OK ]
    - Minimal of 2 responsive nameservers                     [ OK ]
  - Checking default gateway                                  [ DONE ]
  - Getting listening ports (TCP/UDP)                         [ DONE ]
  - Checking promiscuous interfaces                           [ OK ]
  - Checking waiting connections                              [ OK ]
  - Checking status DHCP client                               [ NOT ACTIVE ]
  - Checking for ARP monitoring software                      [ NOT FOUND ]
  - Uncommon network protocols                                [ 0 ]

[+] Printers and Spools
------------------------------------
  - Checking cups daemon                                      [ NOT FOUND ]
  - Checking lp daemon                                        [ NOT RUNNING ]

[+] Software: e-mail and messaging
------------------------------------

[+] Software: firewalls
------------------------------------
  - Checking iptables kernel module                           [ NOT FOUND ]
  - Checking host based firewall                              [ ACTIVE ]

[+] Software: webserver
------------------------------------
  - Checking Apache (binary /usr/sbin/apache2)                [ FOUND ]
      Info: Configuration file found (/etc/apache2/apache2.conf)
      Info: No virtual hosts found
    * Loadable modules                                        [ FOUND (118) ]
        - Found 118 loadable modules
          mod_evasive: anti-DoS/brute force                   [ NOT FOUND ]
          mod_reqtimeout/mod_qos                              [ FOUND ]
          ModSecurity: web application firewall               [ NOT FOUND ]
  - Checking TraceEnable setting in:
      /etc/apache2/conf-enabled/javascript-common.conf        [ NOT FOUND ]
      /etc/apache2/conf-enabled/localized-error-pages.conf    [ NOT FOUND ]
      /etc/apache2/conf-enabled/charset.conf                  [ NOT FOUND ]
      /etc/apache2/conf-enabled/other-vhosts-access-log.conf  [ NOT FOUND ]
      /etc/apache2/conf-enabled/security.conf                 [ FOUND ]
      /etc/apache2/conf-enabled/serve-cgi-bin.conf            [ NOT FOUND ]
      /etc/apache2/sites-enabled/000-default.conf             [ NOT FOUND ]
      /etc/apache2/mods-enabled/mpm_prefork.conf              [ NOT FOUND ]
      /etc/apache2/mods-enabled/negotiation.conf              [ NOT FOUND ]
      /etc/apache2/mods-enabled/reqtimeout.conf               [ NOT FOUND ]
      /etc/apache2/mods-enabled/status.conf                   [ NOT FOUND ]
      /etc/apache2/mods-enabled/mime.conf                     [ NOT FOUND ]
      /etc/apache2/mods-enabled/autoindex.conf                [ NOT FOUND ]
      /etc/apache2/mods-enabled/dir.conf                      [ NOT FOUND ]
      /etc/apache2/mods-enabled/php8.4.conf                   [ NOT FOUND ]
      /etc/apache2/mods-enabled/deflate.conf                  [ NOT FOUND ]
      /etc/apache2/mods-enabled/alias.conf                    [ NOT FOUND ]
      /etc/apache2/mods-enabled/setenvif.conf                 [ NOT FOUND ]
      /etc/apache2/sites-available/default-ssl.conf           [ NOT FOUND ]
      /etc/apache2/sites-available/000-default.conf           [ NOT FOUND ]
      /etc/apache2/ports.conf                                 [ NOT FOUND ]
      /etc/apache2/mods-available/mpm_prefork.conf            [ NOT FOUND ]
      /etc/apache2/mods-available/negotiation.conf            [ NOT FOUND ]
      /etc/apache2/mods-available/php8.2.conf                 [ NOT FOUND ]
      /etc/apache2/mods-available/userdir.conf                [ NOT FOUND ]
      /etc/apache2/mods-available/cgid.conf                   [ NOT FOUND ]
      /etc/apache2/mods-available/reqtimeout.conf             [ NOT FOUND ]
      /etc/apache2/mods-available/status.conf                 [ NOT FOUND ]
      /etc/apache2/mods-available/mime.conf                   [ NOT FOUND ]
      /etc/apache2/mods-available/info.conf                   [ NOT FOUND ]
      /etc/apache2/mods-available/ssl.conf                    [ NOT FOUND ]
      /etc/apache2/mods-available/mpm_worker.conf             [ NOT FOUND ]
      /etc/apache2/mods-available/cache_disk.conf             [ NOT FOUND ]
      /etc/apache2/mods-available/proxy_ftp.conf              [ NOT FOUND ]
      /etc/apache2/mods-available/dav_fs.conf                 [ NOT FOUND ]
      /etc/apache2/mods-available/http2.conf                  [ NOT FOUND ]
      /etc/apache2/mods-available/proxy.conf                  [ NOT FOUND ]
      /etc/apache2/mods-available/mime_magic.conf             [ NOT FOUND ]
      /etc/apache2/mods-available/actions.conf                [ NOT FOUND ]
      /etc/apache2/mods-available/ldap.conf                   [ NOT FOUND ]
      /etc/apache2/mods-available/mpm_event.conf              [ NOT FOUND ]
      /etc/apache2/mods-available/autoindex.conf              [ NOT FOUND ]
      /etc/apache2/mods-available/dir.conf                    [ NOT FOUND ]
      /etc/apache2/mods-available/proxy_html.conf             [ NOT FOUND ]
      /etc/apache2/mods-available/php8.4.conf                 [ NOT FOUND ]
      /etc/apache2/mods-available/deflate.conf                [ NOT FOUND ]
      /etc/apache2/mods-available/proxy_balancer.conf         [ NOT FOUND ]
      /etc/apache2/mods-available/alias.conf                  [ NOT FOUND ]
      /etc/apache2/mods-available/setenvif.conf               [ NOT FOUND ]
      /etc/apache2/mods-available/proxy_connect.conf          [ NOT FOUND ]
      /etc/apache2/apache2.conf                               [ NOT FOUND ]
      /etc/apache2/conf-available/javascript-common.conf      [ NOT FOUND ]
      /etc/apache2/conf-available/localized-error-pages.conf  [ NOT FOUND ]
      /etc/apache2/conf-available/charset.conf                [ NOT FOUND ]
      /etc/apache2/conf-available/other-vhosts-access-log.conf  [ NOT FOUND ]
      /etc/apache2/conf-available/security.conf               [ FOUND ]
      /etc/apache2/conf-available/serve-cgi-bin.conf          [ NOT FOUND ]
  - Checking nginx                                            [ NOT FOUND ]

[+] SSH Support
------------------------------------
  - Checking running SSH daemon                               [ NOT FOUND ]

[+] SNMP Support
------------------------------------
  - Checking running SNMP daemon                              [ NOT FOUND ]

[+] Databases
------------------------------------
    No database engines found

[+] LDAP Services
------------------------------------
  - Checking OpenLDAP instance                                [ NOT FOUND ]

[+] PHP
------------------------------------
  - Checking PHP                                              [ FOUND ]
    - Checking PHP disabled functions                         [ FOUND ]
    - Checking expose_php option                              [ OFF ]
    - Checking enable_dl option                               [ OFF ]
    - Checking allow_url_fopen option                         [ ON ]
    - Checking allow_url_include option                       [ OFF ]
    - Checking listen option                                  [ OK ]

[+] Squid Support
------------------------------------
  - Checking running Squid daemon                             [ NOT FOUND ]

[+] Logging and files
------------------------------------
  - Checking for a running log daemon                         [ OK ]
    - Checking Syslog-NG status                               [ NOT FOUND ]
    - Checking systemd journal status                         [ FOUND ]
    - Checking Metalog status                                 [ NOT FOUND ]
    - Checking RSyslog status                                 [ NOT FOUND ]
    - Checking RFC 3195 daemon status                         [ NOT FOUND ]
    - Checking minilogd instances                             [ NOT FOUND ]
    - Checking wazuh-agent daemon status                      [ NOT FOUND ]
  - Checking logrotate presence                               [ OK ]
  - Checking remote logging                                   [ NOT ENABLED ]
  - Checking log directories (static list)                    [ DONE ]
  - Checking open log files                                   [ DONE ]
  - Checking deleted files in use                             [ FILES FOUND ]

[+] Insecure services
------------------------------------
  - Installed inetd package                                   [ NOT FOUND ]
  - Installed xinetd package                                  [ OK ]
    - xinetd status                                           [ NOT ACTIVE ]
  - Installed rsh client package                              [ OK ]
  - Installed rsh server package                              [ OK ]
  - Installed telnet client package                           [ OK ]
  - Installed telnet server package                           [ NOT FOUND ]
  - Checking NIS client installation                          [ OK ]
  - Checking NIS server installation                          [ OK ]
  - Checking TFTP client installation                         [ SUGGESTION ]
  - Checking TFTP server installation                         [ SUGGESTION ]

[+] Banners and identification
------------------------------------
  - /etc/issue                                                [ FOUND ]
    - /etc/issue contents                                     [ WEAK ]
  - /etc/issue.net                                            [ FOUND ]
    - /etc/issue.net contents                                 [ WEAK ]

[+] Scheduled tasks
------------------------------------
  - Checking crontab and cronjob files                        [ DONE ]

[+] Accounting
------------------------------------
  - Checking accounting information                           [ NOT FOUND ]
  - Checking sysstat accounting data                          [ DISABLED ]
  - Checking auditd                                           [ NOT FOUND ]

[+] Time and Synchronization
------------------------------------
  - NTP daemon found: systemd (timesyncd)                     [ FOUND ]
  - Checking for a running NTP daemon or client               [ OK ]
  - Last time synchronization                                 [ 766s ]

[+] Cryptography
------------------------------------
  - Checking for expired SSL certificates [0/150]             [ NONE ]

  [WARNING]: Test CRYP-7902 had a long execution: 112.655630 seconds

  - Found 0 encrypted and 1 unencrypted swap devices in use.  [ OK ]
  - Kernel entropy is sufficient                              [ YES ]
  - HW RNG & rngd                                             [ NO ]
  - SW prng                                                   [ YES ]
  MOR-bit set                                                 [ YES ]

[+] Virtualization
------------------------------------

[+] Containers
------------------------------------
    - Docker
      - Docker daemon                                         [ RUNNING ]
        - Docker info output (warnings)                       [ NONE ]
      - Containers
        - Total containers                                    [ 3 ]
        - Unused containers                                   [ 3 ]
    - File permissions                                        [ OK ]

[+] Security frameworks
------------------------------------
  - Checking presence AppArmor                                [ FOUND ]
    - Checking AppArmor status                                [ ENABLED ]
        Found 150 unconfined processes
  - Checking presence SELinux                                 [ FOUND ]
    - Checking SELinux status                                 [ DISABLED ]
  - Checking presence TOMOYO Linux                            [ NOT FOUND ]
  - Checking presence grsecurity                              [ NOT FOUND ]
  - Checking for implemented MAC framework                    [ OK ]

[+] Software: file integrity
------------------------------------
  - Checking file integrity tools
  - dm-integrity (status)                                     [ DISABLED ]
  - dm-verity (status)                                        [ DISABLED ]
  - Checking presence integrity tool                          [ NOT FOUND ]

[+] Software: System tooling
------------------------------------
  - Checking automation tooling
  - Automation tooling                                        [ NOT FOUND ]
  - Checking for IDS/IPS tooling                              [ NONE ]

[+] Software: Malware
------------------------------------
  - Checking chkrootkit                                       [ FOUND ]
  - Checking Rootkit Hunter                                   [ FOUND ]
  - Checking ClamAV scanner                                   [ FOUND ]
  - Malware software components                               [ FOUND ]
    - Active agent                                            [ NOT FOUND ]
    - Rootkit scanner                                         [ FOUND ]

[+] File Permissions
------------------------------------
  - Starting file permissions check
    File: /boot/grub/grub.cfg                                 [ OK ]
    File: /etc/crontab                                        [ SUGGESTION ]
    File: /etc/group                                          [ OK ]
    File: /etc/group-                                         [ OK ]
    File: /etc/hosts.allow                                    [ OK ]
    File: /etc/hosts.deny                                     [ OK ]
    File: /etc/issue                                          [ OK ]
    File: /etc/issue.net                                      [ OK ]
    File: /etc/motd                                           [ OK ]
    File: /etc/passwd                                         [ OK ]
    File: /etc/passwd-                                        [ OK ]
    File: /etc/ssh/sshd_config                                [ SUGGESTION ]
    Directory: /root/.ssh                                     [ OK ]
    Directory: /etc/cron.d                                    [ SUGGESTION ]
    Directory: /etc/cron.daily                                [ SUGGESTION ]
    Directory: /etc/cron.hourly                               [ SUGGESTION ]
    Directory: /etc/cron.weekly                               [ SUGGESTION ]
    Directory: /etc/cron.monthly                              [ SUGGESTION ]

[+] Home directories
------------------------------------
  - Permissions of home directories                           [ WARNING ]
  - Ownership of home directories                             [ OK ]
  - Checking shell history files                              [ OK ]

[+] Kernel Hardening
------------------------------------
  - Comparing sysctl key pairs with scan profile
    - dev.tty.ldisc_autoload (exp: 0)                         [ DIFFERENT ]
    - fs.protected_fifos (exp: 2)                             [ DIFFERENT ]
    - fs.protected_hardlinks (exp: 1)                         [ OK ]
    - fs.protected_regular (exp: 2)                           [ OK ]
    - fs.protected_symlinks (exp: 1)                          [ OK ]
    - fs.suid_dumpable (exp: 0)                               [ DIFFERENT ]
    - kernel.core_uses_pid (exp: 1)                           [ OK ]
    - kernel.ctrl-alt-del (exp: 0)                            [ OK ]
    - kernel.dmesg_restrict (exp: 1)                          [ DIFFERENT ]
    - kernel.kptr_restrict (exp: 2)                           [ DIFFERENT ]
    - kernel.modules_disabled (exp: 1)                        [ DIFFERENT ]
    - kernel.perf_event_paranoid (exp: 2 3 4)                 [ OK ]
    - kernel.randomize_va_space (exp: 2)                      [ OK ]
    - kernel.sysrq (exp: 0)                                   [ DIFFERENT ]
    - kernel.unprivileged_bpf_disabled (exp: 1)               [ DIFFERENT ]
    - kernel.yama.ptrace_scope (exp: 1 2 3)                   [ DIFFERENT ]
    - net.core.bpf_jit_harden (exp: 2)                        [ DIFFERENT ]
    - net.ipv4.conf.all.accept_redirects (exp: 0)             [ OK ]
    - net.ipv4.conf.all.accept_source_route (exp: 0)          [ OK ]
    - net.ipv4.conf.all.bootp_relay (exp: 0)                  [ OK ]
    - net.ipv4.conf.all.forwarding (exp: 0)                   [ DIFFERENT ]
    - net.ipv4.conf.all.log_martians (exp: 1)                 [ DIFFERENT ]
    - net.ipv4.conf.all.mc_forwarding (exp: 0)                [ OK ]
    - net.ipv4.conf.all.proxy_arp (exp: 0)                    [ OK ]
    - net.ipv4.conf.all.rp_filter (exp: 1)                    [ DIFFERENT ]
    - net.ipv4.conf.all.send_redirects (exp: 0)               [ DIFFERENT ]
    - net.ipv4.conf.default.accept_redirects (exp: 0)         [ DIFFERENT ]
    - net.ipv4.conf.default.accept_source_route (exp: 0)      [ OK ]
    - net.ipv4.conf.default.log_martians (exp: 1)             [ DIFFERENT ]
    - net.ipv4.icmp_echo_ignore_broadcasts (exp: 1)           [ OK ]
    - net.ipv4.icmp_ignore_bogus_error_responses (exp: 1)     [ OK ]
    - net.ipv4.tcp_syncookies (exp: 1)                        [ OK ]
    - net.ipv4.tcp_timestamps (exp: 0 1)                      [ OK ]
    - net.ipv6.conf.all.accept_redirects (exp: 0)             [ DIFFERENT ]
    - net.ipv6.conf.all.accept_source_route (exp: 0)          [ OK ]
    - net.ipv6.conf.default.accept_redirects (exp: 0)         [ DIFFERENT ]
    - net.ipv6.conf.default.accept_source_route (exp: 0)      [ OK ]

[+] Hardening
------------------------------------
    - Installed compiler(s)                                   [ FOUND ]
    - Installed malware scanner                               [ FOUND ]
    - Non-native binary formats                               [ FOUND ]
```
</details> 

## Tool Screenshots:

<img width="811" height="612" alt="Screenshot_20260622_165921" src="https://github.com/user-attachments/assets/339d9c1b-2d74-4ed5-8de8-0bb3d8767bf4" />
<img width="871" height="728" alt="Screenshot_20260622_170109" src="https://github.com/user-attachments/assets/a39908dd-4f6c-42d1-8ea0-bc750a48e28d" />
<img width="843" height="712" alt="Screenshot_20260622_170211" src="https://github.com/user-attachments/assets/7abd9f8a-5a56-43e0-951e-c1b92835076d" />
<img width="1119" height="810" alt="Screenshot_20260622_170353" src="https://github.com/user-attachments/assets/7d14a5ca-65f2-4a1c-ba72-cdad5d066592" />
<img width="1829" height="714" alt="Screenshot_20260622_170836" src="https://github.com/user-attachments/assets/bcf7163b-3c79-4efc-a393-b24e168ae7ac" />

## Build

### Prerequisites
- `g++` (C++17)
- `cmake` (3.14+)
- `libssl-dev`, `zlib1g-dev`
- `Python` (3.13.12) for controller
- `upx` (optional, for dropper compression)
- Linux kernel headers (for kernel module compilation on target)

**Supported Targets**

  * Architecture    :  x86_64 only
  * Linux Kernel    :  4.17.x – 6.x

## Setup (Google Calendar C2)

### Follow the steps below to configure your Google Cloud project and enable Google Calendar API for C2 communication.

## Part 1 — Google Cloud Project & Service Account (for Calendar C2)

### Create a new Google Cloud Project

   * Go to: https://console.cloud.google.com

<img width="1810" height="942" alt="Screenshot_20260419_220749" src="https://github.com/user-attachments/assets/0144f591-6fd0-4006-822f-c1fd5ec5427e" />

<img width="1809" height="1266" alt="Screenshot_20260419_220814" src="https://github.com/user-attachments/assets/865fda92-96cc-4bae-8d34-847a0de95546" />

   * From the top bar, click the project dropdown → New Project
     
<img width="753" height="517" alt="Screenshot_20260418_234008" src="https://github.com/user-attachments/assets/bc60130b-e2b0-4cb7-8a50-6b60ca1dafb5" />

   * Project name: rat-calendar-c2
   * Click Create

### Create a Service Account (used by your C2)

<img width="1427" height="949" alt="Screenshot_20260418_234437" src="https://github.com/user-attachments/assets/70ef143a-ef6a-4084-80fa-28e52c46f33a" />

   * Sidebar: IAM & Admin → Service Accounts
   * Click + Create Service Account

<img width="1425" height="877" alt="Screenshot_20260418_234602" src="https://github.com/user-attachments/assets/b201cb20-97c4-43f9-9095-c48a930b6107" />

   * Name: c2-server
   * Click Create and Continue

<img width="1424" height="875" alt="Screenshot_20260418_234722" src="https://github.com/user-attachments/assets/67faff30-f9a3-4948-a2c3-bc3139ae2aa9" />
<img width="1425" height="819" alt="Screenshot_20260418_234930" src="https://github.com/user-attachments/assets/4888c67b-143d-4da8-885c-4f95906b18fa" />

   * Role: Owner 
   * Click Continue → Done

### Generate Service Account Key

<img width="1427" height="885" alt="Screenshot_20260418_235009" src="https://github.com/user-attachments/assets/17e8489f-2be4-4f1d-9647-78d0cf369461" />

   * Click your new service account

     <img width="1423" height="872" alt="Screenshot_20260418_235036" src="https://github.com/user-attachments/assets/23e9a05a-54ce-4af3-a853-3af4cd69ac4d" />
     <img width="1421" height="877" alt="Screenshot_20260418_235107" src="https://github.com/user-attachments/assets/c2fde17f-4f1b-49eb-963d-e773251c8e8a" />
     
   * Go to Keys tab → Add Key → Create New Key

<img width="1421" height="877" alt="Screenshot_20260418_235127" src="https://github.com/user-attachments/assets/5186fb04-5775-44e9-9d38-1a00fb1074c2" />

   * Select JSON → Create

<img width="1419" height="903" alt="Screenshot_20260418_235150" src="https://github.com/user-attachments/assets/12550d7a-7d3b-4921-bad4-73ad69fd0d61" />

   * Save file as: c2_creds.json in your C2 directory

### Enable Google Calendar API

   * Go to: https://console.cloud.google.com/apis/library
     
<img width="1423" height="592" alt="Screenshot_20260419_000729" src="https://github.com/user-attachments/assets/7fff30da-d678-4743-8ffb-36c17d620e2f" />

   * Search: Google Calendar API

<img width="1423" height="592" alt="Screenshot_20260419_000756" src="https://github.com/user-attachments/assets/a0b6c88a-b3da-49b4-bb49-89467d73aabe" />

   * Select it → Click Enable

## Part 2 — Share Calendar with Service Account

   * Visit: https://calendar.google.com

     <img width="1427" height="1059" alt="Screenshot_20260418_235416" src="https://github.com/user-attachments/assets/9622b862-465d-4c06-8fa8-0e047635475c" />

   * On the left, find your calendar → click 3-dot menu → Settings and sharing

<img width="1372" height="838" alt="Screenshot_20260419_215802" src="https://github.com/user-attachments/assets/78362a84-e1c7-46d1-a0b7-6df3447bac91" />

   * Click + Add people and groups
 

<img width="1382" height="949" alt="Screenshot_20260419_215332" src="https://github.com/user-attachments/assets/b04f6e47-1e44-4aa2-be77-3978f938f618" />
  
   * Paste your Service Account email
   * Looks like: test-c2@your-project-id.iam.gserviceaccount.com
   * Set permission to: Make changes and manage sharing
   * Click Send

## Part 3 — Google Drive API (for data exfiltration)

## Enable Google Drive API

   * Go to: https://console.cloud.google.com/apis/library
     
<img width="1419" height="871" alt="Screenshot_20260418_235931" src="https://github.com/user-attachments/assets/f86058bf-1807-4de8-a78b-53db49224bb0" />
<img width="1421" height="657" alt="Screenshot_20260419_000001" src="https://github.com/user-attachments/assets/9b2fb074-9d70-43ae-acb9-0288e2f7ffad" />

   * Search: Google Drive API → Enable

### Configure OAuth Consent Screen (required for Drive user auth)

<img width="1409" height="987" alt="Screenshot_20260419_001648" src="https://github.com/user-attachments/assets/bf6cce6f-7199-47e1-b8e2-a7f2d4031db5" />

   * Sidebar: APIs & Services → OAuth consent screen

<img width="1409" height="987" alt="Screenshot_20260419_001722" src="https://github.com/user-attachments/assets/ce394542-377e-4d53-9568-79e6daaff1b2" />

   * click: Get started
     
 <img width="1425" height="733" alt="Screenshot_20260419_001856" src="https://github.com/user-attachments/assets/b0c61445-354d-48a3-8db0-1178ea385e1b" />
 
   * App name: c2-server
   * User support email: your Gmail -> Next

<img width="1421" height="875" alt="Screenshot_20260419_002002" src="https://github.com/user-attachments/assets/8cd6b2b8-524c-4274-ab48-6820361ba03b" />

   * Audience: External -> Next

<img width="1423" height="672" alt="Screenshot_20260419_002055" src="https://github.com/user-attachments/assets/4cde39b7-cd52-49c8-8a8c-c5355955acf7" />

   * Contact Info: your Gmail -> Next

<img width="1427" height="1059" alt="Screenshot_20260419_002226" src="https://github.com/user-attachments/assets/f820285d-65cd-48a4-980f-b4c9d1aec1ef" />

   * Audience → Add users → enter your Gmail address → Save

### Create OAuth Client ID (for Drive exfil)

<img width="1421" height="443" alt="Screenshot_20260419_002406" src="https://github.com/user-attachments/assets/e24dd8ab-5b24-46f6-b578-a81ee7491371" />

   * Sidebar: APIs & Services → Credentials
   * Click + Create Credentials → OAuth Client ID
     
<img width="1425" height="475" alt="Screenshot_20260419_002504" src="https://github.com/user-attachments/assets/a983564b-094c-46a8-b6e0-2c46313323c9" />

   * Application type: Desktop app
   * Name: data exfiltration config
   * Click Create

<img width="1425" height="1025" alt="Screenshot_20260419_002525" src="https://github.com/user-attachments/assets/9e17cb34-6c2b-4427-b2b0-ef42920e52d1" />
     
   * Download the JSON file → rename to data_exfil.json
   * Place it in your Dedsec directory

### Tool Structure:
After completing all steps, your directory should contain:
```code    
DEDSEC_XUAN/
├── c2_creds.json        # Service account key for Calendar C2
├── data_exfil.json      # OAuth credentials for Drive exfiltration
└── dedsec_xuan          # dedsec tool (controller)
```

### INSTALLATION
    git clone https://github.com/0xbitx/DEDSEC_XUAN.git
    cd DEDSEC_XUAN
    sudo pip3 install tabulate
    sudo apt install upx-ucl
    chmod +x dedsec_xuan
    sudo ./dedsec_xuan
    
### TESTED ON FOLLOWING
* Kali Linux 
* Parrot OS
* Ubuntu
  
## Legal Disclaimer

This tool is intended for educational and security research purposes only. Unauthorized usage may be illegal in your jurisdiction. The author is not responsible for any misuse of this tool.
