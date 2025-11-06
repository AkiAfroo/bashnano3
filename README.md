# BASHNANO3
(Every file has its own description of what it does.)
![BASHNANO3](https://i.imgur.com/PeFJxyi.png)
**Version:** `1.0`  
**Target Device:** Avalon Nano 3 (Bitcoin Miner)  
**Language:** Shell Script (`sh`)  
**Embedded device penetration testing of the Avalon Nano3 Obtaining root via command-injection in timezoneconf.cgi (RISC-V, BusyBox Linux)**
---

## Overview of rcS file in /tools/

This is a **complete system initialization script** for the **Avalon Nano 3** Bitcoin miner. It automates the entire boot process, including:

- UBI/UBIFS partition mounting
- First-time backup & subsequent restore of `/mnt` and `/data`
- Automatic **unique worker name generation** from MAC address
- WiFi configuration (`wpa_supplicant.conf`)
- Web UI HTML patching (jsDelivr → Cloudflare CDN with integrity)
- `cgminer.ini` and `systemcfg.ini` generation
- `nanolock` – secure deletion control
- RAM-based logging (flash wear protection)
- SSH server startup (`admin:admin`)
- Startup scripts execution (`/etc/init.d/S??*`)
- Core dump handling and cleanup

> Perfect for **mass deployment, firmware recovery, and headless provisioning**.

---

## Key Features

| Feature | Description |
|--------|-----------|
| **UBI Partition Auto-Attach** | Detects active partition (A/B) via `fw_printenv` |
| **Smart Backup & Restore** | First boot: backup → `/opt`<br>Subsequent boots: restore from `/opt` |
| **Auto Worker Naming** | `DeviceName + last 4 MAC chars` → e.g., `nano-94fa` |
| **HTML CDN Patching** | Replaces `jsdelivr.net` with `cdnjs.cloudflare.com` + SHA512 integrity |
| **nanolock Security** | Prevents accidental deletion of configs |
| **WiFi Watchdog Ready** | Auto-creates `wpa_supplicant.conf` |
| **RAM Logging** | `/data/log` → `/tmp/zlog` symlink |
| **SSH Enabled** | `admin:admin` + auto keygen |
| **Core Dump Control** | Auto-clean `/data/core` if >16MB |

---

## How It Works

### First Boot (No `/opt/bk*` found)
1. Mounts UBI volumes (`/mnt`, `/data`)
2. Patches HTML files in `/mnt/heater/www/html`
3. Backs up `/mnt` → `/opt/bkmnt`, `/data` → `/opt/bkdata`
4. Generates config files with **unique worker name**
5. Sets `nanolock=1` and **reboots**

### Subsequent Boots
1. Restores `/opt/bkmnt` → `/mnt`, `/opt/bkdata` → `/data`
2. Starts SSH, services, logging
3. Runs `/etc/init.d/S??*` scripts

---

## Configuration

Edit these at the top of the script in /tools/rcS:

```sh
wifiname="mywifiname"
wifipass="mypass"
btc_address="btcaddress"
DeviceName="nano3"          # Base name for auto-generated worker
