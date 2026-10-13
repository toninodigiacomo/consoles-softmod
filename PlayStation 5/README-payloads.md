# 8230-ps-hack — Chaîne de payloads PS5

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

## 🔧 Configuration côté console

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

## Full reset procedure
To be followed if the channel is no longer working or after a version update.
**Full restart** = hold down the power button until the device switches off, then switch it back on. Never use sleep mode.
### Phase 0 — Preparation (console switched off)
1. Back up the existing data on the router:
   ```sh
   cd /mnt/nvme0n1/docker/ps-hack/src
   tar czf ~/ps5-backup-$(date +%F).tar.gz ps5
   ```
2. Download each payload from its official repository; note the version and SHA256.
3. In `site.js`: `const AUTO_PAYLOADS = [];`
4. In `payloads/`, keep only `kexp_2026_05_25.bin` and `elfldr-ps5-1360.elf`.

### Phase 1 — Nettoyer la console

5. Jailbreak (Guide utilisateur).
6. Envoyer ftpsrv : `nc -w 3 192.168.1.80 9021 < ftpsrv-ps5.elf`
7. Par FTP (port 2121), renommer dans `/data/` : `pldmgr` → `pldmgr.old`, `etaHEN` → `etaHEN.old`, et tout dossier d'autoloader → `.old`. Supprimer toute icône d'autoloader de l'écran d'accueil.
8. 🔄 **Redémarrage complet.**

### Phase 2 — Exploit seul

9. Jailbreak. Aucune notification de payload ne doit apparaître.
10. `nc -vz -G 2 192.168.1.80 9021` → **succeeded**.
11. Envoyer ftpsrv, vérifier le FTP sur 2121 (accès de secours pour la suite).

### Phase 3 — Payload Manager

12. `nc -w 3 192.168.1.80 9021 < <pldmgr>.elf`
13. Trouver son port (notification ou `nmap -p- -T4 --open 192.168.1.80`), ouvrir son interface.
14. Importer kstuff-lite, ShadowMountPlus et ftpsrv, **sans** activer le chargement automatique.

### Phase 4 — kstuff-lite

15. Lancer depuis Payload Manager, attendre la notification, puis une dizaine de secondes.

### Phase 5 — ShadowMountPlus

16. Lancer, attendre la notification « ShadowMount+ ».
17. Par FTP : `/data/shadowmount/` existe, avec `config.ini` et `debug.log`.

> En cas d'échec en phase 4 ou 5 : noter le message, 🔄 **redémarrage complet**, jailbreak, refaire à la main les étapes réussies, puis retenter. Ne jamais charger deux fois le même payload dans une session.

### Phase 6 — Reproductibilité

18. 🔄 **Redémarrage complet.**
19. Refaire les phases 2 à 5 à la main. Si tout passe une seconde fois, la séquence est fiable.

### Phase 7 — Automatisation

20. Dans Payload Manager, activer le chargement automatique : **kstuff-lite → ShadowMountPlus → ftpsrv**, avec quelques secondes entre chaque.
21. Copier le pldmgr validé dans `payloads/` et le déclarer dans `site.js` :
    ```js
    const AUTO_PAYLOADS = ["<pldmgr>.elf"];
    ```
22. 🔄 **Redémarrage complet.**
23. Jailbreak, ne plus toucher à la console jusqu'à la dernière notification, puis lancer les [vérifications](#vérifications).

---

## Utilisation quotidienne

1. Allumer la PS5.
2. **Paramètres → Guide de l'utilisateur, santé et sécurité → Guide de l'utilisateur.**
3. Accepter le certificat, puis ne plus toucher à rien jusqu'à la dernière notification (ftpsrv).

Le mode repos conserve généralement le jailbreak. Après un redémarrage complet, recommencer.

---

## Jeux sur SSD externe

### Format du SSD

**exFAT** avec table de partition **MBR** (ne jamais utiliser « Formater comme stockage étendu USB » sur la console).

```sh
diskutil list external
sudo diskutil eraseDisk ExFAT PS5 MBR diskN     # remplacer N
sudo mdutil -i off /Volumes/PS5                 # pas d'indexation Spotlight
```

### Format des jeux

| Format | Statut ShadowMountPlus | Création sur Mac |
|---|---|---|
| `.ffpkg` (UFS) | Recommandé par l'auteur | Non (FreeBSD ou Windows UFS2Tool) |
| `.exfat` | Supporté | **Oui**, script officiel |
| `.ffpfs` / `.ffpfsc` | Expérimental | Oui (MkPFS) |

Choix retenu : **`.exfat`**, créé avec le script macOS du dépôt ShadowMountPlus (clusters de 64 Kio) :

```sh
chmod +x mkexfat_macos.sh
./mkexfat_macos.sh ./PPSAxxxxx-app ./PPSAxxxxx.exfat
```

Le dossier source doit contenir `eboot.bin` et `sce_sys/param.json` **à sa racine**, sans dossier intermédiaire.

### Déclaration et montage

1. Copier `PPSAxxxxx.exfat` sur le SSD, éjecter proprement.
2. Ajouter dans `/data/shadowmount/manual.lst` (via FTP) :
   ```
   /mnt/usb0/PPSAxxxxx.exfat
   ```
3. Brancher le SSD sur un port USB 10 Gbit/s (arrière ou USB-C avant), sans hub.
4. Attendre la notification d'installation et l'icône sur l'écran d'accueil.

Images uniquement sur le SSD externe : l'auteur de ShadowMountPlus signale des risques de corruption avec des images sur le stockage interne.

---

## Vérifications

Depuis le Mac :

```sh
# Chargeur et FTP
for p in 9021 2121; do nc -vz -G 2 192.168.1.80 $p; done

# Tous les ports ouverts sur la console
nmap -p- -T4 --open 192.168.1.80

# Contenu de la console
curl -s --list-only ftp://192.168.1.80:2121/data/
```

Par FTP : `/data/shadowmount/debug.log` ne doit pas contenir d'erreur au démarrage.

Côté routeur, pendant un jailbreak :

```sh
docker logs -f 8230-ps-hack 2>&1 | grep -E "site.js|kexp.js|payloads/"
```

Chaque fichier de `payloads/` doit être servi en `200`.

---

## Dépannage

| Symptôme | Cause | Correction |
|---|---|---|
| Le hack ne se charge plus après modification de `AUTO_PAYLOADS` | Fichier absent ou mal nommé (404), version de pldmgr incompatible avec l'envoi du fork, ou ancien `site.js` en cache | Vérifier les logs du conteneur ; revenir à `[]` ; reprendre le pldmgr validé |
| `nc` sur Mac se termine sans rien dire | `nc` macOS ne signale pas les refus sans `-v` | Toujours tester avec `nc -vz` |
| `nc: invalid option` sur le routeur | `nc` BusyBox minimal (ni `-w` ni `-v`) | `nc IP PORT < fichier` puis `Ctrl+C`, ou envoyer depuis le Mac, ou `opkg install netcat` |
| FTP refusé sur 1337 | Port d'etaHEN, absent de la chaîne | Utiliser ftpsrv sur **2121** |
| `/data/shadowmount/` absent | ShadowMountPlus jamais démarré (souvent : kstuff non chargé avant) | Vérifier l'ordre dans Payload Manager |
| Un jeu n'apparaît pas | Image mal structurée, chemin incorrect dans `manual.lst`, copie en cours | Lire `/data/shadowmount/debug.log` |
| Console instable après plusieurs envois | Payload chargé deux fois / état noyau incohérent | Redémarrage complet, rejouer la séquence une seule fois |

**Accès de secours** : si la chaîne automatique échoue, jailbreak puis `nc -w 3 192.168.1.80 9021 < ftpsrv-ps5.elf` depuis le Mac. ftpsrv fonctionne indépendamment de tout le reste.

---

## etaHEN : statut

**Non utilisé.** Testé sur la 13.20, etaHEN démarre (il détecte le firmware `0x1320` et active sa Toolbox), mais son superviseur boucle indéfiniment :

```
runtime supervisor: launching embedded private elfldr via :9021
runtime supervisor: loader recovery failed (1) … (16)
```

Ses services dépendants (FTP 1337, ports 9027, 9081, 9090) ne démarrent jamais.
Hypothèse non vérifiée : conflit avec le chargeur `elfldr-ps5-1360.elf` (nom indiquant une version 13.60) déjà présent sur le port 9021.

Ce qu'etaHEN apporterait : Toolbox (réglages de débogage, cheats, ventilateur), « jailbreak à la demande » pour certains homebrews, installeur de paquets.
Ce qui fonctionne sans lui : jeux via ShadowMountPlus, FTP via ftpsrv, la plupart des homebrews.

À réévaluer : une version d'etaHEN annonçant la 13.20, ou **OnionHEN** (dérivé d'etaHEN, annoncé 4.03 à 13.60). À ajouter **en dernier** dans Payload Manager, après validation manuelle.

---

## Sécurité

Une fois jailbreakée, la console expose sur le LAN des services **sans authentification** :

| Port | Service | Risque |
|---|---|---|
| 9021 | Chargeur d'ELF | Exécution de code noyau par n'importe quel appareil du LAN |
| 2121 | ftpsrv | Lecture/écriture de tout le système de fichiers |
| _variable_ | Interface Payload Manager | Lancement de payloads |
| 10101 | API ShadowMountPlus | Local uniquement (`127.0.0.1`), à conserver ainsi |

Pistes : isoler la PS5 sur un VLAN dédié, ou filtrer ces ports au niveau du routeur pour ne les ouvrir qu'au Mac.

Rappels :

- Rester hors du PSN tant que la console est jailbreakée (déjà bloqué par AdGuard).
- Payloads uniquement depuis les dépôts d'origine, relus avant usage, empreintes notées ci-dessus.
- ShadowMountPlus : uniquement pour des sauvegardes de jeux possédés.

---

## Maintenance

- **Mettre à jour un payload** : télécharger la nouvelle version, la tester **à la main** (envoi par Payload Manager) avant de remplacer l'ancienne dans le chargement automatique ; mettre à jour le tableau [Inventaire](#inventaire-des-fichiers).
- **Ajouter un payload** : toujours en fin de séquence, après validation manuelle.
- **Mettre à jour le fork** : ne remplacer que `kexp.js`, `kexp_*.bin` et `elfldr-*.elf` si nécessaire, puis remettre `AUTO_PAYLOADS`.
- **Nettoyage** : une fois la chaîne stable depuis un moment, supprimer `/data/etaHEN.old` et `/data/pldmgr.old` sur la console.
