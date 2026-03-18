# Pi-hole — Documentation technique
Mars 2026 | Raspberry Pi OS Trixie (Debian 13) | Rédigé par Meddy

---

## Sommaire

1. [Présentation](#1-présentation)
2. [Prérequis et préparation](#2-prérequis-et-préparation)
3. [Installation](#3-installation)
4. [Configuration réseau](#4-configuration-réseau)
5. [Administration](#5-administration)

---

## 1. Présentation

Pi-hole est un bloqueur de publicités et de trackers qui fonctionne au niveau DNS. Au lieu de bloquer les pubs dans le navigateur, il les bloque pour tous les appareils du réseau en interceptant les requêtes DNS vers des domaines connus pour servir de la publicité.

Quand un appareil demande `doubleclick.net`, Pi-hole répond `0.0.0.0` — la requête ne part jamais vers internet, la pub ne s'affiche pas.

**Matériel :** Raspberry Pi ARM64, 4 Go RAM
**OS :** Raspberry Pi OS Trixie (Debian 13)
**IP locale :** 192.168.0.162
**IP Tailscale :** 100.65.XXX.XXX

**Versions installées :**
- Pi-hole Core : v6.4
- Pi-hole Web : v6.4.1
- Pi-hole FTL : v6.5

---

## 2. Prérequis et préparation

### IP statique

Un serveur DNS doit avoir une IP fixe — si l'IP change, tous les appareils du réseau perdent leur DNS. On configure l'IP statique via NetworkManager sur l'interface eth0 (filaire, plus stable que le WiFi pour un serveur).

```bash
# Fixer l'IP statique sur eth0
sudo nmcli con mod "Wired connection 1" \
  ipv4.addresses 192.168.0.162/24 \
  ipv4.gateway 192.168.0.1 \
  ipv4.dns "127.0.0.1 1.1.1.1" \
  ipv4.method manual

sudo nmcli con down "Wired connection 1"
sudo nmcli con up "Wired connection 1"
```

### Problème dhcpcd vs NetworkManager

Pi-hole vérifie `/etc/dhcpcd.conf` pour détecter si l'IP est statique. Sur Trixie avec NetworkManager, ce fichier existe mais n'est pas utilisé — Pi-hole considère alors que l'IP n'est pas statique et bloque l'installation.

Solution : ajouter quand même la config dans dhcpcd.conf pour satisfaire le check de Pi-hole, même si c'est NetworkManager qui gère réellement le réseau.

```bash
sudo tee -a /etc/dhcpcd.conf << 'EOF'

interface eth0
static ip_address=192.168.0.162/24
static routers=192.168.0.1
static domain_name_servers=127.0.0.1 1.1.1.1
EOF
```

### Forcer IPv4

Le Pi a un problème de connectivité IPv6 — il tente IPv6 en premier et timeout avant de tomber sur IPv4. On force IPv4 globalement pour éviter des lenteurs ou échecs de téléchargement.

```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
echo '--ipv4' | sudo tee ~/.curlrc
```

---

## 3. Installation

```bash
curl -sSL https://install.pi-hole.net | bash
```

L'installateur est interactif. Choix effectués :

| Écran | Choix | Raison |
|-------|-------|--------|
| Interface réseau | eth0 | Filaire, plus stable |
| Upstream DNS | Cloudflare (1.1.1.1) | Rapide et respectueux de la vie privée |
| Block lists | Par défaut | Couvre les pubs et trackers courants |
| Web admin | On | Interface de gestion |
| lighttpd | On | Serveur web léger pour l'interface |
| Log queries | On | Historique des requêtes DNS |
| Privacy mode | Show everything | Visibilité complète pour l'admin |

### Changer le mot de passe admin

```bash
sudo pihole setpassword
```

### Import du schéma et mise à jour

Pi-hole installe et configure automatiquement sa base de données SQLite pour stocker l'historique des requêtes DNS. Aucune intervention manuelle nécessaire.

---

## 4. Configuration réseau

### Pi-hole comme DNS sur la box Orange

Pour que tous les appareils du réseau passent automatiquement par Pi-hole sans configuration individuelle, on change le DNS dans la box :

- **DNS primaire** → `192.168.0.162`
- **DNS secondaire** → laisser vide ou mettre le DNS Orange par défaut

> Mettre un DNS secondaire permet un fallback si le Pi est down, mais certains appareils peuvent bypasser Pi-hole et aller directement sur le secondaire. Sans secondaire, si le Pi plante, plus de résolution DNS sur tout le réseau. À choisir selon le niveau de résilience souhaité.

### Désactiver le DNS Tailscale

Tailscale injecte son propre DNS (`100.100.100.100`) et prend la priorité sur la configuration NetworkManager. Sans cette étape, le Pi lui-même n'utilise pas Pi-hole comme DNS.

```bash
sudo tailscale set --accept-dns=false
```

Après cette commande, `dig google.com | grep SERVER` doit retourner `127.0.0.1` — le Pi résout via son propre Pi-hole.

### Forcer le Pi à utiliser son propre DNS

```bash
sudo nmcli con mod "Wired connection 1" ipv4.dns "127.0.0.1"
sudo nmcli con down "Wired connection 1" && sudo nmcli con up "Wired connection 1"
```

### UFW

```bash
sudo ufw enable
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.168.0.0/24
sudo ufw allow in on tailscale0
```

---

## 5. Administration

### Accès

| Interface | URL |
|-----------|-----|
| Web admin | `http://192.168.0.162/admin` |
| Via Tailscale | `http://100.65.XXX.XXX/admin` |

### Vérifier que Pi-hole fonctionne

```bash
# Statut du service
sudo systemctl status pihole-FTL --no-pager | grep "Active:"

# Tester la résolution DNS
dig google.com @192.168.0.162

# Tester le blocage pub (doit retourner 0.0.0.0)
dig doubleclick.net @192.168.0.162
```

### Mettre à jour Pi-hole

```bash
sudo pihole updatePihole
```

### Voir les stats en ligne de commande

```bash
sudo pihole status
sudo pihole -c        # chronometer (stats temps réel)
```

### Ajouter un domaine à la blacklist manuellement

```bash
sudo pihole blacklist domaine.com
```

### Débloquer un domaine (whitelist)

```bash
sudo pihole whitelist domaine.com
```

### Mettre à jour les listes de blocage

```bash
sudo pihole updateGravity
```

### Logs DNS en temps réel

```bash
sudo pihole tail
```

---

## Notes sur la cohabitation avec Tailscale

Tailscale et Pi-hole peuvent entrer en conflit sur la gestion du DNS. Deux points importants :

**DNS Tailscale désactivé** (`--accept-dns=false`) — Tailscale n'écrase plus la config DNS du système. Le Pi utilise Pi-hole pour toutes ses résolutions DNS, y compris pour les connexions qui transitent par Tailscale.

**Accès à l'interface web via Tailscale** — L'interface Pi-hole reste accessible depuis n'importe où via l'IP Tailscale `100.65.XXX.XXX/admin`, ce qui permet d'administrer Pi-hole à distance sans ouvrir de port sur la box.

---

*Pi-hole v6.4 — Mars 2026*
