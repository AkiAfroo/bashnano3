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

## S50watchdog – WiFi Watchdog Init Script

This script ensures **continuous WiFi connectivity** on embedded Linux systems (targeting `wlan0`).

### Functionalities
1. **Active Monitoring**  
   - Checks for IP on `wlan0` + ping to `8.8.8.8` every ~10–15 seconds.  
   - Requires **3 consecutive successful pings** to consider connection stable.

2. **Automatic Recovery**  
   - On failure: brings interface down/up, restarts `wpa_supplicant`, requests DHCP via `udhcpc`.

3. **Failure Escalation**  
   - 60 consecutive failures (~10 minutes) → system reboot.  
   - Up to **3 reboot attempts**. After the 3rd, if failure persists → `poweroff`.

4. **Smart Logging**  
   - Logs only after **≥30 consecutive failures**.  
   - Events saved to `/opt/wlan.logs`.

5. **Daily Scheduled Tasks**  
   - **23:59**: Copies "New best share" lines from `/tmp/zlog/run.log` (current day) to `/opt/best_shares.log`.  
   - **00:00**: Triggers a **scheduled reboot** (once per day).

6. **Robustness & Safety**  
   - **Self-generates** `/bin/watchdog` if missing.  
   - **Lock file** `/tmp/watchdog.lock` → prevents multiple instances.  
   - **Trap cleanup** on exit (INT/TERM).  
   - Clears old `wpa_supplicant` processes.  
   - Disables power-saving (`iwconfig power off`).  
   - Uses control files (`last_copy_hour`, `last_reboot_hour`) to run tasks **once per day**.

---

## S99btcminer – Delayed Bitcoin Miner Startup Script

This init script manages the **`btcminer`** process with **delayed startup**, **control commands**, and **self-generated CLI tool** (`/bin/btcminerctl`).

### Functionalities
1. **Delayed Startup**  
   - Waits **10 seconds** after boot before launching `btcminer`.  
   - Ensures network, WiFi, and system services (like `S50watchdog`) are ready.

2. **Process Management**  
   - `start`: Runs `./btcminer &` from `/mnt/heater/app` in background.  
   - `stop`: `killall btcminer` (graceful process termination).  
   - `restart`: Stop + start.  
   - `status`: Shows running PIDs or "not running".

3. **Self-Generated Control Tool**  
   - Creates `/bin/btcminerctl` if missing.  
   - Allows simple CLI control:  
     ```bash
     btcminerctl start
     btcminerctl stop
     btcminerctl restart
     btcminerctl status

4. **Error Handling & Logging**  
   - Logs to `/opt/nanologs.log` if `btcminer` is missing or not executable.  
   - Silent `cd -` redirection to avoid polluting output.

5. **Init Integration**  
   - LSB-compliant headers (`### BEGIN INIT INFO`).  
   - Depends on `$network` → runs **after networking is up**.  
   - Runlevels: `2 3 4 5` (multi-user + GUI).

6. **Execution Flow**  
   - `S99` → **last in boot order** (after `S50watchdog`, SSH, etc.).  
   - Background subshell + `&` → non-blocking boot.

### Why the Delay?
- Prevents race conditions: miner tries to connect to pool **before WiFi is stable**.  
- `S50watchdog` needs time to recover connectivity on boot.  
- 10-second delay = **safe balance** between speed and reliability.

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
