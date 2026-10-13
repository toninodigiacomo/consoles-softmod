# 8230-ps-hack — Chaîne de payloads PS5

Ce document couvre **ce qui se passe après l'exploit** : quels payloads sont chargés sur la PS5, dans quel ordre, par qui, et comment les maintenir.
L'hébergement de l'exploit (Docker, nginx, NPM, DNS) est décrit dans le [`README.md`](README.md) principal.

> Console : PS5 en firmware **13.20**, IP fixe **192.168.1.80**.
> ⚠️ Ne jamais mettre à jour la console (le 14.00 corrige la faille).

---

## Sommaire

1. [Architecture](#architecture)
2. [Inventaire des fichiers](#inventaire-des-fichiers)
3. [Configuration côté conteneur](#configuration-côté-conteneur)
4. [Configuration côté console](#configuration-côté-console)
5. [Procédure de reconstruction complète](#procédure-de-reconstruction-complète)
6. [Utilisation quotidienne](#utilisation-quotidienne)
7. [Jeux sur SSD externe](#jeux-sur-ssd-externe)
8. [Vérifications](#vérifications)
9. [Dépannage](#dépannage)
10. [etaHEN : statut](#etahen--statut)
11. [Sécurité](#sécurité)
12. [Maintenance](#maintenance)

---

## Architecture

```
Guide utilisateur PS5
 │
 ▼
Relapse (fork fs0ciety404/ps5-relapse, couche exploit uniquement)
 │  kexp.js : exploit noyau (kexp_2026_05_25.bin)
 │          + chargeur d'ELF (elfldr-ps5-1360.elf) sur le port 9021
 │  site.js : AUTO_PAYLOADS → envoie Payload Manager à localhost:9021
 ▼
Payload Manager (pldmgr)
 │  chargement automatique, dans cet ordre, avec délais :
 ├─ 1. kstuff-lite      → patchs noyau (requis par ShadowMountPlus)
 ├─ 2. ShadowMountPlus  → montage automatique des jeux et homebrews
 └─ 3. ftpsrv           → serveur FTP (port 2121)
```

Principe : **le fork ne fait qu'une chose**, lancer l'exploit puis Payload Manager. Tout le reste (choix des payloads, ordre, délais) se gère dans Payload Manager, sur la console.

etaHEN est **volontairement absent** de la chaîne : voir [etaHEN : statut](#etahen--statut).

---

## Inventaire des fichiers

### Sur le routeur : `/mnt/nvme0n1/docker/ps-hack/src/ps5/payloads/`

| Fichier | Rôle | Version | sha256 |
|---|---|---|---|
| `kexp_2026_05_25.bin` | Exploit noyau (chargé par `kexp.js`) | fournie par le fork | `7cfb3a8cb86db67893c360ae3531f460f800e9f24039797038e6e81fc50f18a9` |
| `elfldr-ps5-1360.elf` | Chargeur d'ELF, port 9021 | fournie par le fork | `de5dd480d12637527ba2d75e35de851adbb59c455293fa32588c492ef6b44b81` |
| `<pldmgr>.elf` | Payload Manager (envoyé par `site.js`) | _à compléter_ | _à compléter_ |

Seuls ces trois fichiers doivent se trouver dans le dossier.

### Dans Payload Manager (sur la console)

| Ordre | Payload | Source | Version | sha256 |
|---|---|---|---|---|
| 1 | kstuff-lite | EchoStretch/kstuff-lite | _à compléter_ | _à compléter_ |
| 2 | ShadowMountPlus | drakmor/ShadowMountPlus | _à compléter_ | _à compléter_ |
| 3 | ftpsrv | ps5-payload-dev/ftpsrv | v0.21.1 | _à compléter_ |

Calcul des empreintes sur le Mac : `shasum -a 256 *.elf`.

> Garder une copie de chaque fichier retenu (par exemple dans `payloads-archive/` sur le routeur, hors du dossier servi) pour pouvoir revenir en arrière.

---

## Configuration côté conteneur

`src/ps5/src/site.js`, ligne 51 :

```js
const AUTO_PAYLOADS = ["<pldmgr>.elf"];   // nom exact du fichier présent dans payloads/
```

`src/ps5/src/kexp.js` (ne pas modifier) :

```js
const DEFAULT_KEXP = "kexp_2026_05_25.bin";
const DEFAULT_ELFLDR = "elfldr-ps5-1360.elf";
```

Points d'attention :

- Le nom dans `AUTO_PAYLOADS` doit correspondre **exactement** au fichier dans `payloads/`. Un fichier manquant (404) interrompt le chargement.
- Toutes les versions de Payload Manager ne passent pas forcément par l'envoi du fork. Une version testée et validée doit être conservée.
- Pendant une reconstruction ou un diagnostic : `const AUTO_PAYLOADS = [];` (exploit seul).
- Pas de redémarrage du conteneur nécessaire après modification (volume lu en direct). En revanche, le navigateur de la PS5 peut garder l'ancien `site.js` en cache : vérifier dans les logs ce qui est réellement demandé.

---

## Configuration côté console

| Chemin | Contenu |
|---|---|
| `/data/pldmgr/` | Configuration de Payload Manager (liste et ordre de chargement automatique) |
| `/data/shadowmount/config.ini` | Configuration de ShadowMountPlus (créée au premier démarrage) |
| `/data/shadowmount/debug.log` | Journal de ShadowMountPlus |
| `/data/shadowmount/manual.lst` | Liste des images de jeux à monter (une par ligne) |
| `/data/etaHEN.old/`, `/data/pldmgr.old/` | Anciennes configurations, conservées pendant la reconstruction |

ShadowMountPlus : réglages par défaut conservés, en particulier :

- `api_bind_address=127.0.0.1` : l'API (port 10101) n'écoute qu'en local, **ne pas passer en `0.0.0.0`** ;
- `mount_read_only=1`.

---

## Procédure de reconstruction complète

À suivre si la chaîne ne fonctionne plus ou après un changement de version.
**Redémarrage complet** = maintenir le bouton d'alimentation jusqu'à l'extinction, puis rallumer. Jamais le mode repos.

### Phase 0 — Préparation (console éteinte)

1. Archiver l'existant sur le routeur :
   ```sh
   cd /mnt/nvme0n1/docker/ps-hack/src
   tar czf ~/ps5-backup-$(date +%F).tar.gz ps5
   ```
2. Télécharger chaque payload depuis son dépôt officiel, noter version et sha256.
3. Dans `site.js` : `const AUTO_PAYLOADS = [];`
4. Dans `payloads/`, ne garder que `kexp_2026_05_25.bin` et `elfldr-ps5-1360.elf`.

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
