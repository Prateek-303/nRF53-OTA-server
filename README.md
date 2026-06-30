<div align="center">
  <h1>🚀 NexusOTA Cloud Server</h1>
  <p><strong>Over-The-Air (OTA) Firmware Delivery Network for nRF5340 & nRF7002 DK</strong></p>
</div>

---

## 📌 Overview
This repository serves as the cloud backend for the **NexusOTA** framework. It leverages GitHub Pages to act as a highly-available, secure HTTPS server that delivers signed `.bin` firmware updates and manifest files to edge devices in the field.

By utilizing **Let's Encrypt (ISRG Root X1)** TLS certificates and a dynamic JSON manifest, this server ensures that all edge devices can securely poll, verify, and apply firmware updates automatically.

## ⚙️ Architecture 
1. **Manifest Polling:** Edge devices periodically fetch `/manifest.json` over a secure TLS socket.
2. **Version Resolution:** The device compares its internal `VERSION` against the manifest.
3. **Payload Streaming:** If a newer version is detected, the device streams the `.bin` payload in 1KB chunks directly into the secondary flash slot.
4. **Cryptographic Verification:** The device actively computes the **IEEE CRC-32** and **SHA-256** hashes during the download, comparing them against the manifest.
5. **MCUboot Swap:** Upon a successful cryptographic match, MCUboot swaps the image into the primary slot and boots the new firmware.

---

## 🛠️ Step-by-Step Setup Guide

### 1. Preparing the Cloud Environment
1. **Host on GitHub:** Ensure this repository is hosted on GitHub.
2. **Enable GitHub Pages:** Go to **Settings -> Pages**. Under "Build and deployment", set the source to deploy from the `main` branch. 
3. GitHub will now automatically host `manifest.json` and your `.bin` files via HTTPS.

### 2. Pushing Firmware Updates
Whenever you compile a new firmware version, use the provided `push_server.bat` script (located in your firmware workspace) to deploy it:
1. Compile your Zephyr firmware.
2. The compiler script will inject the SHA256 and CRC32 hashes into `manifest.json`.
3. Run `push_server.bat` to automatically stage, commit, and push the `.bin` and `manifest.json` to this repository.

### 3. Edge Device Upgrades
Once pushed, the edge devices will automatically:
1. Detect the version bump.
2. Display a dynamic `printk` progress bar on the serial terminal: `[##########----------] 50%`
3. Verify the checksums and reboot into the new version.

---

## ⚠️ Things to Keep in Mind

* **GitHub Pages Caching:** GitHub Pages has a global CDN cache that takes about **3 to 5 minutes** to refresh. If you push a new `manifest.json`, the edge device might not see it instantly. Give it a few minutes before assuming the OTA failed!
* **TLS Certificate Expiration:** The firmware relies on the `ISRG Root X1` certificate hardcoded in `github_certs.h`. If this certificate ever expires (or if you move to a non-GitHub server), the firmware will reject the TLS handshake. You must update the certificate *before* it expires using an OTA update.
* **COM Port Conflicts:** When using the deployment `.bat` scripts locally, ensure your Serial Terminal (e.g., PuTTY) is closed during the flashing process, otherwise the build tool will fail to access the COM port.

---

## 🧠 Challenges, Error Codes & Engineering Solutions

Building a robust OTA pipeline on deeply embedded, resource-constrained hardware presented severe architectural challenges. Here are the specific POSIX/Zephyr error codes we encountered and how we engineered solutions:

### 1. Networking Errors (`-113`, `-116`, `-22`)
**Challenge:** During socket creation and HTTP polling, the firmware frequently crashed with raw socket errors:
* **`-113` (EHOSTUNREACH):** The device authenticated with the Wi-Fi router, but the DNS resolver or external gateway failed to route packets to GitHub.
* **`-116` (ETIMEDOUT):** The TCP socket timed out during the massive TLS handshake because asymmetric cryptographic math (RSA/ECC) took up to 5-8 seconds on the nRF5340.
* **`-22` (EINVAL):** Invalid arguments passed to the socket layer, typically occurring when the stack attempted to allocate an IPv6 context while IPv6 was disabled to save memory.

**Solution:** We hardcoded `CONFIG_NET_IPV6=n` to prevent IPv6 socket crashes (fixing `-22`), enforced a robust DNS fallback mechanism, and explicitly extended the Zephyr socket timeout settings (`CONFIG_NET_SOCKETS_CONNECT_TIMEOUT`) to prevent TLS handshakes from returning `-116`. 

### 2. TLS Certificate Verification Failures (Error `-0x2700`)
**Challenge:** When attempting to establish the HTTPS connection to GitHub, mbedTLS threw `mbedtls_ssl_handshake returned -0x2700` (X509 - Certificate verification failed).
**Solution:** Embedded devices do not have an operating system with a built-in trust store. We manually extracted the **DigiCert Global Root G2** (and subsequently the **ISRG Root X1** for GitHub Pages) in PEM format, hardcoded it into `github_certs.h`, and registered it into the Zephyr TLS credentials subsystem using `tls_credential_add()`.

### 3. mbedTLS Heap Exhaustion (Error `-ENOMEM` or `-12`)
**Challenge:** During the TLS handshake, mbedTLS must allocate massive buffers to parse the server's certificate chain. The system crashed with `socket() failed: -12` (Not enough core/memory).
**Solution:** Zephyr's default network heap is too small for modern HTTPS. We aggressively increased the mbedTLS heap to 80KB (`CONFIG_MBEDTLS_HEAP_SIZE=81920`) and the system RAM pool to 120KB (`CONFIG_HEAP_MEM_POOL_SIZE=120000`). We also allocated 8KB specifically for the network TX/RX threads.

### 4. Flash Memory Overflows (Linker Error)
**Challenge:** After adding the Wi-Fi stack, mbedTLS, and MCUboot, the linker failed with `region FLASH overflowed by 45000 bytes`. The firmware was simply too large for the primary application slot.
**Solution:** We aggressively pruned the networking stack. We explicitly disabled IPv6 (`CONFIG_NET_IPV6=n`), disabled WPA3 support (`CONFIG_WIFI_NM_WPA_SUPPLICANT_WPA3=n`), disabled WPA supplicant internal debug strings, and enforced global size optimizations (`CONFIG_SIZE_OPTIMIZATIONS=y`). This freed up over 50KB of flash, allowing the OTA engine to fit cleanly.

### 5. Connection Unreliability & Data Corruption
**Challenge:** Downloading 500+ KB over a noisy Wi-Fi network risks dropped packets or corrupted bytes being written to the flash memory, bricking the device.
**Solution:** Implemented a **Dual-Verification Engine**. During the 1KB chunk stream, the firmware actively calculates a running **IEEE CRC-32** checksum. Once the download is complete, MCUboot performs a final **SHA-256** cryptographic signature check. If either fails, the corrupted payload is erased.

---
*Designed & Architected by Prateek Baraiya for robust IoT Firmware Operations.*
