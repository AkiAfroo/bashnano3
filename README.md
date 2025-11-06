# BASHNANO3 – System Initialization Script

![BASHNANO3](https://i.imgur.com/PeFJxyi.png)

**Version:** `1.2`  
**Target Device:** Avalon Nano 3 (Bitcoin Miner)  
**Language:** Shell Script (`sh`)  
**Root Access:** Achieved via **command injection in `timezoneconf.cgi`** (RISC-V, BusyBox Linux) – **foundation for all system-level improvements**

---

## How It Works

### First Boot (No `/opt/bk*` found)
1. Mounts UBI: `/mnt` ← `ubi1_0`, `/data` ← `ubi2_0`
2. Patches web UI HTML (`sed` + base64 script)
3. Backs up:  
   - `/mnt/*` → `/opt/bkmnt/`  
   - `/data/*` → `/opt/bkdata/`
4. Generates:  
   - `cgminer.ini` (with worker)  
   - `systemcfg.ini` (with WiFi)  
   - `wpa_supplicant.conf` (if missing)
5. Sets `nanolock=1` → **protects backup**
6. `sync && reboot`

### Subsequent Boots
1. `conditional_cleanup()` checks `nanolock`  
   → `0` = **wipe everything** (forces rebuild)  
   → `1` = **keep backup**
2. **Restores** from `/opt` to `/mnt` & `/data`
3. **Regenerates `systemcfg.ini`** with current WiFi
4. Starts SSH, logging, services
5. Executes `/etc/init.d/S??*`

---

## Overview

This is a **Modded system initialization script** (`rcS`) for the **Avalon Nano 3**.

It fully automates the boot process:

- UBI/UBIFS partition mounting (A/B detection via `fw_printenv`)
- **First-boot backup** → `/opt/bkmnt` & `/opt/bkdata`
- **Subsequent boot restore** from `/opt` (fast & flash-safe)
- **Unique worker name** via WiFi MAC (`DeviceName + last 4 chars`)
- WiFi config (`wpa_supplicant.conf` with SSID/PSK)
- Web UI **CDN patching** (`jsDelivr` → `cdnjs.cloudflare.com` + SHA512 integrity)
- `cgminer.ini` & `systemcfg.ini` generation (WiFi, worker, pool)
- `nanolock` – **secure deletion lock** (`/etc/nanolock.conf`)
- **RAM-based logging** (`/data/log` → `/tmp/zlog` symlink)
- SSH server (`admin:admin`, auto keygen)
- `/etc/init.d/S??*` startup scripts
- Core dump control (`/data/core`, auto-clean >16MB)

---

## Key Features

| Feature | Description |
|--------|-----------|
| **Smart Backup & Restore** | First boot: UBI → `/opt`<br>Next boots: `/opt` → `/mnt` & `/data` |
| **Auto Worker Naming** | `DeviceName + last 4 MAC chars` → e.g. `nano3-94fa` |
| **HTML CDN Patching** | `jsDelivr` → `cdnjs.cloudflare.com` + full SHA512 integrity |
| **nanolock Security** | `nanolock 1` = **lock** (no delete)<br>`nanolock 0` = **wipe & rebuild** |
| **WiFi Auto-Config** | `wpa_supplicant.conf` + `systemcfg.ini` **always updated** |
| **RAM Logging** | `/data/log` → `/tmp/zlog` (prevents NAND wear) |
| **SSH Access** | `admin:admin` + auto `ed25519` keygen |
| **Core Dump Cleanup** | Auto-remove if `/data` > 16MB |

---

## Configuration

Edit at the top of `/tools/rcS`:

```sh
wifiname="mywifiname"
wifipass="mypass"
btc_address="btcaddress"
DeviceName="nano3"          # Base name for worker
