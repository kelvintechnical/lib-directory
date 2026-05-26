# Lab: What Is the `/lib` Directory?

- **Series:** linux-ops-mastery — Linux Filesystem Hierarchy Standard
- **Subjects covered:** Filesystem Hierarchy Standard (FHS), `/lib` as the home of essential shared libraries (`libc.so.6`, `ld-linux.so`), how dynamic linking turns ELF binaries into runnable processes, UsrMerge (`/lib → /usr/lib`), the dynamic linker cache (`ldconfig`, `/etc/ld.so.cache`), debug-style library tracing with `ldd` and `nm`, the most common "library not found" failure mode
- **Career arcs covered:** RHCSA (every binary the exam grader runs depends on libraries here), RHCE (Ansible modules link to the same `glibc`), SRE (the "version `GLIBC_2.34` not found" production incident pattern), DevOps (multi-stage Docker builds frequently misplace `.so` files), AI/MLOps (CUDA + PyTorch wheels load `libstdc++`, `libgcc_s`, `libdl` from `/lib`, and the wrong one breaks training)
- **Prerequisite:** Basic `ls`, `cd`, `cat`, and a shell on RHEL 9 / Rocky 9 / Ubuntu
- **Time Estimate:** 20 to 35 minutes
- **Difficulty arc:** Task 1 inspect · 2–3 inventory + read · 4–5 demonstrate purpose · 6 capstone audit

---

## Objective

Move past "libraries are magic stuff binaries need" and treat `/lib` as **a concrete, inspectable directory with a known FHS purpose**: it stores the shared object (`.so`) files that essential binaries in `/bin` and `/sbin` load at runtime. By the end of this lab you can list `.so` files, count them, find which library a binary uses, query the dynamic linker cache, and read symbol tables — the exact loop you would use to debug a `libfoo.so.5: cannot open shared object file` error.

The lab is **inspection-only**. No `ldconfig -p` modifications, no library replacements, no `LD_PRELOAD` games. You will read, count, trace, and audit; nothing changes on disk.

The capstone is the RHCSA-realistic prompt: *"A binary you just built fails at runtime with `error while loading shared libraries: libfoo.so.5: cannot open shared object file`. Use only `ldd`, `ldconfig`, and `find` to locate the library, identify why the linker did not see it, and propose the supported repair (without making the change)."*

---

## Concept: Why `/lib` Exists

```
   ┌────────────────────────────────────────────────────────────────┐
   │  FHS root /                                                    │
   ├────────────────────────────────────────────────────────────────┤
   │  /lib   → essential shared libraries used by /bin and /sbin    │
   │  /lib64 → 64-bit shared libraries on x86_64                    │
   │  /usr/lib   → non-essential shared libraries                   │
   │  /usr/lib64 → non-essential 64-bit shared libraries            │
   │                                                                │
   │  Runtime resolution flow (ld-linux):                           │
   │     binary → ELF DT_NEEDED tags  ─►  /etc/ld.so.cache          │
   │              │                          │                      │
   │              ▼                          ▼                      │
   │       fallback to DT_RPATH      built by ldconfig from         │
   │       and LD_LIBRARY_PATH        /etc/ld.so.conf + .d/         │
   │                                                                │
   │  Modern RHEL (UsrMerge):                                       │
   │     /lib   ── symlink ──► /usr/lib                             │
   │     /lib64 ── symlink ──► /usr/lib64                           │
   └────────────────────────────────────────────────────────────────┘
```

`/lib` exists because **shared libraries** are the entire point of dynamic linking. Linking `libc.so.6` once and letting hundreds of binaries reference it saves disk, RAM (via shared pages), and patch effort (one CVE update fixes every consumer). FHS designates `/lib` as the home of the libraries those essential `/bin` and `/sbin` binaries need — including the **dynamic linker itself**, `ld-linux.so`.

> **Why this matters:** When a binary fails to start with "cannot open shared object," the answer is almost always either (a) the `.so` is not on disk, (b) it is on disk but not in the linker cache, or (c) it is on disk but for a different architecture or `GLIBC_` version. All three diagnoses are made with the same three tools you will use in this lab — `ldd`, `ldconfig`, `find`.

---

## 📜 Why `/lib` Exists — The Story

Early UNIX programs were **statically linked** — every binary carried a private copy of every library function. That was simple, but on a system with hundreds of users and dozens of programs, a fix to `printf()` meant relinking every program. By the mid-1980s, SunOS introduced **shared libraries** and the **dynamic linker** that loads them at runtime; AT&T System V Release 4 standardized the ELF format that made them portable.

Linux inherited shared libraries from day one. The **Filesystem Hierarchy Standard**, first published in **1994** by the linux-fhs group (now maintained by the Linux Foundation), formalized two rules: (1) the shared libraries needed by `/bin` and `/sbin` must live in `/lib`, and (2) less-essential libraries live in `/usr/lib`. The implicit constraint was the same as for binaries — at early boot, `/usr` might still be on a separate, not-yet-mounted disk, so anything required to bring `/usr` online had to be reachable from `/`.

The architecture multiplied things. When 64-bit x86 (`x86_64`) arrived in 2003, distros needed a way to ship both 32-bit and 64-bit libraries on the same system without name collisions. FHS 3.0 added `/lib64` for the 64-bit variant, leaving `/lib` for 32-bit on `x86_64` systems. On native 64-bit-only installs, `/lib64` holds the runnable libraries and `/lib` may be sparse or absent.

The **UsrMove** transition (Fedora 17, 2012; RHEL 7, 2014) folded `/lib` and `/lib64` into `/usr/lib` and `/usr/lib64` as symlinks, just like `/bin` and `/sbin`. From a user's perspective `/lib` still exists and is referenced by the dynamic linker, but the bytes live under `/usr`.

> **The point of the story:** `/lib` is not an organizational courtesy — it is the **entry point of every dynamically-linked program on Linux**. The kernel exec's the binary, the binary's ELF header points at `/lib64/ld-linux-x86-64.so.2`, and that interpreter then resolves every other `.so` listed in `DT_NEEDED`. The directory name is hard-coded into thousands of binaries.

---

## 👪 The `/lib` Family — Who Lives There

### Essential shared libraries (under `/lib64` on x86_64)

| Library | Purpose | Who depends on it |
|---|---|---|
| `libc.so.6` | GNU C library (`glibc`) — `printf`, `malloc`, `open`, `read`, ... | Every dynamic binary |
| `ld-linux-x86-64.so.2` | Dynamic linker / loader (the "interpreter") | Every dynamic binary |
| `libm.so.6` | Math library (`sin`, `cos`, `sqrt`) | Any binary using math.h |
| `libpthread.so.0` | POSIX threads (now part of glibc) | Multi-threaded programs |
| `libdl.so.2` | `dlopen` / `dlsym` for runtime plugin loading | Plugin-based daemons |
| `librt.so.1` | Realtime extensions (timers, async I/O) | Many daemons |
| `libselinux.so.1` | SELinux userland API | RHEL system binaries |
| `libcap.so.2` | POSIX capabilities | Containers, systemd |
| `libcrypt.so.2` | Password hashing (`crypt(3)`) | `login`, `passwd`, PAM |
| `libtinfo.so.6` | terminfo curses backend | `bash`, `vim`, `less` |

### Related directories you will visit

| Directory | Purpose | Relation to `/lib` |
|---|---|---|
| `/lib64` | 64-bit shared libraries on x86_64 | Sibling (or actual location on UsrMerge) |
| `/usr/lib` | Non-essential shared libraries | Symlink target of `/lib` on UsrMerge |
| `/usr/lib64` | 64-bit non-essential libs | Symlink target of `/lib64` |
| `/usr/local/lib` | Locally-installed libraries | Default `make install --prefix=/usr/local` location |
| `/etc/ld.so.conf` | Linker config (extra search dirs) | Tells `ldconfig` where else to look |
| `/etc/ld.so.conf.d/` | Drop-in linker config files | Per-package extra dirs (`mariadb-x86_64.conf`, ...) |
| `/etc/ld.so.cache` | Compiled linker cache | Built by `ldconfig` from the above |

### Tools that interact with `/lib`

| Tool | What it tells you about `/lib` |
|---|---|
| `ldd /bin/ls` | Per-binary library resolution at runtime |
| `ldconfig -p` | Full linker cache (every library the linker can find by SONAME) |
| `ldconfig -v` | Rebuild cache (root only) |
| `file /lib64/libc.so.6` | ELF type and architecture |
| `nm -D /lib64/libc.so.6` | Dynamic symbol table (functions exposed) |
| `objdump -p /bin/ls` | Read `DT_NEEDED`, `DT_RPATH`, `DT_RUNPATH` |
| `readelf -d /bin/ls` | Same, more readable |
| `rpm -qf /lib64/libc.so.6` | Owning RPM (`glibc-...`) |

> **The point of the family tree:** Library debugging is a three-tool job — `ldd` to see what a binary expects, `ldconfig -p` to see what the linker actually knows about, `find` to bridge the gap when those disagree.

---

## 🔬 The Anatomy of `ldd /bin/ls` — In One Diagram

```
$ ldd /bin/ls
	linux-vdso.so.1 (0x00007ffc...)
	libselinux.so.1 => /lib64/libselinux.so.1 (0x00007fbc...)
	libcap.so.2     => /lib64/libcap.so.2     (0x00007fbc...)
	libc.so.6       => /lib64/libc.so.6       (0x00007fbc...)
	libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007fbc...)
	/lib64/ld-linux-x86-64.so.2  (0x00007fbc...)
	         │                       │              │
	         │                       │              └─ Address where the linker mapped the library
	         │                       └─ Absolute path on disk where the linker actually loaded it
	         └─ SONAME the binary asked for (from ELF DT_NEEDED entries)

How `ldd` works internally:
   1. Read binary's PT_INTERP header → /lib64/ld-linux-x86-64.so.2 is the dynamic linker.
   2. Set LD_TRACE_LOADED_OBJECTS=1 in the environment.
   3. Exec the binary. The linker recognizes the env var and prints the resolution map
      instead of jumping to the binary's _start. The binary itself never runs.
   4. The output shows DT_NEEDED entries with their resolved paths.

Special entries:
   linux-vdso.so.1                — virtual shared object provided by the kernel (no on-disk file)
   /lib64/ld-linux-x86-64.so.2    — the dynamic linker itself (its own line, no "=>")
   `not found`                    — what you fear: SONAME present in DT_NEEDED, not resolvable
```

> **Reading rule:** A clean `ldd` output never says `not found`. Any "not found" line is the cause of a "cannot open shared object" failure at exec time. The fix is either install the package, run `ldconfig`, or set `LD_LIBRARY_PATH` — in that order of preference.

---

## 📚 `/lib` Reference Table

| Task | Command | Notes |
|---|---|---|
| Confirm `/lib` is a symlink | `ls -ld /lib` | First char `l` on UsrMerge |
| Print symlink target | `readlink /lib` | Returns `usr/lib` |
| Count `.so` files | `find /usr/lib /usr/lib64 -name '*.so*' \| wc -l` | Several thousand on RHEL 9 |
| Library deps of a binary | `ldd /bin/ls` | The everyday tool |
| Inspect linker cache | `ldconfig -p \| head` | Lists every SONAME the linker knows |
| Filter cache by name | `ldconfig -p \| grep libc.so` | Find a specific library |
| Find a library by name | `find /usr/lib* -name 'libfoo.so*'` | On-disk search |
| Show DT_NEEDED entries | `objdump -p /bin/ls \| grep NEEDED` | What the binary actually asked for |
| Show ELF arch | `file /lib64/libc.so.6` | ELF 64-bit LSB shared object |
| Symbol query | `nm -D /lib64/libc.so.6 \| grep printf` | Dynamic symbols exported |

> **Rule one of `/lib`:** Libraries here are **read-only system property**. Never `rm`, `mv`, or `chmod` a `.so` file on a running system — you will brick the box mid-command.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Failing to run a binary because of a missing library is a common exam recovery scenario; you must know `ldd` and `ldconfig` cold. |
| **RHCE candidate** | Ansible modules call out to libraries via Python's `ctypes` — knowing where `.so` files live makes broken-module debugging tractable. |
| **SRE / Platform** | "version `GLIBC_2.34` not found" is the canonical "wrong base image" production incident. The diagnosis loop lives here. |
| **DevOps** | Multi-stage Docker builds where you `COPY --from=builder /app /app` and forget the dynamic libraries — same diagnosis loop. |
| **AI / MLOps** | PyTorch and TensorFlow wheels carry `.so` files that resolve against host `libstdc++` and `libgcc_s`. Mismatch = `undefined symbol`. |

---

## 🔧 The 6 Tasks

> Six inspection-only phases that build the **identify → enumerate → trace → cache → audit** habit for shared libraries.

---

### Task 1 — Inspect `/lib` itself

**Purpose:** Confirm whether `/lib` is a directory or a symlink on your RHEL 9 system and capture the metadata you will refer to for the rest of the lab.

```bash
ls -ld /lib /lib64
stat /lib
readlink /lib
file /lib /lib64
readlink -f /lib /lib64
```

**Human-Readable Breakdown:** `ls -ld` of both `/lib` and `/lib64` reveals their type in one line each. `stat` shows the full inode. `readlink` gives the raw target. `file` classifies the path. `readlink -f` resolves the canonical absolute path.

**Reading it left to right:** On RHEL 9 the typical output is `/lib -> usr/lib` and `/lib64 -> usr/lib64`. The relative target string keeps the symlink valid even inside chroots. `readlink -f` follows the link and prints `/usr/lib`.

**The story:** Just like `/bin`, `/lib` is a compatibility name on modern RHEL. The bytes live in `/usr/lib`. The directory only exists because thousands of binaries' ELF headers hard-code `/lib64/ld-linux-x86-64.so.2`.

**Expected output:**

```text
lrwxrwxrwx. 1 root root 7 Apr 12  2024 /lib -> usr/lib
lrwxrwxrwx. 1 root root 9 Apr 12  2024 /lib64 -> usr/lib64
  File: /lib -> usr/lib
  Size: 7         	Blocks: 0          IO Block: 4096   symbolic link
Device: fd00h/64768d	Inode: 1572867     Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
usr/lib
/lib:   symbolic link to usr/lib
/lib64: symbolic link to usr/lib64
/usr/lib
/usr/lib64
```

**Switches**

| Token | Meaning |
|---|---|
| `ls -ld` | Long format, do not descend |
| `stat` | Inode metadata |
| `readlink` | Raw symlink target |
| `readlink -f` | Canonicalized full path |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `/lib` is a directory | Non-UsrMerge distro — fine, lab continues |
| `/lib` missing entirely | Corrupted symlink — boot rescue media |
| `/lib64` empty | Pure 32-bit distro or container — substitute `/lib` everywhere below |

---

### Task 2 — Count and sample `.so` files

**Purpose:** Get a sense of **how many** shared libraries the system ships and **what their naming convention looks like** — major SONAME, symlink chain, real ELF file.

```bash
find /usr/lib64 -maxdepth 1 -name '*.so*' | wc -l
ls /usr/lib64/libc.so* 2>/dev/null
ls -l /usr/lib64/libc.so* 2>/dev/null
file /usr/lib64/libc.so.6
ls /usr/lib64 | head -n 20
```

**Human-Readable Breakdown:** Total `.so*` count under `/usr/lib64`, the `libc.so*` family showing the symlink chain (the canonical example), long-format detail revealing real file vs. symlink, `file` on the canonical SONAME, and a sample of the alphabetical head of `/usr/lib64`.

**Reading it left to right:** Shared libraries on Linux follow a three-name convention: a real file `libfoo.so.X.Y.Z`, a soname symlink `libfoo.so.X` (this is what binaries record), and a linker name `libfoo.so` (this is what `gcc -lfoo` uses at link time). Counting `*.so*` covers all three.

**The story:** RHEL 9 ships several thousand `.so` files under `/usr/lib64`. Each library typically has 2-3 names. Knowing the **three-name convention** is what lets you understand why `ls libc.so*` shows you `libc.so.6` (symlink) and `libc-2.34.so` (real file) — they refer to the same library at different naming stability tiers.

**Expected output:**

```text
4127
/usr/lib64/libc.so.6
lrwxrwxrwx. 1 root root      12 Mar 22  2024 /usr/lib64/libc.so.6 -> libc-2.34.so
/usr/lib64/libc.so.6: symbolic link to libc-2.34.so
ld-linux-x86-64.so.2
ld-lsb-x86-64.so.3
libBrokenLocale-2.34.so
libBrokenLocale.so.1
libCheckIcmp.so
libICE.so.6
libICE.so.6.3.0
libSM.so.6
libSM.so.6.0.1
libX11-xcb.so.1
libX11.so.6
libX11.so.6.4.0
libXau.so.6
libXau.so.6.0.0
libXcomposite.so.1
libXcomposite.so.1.0.0
libXcursor.so.1
libXcursor.so.1.0.2
libXdamage.so.1
```

**Switches**

| Token | Meaning |
|---|---|
| `find -maxdepth 1` | Do not recurse below `/usr/lib64` |
| `-name '*.so*'` | All shared-library naming variants |
| `ls -l libc.so*` | Reveal symlink chain |
| `file PATH` | Classify by magic bytes |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `find` returns 0 | Wrong path on 32-bit distro — use `/usr/lib` |
| `libc.so.6` not a symlink | Static-glibc system (unusual) — fine, real file |
| `file` says "data" | Corrupt `.so` — `dnf reinstall glibc` |

---

### Task 3 — Find which library a binary needs (`ldd` deep dive)

**Purpose:** Use `ldd` to read the full library resolution map of a real binary, then cross-check against the binary's actual ELF `DT_NEEDED` entries to prove that what `ldd` shows is the truth.

```bash
ldd /bin/ls
echo "---"
objdump -p /bin/ls 2>/dev/null | grep -E 'NEEDED|RUNPATH|RPATH'
echo "---"
readelf -d /bin/ls 2>/dev/null | grep -E 'NEEDED|RUNPATH|RPATH'
echo "---"
ldd /bin/ls | awk '/=>/ {print $3}' | xargs -I{} ls -l {} 2>/dev/null | head
```

**Human-Readable Breakdown:** Run `ldd` to see resolved paths, run `objdump -p` and `readelf -d` to read the raw ELF `DT_NEEDED` entries (the source of truth — what the binary actually asks for), then use the resolved paths from `ldd` to `ls -l` each library file.

**Reading it left to right:** `objdump -p` and `readelf -d` both print the dynamic section of the ELF file. `DT_NEEDED` is the linker tag that lists each SONAME the binary requires. `DT_RUNPATH`/`DT_RPATH` are extra search hints — usually empty for distro binaries, often populated for third-party software like Oracle or NVIDIA. `awk '/=>/ {print $3}'` extracts the resolved path column from `ldd` output, and `xargs ls -l` shows their on-disk metadata.

**The story:** When `ldd` and `readelf -d` agree, the binary is healthy. When they disagree — `readelf -d` lists `libfoo.so.5` but `ldd` says `not found` — the library is missing or out of the linker's reach. That mismatch is the diagnosis half of every "shared object" failure.

**Expected output:**

```text
	linux-vdso.so.1 (0x00007ffc...)
	libselinux.so.1 => /lib64/libselinux.so.1 (0x00007fbc...)
	libcap.so.2 => /lib64/libcap.so.2 (0x00007fbc...)
	libc.so.6 => /lib64/libc.so.6 (0x00007fbc...)
	libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007fbc...)
	/lib64/ld-linux-x86-64.so.2 (0x00007fbc...)
---
  NEEDED               libselinux.so.1
  NEEDED               libcap.so.2
  NEEDED               libc.so.6
---
 0x0000000000000001 (NEEDED)             Shared library: [libselinux.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libcap.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
---
lrwxrwxrwx. 1 root root  17 Mar 22  2024 /lib64/libselinux.so.1 -> libselinux.so.1
-rwxr-xr-x. 1 root root 174216 Mar 22  2024 /lib64/libcap.so.2
lrwxrwxrwx. 1 root root  12 Mar 22  2024 /lib64/libc.so.6 -> libc-2.34.so
lrwxrwxrwx. 1 root root  20 Mar 22  2024 /lib64/libpcre2-8.so.0 -> libpcre2-8.so.0.11.0
```

**Switches**

| Token | Meaning |
|---|---|
| `ldd PATH` | Print resolved library map (executes binary's loader) |
| `objdump -p` | Dump dynamic section (does not execute) |
| `readelf -d` | Dump dynamic entries (does not execute) |
| `xargs -I{}` | Substitute each input line as `{}` |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ldd` shows `not found` | Library missing or out of linker reach — Task 4 |
| `objdump` errors with "no symbols" | Stripped binary — use `readelf -d` instead |
| `RUNPATH` or `RPATH` non-empty | Third-party software sets a private search dir — note for production deploys |

---

### Task 4 — Query the dynamic linker cache (`ldconfig`)

**Purpose:** Learn that **the linker does not search `/usr/lib*` at runtime** — it consults the precompiled `/etc/ld.so.cache`. Querying that cache tells you exactly what the linker thinks is available, which is what matters for `ldd not found` debugging.

```bash
ldconfig -p | head
ldconfig -p | wc -l
ldconfig -p | grep -E 'libc\.so\.6'
ldconfig -p | grep -E 'libselinux\.so\.1'
ls -1 /etc/ld.so.conf.d/ | head
cat /etc/ld.so.conf
```

**Human-Readable Breakdown:** Show the header + first entries of the linker cache, count total entries, locate `libc.so.6` and `libselinux.so.1` to confirm they are cached, then peek at the configuration that determines what `ldconfig` indexes.

**Reading it left to right:** `ldconfig -p` reads `/etc/ld.so.cache` (binary file). Each entry maps SONAME → architecture flags → absolute path. The cache is rebuilt by `sudo ldconfig` whenever package install scripts add `.so` files. `/etc/ld.so.conf.d/*.conf` adds extra search dirs (one per package).

**The story:** `ldd` says "not found" usually because the `.so` exists on disk but `ldconfig` was never told about its location. The fix is `echo /opt/custom/lib > /etc/ld.so.conf.d/custom.conf && ldconfig`. RHCSA examiners love this troubleshoot.

**Expected output:**

```text
4127 libs found in cache `/etc/ld.so.cache'
	libzstd.so.1 (libc6,x86-64) => /lib64/libzstd.so.1
	libz.so.1 (libc6,x86-64) => /lib64/libz.so.1
	libyajl.so.2 (libc6,x86-64) => /lib64/libyajl.so.2
	libxshmfence.so.1 (libc6,x86-64) => /lib64/libxshmfence.so.1
	libxslt.so.1 (libc6,x86-64) => /lib64/libxslt.so.1
	libxml2.so.2 (libc6,x86-64) => /lib64/libxml2.so.2
	libxml.so.2 (libc6,x86-64) => /lib64/libxml.so.2
	libxkbfile.so.1 (libc6,x86-64) => /lib64/libxkbfile.so.1
	libxkbcommon.so.0 (libc6,x86-64) => /lib64/libxkbcommon.so.0
4128
	libc.so.6 (libc6,x86-64, OS ABI: Linux 3.2.0) => /lib64/libc.so.6
	libselinux.so.1 (libc6,x86-64) => /lib64/libselinux.so.1
bind-export.conf
dyninst-x86_64.conf
kernel-5.14.0-427.13.1.el9_4.x86_64.conf
libiscsi-x86_64.conf
mariadb-x86_64.conf
include ld.so.conf.d/*.conf
```

**Switches**

| Token | Meaning |
|---|---|
| `ldconfig -p` | Print cache (read-only) |
| `ldconfig` | Rebuild cache (root only — do NOT run in this lab) |
| `/etc/ld.so.conf.d/` | Per-package extra search dirs |
| `include` | Pulls in all `*.conf` files in conf.d |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ldconfig: command not found` as non-root | Use `/sbin/ldconfig -p` with absolute path |
| Cache count is 0 | Cache file missing — `sudo ldconfig` would rebuild it |
| Library missing from cache but present on disk | Add directory to `/etc/ld.so.conf.d/*.conf`, rerun `ldconfig` |

---

### Task 5 — Symbol lookup inside a library with `nm`

**Purpose:** Look inside a shared library and list the symbols (function names) it exports. This is how you confirm whether a particular function (e.g. `printf`, `dlopen`, `__libc_start_main`) is provided by the version of glibc on the system.

```bash
nm -D /lib64/libc.so.6 | head
nm -D /lib64/libc.so.6 | wc -l
nm -D /lib64/libc.so.6 | grep -w ' T printf'
nm -D /lib64/libc.so.6 | grep -w 'T memcpy' | head
nm -D --with-symbol-versions /lib64/libc.so.6 | grep 'GLIBC_2.34' | head
```

**Human-Readable Breakdown:** `nm -D` lists **dynamic** symbols (the ones exposed to consumers). The first column is the symbol type — `T` means a defined text (function) symbol, `U` means undefined (consumed from elsewhere), `D` is initialized data. Filtering for `printf`, `memcpy`, and `GLIBC_2.34` versioned symbols shows you exactly what this library provides.

**Reading it left to right:** `nm -D` reads the dynamic symbol table. `--with-symbol-versions` appends the `@@GLIBC_X.YZ` version tag, which is how glibc maintains binary compatibility — every binary asks for a specific symbol version, and the library provides multiple versions.

**The story:** The legendary "version `GLIBC_2.34` not found" error means the binary was built against a newer glibc than the system provides. The fix is either rebuild the binary, ship a newer glibc, or run the binary in a container with the right base image. `nm -D --with-symbol-versions` is how you confirm which versions are present.

**Expected output:**

```text
                 w __cxa_finalize@GLIBC_2.2.5
                 w __cxa_thread_atexit_impl@GLIBC_2.18
                 U __gmon_start__
                 w __pthread_key_create@GLIBC_2.2.5
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
0000000000028a30 T abort
00000000000a14b0 T abs
00000000000ad440 T accept
00000000000ad470 T accept4
2497
00000000000639c0 T printf
0000000000091fa0 T memcpy
0000000000091fa0 T __memcpy_chk@GLIBC_2.3.4
0000000000091fa0 T memcpy_chk
0000000000028a30 T abort@@GLIBC_2.2.5
00000000000639c0 T printf@@GLIBC_2.2.5
00000000000a14b0 T abs@@GLIBC_2.2.5
0000000000028a30 T abort@GLIBC_2.34
```

**Switches**

| Token | Meaning |
|---|---|
| `nm -D` | Dynamic symbol table (vs static `nm` for archives) |
| `T` | Defined text (function) symbol |
| `U` | Undefined (consumed) |
| `--with-symbol-versions` | Append GLIBC version tags |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `nm: no symbols` | Library was fully stripped — use `objdump -T` instead |
| Symbol missing | Binary requires a newer glibc — match container base image to target |
| Version conflict | Versioned-symbol resolution — diagnose with `LD_DEBUG=symbols cmd` |

---

### Task 6 — Capstone: diagnose a missing-library error

**Task statement:** *"A binary fails at runtime with `error while loading shared libraries: libfoo.so.5: cannot open shared object file`. Without changing the system, locate any installed `libfoo.so.5`, determine why the dynamic linker did not see it, and propose the supported repair."*

**Purpose:** Combine `ldd`, `ldconfig`, `find`, and `readelf` into the canonical diagnosis loop. The lab uses an **already-installed** library so nothing must be modified.

```bash
TARGET=/bin/ls
LIB=libc.so.6
echo "=== Library diagnosis for $LIB needed by $TARGET ==="
echo
echo "--- 1. What does $TARGET ask for? (DT_NEEDED) ---"
readelf -d "$TARGET" 2>/dev/null | grep NEEDED
echo
echo "--- 2. Does ldd resolve $LIB? ---"
ldd "$TARGET" 2>/dev/null | grep -E "\b${LIB}\b"
echo
echo "--- 3. Is $LIB in the linker cache? ---"
ldconfig -p | grep -E "\s${LIB}\s"
echo
echo "--- 4. Where on disk does $LIB live? ---"
find /usr/lib /usr/lib64 -name "${LIB}" 2>/dev/null
echo
echo "--- 5. What package owns it? ---"
rpm -qf "$(ldconfig -p | grep -E "\s${LIB}\s" | awk '{print $NF}' | head -n1)" 2>/dev/null
echo
echo "--- 6. Proposed repair (NOT executed) ---"
echo "If steps 1+2+3 disagree, possible repairs are:"
echo "  a) Install the missing package:    sudo dnf install <pkg>"
echo "  b) Refresh the linker cache:       sudo ldconfig"
echo "  c) Add custom dir to ld.so.conf.d: echo /opt/custom/lib >/etc/ld.so.conf.d/custom.conf"
echo "                                     then sudo ldconfig"
echo "  d) Last resort (script-local):     LD_LIBRARY_PATH=/opt/custom/lib $TARGET"
```

**Human-Readable Breakdown:** Six-section diagnosis. Section 1 reads the binary's stated requirement. Section 2 asks whether `ldd` resolves it. Section 3 asks whether the linker cache lists it. Section 4 finds it on disk regardless of cache. Section 5 identifies the owning package. Section 6 enumerates the four supported repairs in order of preference.

**Reading it left to right:** The diagnosis loop converges on the answer in 30 seconds. If sections 1-3 all show the library and section 4 confirms presence, the library is fine; failure is elsewhere. If section 4 finds the library but section 3 (cache) does not, the fix is `ldconfig`. If section 4 returns nothing, install the package.

**Layer stack you traversed:**

```text
Binary (DT_NEEDED)        ← readelf -d
   │
   ▼
Dynamic linker            ← ld-linux-x86-64.so.2
   │
   ▼
Linker cache              ← /etc/ld.so.cache (ldconfig -p)
   │
   ▼
On-disk library           ← find /usr/lib*
   │
   ▼
Owning RPM                ← rpm -qf
```

**The story:** This is the **interview answer** when someone asks "walk me through a `libfoo.so.5 not found` debug session." The order is fixed: binary → linker → cache → disk → package. Skipping a step is where junior engineers waste 30 minutes.

**Expected output:**

```text
=== Library diagnosis for libc.so.6 needed by /bin/ls ===

--- 1. What does /bin/ls ask for? (DT_NEEDED) ---
 0x0000000000000001 (NEEDED)             Shared library: [libselinux.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libcap.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]

--- 2. Does ldd resolve libc.so.6? ---
	libc.so.6 => /lib64/libc.so.6 (0x00007fbc...)

--- 3. Is libc.so.6 in the linker cache? ---
	libc.so.6 (libc6,x86-64, OS ABI: Linux 3.2.0) => /lib64/libc.so.6

--- 4. Where on disk does libc.so.6 live? ---
/usr/lib64/libc.so.6

--- 5. What package owns it? ---
glibc-2.34-100.el9_4.2.x86_64

--- 6. Proposed repair (NOT executed) ---
If steps 1+2+3 disagree, possible repairs are:
  a) Install the missing package:    sudo dnf install <pkg>
  b) Refresh the linker cache:       sudo ldconfig
  c) Add custom dir to ld.so.conf.d: echo /opt/custom/lib >/etc/ld.so.conf.d/custom.conf
                                     then sudo ldconfig
  d) Last resort (script-local):     LD_LIBRARY_PATH=/opt/custom/lib /bin/ls
```

**Cleanup**

```bash
# nothing destructive — this lab was inspection-only
# no /tmp files, no ldconfig refresh, no library changes
echo "diagnosis complete — /lib untouched"
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Section 4 empty | Library truly missing — `dnf provides */libfoo.so.5` to find package |
| Section 2 says `not found` but section 4 finds it | `ldconfig` cache is stale or path not configured |
| Section 5 empty | Library installed outside RPM — third-party tarball; trace install logs |
| `LD_LIBRARY_PATH` workaround "fixes" but `sudo` fails | `sudo` strips it (env_reset); add path via `/etc/ld.so.conf.d` instead |

---

## 🔍 `/lib` Decision Guide

```
Binary fails with "cannot open shared object: libfoo.so.5"?
  │
  ├── "Does ldd resolve it?"
  │       ├── yes → not a library problem; check perms or SELinux
  │       └── no → continue
  │
  ├── "Is it in ldconfig -p?"
  │       ├── yes → cache mismatch — rare; try ldd with absolute path
  │       └── no → continue
  │
  ├── "Is the .so file on disk?"
  │       ├── yes → directory not in /etc/ld.so.conf.d/*.conf
  │       │        Fix: add path, ldconfig
  │       └── no  → continue
  │
  ├── "Does any package provide it?"
  │       └── dnf provides '*/libfoo.so.5'
  │              ├── yes → dnf install <pkg>
  │              └── no  → third-party tarball; reinstall properly
  │
  └── "Last resort"
          └── LD_LIBRARY_PATH=/opt/custom/lib <cmd>   (do not use for sudo)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Inspect `/lib` and `/lib64` with `ls -ld`, `stat`, `readlink`, `file`
- [ ] 02 Count `.so` files and decode the three-name `libfoo.so.X` convention
- [ ] 03 Run `ldd` plus `readelf -d` to confirm a binary's library requirements
- [ ] 04 Query the linker cache (`ldconfig -p`) and inspect `ld.so.conf.d`
- [ ] 05 Use `nm -D` to list dynamic symbols and GLIBC version tags
- [ ] 06 Run the six-section missing-library diagnosis capstone

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Manually copying `.so` into `/lib` | Inconsistent state, RPM verify fails | Install the owning package instead |
| Running `ldd` on untrusted binaries | Code execution risk | Use `readelf -d` or `objdump -p` |
| Setting `LD_LIBRARY_PATH` globally | Breaks SUID binaries (silently ignored) | Use `/etc/ld.so.conf.d/*.conf` + `ldconfig` |
| Confusing `/lib` (32-bit) with `/lib64` | `wrong ELF class` errors | Match arch to binary with `file` |
| Forgetting `ldconfig` after adding lib path | `ldd not found` persists | `sudo ldconfig` after editing conf |
| Editing `/etc/ld.so.cache` directly | Binary format — corruption | Cache is generated; edit `/etc/ld.so.conf.d/` |
| Trusting `ldd` for static binaries | "not a dynamic executable" | Static binaries do not have library deps |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Know that `/lib`, `/lib64` are symlinks into `/usr` on UsrMerge systems. Be ready to read `ldd` output line by line.

**RHCE candidate**
- Ansible roles that install third-party software should drop an `ld.so.conf.d` file and notify `ldconfig` — write the handler as a one-liner.

**SRE / Platform interview**
- "Production binary fails with `GLIBC_2.34 not found`" → walk the six-section diagnosis. Identify the root cause (binary built against newer glibc), propose container or chroot.

**DevOps**
- Multi-stage Docker builds: `COPY --from=builder /app/bin /app/bin` is incomplete if the binary needs `libstdc++.so.6` — copy `/lib64/libstdc++.so.6` too or stay on the same base image.

**AI / MLOps**
- PyTorch wheels resolve against host `libstdc++`. If the host is `manylinux2014` and the wheel was built for `manylinux_2_28`, `nm -D` against both surfaces the version mismatch.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| [/bin](https://github.com/kelvintechnical/bin-directory) | Binaries that load these libraries |
| [/sbin](https://github.com/kelvintechnical/sbin-directory) | Admin binaries — same dependency chain |
| [/lib64](https://github.com/kelvintechnical/lib64-directory) | 64-bit sibling — actual home of the runnable libs |
| [/usr](https://github.com/kelvintechnical/usr-directory) | Canonical home after UsrMerge |
| [/etc](https://github.com/kelvintechnical/etc-directory) | `/etc/ld.so.conf` and `ld.so.conf.d/` live here |
| [/boot](https://github.com/kelvintechnical/boot-directory) | Kernel images that initialize the dynamic loader |
| [/home](https://github.com/kelvintechnical/home-directory) | User shells that use the libraries |
| [/root](https://github.com/kelvintechnical/root-directory) | Root home — first place to land in rescue |
| [/var](https://github.com/kelvintechnical/var-directory) | Logs that record `ld.so` errors via journald |
| [/tmp](https://github.com/kelvintechnical/tmp-directory) | Scratch space for build artifacts |
| [/opt](https://github.com/kelvintechnical/opt-directory) | Third-party software often ships its own `.so` files |
| [/srv](https://github.com/kelvintechnical/srv-directory) | Service data — daemons load libs from `/lib` |
| [/dev](https://github.com/kelvintechnical/dev-directory) | Device nodes the libraries call into |
| [/proc](https://github.com/kelvintechnical/proc-directory) | `/proc/<pid>/maps` shows loaded library addresses |
| [/sys](https://github.com/kelvintechnical/sys-directory) | Kernel object exposure |
| [/run](https://github.com/kelvintechnical/run-directory) | Runtime state for dynamic-linker daemons |
| [/media](https://github.com/kelvintechnical/media-directory) | Removable media — rare host for libs |
| [/mnt](https://github.com/kelvintechnical/mnt-directory) | Manual mount points — chroots load own libs |
| [/afs](https://github.com/kelvintechnical/afs-directory) | AFS mount; libraries may be served over network |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
