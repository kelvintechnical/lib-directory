# What is the /lib Directory?

**Linux Filesystem Hierarchy Standard (FHS)** | Lab 03  
**Part of:** [Linux-Filesystem-Hierarchy-Standard](https://github.com/kelvintechnical/Linux-Filesystem-Hierarchy-Standard-)  
**Previous Lab:** [What is the /sbin directory?](https://github.com/kelvintechnical/sbin-directory)  
**Next Lab:** [What is the /lib64 directory?](https://github.com/kelvintechnical/what-is-the-lib64-directory)  
**Time Estimate:** 45–60 minutes (5 labs)

---

## 🎯 Objective

Understand what the `/lib` directory is, what lives inside it, why it exists, and how it powers every single command on your system — through five hands-on labs that take you from "I've never opened it" to "I can troubleshoot a missing library at 2 a.m."

---

## 🧠 Big Idea — What is /lib?

`/lib` stands for **library**. It contains the shared **library files** and kernel **modules** that essential binaries in `/bin` and `/sbin` need to run.

> Think of `/lib` as the **engine room** of the operating system. `/bin` and `/sbin` are the steering wheels and dashboards — `/lib` is the actual machinery making them work. Remove `/lib`, and almost nothing on the system starts.

**Key rule:** If a command in `/bin` or `/sbin` needs **shared code** (math routines, network protocols, crypto, terminal handling), that code lives in `/lib` as a `.so` ("shared object") file.

**One-sentence definition:**
> `/lib` holds the **shared libraries** (`.so` files) and **kernel modules** that the binaries in `/bin` and `/sbin` link against at runtime so they don't each have to ship a private copy of `printf`, `malloc`, or `openssl`.

---

## 🏗️ Why /lib Exists — The Story

In the early Unix days, every binary statically embedded all the code it needed. Result: `ls`, `cp`, and `cat` each carried their **own copy** of every helper routine — duplicated megabytes everywhere, and patching a security bug meant rebuilding every binary on the system.

Linux solved that with **shared libraries**: one copy of `libc.so.6` on disk, loaded into RAM once, and **shared** by every running process that needs it. `/lib` is the official home of those shared copies. When you patch `glibc`, you patch one file in `/lib` and every program on the system instantly benefits on its next launch.

---

## 📚 Command Decision Map

| Task | Command |
|---|---|
| List contents of /lib | `ls /lib` |
| Count how many libraries are in /lib | `ls /lib \| wc -l` |
| Confirm /lib is a symlink on modern RHEL | `ls -la / \| grep lib` |
| See what type of file a library is | `file /lib/libc.so.6` |
| List the libraries a binary needs | `ldd /bin/ls` |
| Rebuild the library cache | `sudo ldconfig` |
| Show what `ldconfig` knows about | `ldconfig -p \| less` |
| List kernel modules | `ls /lib/modules/$(uname -r)/` |

---

## 🗺️ The /lib Family — Who Lives Where

```
/                              ← root
├── lib            -> usr/lib  ← symlink on modern RHEL/Fedora
├── lib64          -> usr/lib64
└── usr/
    ├── lib/                   ← 32-bit + arch-independent shared libs
    ├── lib64/                 ← 64-bit shared libs (the real engine room)
    └── lib/modules/<kernel>/  ← kernel modules (drivers) for the running kernel
```

| Path | What's there |
|---|---|
| `/lib` | Shared libraries needed by `/bin` and `/sbin` (often a symlink to `/usr/lib`) |
| `/lib64` | 64-bit shared libraries (often a symlink to `/usr/lib64`) |
| `/usr/lib` | The real location after UsrMerge — most `.so` files live here |
| `/lib/modules/$(uname -r)` | **Kernel modules** — drivers loaded by `modprobe`/`insmod` |
| `/lib/firmware` | Binary firmware blobs the kernel hands to hardware on boot |
| `/lib/systemd` | Systemd unit files and helpers |

---

# 🔧 Lab 1 — Navigate to /lib

**Goal:** Get to `/lib` from anywhere on the system and confirm where you actually are. Sounds trivial — but on modern RHEL/Fedora, `/lib` is a **symlink**, so what you see and where you stand on disk are two different things.

---

### Task 1.1 — `cd /lib` and confirm with `pwd`

**Purpose:** The first move. Land in `/lib` with an absolute path so it works no matter where you started.

```bash
cd /lib
pwd
pwd -P
```

**Expected output (RHEL 9 / Fedora):**

```
/lib
/usr/lib
```

**Switches**

| Token | Meaning |
|---|---|
| `cd` | **C**hange **D**irectory — shell built-in, no separate binary |
| `/lib` | Absolute path — leading `/` means "start from root" |
| `pwd` | **P**rint **W**orking **D**irectory — default = `-L` (logical) |
| `pwd -P` | **P**hysical path — resolves every symlink along the way |

**Output decoded**

| Line | What it tells you |
|---|---|
| `/lib` | The **logical** name your shell walked through — `/lib` looks like a real directory |
| `/usr/lib` | The **physical** location on disk — `/lib` is actually a symlink to `/usr/lib` |

> **The Story:** You typed `cd /lib`. The shell asked the kernel "is `/lib` a directory?" The kernel answered "it's a symbolic link — follow it." The shell quietly followed the arrow to `/usr/lib`, but it **remembered the name you used** (`/lib`) so `pwd` shows the name you walked, while `pwd -P` reveals the actual disk address you're standing on. This is the **UsrMerge** at work — the same change that unified `/bin` and `/usr/bin`.

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| The leading `/` (`cd lib`) | Shell looks for `lib` in your **current** directory — fails unless you happened to be in a parent that has one |
| `pwd` after `cd` | You don't verify the move — and on shared systems you might be one directory off without knowing |
| `pwd -P` | You never learn that `/lib` is a symlink — and later you'll be confused why `ls /lib` and `ls /usr/lib` look identical |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `bash: cd: /lib: No such file or directory` | Extremely broken system — `/lib` is one of the most essential directories. Boot a rescue disk |
| `pwd` and `pwd -P` show the same thing | Older distro (pre-UsrMerge) — `/lib` is a real directory there, not a symlink |
| `pwd -P` says `/usr/lib64` instead of `/usr/lib` | You're probably on a system where `/lib` was made a symlink to `/usr/lib64` — rare but legal |

---

### Task 1.2 — Prove /lib is a symlink with `ls -la / | grep lib`

**Purpose:** Don't just take `pwd -P`'s word for it. Look at the **filesystem entry** for `/lib` and read the symlink directly.

```bash
ls -la / | grep lib
```

**Expected output (RHEL 9):**

```
lrwxrwxrwx.   1 root root     7 May 11 14:02 lib -> usr/lib
lrwxrwxrwx.   1 root root     9 May 11 14:02 lib64 -> usr/lib64
```

**Command explained — the bouncer breakdown**

| Part | Meaning |
|---|---|
| `ls` | "Show me what's in this folder" |
| `-l` | Long listing — full metadata, one entry per line |
| `-a` | **A**ll — include entries starting with `.` (not relevant here, but standard) |
| `/` | The directory we're listing — root |
| `\|` | Pipe — send `ls`'s output to the next command's stdin |
| `grep` | "Search and keep" — read every line, keep ones that match |
| `lib` | The pattern grep is looking for |

**Output decoded — every column of `lrwxrwxrwx.   1 root root     7 May 11 14:02 lib -> usr/lib`**

| Column | Value | Meaning |
|---|---|---|
| 1 (char 1) | `l` | **L**ink — this is a **symbolic link** (not a file `-`, not a directory `d`) |
| 1 (chars 2–10) | `rwxrwxrwx` | Symlinks always show wide-open permissions — the **target's** permissions are what actually matter |
| 1 (char 11) | `.` | Has an SELinux context attached |
| 2 | `1` | Hard link count |
| 3 | `root` | Owner |
| 4 | `root` | Group |
| 5 | `7` | **Size of the symlink's target string** — `usr/lib` is 7 characters |
| 6 | `May 11 14:02` | Last modification time of the link itself |
| 7 | `lib -> usr/lib` | Name of the link → what it points to |

> **The bouncer's job:** `grep lib` reads every line `ls -la /` produces. Most lines (`bin`, `boot`, `dev`, `etc`, `home`...) don't contain "lib" anywhere, so they get thrown away. The lines `lib -> usr/lib` and `lib64 -> usr/lib64` both contain the letters `lib`, so they get waved through to your screen.

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| `-l` from `ls -la` | You see just the names — no `l` prefix means you can't tell symlinks from real dirs |
| `-a` | `lib` and `lib64` still show (they're not dotfiles), but you've lost the habit of seeing dotfiles |
| `\| grep lib` | Wall of output — you have to eyeball 25+ entries to find the two you care about |
| `/` after `ls -la` | `ls` lists your current directory instead of root — you'd miss `/lib` entirely unless you happened to be at `/` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `lib` shows `drwxr-xr-x` instead of `l...` | You're on an older distro (CentOS 7, Debian 9) — `/lib` is a real directory there |
| `grep` returns nothing | You typo'd `lib` or your `/` is broken — try `ls / \| less` and look manually |
| `lib -> /usr/lib` (absolute) vs `lib -> usr/lib` (relative) | Both are legal — relative is preferred so the link still works inside chroots |

---

### Task 1.3 — Walk into /lib relative-style from /

**Purpose:** Reinforce that `/lib` behaves like a directory even though it's a symlink. The kernel resolves it for you.

```bash
cd /
cd lib
pwd
pwd -P
```

**Expected output:**

```
/lib
/usr/lib
```

**Switches**

| Token | Meaning |
|---|---|
| `cd /` | Move to the root directory |
| `cd lib` | Relative — no leading `/`, so shell appends to current location (`/` + `lib` = `/lib`) |

**Output decoded**

| Line | What it tells you |
|---|---|
| `/lib` | Logical path — shell remembers you walked through the name `lib` |
| `/usr/lib` | Physical location after the symlink is resolved |

> **Why this matters on the exam:** RHCSA tasks often write paths like `/lib/modules/...`. You don't need to mentally rewrite them as `/usr/lib/modules/...` — the kernel handles the redirection for you.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `cd lib` fails after `cd /` | `/lib` was deleted (catastrophic) — boot rescue media |
| You wanted to verify the symlink works in **both directions** | `cd /usr/lib && pwd -P` should also show `/usr/lib` — the symlink only points one way |

---

### Task 1.4 — Bounce between /lib and /lib64

**Purpose:** Most modern systems run 64-bit code, so `/lib64` is the more interesting directory in practice. `cd -` lets you bounce.

```bash
cd /lib
cd /lib64
pwd -P
cd -
pwd -P
```

**Expected output:**

```
/usr/lib64
/lib
/usr/lib
```

**Switches**

| Token | Meaning |
|---|---|
| `cd -` | Toggle to the **previous** directory (`$OLDPWD`) |

**Output decoded**

| Line | What it tells you |
|---|---|
| `/usr/lib64` | First `pwd -P` — you're in the 64-bit library directory |
| `/lib` | `cd -` echoes the directory it returned you to |
| `/usr/lib` | Confirmation — you're back in `/lib` (which resolves to `/usr/lib`) |

> **Why a sysadmin needs this:** On CKA and RHCSA, you bounce between `/lib/modules/$(uname -r)` and `/lib64` constantly when investigating "why won't this driver load?" Issues. `cd -` is faster than retyping.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `bash: cd: OLDPWD not set` | First `cd` of your session — run any `cd` first, then `cd -` works |
| Echoed path doesn't match `pwd` | You're using a non-standard shell — `bash` always echoes after `cd -`, `dash` does not |

---

# 🔧 Lab 2 — List and Inspect /lib

**Goal:** Look inside `/lib` and learn to read what you see. You'll meet `.so` files, version numbers, kernel modules, and firmware.

---

### Task 2.1 — Plain `ls /lib` to see the landscape

**Purpose:** Get the lay of the land. What kinds of files and directories live here?

```bash
ls /lib | head -20
```

**Expected output (truncated):**

```
binfmt.d
cpp
crypto-policies
dracut
firewalld
firmware
grub
gssproxy
kbd
kernel
modprobe.d
modules
modules-load.d
NetworkManager
os-release
python3.9
sysctl.d
systemd
sysusers.d
tmpfiles.d
```

**Switches**

| Token | Meaning |
|---|---|
| `ls` | List — no flags = short, multi-column, alphabetical, no dotfiles |
| `/lib` | Target directory (resolves to `/usr/lib`) |
| `\| head -20` | Show only the first 20 lines so the screen doesn't drown |

**Output decoded**

| Entry pattern | Meaning |
|---|---|
| `firmware/` | Binary blobs handed to hardware (Wi-Fi, GPU, NIC) |
| `modules/` | Kernel modules (drivers) keyed by kernel version |
| `systemd/` | systemd unit files, generators, and helpers |
| `python3.9/` | Python standard library — language runtimes live here too |
| `NetworkManager/` | Plugins and helpers for `NetworkManager.service` |
| `crypto-policies/` | System-wide crypto policy (RHEL 8+) |

> **The Story:** `/lib` isn't just "old C libraries." It's the entire support cabinet for every service on your system: kernel drivers, firmware, language runtimes, systemd internals, init scripts, GUI toolkits, package manager plugins. Most of these are **directories**, not `.so` files — the `.so` files mostly live in `/usr/lib64`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Output is just `/usr/lib` | You typed `ls -l /lib` and the symlink resolved — that's normal |
| Empty output | `/lib` itself is empty — extremely broken system |
| Way more entries than 20 | Use `ls /lib \| wc -l` to count, then `ls /lib \| less` to scroll |

---

### Task 2.2 — Count entries with `wc -l`

**Purpose:** Quantify what you just saw. On RHEL 9 there are typically 80–150 entries in `/lib`.

```bash
ls /lib | wc -l
```

**Expected output:**

```
112
```

**Command explained**

| Part | Meaning |
|---|---|
| `ls /lib` | List entries (one per line when piped) |
| `\|` | Pipe — send stdout to next command |
| `wc` | **W**ord **C**ount — counts words, lines, or characters |
| `-l` | Count **l**ines only — and since `ls` prints one entry per line through a pipe, that's our entry count |

**Output decoded**

| Token | Meaning |
|---|---|
| `112` | Number of top-level entries in `/lib` — files + directories combined |

> **The bouncer reframed:** `wc -l` isn't a bouncer that judges lines — it's a turnstile. Every line that comes through gets counted, then the final tally prints. It doesn't care what the lines say.

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| The pipe | `wc -l /lib` would try to read `/lib` as a file — but `/lib` is a directory, so you'd get an error |
| `-l` | `wc` prints lines, words, and bytes — three numbers — instead of just the count you wanted |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `wc -l: /lib: Is a directory` | You forgot the pipe — `ls /lib \| wc -l` is the idiom |

---

### Task 2.3 — Look at one specific library with `file`

**Purpose:** A `.so` file is just a name. `file` opens the first few bytes and tells you what kind of file it really is — ELF binary, script, symlink, data.

```bash
file /lib64/libc.so.6
file -L /lib64/libc.so.6
```

**Expected output:**

```
/lib64/libc.so.6: symbolic link to libc-2.34.so
/lib64/libc.so.6: ELF 64-bit LSB shared object, x86-64, version 1 (GNU/Linux), dynamically linked, ...
```

**Switches**

| Token | Meaning |
|---|---|
| `file` | Identify a file by inspecting its magic bytes (header) |
| `/lib64/libc.so.6` | The C library — used by literally every dynamically-linked binary |
| `-L` | **L**ook through symlinks — report on the **target**, not the link itself |

**Output decoded — first line**

| Token | Meaning |
|---|---|
| `symbolic link to libc-2.34.so` | `libc.so.6` is a versioned symlink pointing at the real library file |

**Output decoded — second line**

| Token | Meaning |
|---|---|
| `ELF` | **E**xecutable and **L**inkable **F**ormat — the standard Linux binary format |
| `64-bit` | Compiled for 64-bit CPUs |
| `LSB` | Little-endian byte order (x86-64 standard) |
| `shared object` | A `.so` library — meant to be loaded by other programs, not run directly |
| `x86-64` | CPU architecture |
| `dynamically linked` | Yes — even `libc` itself links to other things (`ld-linux-x86-64.so.2`) |

> **The Story:** When a program like `ls` starts, the kernel loads it into memory and sees it wants `libc.so.6`. The kernel hands off to the **dynamic linker** (`/lib64/ld-linux-x86-64.so.2`), which finds `libc.so.6` in `/lib64`, follows the symlink to `libc-2.34.so`, maps it into the process's memory, and patches in the function addresses `ls` needs. All of this happens in milliseconds, every time you run any command.

**Why versioned symlinks exist:**

```
libc.so          → libc.so.6        ← compile-time link (developers)
libc.so.6        → libc-2.34.so     ← runtime link (loader)
libc-2.34.so                        ← the actual file on disk
```

This three-step chain lets the system upgrade glibc from 2.34 → 2.35 by only swapping the bottom file and re-pointing the middle symlink. Programs compiled against `libc.so.6` keep working.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `file: cannot open: No such file or directory` | Library name has changed — try `ls /lib64/libc*` |
| Output says `ASCII text` instead of `ELF` | You've found a linker script (rare, but glibc has a few) — `cat` it to see |

---

### Task 2.4 — Browse kernel modules in /lib/modules

**Purpose:** Drivers (modules) are versioned per-kernel and live under `/lib/modules/$(uname -r)/`. This is where `modprobe` looks.

```bash
uname -r
ls /lib/modules/$(uname -r)/ | head -10
ls /lib/modules/$(uname -r)/kernel/drivers/ | head -10
```

**Expected output:**

```
5.14.0-503.40.1.el9_5.x86_64
bls.conf
build
config-5.14.0-503.40.1.el9_5.x86_64
modules.alias
modules.alias.bin
modules.builtin
modules.dep
modules.dep.bin
modules.devname
modules.order

acpi
ata
block
bluetooth
char
cpufreq
dma
firmware
gpu
hid
```

**Switches**

| Token | Meaning |
|---|---|
| `uname -r` | Print the running kernel's release string |
| `$(...)` | **Command substitution** — bash runs `uname -r` first and pastes the result into the outer command |
| `/lib/modules/$(uname -r)/` | The folder of drivers for the kernel **currently running** |

**Output decoded**

| Entry | Meaning |
|---|---|
| `kernel/` | Where the actual `.ko` driver files live |
| `modules.dep` | Dependency map — which module needs which other module |
| `modules.alias` | "If hardware reports PCI ID X, load module Y" |
| `modules.builtin` | Modules baked into the kernel image (not loadable, but listed) |
| `build/` | Symlink to kernel headers — used when building out-of-tree modules |

> **Why a sysadmin needs this:** When `modprobe nvidia` says "module not found," `ls /lib/modules/$(uname -r)/extra/` is your first stop. When you upgrade the kernel, this whole directory tree gets recreated for the new version — old modules don't carry over unless rebuilt.

**The bouncer reframed for `$(uname -r)`:**

```
You wrote:        ls /lib/modules/$(uname -r)/
Bash sees:        ls /lib/modules/$(uname -r)/
Bash runs uname:  uname -r  →  5.14.0-503.40.1.el9_5.x86_64
Bash substitutes: ls /lib/modules/5.14.0-503.40.1.el9_5.x86_64/
Final command:    ls /lib/modules/5.14.0-503.40.1.el9_5.x86_64/
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ls: cannot access '/lib/modules/...'` | You upgraded the kernel but didn't reboot — `uname -r` reports the **running** kernel, not the installed one |
| Empty `extra/` | Normal — only out-of-tree drivers (NVIDIA, VirtualBox, ZFS) land there |

---

# 🔧 Lab 3 — Understand Shared Libraries with `ldd`

**Goal:** Open the hood on any binary and see which `.so` files it needs. This is the single most useful `/lib` skill.

---

### Task 3.1 — Run `ldd /bin/ls`

**Purpose:** See the full dependency list of one of the most-used commands on your system.

```bash
ldd /bin/ls
```

**Expected output:**

```
linux-vdso.so.1 (0x00007ffc8f9d6000)
libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f3a4c200000)
libcap.so.2 => /lib64/libcap.so.2 (0x00007f3a4c100000)
libc.so.6 => /lib64/libc.so.6 (0x00007f3a4c000000)
libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007f3a4be00000)
/lib64/ld-linux-x86-64.so.2 (0x00007f3a4c4d0000)
```

**Switches**

| Token | Meaning |
|---|---|
| `ldd` | **L**ist **D**ynamic **D**ependencies — show what shared libraries a binary needs |
| `/bin/ls` | The binary to inspect |

**Output decoded — line by line**

| Line | Meaning |
|---|---|
| `linux-vdso.so.1` | **Virtual** DSO — not on disk, injected by the kernel for fast syscalls (no `=>` path) |
| `libselinux.so.1 => /lib64/libselinux.so.1` | SELinux library — found in `/lib64`, will be loaded at `0x7f3a...` |
| `libcap.so.2` | Linux capabilities library |
| `libc.so.6` | The C standard library — every C program needs this |
| `libpcre2-8.so.0` | PCRE2 (regex engine) — `ls` uses it for pattern matching |
| `/lib64/ld-linux-x86-64.so.2` | The **dynamic linker itself** — the program that loads all the others |

**The address column `(0x00007f3a4c200000)`:**

> The randomized memory address where the library will be mapped when this binary runs. The randomization is called **ASLR** (Address Space Layout Randomization) — it makes attacks harder by ensuring `libc` lands at a different address every run.

> **The Story:** When you type `ls`, the kernel reads `/bin/ls` and sees a list of "I need these libraries: libselinux, libcap, libc, libpcre2." The kernel hands off to `ld-linux-x86-64.so.2` (the dynamic linker — itself a tiny library). The linker walks `/etc/ld.so.cache` (a binary index of every library on the system), finds each `.so` file in `/lib64`, maps them into memory, and **patches** the missing function addresses in `/bin/ls`. Only then does `ls` start executing your command. All of this happens in under 5 milliseconds, every time.

**What breaks without each piece**

| If you removed... | What goes wrong |
|---|---|
| `libc.so.6` from `/lib64` | **Nothing runs.** Every dynamically-linked binary fails with "error while loading shared libraries" |
| `ld-linux-x86-64.so.2` | Same as above — the kernel can't even start the linker |
| `libselinux.so.1` | `ls` itself fails to start — on SELinux systems, `ls` needs to read file contexts |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ldd: warning: you do not have execution permission for...` | Set the executable bit — `chmod +x` |
| `not a dynamic executable` | The binary is statically linked (rare) — try `file <binary>` to confirm |
| One library shows `=> not found` | Library missing — `sudo dnf provides */libname` to find which package ships it |

---

### Task 3.2 — Compare two commands' dependencies

**Purpose:** Why does `cat` start faster than `ls`? Compare their dependency counts.

```bash
ldd /bin/cat | wc -l
ldd /bin/ls | wc -l
```

**Expected output:**

```
4
6
```

**Output decoded**

| Token | Meaning |
|---|---|
| `4` | `cat` needs 4 things loaded (libc + vdso + linker + 1 more) |
| `6` | `ls` needs 6 things — including SELinux and PCRE2 |

> **Why this matters:** Every `.so` is a disk read + memory map at startup. More dependencies = slower cold start. That's why initramfs uses statically-linked busybox builds: zero `.so` lookups, instant launch.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Both return the same number | Both binaries happen to have the same dependency count — check with `ldd` directly to confirm |

---

### Task 3.3 — Find the package that owns a library

**Purpose:** "Where did this library come from?" RPM tracks every file.

```bash
rpm -qf /lib64/libc.so.6
rpm -qf /lib64/libselinux.so.1
```

**Expected output:**

```
glibc-2.34-100.el9_5.x86_64
libselinux-3.5-1.el9.x86_64
```

**Switches**

| Token | Meaning |
|---|---|
| `rpm` | RPM package manager query tool |
| `-q` | **Q**uery mode |
| `-f` | Query the package that owns **f**ile |

**Output decoded**

| Token | Meaning |
|---|---|
| `glibc-2.34-100.el9_5.x86_64` | The GNU C Library package, version 2.34, build 100, RHEL 9.5, x86_64 arch |

> **Why a sysadmin needs this:** When a CVE drops for "glibc 2.34 buffer overflow," you need to know which file to check. `rpm -qf` reverses the lookup. On Debian/Ubuntu the equivalent is `dpkg -S /lib/x86_64-linux-gnu/libc.so.6`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `file ... is not owned by any package` | Hand-installed library outside the package manager — investigate, this is a security flag |

---

# 🔧 Lab 4 — ldconfig and the Library Cache

**Goal:** Understand `/etc/ld.so.cache`, the index every program uses to find `.so` files at startup, and how `ldconfig` keeps it accurate.

---

### Task 4.1 — Inspect the cache with `ldconfig -p`

**Purpose:** See what the dynamic linker knows about. This is the single source of truth for "what libraries exist on this system."

```bash
ldconfig -p | head -10
ldconfig -p | wc -l
ldconfig -p | grep libssl
```

**Expected output:**

```
1842 libs found in cache `/etc/ld.so.cache'
        libzstd.so.1 (libc6,x86-64) => /lib64/libzstd.so.1
        libz.so.1 (libc6,x86-64) => /lib64/libz.so.1
        libyaml-0.so.2 (libc6,x86-64) => /lib64/libyaml-0.so.2
        ...
1843
        libssl3.so (libc6,x86-64) => /lib64/libssl3.so
        libssl.so.3 (libc6,x86-64) => /lib64/libssl.so.3
```

**Switches**

| Token | Meaning |
|---|---|
| `ldconfig` | The cache builder for the dynamic linker |
| `-p` | **P**rint the current cache — does not modify it |

**Output decoded**

| Token | Meaning |
|---|---|
| `1842 libs found` | Total number of libraries the linker can find without searching disk |
| `libzstd.so.1` | The **soname** — the name programs ask for |
| `(libc6,x86-64)` | ABI (glibc 2.x) and architecture |
| `=> /lib64/libzstd.so.1` | The actual path where this library lives |

> **The Story:** Every dynamically-linked binary asks the dynamic linker "where is `libc.so.6`?" If the linker had to search `/lib`, `/lib64`, `/usr/lib`, `/usr/lib64`, and every `LD_LIBRARY_PATH` entry every single time, every command would be slow. So `ldconfig` builds a **binary index file** at `/etc/ld.so.cache` — like a phonebook of libraries — and the linker reads that file (memory-mapped, instant lookup) instead of searching directories. The cache is rebuilt when you install or remove packages.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `0 libs found in cache` | Cache file empty or corrupt — `sudo ldconfig` rebuilds it |
| Library you just installed isn't there | You installed it manually — see Task 4.3 |

---

### Task 4.2 — Read the search path config

**Purpose:** `ldconfig` reads `/etc/ld.so.conf` and `/etc/ld.so.conf.d/*` to decide which directories to scan. Look at them.

```bash
cat /etc/ld.so.conf
ls /etc/ld.so.conf.d/
cat /etc/ld.so.conf.d/*.conf
```

**Expected output:**

```
include ld.so.conf.d/*.conf

dyninst-x86_64.conf
kernel-5.14.0-503.40.1.el9_5.x86_64.conf

/usr/lib64/dyninst
```

**Switches**

| Token | Meaning |
|---|---|
| `cat` | Print files to stdout |
| `/etc/ld.so.conf` | Master config — usually a one-line `include` |
| `/etc/ld.so.conf.d/*.conf` | Drop-in directory — each `.conf` adds a search path |

**Output decoded**

| Entry | Meaning |
|---|---|
| `include ld.so.conf.d/*.conf` | "Pull in every `.conf` file from that subdirectory" |
| `dyninst-x86_64.conf` | Drop-in from the Dyninst package adding its lib path |
| `/usr/lib64/dyninst` | The actual extra directory to scan |

> **Note:** `/lib64` and `/usr/lib64` are **always** scanned even without config — they're built into the linker. The `.conf.d` files exist for **non-standard** locations.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Empty `/etc/ld.so.conf.d/` | Normal on minimal systems — only matters when third-party packages install libs outside `/lib64` |

---

### Task 4.3 — Add a custom library directory

**Purpose:** When you install software outside the package manager (e.g., compiled-from-source apps in `/opt/myapp/lib`), tell `ldconfig` about it.

```bash
sudo mkdir -p /opt/myapp/lib
echo "/opt/myapp/lib" | sudo tee /etc/ld.so.conf.d/myapp.conf
sudo ldconfig
ldconfig -p | grep myapp
```

**Expected output:**

```
/opt/myapp/lib
(no output from grep — directory is empty, but it's now scanned)
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Make directory, plus all parents if needed |
| `echo "..." \| sudo tee` | Bash trick — `tee` writes stdin to a file with sudo privileges (plain `>` wouldn't work after `sudo`) |
| `ldconfig` (no flags) | **Rebuild** `/etc/ld.so.cache` from scratch |

**Output decoded**

| Step | What happened |
|---|---|
| `mkdir -p` | Created `/opt/myapp/lib` |
| `tee` | Wrote `/opt/myapp/lib` into `/etc/ld.so.conf.d/myapp.conf` |
| `sudo ldconfig` | Rebuilt the cache, scanning all configured paths plus the new one |
| `grep myapp` | No matches — the directory is empty, so no `.so` files were cached |

> **The bouncer rewrite:**
> 1. The package you installed dropped a `.so` into `/opt/myapp/lib`
> 2. You tell `ldconfig` "scan here too" by adding a `.conf` file
> 3. You run `ldconfig` — it scans **all** configured paths and rebuilds the cache
> 4. Now any program looking for that library finds it instantly

**What breaks without each piece**

| If you skipped... | What goes wrong |
|---|---|
| Creating the `.conf` file | `ldconfig` never scans `/opt/myapp/lib`, programs say "library not found" |
| Running `sudo ldconfig` | The `.conf` exists but the cache hasn't been updated — same failure |
| `sudo` on either step | `tee` and `ldconfig` need write access to `/etc/` and `/etc/ld.so.cache` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ldconfig -p \| grep myapp` shows nothing even after a `.so` is dropped in | Re-run `sudo ldconfig`; check the `.so` has a proper soname (`objdump -p file.so \| grep SONAME`) |
| `Permission denied` writing the conf file | You forgot `sudo tee` — plain `>` runs as your user even after `sudo` |

---

# 🔧 Lab 5 — Real-World Troubleshooting

**Goal:** Diagnose and fix the most common `/lib`-related failure: **"error while loading shared libraries."** This is the senior-engineer payoff.

---

### Task 5.1 — Simulate a missing-library error

**Purpose:** Practice in a controlled way so you know what the failure looks like before it bites you in production.

```bash
ldd /bin/ls
LD_LIBRARY_PATH=/nonexistent /bin/ls 2>&1 | head -5
```

**Expected output (second command):**

```
adjtime  alternatives  ...
```

> **Wait — it still worked?** Yes. `LD_LIBRARY_PATH` **prepends** to the search path; it doesn't replace it. The linker tried `/nonexistent`, didn't find `libc.so.6` there, then fell back to `/lib64` and succeeded. To force a real failure, we need a different trick.

**Now force a real failure:**

```bash
/lib64/ld-linux-x86-64.so.2 --library-path /nonexistent /bin/ls 2>&1 | head -3
```

**Expected output:**

```
/bin/ls: error while loading shared libraries: libselinux.so.1: cannot open shared object file: No such file or directory
```

**Output decoded**

| Token | Meaning |
|---|---|
| `error while loading shared libraries` | The dynamic linker failed before `ls` ran a single line of code |
| `libselinux.so.1` | The first missing library it found |
| `cannot open shared object file` | The library file isn't where the linker expected |
| `No such file or directory` | Standard ENOENT errno from the kernel |

> **The Story:** You hijacked `/bin/ls` by running the dynamic linker manually with `--library-path /nonexistent`, which **replaces** (not prepends) the search path. The linker dutifully looked only in `/nonexistent`, found nothing, and reported the first missing library. This is exactly what you'd see in a real outage where someone deleted `/lib64` or mounted the wrong filesystem.

**Troubleshoot — what's actually breaking in production**

| Symptom | Real cause | Fix |
|---|---|---|
| `error while loading shared libraries: libfoo.so.5` | Library was removed or renamed | `rpm -qf` to find owning package, reinstall it |
| Same error after fresh install of the package | Cache stale | `sudo ldconfig` |
| Works as root, fails as user | Permissions on the `.so` file | `ls -l /path/to/libfoo.so.5` — should be world-readable (`r--r--r--`) |
| Works locally, fails after `sudo -i` | `LD_LIBRARY_PATH` set in user env but stripped by `sudo` | Move the library to a directory in `/etc/ld.so.conf.d/` |

---

### Task 5.2 — Find which package would restore a missing library

**Purpose:** Reverse a missing-library error into an action item: **what do I install?**

```bash
sudo dnf provides '*/libselinux.so.1'
```

**Expected output (truncated):**

```
libselinux-3.5-1.el9.x86_64 : SELinux library and simple utilities
Repo        : @System
Matched from:
Filename    : /usr/lib64/libselinux.so.1
```

**Switches**

| Token | Meaning |
|---|---|
| `dnf provides` | Search every available package for one that ships a file matching the pattern |
| `'*/libselinux.so.1'` | Glob pattern — match any path ending in `libselinux.so.1` |

**Output decoded**

| Field | Meaning |
|---|---|
| `libselinux-3.5-...` | Package name and version |
| `@System` | Source — already installed (so this would be `sudo dnf reinstall libselinux`) |
| `Filename` | The exact path the package owns |

> **Why a sysadmin needs this:** When you walk into a server that throws "libfoo.so.X not found," `dnf provides` reverses the unknown into a package name in seconds. On Debian/Ubuntu: `apt-file search libselinux.so.1`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `No matches found` | Library isn't in any enabled repo — check enabled repos (`dnf repolist`) or it's from an out-of-tree source |
| Multiple packages match | Pick the one whose name matches what your binary expects — usually the obvious one |

---

### Task 5.3 — Audit which libraries a service actually loads at runtime

**Purpose:** `ldd` shows compile-time dependencies. To see what a **running** process is using, look at `/proc`.

```bash
pgrep -x sshd | head -1 | xargs -I {} cat /proc/{}/maps | grep '\.so' | awk '{print $NF}' | sort -u | head -10
```

**Expected output:**

```
/usr/lib64/ld-linux-x86-64.so.2
/usr/lib64/libaudit.so.1.0.0
/usr/lib64/libc.so.6
/usr/lib64/libcap-ng.so.0.0.0
/usr/lib64/libcrypto.so.3.0.7
/usr/lib64/libgssapi_krb5.so.2.2
/usr/lib64/libkeyutils.so.1.10
/usr/lib64/libpam.so.0.85.1
/usr/lib64/libselinux.so.1
/usr/lib64/libz.so.1.2.11
```

**Command explained — the full pipeline**

| Stage | What it does |
|---|---|
| `pgrep -x sshd` | Find PIDs of processes **exactly** named `sshd` |
| `\| head -1` | Take the first PID only |
| `\| xargs -I {} cat /proc/{}/maps` | Build the command `cat /proc/<pid>/maps` and run it |
| `\| grep '\.so'` | Keep only lines mentioning a shared library file |
| `\| awk '{print $NF}'` | Print only the **last field** (the path) of each line |
| `\| sort -u` | Sort and remove duplicates (`maps` repeats each lib once per memory segment) |
| `\| head -10` | First 10 results to keep the screen sane |

> **The bouncer breakdown:** Each command in the pipeline is a bouncer with a specific job. `pgrep` finds the right person. `head` keeps only the first match. `xargs` builds the next command using that match. `grep` keeps only library lines. `awk` keeps only the path field. `sort -u` removes duplicates. `head` trims the final list. This is the unix philosophy in action — small, sharp tools chained together.

**Output decoded**

| Token | Meaning |
|---|---|
| `/proc/<pid>/maps` | A virtual file showing every memory region of a running process |
| `libcrypto.so.3.0.7` | The actual versioned file (not the soname `libcrypto.so.3`) currently mapped into sshd |
| `libgssapi_krb5.so.2.2` | Kerberos GSSAPI — sshd uses this for kerberized auth |
| `libpam.so.0.85.1` | PAM — pluggable authentication, every login goes through here |

> **Why this beats `ldd`:** `ldd /usr/sbin/sshd` shows what sshd **could** load. `/proc/<pid>/maps` shows what it **actually** loaded — including libraries dlopen()'d later (like PAM modules, krb5 plugins). On a hardened or chrooted service this catches things `ldd` misses.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `pgrep` returns nothing | Service isn't running — `sudo systemctl start sshd` first |
| `Permission denied` on `/proc/<pid>/maps` | Run as root — process memory maps are sensitive |

---

### Task 5.4 — Fix a "library not found" error end-to-end

**Purpose:** Tie all four prior labs together in one realistic scenario.

**Scenario:** You log into a server. You run `httpd -v`. You see:

```
httpd: error while loading shared libraries: libapr-1.so.0: cannot open shared object file: No such file or directory
```

**Solution walkthrough:**

```bash
ldd $(which httpd) | grep 'not found'
sudo dnf provides '*/libapr-1.so.0'
sudo dnf install apr
sudo ldconfig
ldd $(which httpd) | grep apr
httpd -v
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `ldd $(which httpd) \| grep 'not found'` | Confirm which library is missing — could be more than one |
| `sudo dnf provides '*/libapr-1.so.0'` | Find which package ships the missing file |
| `sudo dnf install apr` | Install the package — drops the library into `/lib64` |
| `sudo ldconfig` | Refresh the cache so the linker sees the new file |
| `ldd $(which httpd) \| grep apr` | Verify the library now resolves |
| `httpd -v` | Confirm the original failing command now runs |

**Expected final output:**

```
Server version: Apache/2.4.62 (Red Hat Enterprise Linux)
Server built:   Sep 27 2024 00:00:00
```

> **The Story:** A real-world `/lib` problem rarely starts at `/lib`. It starts with a failing command. You read the error, identify the missing library, locate the package that owns it, install the package, refresh the cache, and verify. **Five commands, one fix.** This is the rhythm every senior Linux engineer has burned into muscle memory.

---

## ✅ Lab Checklist — 5 Labs, 17 Tasks

**Lab 1 — Navigate**
- [ ] 1.1 `cd /lib && pwd && pwd -P` shows the symlink in action
- [ ] 1.2 `ls -la / \| grep lib` shows `lib -> usr/lib` and `lib64 -> usr/lib64`
- [ ] 1.3 `cd /` then `cd lib` works relatively
- [ ] 1.4 `cd -` bounces between `/lib` and `/lib64`

**Lab 2 — List and inspect**
- [ ] 2.1 `ls /lib \| head -20` shows directories like `modules`, `firmware`, `systemd`
- [ ] 2.2 `ls /lib \| wc -l` returns a count
- [ ] 2.3 `file /lib64/libc.so.6` reveals the ELF binary
- [ ] 2.4 `ls /lib/modules/$(uname -r)/` lists kernel modules

**Lab 3 — Shared libraries**
- [ ] 3.1 `ldd /bin/ls` lists every shared library
- [ ] 3.2 `ldd /bin/cat \| wc -l` compared to `ldd /bin/ls \| wc -l`
- [ ] 3.3 `rpm -qf /lib64/libc.so.6` returns `glibc-...`

**Lab 4 — ldconfig**
- [ ] 4.1 `ldconfig -p \| head -10` shows the cache
- [ ] 4.2 `cat /etc/ld.so.conf` and `ls /etc/ld.so.conf.d/`
- [ ] 4.3 Added a custom dir to `/etc/ld.so.conf.d/` and ran `sudo ldconfig`

**Lab 5 — Troubleshooting**
- [ ] 5.1 Reproduced "library not found" with the manual linker invocation
- [ ] 5.2 Used `dnf provides` to find the owning package
- [ ] 5.3 Inspected `/proc/<pid>/maps` of a running service
- [ ] 5.4 Walked the full "library missing → fix" cycle end-to-end

---

## 🧠 /lib vs /lib64 vs /usr/lib — What's the Difference?

| Directory | Purpose | On modern RHEL 9 |
|---|---|---|
| `/lib` | Historical home of shared libraries needed by `/bin` and `/sbin` | Symlink → `/usr/lib` |
| `/lib64` | 64-bit shared libraries (architecture-specific) | Symlink → `/usr/lib64` |
| `/usr/lib` | Arch-independent libs, kernel modules, systemd files | The **real** location |
| `/usr/lib64` | The real 64-bit `.so` files (libc, openssl, etc.) | The **real** location |
| `/usr/local/lib` | Libraries installed manually outside the package manager | Separate — outside dnf/rpm |

> **One sentence:** On modern RHEL/Fedora, `/lib` and `/lib64` exist for backward compatibility; the **real** libraries live in `/usr/lib` and `/usr/lib64`.

---

## ⚠️ Common Pitfalls

| Mistake | Fix |
|---|---|
| Confusing `/lib` with `/lib64` | `/lib` historically held 32-bit libs; `/lib64` is for 64-bit. On modern symlinked systems both target `/usr/lib*` |
| Deleting files in `/lib` to "save space" | **Never.** You will brick the system. Use `dnf` to remove packages |
| Editing `/etc/ld.so.cache` by hand | Don't — it's a **binary** file. Edit `/etc/ld.so.conf.d/*.conf` and run `sudo ldconfig` |
| Forgetting `sudo ldconfig` after dropping a `.so` into a custom dir | Cache stays stale; programs report "library not found" even though the file is there |
| Setting `LD_LIBRARY_PATH` permanently in `~/.bashrc` | Security risk + breaks `sudo` (which strips it). Use `/etc/ld.so.conf.d/` instead |
| Expecting `ldd` to show **all** runtime libraries | `ldd` only shows compile-time deps. `dlopen()`'d libs (PAM, plugins) appear only at runtime — check `/proc/<pid>/maps` |
| `cd /lib/modules` without `$(uname -r)` | Lands you in a directory of every installed kernel — confusing when troubleshooting the running one |

---

## 📌 Exam Tips (RHCSA / RHCE / CKA)

- **RHCSA EX200:** Know that `/lib` and `/lib64` are symlinks on RHEL 9 (UsrMerge). Be ready to confirm with `ls -la /`.
- **RHCSA EX200:** When asked to "make a library available," the answer is almost always `/etc/ld.so.conf.d/*.conf` + `sudo ldconfig`.
- **RHCE EX294 (Ansible):** Ansible modules ship libraries under `/usr/lib/python3.X/site-packages/ansible/` — that's still under `/lib` semantically.
- **CKA:** Container runtime libraries (`runc`, `containerd-shim`) live in `/usr/lib64` and `/usr/libexec`. CNI plugins in `/opt/cni/bin` use `.so` files from `/usr/lib64`.
- **RHCA RH342:** First debug move when a service won't start: `journalctl -u svc -n 50` → if you see `error while loading shared libraries` → `ldd` → `dnf provides` → reinstall.
- Memorize the **five-command fix:** `ldd` → `dnf provides` → `dnf install` → `ldconfig` → re-test.

---

## 🔍 /lib Decision Guide

```
What does this binary need?    → ldd /path/to/binary
What package owns this .so?    → rpm -qf /lib64/libfoo.so.X
What package would provide it? → dnf provides '*/libfoo.so.X'
Where does the linker look?    → cat /etc/ld.so.conf.d/*.conf
What's cached right now?       → ldconfig -p | grep libfoo
Refresh the cache?             → sudo ldconfig
What's a running process using?→ cat /proc/<pid>/maps | grep '\.so'
What kernel modules exist?     → ls /lib/modules/$(uname -r)/kernel/
What's the firmware blob?      → ls /lib/firmware/ | grep <hardware>
```

---

## 🔗 Series Index

| # | Directory | Repo |
|---|---|---|
| 01 | `/bin` | [bin-directory](https://github.com/kelvintechnical/bin-directory) |
| 02 | `/sbin` | [sbin-directory](https://github.com/kelvintechnical/sbin-directory) |
| 👉 03 | `/lib` | **You are here** |
| 04 | `/lib64` | [what-is-the-lib64-directory](https://github.com/kelvintechnical/what-is-the-lib64-directory) |
| 05 | `/usr` | [what-is-the-usr-directory](https://github.com/kelvintechnical/what-is-the-usr-directory) |
| 06 | `/etc` | [what-is-the-etc-directory](https://github.com/kelvintechnical/what-is-the-etc-directory) |
| 07 | `/boot` | [what-is-the-boot-directory](https://github.com/kelvintechnical/what-is-the-boot-directory) |
| 08 | `/home` | [what-is-the-home-directory](https://github.com/kelvintechnical/what-is-the-home-directory) |
| 09 | `/root` | [what-is-the-root-directory](https://github.com/kelvintechnical/what-is-the-root-directory) |
| 10 | `/var` | [what-is-the-var-directory](https://github.com/kelvintechnical/what-is-the-var-directory) |
| 11 | `/tmp` | [what-is-the-tmp-directory](https://github.com/kelvintechnical/what-is-the-tmp-directory) |
| 12 | `/opt` | [what-is-the-opt-directory](https://github.com/kelvintechnical/what-is-the-opt-directory) |
| 13 | `/srv` | [what-is-the-srv-directory](https://github.com/kelvintechnical/what-is-the-srv-directory) |
| 14 | `/dev` | [what-is-the-dev-directory](https://github.com/kelvintechnical/what-is-the-dev-directory) |
| 15 | `/proc` | [what-is-the-proc-directory](https://github.com/kelvintechnical/what-is-the-proc-directory) |
| 16 | `/sys` | [what-is-the-sys-directory](https://github.com/kelvintechnical/what-is-the-sys-directory) |
| 17 | `/run` | [what-is-the-run-directory](https://github.com/kelvintechnical/what-is-the-run-directory) |
| 18 | `/media` | [what-is-the-media-directory](https://github.com/kelvintechnical/what-is-the-media-directory) |
| 19 | `/mnt` | [what-is-the-mnt-directory](https://github.com/kelvintechnical/what-is-the-mnt-directory) |
| 20 | `/afs` | [what-is-the-afs-directory](https://github.com/kelvintechnical/what-is-the-afs-directory) |

---

## 🔗 Part of Linux Ops Mastery

- [Linux-Filesystem-Hierarchy-Standard](https://github.com/kelvintechnical/Linux-Filesystem-Hierarchy-Standard-)
- [Linux Ops Mastery](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
