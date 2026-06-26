# dhcpoptinj (OpenWrt Package Feed)

[![License: GPL v3+](https://img.shields.io/badge/License-GPLv3%2B-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

Intercept DHCP packets on a bridge interface and inject arbitrary DHCP options into them using netfilter queues.

## Overview

**dhcpoptinj** is a lightweight C daemon that hooks into the Linux netfilter queue system to intercept, modify, and re-inject DHCP packets in transit. It is designed for OpenWrt routers acting as bridges where you need to inject DHCP options (such as Relay Agent Information / Option 82, custom hostnames, DNS servers, etc.) without modifying the DHCP client or server.

This repository contains the **OpenWrt package feed**. This includes: the build recipe, init script, UCI configuration, and helper tools needed to integrate dhcpoptinj into an OpenWrt firmware build or install it on a running router.

The upstream source is maintained at [github.com/misje/dhcpoptinj](https://github.com/misje/dhcpoptinj).

## Features

- Intercept DHCP requests (port 67), replies (port 68), or both, on a bridge interface
- Inject any DHCP option by specifying its code and hex-encoded payload
- Remove conflicting options before injection
- Automatic nftables rule management via procd init script
- Automatic DHCP option length byte insertion
- UCI-compatible configuration with `service reload` support
- Respawn on crash via procd
- Bypass queue on daemon failure (traffic not interrupted)

## How It Works

1. nftables rules in the `bridge` family queue matching DHCP packets (udp dport 67/68) to a netfilter queue number.
2. `dhcpoptinj` reads packets from the queue, parses the BOOTP/DHCP payload, injects the configured DHCP options, recalculates the IP header checksum, and re-injects the modified packet.
3. The modified packet continues through the bridge path to its destination.

## Requirements

- OpenWrt (any recent release with nftables support)
- Kernel with `kmod-nft-queue`
- `libnetfilter-queue`
- `nftables`

## Building with the OpenWrt Build System

### Method 1: As a package feed (recommended)

Add this repository as a feed to your OpenWrt build environment:

```bash
# Clone the feed
git clone https://github.com/LuisMitaHL/dhcpoptinj-openwrt.git feeds/dhcpoptinj

# Or add it to feeds.conf.default / feeds.conf
echo "src-git dhcpoptinj https://github.com/LuisMitaHL/dhcpoptinj-openwrt.git" >> feeds.conf.default

# Update and install the feed
./scripts/feeds update dhcpoptinj
./scripts/feeds install dhcpoptinj

# Select the package in menuconfig
make menuconfig
# Navigate to: Network → dhcpoptinj  and enable it as a module (*)

# Build the package (or full firmware)
make package/dhcpoptinj/compile V=s
```

The resulting `.apk` will be placed at `bin/packages/ARCH/packages/` or similar, depending on your target architecture.

### Method 2: Manual integration

Copy the `net/dhcpoptinj` directory into your OpenWrt source tree under `package/`:

```bash
cp -r net/dhcpoptinj /path/to/openwrt/package/dhcpoptinj
cd /path/to/openwrt
make menuconfig  # select Network → dhcpoptinj
make package/dhcpoptinj/compile V=s
```

### Method 3: Standalone build (sdk)

If you have an OpenWrt SDK for your target:

```bash
# Prepare the SDK feed
echo "src-git dhcpoptinj https://github.com/LuisMitaHL/dhcpoptinj-openwrt.git" >> feeds.conf
./scripts/feeds update dhcpoptinj
./scripts/feeds install dhcpoptinj

# Build
make package/dhcpoptinj/compile V=s
```

## Installation on a Running Router

After building, transfer the `.apk` to your router and install:

```bash
apk add dhcpoptinj_*.apk --allow-untrusted
```

## Configuration

### UCI

Edit `/etc/config/dhcpoptinj`:

```uci
config dhcpoptinj 'main'
    option enabled '0'           # Enable the service (1 = on)
    option queue '0'             # Netfilter queue number (must match nftables rule)
    option snoop 'both'          # Packets to intercept: 'request', 'reply', 'both', 'none'
    option interface ''          # Bridge interface to match (e.g. 'br-lan'; empty = all)
    # list exclude_interface 'eth0'   # Bridge interfaces to exclude (empty = none)
    option debug '0'             # Enable debug output
    option forward_on_fail '0'   # Forward packet if processing fails
    option conflict ''           # Conflict handling: '' (off), 'ignore', or 'remove'
    option pid_file ''           # PID file path (empty = default)
    list option '0C:66:6A:61:73:65:68:6F:73:74'   # DHCP options to inject (hex)
```

**Option hex format**: the first byte is the DHCP option code, followed by the payload. The length byte is inserted automatically. Any non-hex delimiter is ignored (spaces, colons, hyphens all work).

Examples:
| Option | Hex |
|---|---|
| Hostname "fjasehost" (opt 12) | `0C:66:6A:61:73:65:68:6F:73:74` |
| Relay Agent Info (opt 82) | `52:01:04:46:6A:61:73` |
| Request IP 10.20.30.40 (opt 50) | `32:0A:14:1E:28` |
| DNS 8.8.8.8, 8.8.4.4 (opt 6) | `06:08:08:08:08:08:08:04:04` |

The pad option (code 0) is the only option allowed without payload. Option 255 (END) is automatically appended by dhcpoptinj and cannot be specified.

## Usage

### Service management

```bash
/etc/init.d/dhcpoptinj start
/etc/init.d/dhcpoptinj stop
/etc/init.d/dhcpoptinj restart
/etc/init.d/dhcpoptinj reload    # Re-read UCI config without full restart
```

The init script automatically creates and destroys the nftables `bridge dhcpoptinj` table. The queue action is set to `bypass` so DHCP traffic continues to flow even if dhcpoptinj crashes.

### Enc82 helper

The package includes `dhcpoptinj-enc82`, a helper to encode DHCP Option 82 (Relay Agent Information) sub-options into the hex format expected by `dhcpoptinj -o`:

```bash
dhcpoptinj-enc82 -c "my-relay" -r "switch-01"
# Output: 52:01:08:6D:79:2D:72:65:6C:61:79:02:09:73:77:69:74:63:68:2D:30:31
```

Use the output directly in `list option` in the UCI config.

## Files

| Path | Purpose |
|---|---|
| `/usr/bin/dhcpoptinj` | The DHCP option injection daemon |
| `/usr/bin/dhcpoptinj-enc82` | Helper to encode Option 82 sub-options |
| `/etc/config/dhcpoptinj` | UCI configuration file (preserved on upgrade) |
| `/etc/init.d/dhcpoptinj` | procd init script |

## License

GNU General Public License v3 or later (GPL-3.0-or-later). See `LICENSE` in the OpenWrt build tree or the upstream repository for details.
