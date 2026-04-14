# nspawn.sh

A ~500-line POSIX shell script that implements minimal Linux containers using namespaces, bind mounts, and `pivot_root`.

No daemons. No D-Bus. No dependencies beyond a POSIX shell, a few coreutils (or busybox), and a kernel ≥ 4.19.

## Quick start

### Install:
```sh
curl -fL https://raw.githubusercontent.com/ponces/nspawn.sh/main/install.sh | sh
```

### Get a rootfs:

```sh
# Download and extract a Debian rootfs to current directory
sudo getroot debian

# For older version
sudo getroot debian:12

# For Ubuntu, Alpine, Arch, Fedora, ...
sudo getroot alpine
sudo getroot ubuntu:24
```

### Run it:

```sh
# Basic container (host network)
sudo nspawn debian_trixie

# With network namespace
sudo nspawn --net debian_trixie

# Run a specific command
sudo nspawn debian_trixie /bin/bash -l

# Port forwarding from wifi to container:
nspawn --net --port-range 80:8080 <rootfs>

# Android: Custom route when using multi-wan
nspawn --net --route-via wlan0 <rootfs>
```

## Why

Containers are just namespaces + bind mounts + `pivot_root`. That's it. Everything else is optional. This script exists to prove that point.

The goal is to do the hard part once so you never have to again. We set up the namespace
plumbing so you get a clean Linux environment,then get out of the way and let you work natively inside it.

## Android is a first-class concern

Smartphones are the computers people actually have. Over 6 billion of them exist.
A Snapdragon 865 from 2020 matches an old desktop i5 while sipping single-digit watts.
These are real computers — more powerful than the servers that ran the early internet.

A phone with a cracked screen and a dead battery is still a perfectly good 8-core Linux server
with no moving parts that fits in your pocket. Billions of them get landfilled every year.
The hardware is fine; the software ecosystem threw it away.

If your rooted phone has a kernel ≥ 4.19 with namespace support, this script will run a full Linux userspace on it — Debian, Arch, Alpine, whatever you need.

## What's in the box

| File | Lines | Description |
|------|-------|-------------|
| `nspawn` | ~670 | Full container runtime with network namespace support |
| `nspawn-mini` | ~230 | Stripped version — namespaces only, no networking |
| `getroot` | ~360 | Rootfs downloader from images.linuxcontainers.org |

## How it works

The script runs in two phases inside a single file:

```
Phase 1 (root, on the host)             Phase 2 (inside new namespaces)
┌─────────────────────────┐             ┌────────────────────────────┐
│ Parse arguments          │             │ Bind mount rootfs           │
│ Set up bridge + NAT      │──unshare──▶│ Mount /proc /sys /dev /tmp  │
│ Create netns + veth pair │             │ Create device nodes         │
│ Detect namespace support │             │ Mask sensitive paths        │
│                          │             │ pivot_root into rootfs      │
│                          │             │ exec shell or command       │
└─────────────────────────┘             └────────────────────────────┘
```

**The mount namespace is the key insight.** By doing all bind mounts inside a new mount namespace, cleanup is automatic.
When the process exits, the namespace is destroyed, and every mount disappears with it. No cleanup code needed.

### Namespaces

The script probes for available namespace support and uses whatever the kernel offers:

| Namespace | Flag | Purpose |
|-----------|------|---------|
| Mount | `-m` | Isolated filesystem view (required) |
| PID | `-p` | Container gets its own PID 1 |
| IPC | `-i` | Isolated shared memory / semaphores |
| UTS | `-u` | Container gets its own hostname |
| Network | `-n` | Isolated network stack (via `--net`) |
| Cgroup | `-C` | Isolated cgroup tree |
| Time | `-T` | Isolated boot/monotonic clocks |

Only mount namespace is required. Everything else is used if available, skipped if not.

### Networking (`--net`)

When `--net` is passed, the script creates a full network stack for the container:

```
                    Host
                 ┌─────────┐
    Internet ◄───┤ iptables ├──── MASQUERADE
                 │   NAT    │
                 └────┬─────┘
                      │
               ┌──────┴──────┐
               │  nspawn_br1  │  10.11.0.1/24
               │   (bridge)   │  fd11::1/64
               └──┬───────┬──┘
                  │       │
            ┌─────┴──┐ ┌──┴─────┐
            │ veth_a  │ │ veth_b │   ← veth pairs
            └─────┬──┘ └──┬─────┘
                  │       │
           ┌──────┴──┐ ┌──┴──────┐
           │Container │ │Container│
           │  .2      │ │  .3     │
           └─────────┘ └─────────┘
```

Each container gets a unique IP (10.11.0.2, .3, .4, ...) with full IPv4 and IPv6 dual-stack.
The bridge, NAT, and IP forwarding are set up automatically.

**Per-container routing** with `--route-via` lets you pin specific containers to specific interfaces:

```sh
# This container goes through VPN
sudo nspawn --net --route-via tun0 vpn_rootfs

# This one uses WiFi directly
sudo nspawn --net --route-via wlan0 wifi_rootfs
```

When an interface drops, the kernel falls through to the next routing rule. Free failover, zero code.

**Port forwarding** with `--port-range` exposes container ports to the host's WAN interface:

```sh
sudo nspawn --net --port-range 80:8080 my_server
```

### Android specifics

On Android, the script handles:

- **toybox `pivot_root`** — yes, toybox has it
- **busybox `mount`** — toybox mount is missing required features
- **`/system/bin/ip`** — Android ships iproute2
- **Policy-based routing** — VPN (`tun0/tun1`) > WiFi (`wlan0`) > Mobile (`rmnet_data*`), with configurable priorities
- **`/data/media/0`** bind mount — exposes internal storage at `/mnt/storage` inside the container
- **`/data/misc/net/rt_tables`** — makes `ip rule` show human-readable table names
- **Termux compatibility** — chroot immediately after `pivot_root` for `tsu` environments

### Path masking

This is **not** a security boundary. Think of it like `docker run --privileged` or a privileged LXC container.
The point is to have a working Linux environment. If you want real isolation, install Docker/nsjail inside.

Some host paths are masked or read-only to prevent accidents:

- **Masked**: `/proc/keys`, `/proc/kmsg`, `/proc/sysrq-trigger`, `/sys/firmware`, `/sys/power`, etc.
- **Read-only**: `/proc/bus`, `/proc/fs`, `/proc/irq`

This is just "don't accidentally write to `/proc/sysrq-trigger` and crash your phone" level protection.

## getroot

A companion rootfs downloader. Fetches pre-built images from [images.linuxcontainers.org](https://images.linuxcontainers.org).

```sh
# Download by name:release
sudo getroot debian:13
sudo getroot ubuntu:24.04
sudo getroot alpine:edge

# Specify output directory
sudo getroot debian:13 -o my_debian

# List all available images
sudo getroot --list

# Search releases for a distro
sudo getroot --search debian
```

Supported distros include: Debian, Ubuntu, Alpine, Arch, Fedora, CentOS, Kali, Gentoo,
Void, openSUSE, NixOS, Amazon Linux, Rocky, Alma, Oracle, OpenWrt, and more.

Release aliases work — `debian:13` resolves to `trixie`, `ubuntu:24` to `noble`. Architecture is auto-detected (amd64, arm64, armhf, riscv64).

The default root password is set to `1`. DNS is configured to `1.1.1.1`.

## Requirements

- POSIX shell (`/bin/sh`)
- `unshare` (coreutils or busybox)
- `mount` (coreutils or busybox — not toybox)
- `pivot_root` (coreutils or toybox)
- Kernel ≥ 4.19 with at least mount namespace support
- For `--net`: `ip` from iproute2 (not busybox) + `iptables`
- For `getroot`: `curl` or `wget` + `tar` with xz support
