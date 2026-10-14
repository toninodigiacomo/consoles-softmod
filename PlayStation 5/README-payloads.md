# ps-hack — Chaîne de payloads PS5

This document covers **what happens after the exploit**: which payloads are loaded onto the PS5, in what order, by whom, and how to maintain them.
The exploit’s hosting setup (Docker, nginx, NPM, DNS) is described in the main [`README.md`](README.md).

> Console: PS5 running firmware **13.20**, static IP **192.168.1.80**.
> ⚠️ Never update the console (version 14.00 patches the vulnerability).

---

## 🏛️ Architecture

```
Guide utilisateur PS5
 │
 ▼
Relapse (fork fs0ciety404/ps5-relapse, exploit layer only)
 │  kexp.js : kernel exploit (kexp_2026_05_25.bin)
 │          + ELF loader (elfldr-ps5-1360.elf) on port 9021
 │  site.js: AUTO_PAYLOADS → sends Payload Manager to localhost:9021
 ▼
Payload Manager (pldmgr)
 │  Automatic loading, in this order, with 3 sec. delays:
 ├─ 1. kstuff-lite      → kernel patches (required by ShadowMountPlus)
 ├─ 2. ShadowMountPlus  → automatic mounting of games and homebrew
 └─ 3. ftpsrv           → FTP server (port 2121)
```

Principle: **The fork does just one thing**: launch the exploit and then the Payload Manager. Everything else (choice of payloads, order, timing) is managed in the Payload Manager, via the console or the web manager.  
etaHEN is **deliberately absent** from the channel: see [etaHEN: status](#etahen--status).

---

## 🗒️ File inventory
### On the router: `/mnt/nvme0n1/docker/ps-hack/src/ps5/payloads/`

| File | Role | Version | sha256 |
|---|---|---|---|
| `kexp_2026_05_25.bin` | Kernel exploit (loaded by `kexp.js`) | provided by the fork | `7cfb3a8cb86db67893c360ae3531f460f800e9f24039797038e6e81fc50f18a9` |
| `elfldr-ps5-1360.elf` | ELF loader, port 9021 | provided by the fork | `de5dd480d12637527ba2d75e35de851adbb59c455293fa32588c492ef6b44b81` |
| `pldmgr_v0.5.2.elf` | Payload Manager (sent by `site.js`) | 0.5.2 | `62b3ba2a4937c2afc502f9a4e7242cca538610ebb4ae2800c7c6f72e7f268e7c` |

Only these three files should be in the folder.

### In Payload Manager (on the console)

| Order | Payload | Source | Version | sha256 |
|---|---|---|---|---|
| 1 | kstuff | EchoStretch/kstuff-lite | 1.11 Beta | ab9a6cb4d3b1daf139d4d646e402b1cf569071acd64599c936d7a3a6164dc779 |
| 2 | ShadowMountPlus | drakmor/ShadowMountPlus | 1.7beta5 | 0d54a8467ec3544e1c7b365c9f100b2bf42bf4633cb768bde7c8f16b6cfb5375 |
| 3 | ftpsrv | ps5-payload-dev/ftpsrv | v0.21.1 | 7d4b31c83eae4e056580482a3a074e1db25922efc74ac1e439473b45d71a5938 |

Calculating hashes on a Mac: `shasum -a 256 *.elf`.

---

## 🔧 Container-side configuration

`src/ps5/src/site.js`, line 51 :

```js
const AUTO_PAYLOADS = ["<pldmgr>.elf"];   // exact name of the file in payloads/
```

`src/ps5/src/kexp.js` (Do not change) :

```js
const DEFAULT_KEXP = "kexp_2026_05_25.bin";
const DEFAULT_ELFLDR = "elfldr-ps5-1360.elf";
```

Points to note:
- The name in `AUTO_PAYLOADS` must match the file in `payloads/` **exactly**. A missing file (404) will interrupt the loading process.
- Not all versions of Payload Manager necessarily involve sending the fork. A tested and validated version must be retained.
- During a rebuild or diagnostic check: `const AUTO_PAYLOADS = [];` (exploit only).
- No need to restart the container after making changes (volume read live). However, the PS5’s browser may still have the old `site.js` cached: check the logs to see what is actually being requested.

---

## 🔧 Console-side configuration

| Path | Content |
|---|---|
| `/data/pldmgr/` | Payload Manager configuration (list and order of automatic loading) |
| `/data/shadowmount/config.ini` | ShadowMountPlus configuration (created on first start-up) |
| `/data/shadowmount/debug.log` | ShadowMountPlus log |
| `/data/shadowmount/manual.lst` | List of game images to mount (one per line) |
| `/data/etaHEN.old/`, `/data/pldmgr.old/` | Old configurations, retained during reconstruction |

ShadowMountPlus : réglages par défaut conservés, en particulier :
- `api_bind_address=127.0.0.1`: the API (port 10101) only listens locally; **do not set this to `0.0.0.0`**;
- `mount_read_only=1`.

---

## 🆘 Full reset procedure
To be followed if the channel is no longer working or after a version update.
**Full restart** = hold down the power button until the device switches off, then switch it back on. Never use sleep mode.
### Step 0 — Preparation (console switched off)
1. Back up the existing data:
   ```sh
   cd ./ps-hack/src
   tar czf ~/ps5-backup-$(date +%F).tar.gz ps5
   ```
2. Download each payload from its official repository; note the version and SHA256.
3. In `site.js`: `const AUTO_PAYLOADS = [];`
4. In `payloads/`, keep only `kexp_2026_05_25.bin` and `elfldr-ps5-1360.elf`.

### Step 1 — Clean the console
5. Jailbreak (User Guide Menu).
6. Start ftpsrv: `nc -w 3 192.168.X.Y 9021 < ftpsrv-ps5.elf`
7. Via FTP (port 2121), rename the following in `/data/`: `pldmgr` → `pldmgr.old`, `etaHEN` → `etaHEN.old`, and any autoloader folders → `.old`. Remove any autoloader icons from the home screen.
8. 🔄 **Full restart.**

### Step 2 — Exploit only
9. Jailbreak. No payload notification should appear.
10. `nc -vz -G 2 192.168.1.80 9021` → **succeeded**.
11. Start ftpsrv, check FTP on port 2121 (backup access for later).

### Step 3 — Payload Manager
12. `nc -w 3 192.168.1.80 9021 < <pldmgr>.elf`
13. Find the port (using the notification or `nmap -p- -T4 --open 192.168.X.Y`), then open the interface.
14. Import kstuff-lite, ShadowMountPlus and ftpsrv, **without** enabling automatic loading.

### Step 4 — kstuff-lite
15. Launch from Payload Manager, wait for the notification, then wait for about ten seconds.

### Step 5 — ShadowMountPlus
16. Launch the app and wait for the ‘ShadowMount+’ notification.
17. Via FTP: the `/data/shadowmount/` directory exists, containing `config.ini` and `debug.log`.
  
> If the process fails at stage 4 or 5: make a note of the message, 🔄 **full restart**, jailbreak, manually repeat the steps that were successful, then try again.
> Never load the same payload twice in a single session.

### Step 6 — Reproducibility
18. 🔄 **Full restart.**
19. Repeat steps 2 to 5 manually. If everything works a second time, the sequence is reliable.

### Step 7 — Automation
20. In Payload Manager, enable automatic loading: **kstuff-lite → ShadowMountPlus → ftpsrv**, with a few (3) seconds between each.
21. Copy the validated pldmgr to `payloads/` and declare it in `site.js`:
    ```js
    const AUTO_PAYLOADS = [‘<pldmgr>.elf’];
    ```
22. 🔄 **Full reboot.**
23. Jailbreak; do not touch the console until the final notification appears, then run the [checks](#checks).

---

## ✳️ Day-to-day use
1. Switch on the PS5.
2. **Settings → User Guide, Health and Safety → User Guide.**
3. Accept the certificate, then do not touch anything until the final notification (ftpsrv) appears.
  
> **Rest mode usually preserves the jailbreak.**
> After a full restart, repeat the process.

---

## 🕹️ Games on an external SSD
### SSD format
**exFAT** with an **MBR** partition table (never use ‘Format as USB mass storage’ on the console).
```sh
diskutil list external
sudo diskutil eraseDisk ExFAT PS5 MBR diskN     # replace N
sudo mdutil -i off /Volumes/PS5                 # no Spotlight indexing
```
  
### Format des jeux
| Format | Statut ShadowMountPlus | Création sur Mac |
|---|---|---|
| `.ffpkg` (UFS) | Recommended by the author | No (FreeBSD or Windows UFS2Tool) |
| `.exfat` | Supported | **Yes**, official script |
| `.ffpfs` / `.ffpfsc` | Experimental | Yes (MkPFS) |
  
Selected option: **`.exfat`**, created using the macOS script from the ShadowMountPlus repository (64 Kio clusters):
```sh
chmod +x mkexfat_macos.sh
./mkexfat_macos.sh ./PPSAxxxxx-app ./PPSAxxxxx.exfat
```
  
> The source directory **must contain** `eboot.bin` and `sce_sys/param.json` **at its root**, with no intermediate directories.
  
### Configuration and setup
1. Copy `PPSAxxxxx.exfat` to the SSD, then eject it safely.
2. Add the following to `/data/shadowmount/manual.lst` (via FTP):
   ```
   /mnt/usb0/PPSAxxxxx.exfat
   ```
3. Connect the SSD to a 10 Gbit/s USB port (rear or front USB-C), without using a hub.
4. Wait for the installation notification and for the icon to appear on the home screen.
  
> ❗️Images on the external SSD only: the developer of ShadowMountPlus warns of a risk of corruption when images are stored on internal storage.

---

## Checks
From the Mac:
```sh
# Downloader and FTP
for p in 9021 2121; do nc -vz -G 2 192.168.X.Y $p; done
# All open ports on the console
nmap -p- -T4 --open 192.168.1.80
# Console contents
curl -s --list-only ftp://192.168.X.Y:2121/data/
```
  
Via FTP: `/data/shadowmount/debug.log` must not contain any errors on start-up.
  
On the router side, during a jailbreak:
```sh
docker logs -f 8230-ps-hack 2>&1 | grep -E ‘site.js|kexp.js|payloads/’
```
Each file in `payloads/` must be served with a `200` status code.

---

## ⚠️ Troubleshooting

| Symptom | Cause | Solution |
|---|---|---|
| The hack no longer loads after modifying `AUTO_PAYLOADS` | File missing or incorrectly named (404), pldmgr version incompatible with the fork submission, or old `site.js` cached | Check the container logs; reset to `[]`; revert to the validated pldmgr |
| `nc` on Mac exits without any message | macOS `nc` does not report rejections without `-v` | Always test with `nc -vz` |
| `nc: invalid option` on the router | Minimal BusyBox `nc` (neither `-w` nor `-v`) | `nc IP PORT < file` then `Ctrl+C`, or send from the Mac, or `opkg install netcat` |
| FTP refused on 1337 | etaHEN port, missing from the string | Use ftpsrv on **2121** |
| `/data/shadowmount/` missing | ShadowMountPlus never started (often: kstuff not loaded beforehand) | Check the order in Payload Manager |
| A game does not appear | Image structured incorrectly, incorrect path in `manual.lst`, copying in progress | Read `/data/shadowmount/debug.log` |
| Console unstable after several transfers | Payload loaded twice / inconsistent kernel state | Full reboot, repeat the sequence just once |

**Fallback method**: if the automatic chain fails, jailbreak the device and then run `nc -w 3 192.168.X.Y 9021 < ftpsrv-ps5.elf` from the Mac. ftpsrv runs independently of everything else.

---

## ⁉️ etaHEN : status
**Not used.** Tested on 13.20: etaHEN starts up (it detects firmware `0x1320` and activates its Toolbox), but its supervisor loops indefinitely:
```
runtime supervisor: launching embedded private elfldr via :9021
runtime supervisor: loader recovery failed (1) … (16)
```
Its dependent services (FTP 1337, ports 9027, 9081, 9090) never start.
Unverified hypothesis: conflict with the `elfldr-ps5-1360.elf` loader (name indicating version 13.60) already present on port 9021.
  
What etaHEN would provide: Toolbox (debugging settings, cheats, fan control), ‘on-demand jailbreak’ for certain homebrews, package installer.
What works without it: games via ShadowMountPlus, FTP via ftpsrv, most homebrews.
   
**To be reassessed:** a version of etaHEN announcing 13.20, or **OnionHEN** (a derivative of etaHEN, announced as supporting 4.03 to 13.60). To be added **last** in Payload Manager, after manual validation.

---

## 👮‍♀️ Security

Once jailbroken, the console exposes services on the LAN **without authentication**:

| Port | Service | Risk |
|---|---|---|
| 9021 | ELF Loader | Execution of kernel code by any device on the LAN |
| 2121 | ftpsrv | Read/write access to the entire file system |
| _variable_ | Payload Manager interface | Launching payloads |
| 10101 | ShadowMountPlus API | Local only (`127.0.0.1`), should be kept this way |
  
**Possible solutions:** isolate the PS5 on a dedicated VLAN, or filter these ports at router level so that they are only accessible from the Mac.
  
Reminders:
- Stay off the PSN whilst the console is jailbroken (already blocked by AdGuard).
- Payloads only from official repositories; check them before use; fingerprints listed above.
- ShadowMountPlus: for save files of games you own only.

---

## ☢️ Maintenance
- **Updating a payload**: download the new version, test it **manually** (by sending it via Payload Manager) before replacing the old one in the automatic upload; update the [Inventory](#file-inventory) table.
- **Add a payload**: always at the end of the sequence, after manual validation.
- **Update the fork**: only replace `kexp.js`, `kexp_*.bin` and `elfldr-*.elf` if necessary, then reset `AUTO_PAYLOADS`.
- **Clean-up**: once the chain has been stable for a while, delete `/data/etaHEN.old` and `/data/pldmgr.old` on the console.
