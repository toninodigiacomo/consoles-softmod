# PS5 Hack: Architecture and components
## ⚒️ ps-hack
Local host for console web exploits, hosted on a Raspberry Pi 5, Docker ```192.168.1.2```. Accessible only from the LAN. **First target:** PS5 running firmware 13.20 with the Relapse exploit.
> [!CAUTION]
> **WARNING:** Never update your PS5: Firmware 14.00 patches the vulnerability exploited by Relapse, and there is no way to downgrade.

---

## 🏛️Architecture
```txt
PS5 (Manual DNS 192.168.1.2)
 │  User Guide → https://manuals.playstation.net/document/fr/ps5/...
 ▼
AdGuard Home
 │  rewrite: manuals.playstation.net → 192.168.1.2
 ▼
Nginx Proxy Manager (192.168.1.2:443)
 │  self-signed certificate + “LAN only” access list
 │  forward http://ps-hack:80 (Docker network root_default)
 ▼
ps-hack (nginx:alpine)
 │  routing by User-Agent: PlayStation 5 → /ps5/
 ▼
Relapse (fork of fs0ciety404/ps5-relapse)
 │  WebKit + kernel exploit → elfldr on localhost:9021
 ▼
Automatic loading of payloads (kstuff, ShadowMountPlus, etaHEN)
```

### Key points
- The OpenWRT router (192.168.1.2) is not the LAN gateway (that’s the box at 192.168.1.1). Therefore, the container cannot be reached via its Docker IP from the LAN: everything goes through **NPM** at 192.168.1.2.
- TLS is handled by **NPM**. The container only serves HTTP on the Docker network.
- **AdGuard Home** listens on port 53 on the LAN, in front of dnsmasq (port 54).

---

## 🧰 Directory Structure
```./docker/8230-ps-hack/```
```txt
ps-hack/
├── README.md
├── nginx/
│   └── default.conf        # User-Agent routing, MIME types
├── certs/                  # self-signed certificate (imported into NPM, not in Git)
│   ├── ps-hack.crt
│   └── ps-hack.key
└── src/                    # web root (mounted as read-only)
    ├── menu.html           # fallback page (unrecognized browser)
    ├── ps5/                # Relapse (fork of fs0ciety404/ps5-relapse)
    ├── ps4/                # coming soon, maybe
    └── vita/               # coming soon, maybe
```
certs/*.key should not be versioned. Add to .gitignore:
```txt
certs/
```

---

## ⚒️ Container (ps-hack)
Basic common ```compose.yml``` container:
```yaml
services:
  ps-hack:
    container_name: ps-hack
    image: nginx:alpine
    ports:
      - 8240:80
      # HTTPS ends with NPM (proxy host manuals.playstation.net + ps-hack.<domain>.lan)
    volumes:
      - ./docker/ps-hack/src:/usr/share/nginx/html:ro
      - ./docker/ps-hack/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
```
**Getting Started**
```sh
docker compose -f compose.yml up -d --force-recreate ps-hack
docker logs --tail 5 ps-hack   # should end with “start worker process”
```
