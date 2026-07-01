<div align="center">

# 🚀 OTA Cloud Server

**Over-The-Air (OTA) Firmware Delivery Network for nRF7002 DK (nRF5340)**

[![GitHub Repo](https://img.shields.io/badge/Server%20Repo-nRF53--OTA--server-blue?logo=github)](https://github.com/Prateek-303/nRF53-OTA-server)
[![Firmware Repo](https://img.shields.io/badge/Firmware%20Repo-Firmware--OTA-informational?logo=github)](https://github.com/Prateek-303/Firmware-OTA)
[![SDK](https://img.shields.io/badge/nRF%20Connect%20SDK-v3.2.4-orange)](https://developer.nordicsemi.com)
[![Board](https://img.shields.io/badge/Board-nRF7002%20DK-green)](https://www.nordicsemi.com/Products/Development-hardware/nRF7002-DK)
[![TLS](https://img.shields.io/badge/TLS-1.2%20%2F%20mbedTLS-red)](https://tls.mbed.org)

*Designed & Architected by **Prateek Baraiya***

</div>

---

## 📌 Overview

This repository is the **cloud backend** for the NexusOTA framework. It uses a **plain GitHub repository** served via `raw.githubusercontent.com` as a zero-cost, globally available, TLS-secured HTTPS server to deliver signed firmware binaries and JSON manifests to nRF5340-based edge devices in the field. No GitHub Pages, no extra configuration — just push to `main` and the files are instantly live.

> **Firmware Repository:** The source code for the firmware lives at:
> 👉 [github.com/Prateek-303/Firmware-OTA](https://github.com/Prateek-303/Firmware-OTA)

The system supports:
- Automatic version detection and upgrade
- Real-time streaming download with a live progress bar
- Dual cryptographic verification (IEEE CRC-32 live + SHA-256 post-download)
- MCUboot-managed atomic swap with automatic rollback failsafe
- MAC address-based device authorization whitelist

---

## 📁 Repository Structure

```
📦 nrf54-OTA  (this GitHub repository)
 ┣ 📜 manifest.json              ← Auto-updated: version, size, SHA256, CRC32
 ┣ 📜 authorized_devices.json   ← MAC address whitelist for security
 ┣ 📜 app_update_1.0.1.bin      ← V1 signed firmware (LED blink only)
 ┣ 📜 app_update_1.0.2.bin      ← V1.1 signed firmware (I2C pin fix)
 ┣ 📜 app_update_2.0.0.bin      ← V2 signed firmware (LED + BMP280 + TMP117)
 ┣ 📜 LICENSE
 └ 📜 README.md
```

---

## ⚙️ System Architecture

### High-Level Block Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER MACHINE                            │
│                                                                     │
│  ┌──────────────┐    build_ota.bat     ┌──────────────────────────┐│
│  │  VS Code /   │ ─────────────────▶  │   west build             ││
│  │  Source Code │                     │   (Zephyr + nRF SDK)     ││
│  └──────────────┘                     └───────────┬──────────────┘│
│                                                   │               │
│                                         zephyr.signed.bin         │
│                                                   │               │
│                                       ┌───────────▼──────────────┐│
│                                       │  update_manifest.py      ││
│                                       │  • Compute SHA-256        ││
│                                       │  • Compute IEEE CRC-32   ││
│                                       │  • Write manifest.json   ││
│                                       └───────────┬──────────────┘│
│                                                   │               │
│                                          git push origin main     │
└───────────────────────────────────────────────────┼───────────────┘
                                                    │
                                    ┌───────────────▼───────────────┐
                                    │     GITHUB REPOSITORY         │
                                    │  (raw.githubusercontent.com)  │
                                    │                               │
                                    │  ┌─────────────────────────┐  │
                                    │  │  manifest.json           │  │
                                    │  │  app_update_X.Y.Z.bin   │  │
                                    │  │  authorized_devices.json│  │
                                    │  └─────────────────────────┘  │
                                    └───────────────┬───────────────┘
                                                    │  HTTPS / TLS 1.2
                                                    │  (every 24 hours)
                                    ┌───────────────▼───────────────┐
                                    │         nRF7002 DK            │
                                    │       (nRF5340 SoC)           │
                                    │                               │
                                    │  1. Wi-Fi connect             │
                                    │  2. Fetch manifest.json       │
                                    │  3. Compare versions          │
                                    │  4. Stream .bin (1KB chunks)  │
                                    │  5. Live CRC-32 check         │
                                    │  6. Final SHA-256 check       │
                                    │  7. MCUboot image swap        │
                                    │  8. Reboot → new firmware ✅  │
                                    └───────────────────────────────┘
```

### OTA State Machine

```
  BOOT
   │
   ▼
[Confirm image] ──fail──▶ MCUboot rolls back to previous version
   │ ok
   ▼
[Register TLS CA cert]
   │
   ▼
[Wi-Fi connect] ──fail──▶ retry indefinitely
   │ IP assigned
   ▼
[Spawn OTA thread] ◀──────────────────────────────────┐
   │                                                  │
   ▼                                                  │
[HTTPS GET /manifest.json]                            │
   │                                                  │
   ├── parse version ──▶ same as running? ──YES──▶ sleep 24h ──┘
   │
   NO (newer version found)
   │
   ▼
[Check authorized_devices.json] ──not listed──▶ abort
   │ authorized
   ▼
[HTTPS GET /app_update_X.Y.Z.bin]
   │ stream 1KB chunks
   ▼
[Live CRC-32 accumulation] ──mismatch──▶ erase flash, abort
   │ match
   ▼
[SHA-256 final verify] ──mismatch──▶ erase flash, abort
   │ match
   ▼
[MCUboot: mark image for swap]
   │
   ▼
[sys_reboot()] ──▶ MCUboot swaps primary ↔ secondary
   │
   ▼
  BOOT (new firmware) ──▶ confirm image ──▶ running ✅
```

---

## 🛠️ Full Step-by-Step Execution Guide

### ─── PHASE 0: One-Time Environment Setup ───

#### Step 0.1 — Install Prerequisites
Make sure all of the following are installed and on PATH:

| Tool | Version | Check Command |
|------|---------|---------------|
| nRF Connect SDK | v3.2.4 | `west --version` |
| nRF Connect Toolchain | fd21892d0f | Located at `C:\ncs\toolchains\` |
| Python | 3.x | `python --version` |
| Git | Any modern | `git --version` |

#### Step 0.2 — Cache GitHub Credentials (Do Once)
So `git push` never asks for a password interactively:

```powershell
git config --global credential.helper manager
```

Then manually run any `git push` once to trigger the Windows login prompt. After that, all future pushes are fully silent and automatic.

#### Step 0.3 — Verify the Repository is Public
1. Open this repository on GitHub.
2. Go to **Settings → General → Danger Zone**.
3. Ensure the repository visibility is set to **Public**.

> This is required because the firmware fetches files via `raw.githubusercontent.com` without any authentication token. Private repos will return HTTP 404 and the OTA will silently fail.

Files are served directly over HTTPS at:
```
https://raw.githubusercontent.com/Prateek-303/nrf54-OTA/main/manifest.json
https://raw.githubusercontent.com/Prateek-303/nrf54-OTA/main/app_update_1.0.2.bin
```

> 💡 No GitHub Pages setup needed. Just push to `main` and files are immediately accessible.

---

### ─── PHASE 1: First Build (Clean) ───

#### Step 1.1 — Open nRF Connect SDK Terminal
Open **PowerShell** and activate the toolchain environment:

```powershell
cd D:\Nordic\OTA_FIRM
C:\ncs\toolchains\fd21892d0f\opt\bin\activate.bat
```

> ⚠️ You must activate this every time you open a new terminal. Without it, `west` is not found.

#### Step 1.2 — Clean Old Build Artifacts
For the very first build, or after adding/removing source files:

```powershell
Remove-Item -Recurse -Force build
```

#### Step 1.3 — Check VERSION File
Open `D:\Nordic\OTA_FIRM\VERSION` and ensure it starts at `1.0.0`:

```
VERSION_MAJOR = 1
VERSION_MINOR = 0
PATCHLEVEL = 0
VERSION_TWEAK = 0
```

#### Step 1.4 — Run the Build Script

```powershell
.\build_ota.bat
```

**What happens internally:**
```
build_ota.bat
  │
  ├─▶ west build -b nrf7002dk/nrf5340/cpuapp --sysbuild
  │     └─▶ produces: build\OTA_FIRM\zephyr\zephyr.signed.bin
  │
  ├─▶ Read VERSION file → sets VERSION_STR = 1.0.0
  │
  ├─▶ Clone deploy_repo from GitHub (if not already cloned)
  │     └─▶ git clone https://github.com/Prateek-303/nrf54-OTA.git deploy_repo
  │
  ├─▶ Copy zephyr.signed.bin → deploy_repo\app_update_1.0.0.bin
  │
  ├─▶ python update_manifest.py 1.0.0 app_update_1.0.0.bin
  │     ├─▶ Compute SHA-256 of binary
  │     ├─▶ Compute IEEE CRC-32 of binary
  │     └─▶ Write manifest.json with version, size, hashes, image path
  │
  └─▶ cd deploy_repo
        git add app_update_1.0.0.bin manifest.json
        git commit -m "Auto OTA Update to v1.0.0"
        git push origin main
```

#### Step 1.5 — Flash to Board (First Time Only)
The first flash must be done via USB cable using nRF Connect for Desktop or west:

```powershell
west flash --recover
```

This flashes both MCUboot and your application into the board permanently.

---

### ─── PHASE 2: Deploying an OTA Update ───

Follow this every time you want to push a new firmware version to the fleet wirelessly.

#### Step 2.1 — Make Your Code Changes
Edit the source files in `D:\Nordic\OTA_FIRM\src\`:

| File | Purpose | When to Edit |
|------|---------|--------------|
| `user_applications.c` | Orchestrator — toggles V1/V2 | Every version bump |
| `Led_Blink.c` | LED blink control | V1 and V2 |
| `Sensors_OTA.c` | BMP280 + TMP117 I2C logic | V2 only |
| `ota_http.c` | OTA engine (rarely touched) | Advanced changes |
| `wifi_mgr.c` | Wi-Fi lifecycle (rarely touched) | Advanced changes |

**V1 → V2 Switch** (uncomment in `user_applications.c`):
```c
// V1: Only LED
// V2: Uncomment the lines below
#include "Sensors_OTA.h"        // ← uncomment
Sensors_OTA_init();             // ← uncomment in init
Sensors_OTA_read_and_log();     // ← uncomment in run loop
```

#### Step 2.2 — Bump the Version
Open `D:\Nordic\OTA_FIRM\VERSION` and increment:

```
VERSION_MAJOR = 1       ← Major: breaking/architectural changes
VERSION_MINOR = 0       ← Minor: new features (e.g., V2 sensors)
PATCHLEVEL = 2          ← Patch: bug fixes, pin fixes, etc.
```

> ⚠️ The OTA engine only downloads if manifest version > running version. Same version = no update.

**Version Strategy:**
```
1.0.x → V1 firmware  (LED blink only, sensors commented out)
2.0.x → V2 firmware  (LED + BMP280 + TMP117 sensor data)
```

#### Step 2.3 — Build & Push

```powershell
cd D:\Nordic\OTA_FIRM
.\build_ota.bat
```

Watch the terminal output — successful end looks like:
```
==========================================
[SUCCESS] Update v1.0.2 has been automatically pushed to GitHub!
Wait 5 minutes for GitHub cache to update, then hit RESET on your board!
==========================================
```

#### Step 2.4 — Wait for GitHub Cache (Important!)
`raw.githubusercontent.com` has a short server-side cache. After every `git push`, wait about **1 to 2 minutes** before resetting the board. Unlike GitHub Pages (which can take 5 minutes), raw GitHub is much faster to propagate — but not instant.

#### Step 2.5 — Trigger OTA on the Board
Press the **RESET** button on the nRF7002 DK.

The board will:
1. Boot and confirm the current image.
2. Connect to Wi-Fi (watch for `[NET] Ready — IP: 192.168.x.x`).
3. Poll `manifest.json` and detect the new version.
4. Begin downloading with a live progress bar:

```
[OTA] Detected update: v1.0.1 → v1.0.2
[OTA] Binary size: 652400 bytes
[############################--------------------] 55% (358400 / 652400 bytes)
[OTA] Download complete.
[OTA] CRC32 verified  ✓
[OTA] SHA256 verified ✓
[OTA] Requesting MCUboot swap...
*** Booting My Application v1.0.2 ***
```

---

## 📋 manifest.json — Field Reference

The manifest is the single source of truth for the OTA engine. It is auto-generated by `update_manifest.py`. **Never edit it manually.**

```json
{
    "version": "1.0.2",
    "file_size": 652400,
    "image": "/app_update_1.0.2.bin",
    "sha256": "c4d0505dc6c43e1d11bd4037052c01423ed872f436e9cf6791fa7327b22948c6",
    "crc32": "3c5ce300"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `version` | string | Semantic version. Board downloads only if this > running version. |
| `file_size` | int | Exact byte count. Used to detect incomplete downloads. |
| `image` | string | URL path relative to GitHub Pages root. |
| `sha256` | string | Full SHA-256 hex digest. Verified by MCUboot post-download. |
| `crc32` | string | IEEE 802.3 CRC-32 hex. Verified live during streaming to catch bit errors. |

---

## 🔐 Device Authorization

Only MAC addresses listed in `authorized_devices.json` will receive updates:

```json
{
  "authorized_devices": [
    "bf77015cc222c893"
  ]
}
```

To authorize a new board:
1. Boot the board and open the serial terminal at 115200 baud.
2. Find the line: `[OTA] Device MAC: bf77015cc222c893`
3. Add that MAC string to the array in `authorized_devices.json`.
4. Commit and push.

---

## 🧠 Challenges, Error Codes & Engineering Solutions

### ❌ `-113` — EHOSTUNREACH (Host Unreachable)
**Symptom:** `socket connect failed: -113`  
**Cause:** Board connected to Wi-Fi but DNS/gateway cannot reach `raw.githubusercontent.com`.  
**Fix:** Verify router has internet. Check if ISP or corporate firewall blocks GitHub. The OTA thread retries automatically every 24 hours.

---

### ❌ `-116` — ETIMEDOUT (Connection Timed Out)
**Symptom:** `mbedtls_ssl_handshake returned: -116`  
**Cause:** TLS handshake RSA/ECC asymmetric math takes 5–8 seconds on nRF5340. TCP socket default timeout was shorter.  
**Fix:** Extended in `prj.conf` via `CONFIG_NET_MGMT_EVENT_QUEUE_TIMEOUT=5000`. Ensure strong Wi-Fi signal — weak signal multiplies TCP retransmissions which compound TLS delay.

---

### ❌ `-22` — EINVAL (Invalid Argument)
**Symptom:** `socket() failed: -22`  
**Cause:** Networking stack tried to open an IPv6 socket when `CONFIG_NET_IPV6=n` was set to save memory.  
**Fix:** Resolved in `prj.conf` with `CONFIG_NET_IPV4=y` and `CONFIG_NET_IPV6=n`.

---

### ❌ `-12` / `-ENOMEM` (Out of Memory)
**Symptom:** `socket() failed: -12` or system crash during handshake  
**Cause:** mbedTLS allocates large buffers to parse GitHub's full X.509 certificate chain. Default Zephyr heap is too small.  
**Fix:** Resolved with `CONFIG_MBEDTLS_HEAP_SIZE=81920` (80 KB) and `CONFIG_HEAP_MEM_POOL_SIZE=120000` (120 KB).

---

### ❌ `-0x2700` — mbedTLS X509 Certificate Verify Failed
**Symptom:** `mbedtls_ssl_handshake returned -0x2700`  
**Cause:** Embedded devices have no OS trust store. The board rejected GitHub's certificate because it had no trusted root CA to verify it against.  
**Fix:** Manually extracted the **ISRG Root X1** PEM certificate from Let's Encrypt, hardcoded it in `github_certs.h`, and registered it with `tls_credential_add(TLS_CREDENTIAL_CA_CERTIFICATE)`.

---

### ❌ `region FLASH overflowed` (Linker Error)
**Symptom:** Build fails: `region FLASH overflowed by 45000 bytes`  
**Cause:** Wi-Fi stack (WPA Supplicant) + mbedTLS + MCUboot + application exceeded the primary slot size.  
**Fix:** Pruned the stack aggressively in `prj.conf`:
```
CONFIG_SIZE_OPTIMIZATIONS=y
CONFIG_NET_IPV6=n
CONFIG_WIFI_NM_WPA_SUPPLICANT_WPA3=n
CONFIG_WPA_SUPP_LOG_LEVEL_NONE=y
```
Saved 50+ KB of flash.

---

### ❌ `BMP280 ID read failed` (I2C Sensor Error)
**Symptom:** `<err> Sensors_OTA: BMP280 ID read failed` on every boot  
**Cause 1:** SCL/SDA pins swapped in the device tree overlay.  
**Cause 2:** BMP280 address mismatch — SDO pin state sets address to `0x76` or `0x77`.  
**Cause 3:** Missing internal pull-up resistors on the I2C lines.  
**Fix:** Corrected overlay to `SCL=P1.02, SDA=P1.03` with `bias-pull-up`. Added auto-detection fallback to try both `0x76` and `0x77` in firmware.

---

### ❌ `Cannot find source file: src/app_logic.c` (CMake Error)
**Symptom:** Build fails with CMake error about missing source file  
**Cause:** `CMakeLists.txt` listed old filenames after source files were renamed.  
**Fix:** Updated `CMakeLists.txt` `target_sources()` block to match the actual filenames: `user_applications.c`, `Led_Blink.c`, `Sensors_OTA.c`.

---

### ❌ Net Socket Log Flooding (UI Issue)
**Symptom:** Serial terminal was flooded with hundreds of `net_sock` debug lines per second, destroying the progress bar display.  
**Fix:** Disabled in `prj.conf` with `CONFIG_NET_LOG=n`. Implemented a carriage-return (`\r`) overwriting `printk` progress bar so it updates in-place on the same terminal line.

---

## ⚠️ Important Things to Keep in Mind

| Warning | Detail |
|---------|--------|
| ⏱️ **GitHub Cache Delay** | Wait 1–2 minutes after `git push` before resetting the board. `raw.githubusercontent.com` propagates fast but not instantly. |
| 🔐 **TLS Cert Expiry** | The ISRG Root X1 cert in `github_certs.h` has a long validity but must be renewed via OTA before it expires. |
| 🔌 **COM Port Conflict** | Close PuTTY / any serial terminal before running `west flash` or `build_ota.bat`. |
| 🗑️ **Clean Builds** | Delete `build\` folder when adding new `.c` files or changing Kconfig options. |
| 🔢 **Version Must Increase** | Pushing the same version number triggers no OTA. Always bump `PATCHLEVEL` at minimum. |
| 🔄 **MCUboot Rollback** | If the new firmware boots but does not call `boot_write_img_confirmed()` within 10 seconds, MCUboot automatically reverts to the previous image. |
| 📡 **Poll Interval** | OTA thread polls every 24 hours. Press RESET to trigger an immediate check. |
| 🔒 **Repo Must Be Public** | The firmware fetches files without auth tokens. A private repo returns HTTP 404 and OTA silently fails. |

---

## 🔗 Reference Links

- [nRF Connect SDK Docs](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest/nrf/index.html)
- [Zephyr MCUboot / DFU Guide](https://docs.zephyrproject.org/latest/services/dfu/index.html)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)
- [mbedTLS Error Code Reference](https://tls.mbed.org/api/group__mbedtls__error.html)
- [nRF7002 DK Product Page](https://www.nordicsemi.com/Products/Development-hardware/nRF7002-DK)

---

<div align="center">
<em>Designed & Architected by <strong>Prateek Baraiya</strong> — nRF5340 NexusOTA Firmware System</em>
</div>
