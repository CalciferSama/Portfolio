# Borgmatic — Documentation technique
Mars 2026 | Raspberry Pi OS Trixie (Debian 13) | Rédigé par Meddy

---

## Sommaire

1. [Présentation](#1-présentation)
2. [Architecture](#2-architecture)
3. [Installation serveur (Raspberry Pi)](#3-installation-serveur-raspberry-pi)
4. [Configuration serveur](#4-configuration-serveur)
5. [Installation client (machine à sauvegarder)](#5-installation-client-machine-à-sauvegarder)
6. [Administration](#6-administration)

---

## 1. Présentation

Borgmatic est un wrapper autour de BorgBackup qui simplifie la configuration et l'automatisation des sauvegardes. BorgBackup est un outil de sauvegarde avec déduplication, compression et chiffrement — concrètement, il ne stocke qu'une seule fois les données identiques entre plusieurs sauvegardes, ce qui économise énormément d'espace.

Dans cette configuration, le Raspberry Pi joue le rôle de **serveur de backup** : il reçoit et stocke les sauvegardes. Les machines du réseau se connectent en SSH pour y déposer leurs archives.

**Versions installées :**
- BorgBackup : 1.4.0
- Borgmatic : 2.1.3

---

## 2. Architecture

```
Machine client (Linux)              Raspberry Pi (serveur backup)
┌─────────────────────┐             ┌─────────────────────────┐
│                     │             │                          │
│  Borgmatic          │──SSH/Borg──▶│  /backup/borg            │
│  /etc/borgmatic/    │             │  (dépôt Borg chiffré)    │
│  config.yaml        │             │                          │
│                     │             │  Cron : 6h + 18h         │
└─────────────────────┘             └─────────────────────────┘
```

Ce qui est sauvegardé depuis le Pi lui-même :
- `/home` — dossiers utilisateurs
- `/etc/pihole` — configuration Pi-hole

Ce qui peut être sauvegardé depuis une machine client :
- Dossiers personnalisés
- Bases de données (dump automatique avant backup)

---

## 3. Installation serveur (Raspberry Pi)

### Dépendances

```bash
sudo apt install -y borgbackup
sudo pip install borgmatic --break-system-packages
```

> Le warning pip sur l'utilisation en root est normal et sans conséquence ici.

### Initialisation du dépôt

```bash
# Création du dossier de stockage
sudo mkdir -p /backup/borg

# Initialisation avec chiffrement repokey
# La clé est stockée dans le dépôt lui-même, protégée par la passphrase
sudo borg init --encryption=repokey /backup/borg
```

Le mode `repokey` stocke la clé de chiffrement dans le dépôt. Cela signifie que la passphrase seule suffit pour déchiffrer — mais si le dépôt est perdu, les données sont irrécupérables. Il faut donc exporter la clé :

```bash
# Export de la clé de chiffrement (à conserver précieusement ailleurs)
sudo borg key export /backup/borg /root/borg-key-backup.txt
```

### Accès SSH pour les clients

Pour qu'une machine distante puisse sauvegarder vers le Pi, il faut lui donner accès en SSH :

```bash
# Sur le Pi : créer un utilisateur dédié aux backups
sudo useradd -m -s /bin/bash borgbackup
sudo mkdir -p /home/borgbackup/.ssh
sudo chmod 700 /home/borgbackup/.ssh

# Créer un dépôt dédié pour la machine cliente
sudo mkdir -p /backup/borg-clients/machine1
sudo borg init --encryption=repokey /backup/borg-clients/machine1

# Donner les droits à l'utilisateur borgbackup
sudo chown -R borgbackup:borgbackup /backup/borg-clients
```

---

## 4. Configuration serveur

### Fichier de configuration Borgmatic

`/etc/borgmatic/config.yaml`

```yaml
repositories:
    - path: /backup/borg
      label: local

source_directories:
    - /home
    - /etc/pihole

commands:
    - before: action
      when:
          - create
      run:
          - echo "Début backup $(date)"

    - after: action
      when:
          - create
      states:
          - finish
      run:
          - echo "Backup terminé $(date)"

    - after: action
      states:
          - fail
      run:
          - echo "ERREUR backup $(date)"

retention:
    keep_within: 7d    # Tout garder pendant les 7 premiers jours
    keep_daily: 30     # 1 backup par jour entre 7 et 30 jours
    keep_monthly: 6    # 1 backup par mois entre 30 jours et 6 mois

compression: lz4       # Compression rapide, bon compromis vitesse/taille

encryption_passphrase: "changeme"

checks:
    - name: repository
      frequency: 2 weeks
    - name: archives
      frequency: 2 weeks
```

### Politique de rétention dégressive

La rétention est dégressive — plus une archive est ancienne, moins on en garde. Ça permet d'avoir une granularité fine sur le court terme tout en limitant l'espace utilisé sur le long terme.

| Période | Ce qu'on garde | Pourquoi |
|---------|----------------|---------|
| **0 → 7 jours** | Tous les backups (2x/jour = 14 archives) | Granularité maximale, erreurs récentes facilement récupérables |
| **7 → 30 jours** | 1 backup par jour | On peut revenir jusqu'à un mois en arrière |
| **30 jours → 6 mois** | 1 backup par mois | Filet de sécurité longue durée |
| **après 6 mois** | Supprimé automatiquement | Libère l'espace disque |

Les archives hors politique sont supprimées automatiquement à chaque exécution de borgmatic (action `prune`).

### Compression

`lz4` est choisi pour sa rapidité. Sur un Pi, la compression CPU est un facteur limitant. Les alternatives :

| Algorithme | Vitesse | Taux de compression |
|------------|---------|---------------------|
| lz4 | Très rapide | Moyen |
| zstd | Rapide | Bon |
| zlib | Moyen | Bon |
| lzma | Lent | Excellent |

### Cron — automatisation 2x par jour

`/etc/cron.d/borgmatic`

```
0 6,18 * * * root /usr/local/bin/borgmatic --verbosity 0 --syslog-verbosity 1
```

Le backup tourne à 6h et 18h tous les jours. `--syslog-verbosity 1` envoie les résultats dans syslog, consultable avec `journalctl`.

---

## 5. Installation client (machine à sauvegarder)

### Prérequis

La machine cliente doit pouvoir se connecter au Pi en SSH sans mot de passe (authentification par clé).

```bash
# Sur la machine cliente : générer une clé SSH si pas déjà fait
ssh-keygen -t ed25519 -C "borgmatic-client"

# Copier la clé publique vers le Pi
ssh-copy-id borgbackup@192.168.0.162

# Tester la connexion
ssh borgbackup@192.168.0.162
```

### Installation Borgmatic sur le client

```bash
sudo apt install -y borgbackup
sudo pip install borgmatic --break-system-packages
```

### Initialisation du dépôt distant

```bash
# Initialiser le dépôt sur le Pi depuis la machine cliente
borg init --encryption=repokey \
  ssh://borgbackup@192.168.0.162/backup/borg-clients/machine1
```

### Configuration Borgmatic client

`/etc/borgmatic/config.yaml`

```yaml
repositories:
    - path: ssh://borgbackup@192.168.0.162/backup/borg-clients/machine1
      label: raspserver

# Adapter les dossiers selon ce qu'on veut sauvegarder
source_directories:
    - /home
    - /var/www
    - /opt/monapp

# Dump automatique des bases de données avant backup
mariadb_databases:
    - name: all
      username: root
      password: VOTRE_MOT_DE_PASSE_DB

# Exclusions classiques
exclude_patterns:
    - /home/*/.cache
    - /home/*/.local/share/Trash
    - '*.tmp'
    - '*.log'

commands:
    - before: action
      when:
          - create
      run:
          - echo "Début backup client $(date)"

    - after: action
      when:
          - create
      states:
          - finish
      run:
          - echo "Backup client terminé $(date)"

    - after: action
      states:
          - fail
      run:
          - echo "ERREUR backup client $(date)"

retention:
    keep_within: 7d    # Tout garder pendant les 7 premiers jours
    keep_daily: 30     # 1 backup par jour entre 7 et 30 jours
    keep_monthly: 6    # 1 backup par mois entre 30 jours et 6 mois

compression: lz4

encryption_passphrase: "changeme"

checks:
    - name: repository
      frequency: 2 weeks
    - name: archives
      frequency: 2 weeks
```

### Cron client

```bash
sudo tee /etc/cron.d/borgmatic << 'EOF'
0 6,18 * * * root /usr/local/bin/borgmatic --verbosity 0 --syslog-verbosity 1
EOF
```

---

## 6. Administration

### Commandes courantes

```bash
# Lancer un backup manuellement
sudo borgmatic create --verbosity 1

# Lister toutes les archives
sudo borgmatic list --match-archives "*"

# Infos sur l'espace utilisé
sudo borgmatic info --match-archives "*"

# Vérifier l'intégrité du dépôt
sudo borgmatic check

# Voir les logs dans syslog
sudo journalctl -u cron | grep borgmatic
```

### Restaurer un fichier

```bash
# Monter le dépôt pour naviguer dedans
sudo borgmatic mount --mount-point /mnt/restore

# Naviguer et copier ce dont on a besoin
ls /mnt/restore
cp /mnt/restore/home/meddy/fichier.txt /home/meddy/fichier-restauré.txt

# Démonter
sudo borgmatic umount --mount-point /mnt/restore
```

### Restaurer une archive complète

```bash
# Lister les archives disponibles
sudo borgmatic list --match-archives "*"

# Restaurer une archive spécifique dans /tmp/restore
sudo borgmatic restore \
  --archive raspserver-2026-03-18T06:00:00 \
  --destination /tmp/restore
```

### En cas de backup interrompu

Si un backup est interrompu (Ctrl+C, coupure réseau...), Borg laisse un verrou qui bloque les backups suivants :

```bash
sudo borg break-lock /backup/borg
```

### Vérifier l'espace utilisé sur la SD card

```bash
df -h /backup
sudo borgmatic info --match-archives "*"
```

---

*Borgmatic v2.1.3 — BorgBackup v1.4.0 — Mars 2026*
