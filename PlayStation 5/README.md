# PS5 Hack: Architecture and components
## ⚒️ ps-hack
Local host for console web exploits, hosted on a Raspberry Pi 5, Docker ```192.168.1.2```. Accessible only from the LAN. **First target:** PS5 running firmware 13.20 with the Relapse exploit.
> [!CAUTION]
> **WARNING:** Never update PS5. Firmware 14.00 patches the vulnerability exploited by Relapse, and there is no way to downgrade.

---

## 🏛️ Architecture
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
      # HTTPS ends with NPM (proxy host ```manuals.playstation.net``` + ```ps-hack.<domain>.lan```)
    volumes:
      - ./docker/ps-hack/src:/usr/share/nginx/html:ro
      - ./docker/ps-hack/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
```
**Getting Started**
```sh
docker compose -f compose.yml up -d --force-recreate ps-hack
docker logs --tail 5 ps-hack   # should end with “start worker process”
```

---

## 🌍 Nginx Configuration

```nginx/default.conf```
```html
nginx
# Console routing based on the browser's User-Agent
map $http_user_agent $console_home {
    “~*PlayStation 5”     /ps5/;
    “~*PlayStation 4”     /ps4/;
    “~*PlayStation Vita”  /vita/;
    default               /menu.html;
}

server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Relative redirects (prevents nginx from rewriting to http:// behind NPM)
    absolute_redirect off;

    access_log /var/log/nginx/access.log;

    # Root and paths for the User Guide (/document/fr/ps5/... etc.)
    location = / {
        return 302 $console_home;
    }
    location /document/ {
        return 302 $console_home;
    }

    # Application Cache (offline mode for certain exploit hosts)
    location ~* \.(appcache|manifest)$ {
        types { }
        default_type text/cache-manifest;
        add_header Cache-Control “no-cache”;
    }

    # Binary payloads
    location ~* \. (elf|bin)$ {
        types { }
        default_type application/octet-stream;
    }

    autoindex off;
}
```
> [!NOTE]
> In a map block, quotation marks enclose the entire pattern, including the operator: “~*PlayStation 5”, not ~*“PlayStation 5” (otherwise: invalid number of map parameters).  
> ```absolute_redirect off``` is essential when used with **NPM**: without it, the redirect would point to http://… and the PS5 would leave the HTTPS connection.  
> The message ```10-listen-on-ipv6-by-default.sh: cannot modify … (read-only file system?)``` at startup is normal (mounted as ```:ro```).

---

## 🚦 DNS (AdGuard Home + dnsmasq)
AdGuard Home (```/etc/adguardhome.yaml```) listens on port **53** and serves the LAN. **dnsmasq** listens on port **54**.  
  
**In AdGuard Home**
DNS Rewrites (Filters → DNS Rewrites):
| Domain | Response |
| :--- | :--- |
| manuals.playstation.net | 192.168.1.2 |
| *.update.playstation.net | 192.168.1.2 (blocks updates) |
| (other Sony domains already present) | 192.168.1.2 |
  
Custom filtering rules already in place: blocking ```playstation.net```, ```playstation.com```, ```playstation.org```, and Sony’s ```Akamai``` domain. These rules block **PSN** for the entire LAN.  
To target only the PS5, add ```$client=<PS5_IP>``` to the rules.  
> [!NOTE]
> Edit AdGuard via its web interface, not by editing the YAML file directly (AdGuard rewrites the file).  

**In dnsmasq**
Local name for access from the LAN, following the same pattern as the other services:
```txt
dhcp.@cname[N].cname=‘ps-hack.<domain>.lan’
dhcp.@cname[N].target='<domain>.lan'
```
AdGuard must also resolve ```ps-hack.<domain>.lan``` to **192.168.1.2**.  
After making any changes: clear AdGuard’s cache (otherwise it will continue to return an old NXDOMAIN).

---

## 🚥 Nginx Proxy Manager
**Self-signed certificate**
Let's Encrypt is not available for these domain names. A 10-year certificate covering both domain names:
```sh
openssl req -x509 -newkey rsa:2048 -nodes -days 3650 \
  -keyout ps-hack.key -out ps-hack.crt \
  -subj “/CN=manuals.playstation.net” \
  -addext “subjectAltName=DNS:manuals.playstation.net,DNS:ps-hack.di-giacomo.lan”
```
**Import:** SSL Certificates → Add SSL Certificate → Custom (key + certificate, no intermediate certificate).  
  
**“LAN Only” Access List**
Access Lists → Add Access List:  
**Details:** name **LAN only**; **Satisfy Any** and **Pass Auth to Host** unchecked.
**Authorization:** blank.
> [!NOTE]
> The browser may pre-fill the username and password fields using autocomplete; clear them before saving, otherwise **NPM** will require Basic HTTP authentication (error 401), which the User Guide does not support.
**Access:** allow 192.168.1.0/24, then deny all.

### Proxy host
| Field | Value |
| :--- | :--- |
| Domain names | ```manuals.playstation.net```, ```ps-hack.<domain>.lan``` |
| Scheme | http |
| Forward Hostname / IP | 192.168.1.2 |
| Forward Port | 8240 |
| Access List | LAN only |
| SSL Certificate | custom certificat hereabow |
| Force SSL | no |
| HSTS | no |

---

## ⚒️ Relapse (PS5)
Fork used: ```fs0ciety404/ps5-relapse```
Why this fork and not the original Relapse: on 13.x, the ELF loader (port 9021) only accepts payloads locally.  
This fork sends the payloads to ```localhost:9021``` itself via the ROP chain, which eliminates the need to send them via ```nc``` from a PC.  
  
### Payloads chargés automatiquement
- **kstuff-lite** kernel patches ;
- **ShadowMountPlus** Setting Up Game Backups ;
- **etaHEN** Homebrew Enabler, Debug Settings, Toolbox.
  
**Installation:** Copy the contents of the repository (the folder containing ```index.html```) into ```src/ps5/```  

---

## 🚀 Réglages de la PS5
- **Firmware** 13.20
- Automatic updates disabled (Settings → System → System Software).
- **Network** Manual DNS, primary ```192.168.1.2```. Without this, the console may receive the DNS from the router (especially over IPv6) and bypass AdGuard.
- **PSN account** Stay offline as long as the console is jailbroken.
- **After changing the DNS** fully restart the console (do not use sleep mode), otherwise the old DNS will remain cached.

---

## ⚡ Usage
- Close all applications.
- Settings → User Guide, Health & Safety → User Guide.
- Accept the certificate warning.
- **Wait** the Relapse page will load, and then the payloads will load automatically (notifications).
  - If the device freezes or crashes: restart and try again; it may take several attempts.
> [!TIP]
> The jailbreak does not survive a full reboot: repeat the procedure after each startup.

---

# Tip and Tricks
### Checks
From the router or a PC on the LAN:
```sh
# Container reachable from NPM
docker exec nginx-proxy-manager curl -sI http://192.168.1.2:8240/
# → 302, Location: /menu.html

# Full chain via NPM
curl -vk --resolve manuals.playstation.net:443:192.168.1.2 https://manuals.playstation.net/
# → certificate CN=manuals.playstation.net, 302 redirect to /menu.html

# PS5 User-Agent redirection
curl -sk -o /dev/null -w ‘%{redirect_url}\n’ \
  -A "Mozilla/5.0 (PlayStation 5 13.20) AppleWebKit/605.1.15" \
  --resolve manuals.playstation.net:443:192.168.1.2 \
  https://manuals.playstation.net/document/fr/ps5/index.html
# → .../ps5/

# Relapse present
curl -sk -o /dev/null -w ‘%{http_code}\n’ \
  --resolve manuals.playstation.net:443:192.168.1.2 https://manuals.playstation.net/ps5/
# → 200

# DNS
nslookup manuals.playstation.net 192.168.1.2     # → 192.168.1.2
nslookup ps-hack.<domain>.lan 192.168.1.2      # → 192.168.1.2
```
### Follow a live attempt:
```sh
tail -f /mnt/nvme0n1/docker/npm/data/logs/proxy-host-54_access.log &
docker logs -f --since 1m ps-hack
```
### Troubleshooting
| Symptom | Cause | Solution |
| :--- | :--- | :--- |
| ```Error mounting … default.conf … not a directory``` | The file didn't exist on the first startup: Docker created a **folder** in its place | ```rm -rf nginx/default.conf```, copy the file back, restart |
| ```Invalid number of map parameters``` | Misplaced quotes in the ```map``` | ```~*PlayStation 5``` |
| **dnsmasq** rules have no effect | AdGuard is listening on port **53**; dnsmasq is behind it (port 54) | Configure the rewrites in AdGuard |
| ```curl``` stuck on ```Trying 172.18.0.230``` | The router is not the LAN gateway | Route through **NPM** on ```192.168.1.2``` |
| ```NXDOMAIN``` on ```ps-hack.di-giacomo.lan``` even though **dnsmasq** responds | AdGuard wasn’t forwarding / negative cache | Configure resolution in AdGuard, clear its cache |
| ```curl``` doesn’t resolve, but **nslookup``` does (macOS)	| System resolver cache or DNS cache on the primary router | ```sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder```, check ```scutil --dns``` |
| ```401``` Authorization Required | Username pre-filled by browser autocomplete in the access list | Clear the Authorization tab |
| ```502``` Bad Gateway, ```wrong version number``` | HTTPS scheme pointing to an HTTP container | Use HTTP scheme |
| ```MOZILLA_PKIX_ERROR_SELF_SIGNED_CERT``` | Self-signed certificate (expected) | Accept the exception, or import ```ps-hack.crt``` |
| Blank page in the User Guide | PS5 not yet on the new DNS	Full console reboot |
| Listening on port **9021** but no effect | Loader 13.x local only; etaHEN 2.5B predates 13.x | Fork ```fs0ciety404/ps5-relapse``` (autoload) |
### Maintenance
- **Update Relapse** Replace the contents of ```src/ps5/``` with the new version of the fork (after review). Nothing else needs to be changed.
- **Renew the certificate** valid through 2036.
### Security
- **No internet exposure** Port ```8240``` is bound only to ```192.168.1.2```; port 443 is not exposed by the container; LAN-only access list on the NPM proxy host (NPM can see the actual client IPs).
- **Local DNS resolution** No third-party DNS on the console, contrary to public tutorials.
  - Sony updates are blocked at the AdGuard level.
- **Third-party code** Review Relapse and payloads before each update; use only the original repositories.

