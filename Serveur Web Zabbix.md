# RaspServer — Documentation technique
Mars 2026 | Raspberry Pi OS Trixie (Debian 13) | Rédigé par Meddy

---

## Sommaire

1. [Présentation du projet](#1-présentation-du-projet)
2. [Système de base](#2-système-de-base)
3. [Apache](#3-apache)
4. [PHP et MariaDB](#4-php-et-mariadb)
5. [SSL](#5-ssl)
6. [ModSecurity](#6-modsecurity)
7. [Zabbix](#7-zabbix)
8. [Fail2ban et UFW](#8-fail2ban-et-ufw)
9. [Tuning 24/7](#9-tuning-247)

---

## 1. Présentation du projet

L'objectif est de transformer un Raspberry Pi en serveur web complet, capable de tourner 24h/24 de façon stable et sécurisée. Le Pi est uniquement accessible sur le réseau local (pas de nom de domaine public), ce qui implique d'utiliser un certificat SSL auto-signé plutôt que Let's Encrypt.

**Matériel :** Raspberry Pi ARM64, 4 Go RAM
**OS :** Raspberry Pi OS Trixie, basé sur Debian 13
**IP locale :** 192.168.0.162
**IP Tailscale :** 100.65.247.110 (accès distant via VPN)

La stack choisie est classique pour ce type de serveur : Apache comme serveur web, PHP-FPM pour exécuter le code PHP, MariaDB comme base de données, et Zabbix pour surveiller l'état du serveur. ModSecurity et Fail2ban sont là pour la sécurité applicative et réseau.

---

## 2. Système de base

### Problème IPv6

Dès le début de l'installation, on a constaté que les téléchargements depuis internet échouaient ou étaient très lents. Le Pi tentait de se connecter en IPv6 en premier, ce qui timeout systématiquement, avant de finalement tomber sur IPv4. Pour éviter ce problème sur toute la durée de l'installation (et en production), on force IPv4 globalement :

```bash
# Pour apt
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4

# Pour curl et wget
echo '--ipv4' | sudo tee ~/.curlrc
```

Ces deux fichiers font en sorte que tous les outils systèmes utilisent IPv4 par défaut, sans avoir à le préciser à chaque commande.

### Swap

Le Pi a 4 Go de RAM, ce qui est suffisant pour la stack complète. On a quand même configuré 2 Go de swap. Le swap sur un Pi est sur la carte SD, donc on configure `vm.swappiness=10` pour que le système l'utilise le moins possible et préserve la SD card. Le swap reste là comme filet de sécurité si Zabbix ou MariaDB consomment plus que prévu.

---

## 3. Apache

Apache est le point d'entrée de tout le trafic web. On l'a configuré en mode **MPM Event** plutôt que Prefork. La différence : Prefork crée un processus par connexion (lourd en RAM), Event gère plusieurs connexions par thread (bien plus efficace). Sur un Pi avec de la RAM limitée, c'est important.

On a activé plusieurs modules :

- `ssl` — pour le HTTPS
- `http2` — pour servir les pages plus vite (HTTP/2 est le protocole moderne)
- `headers` — pour ajouter des headers de sécurité sur toutes les réponses
- `rewrite` — pour la redirection HTTP → HTTPS
- `proxy_fcgi` — pour que Apache puisse parler à PHP-FPM (sans ça, Apache ne sait pas exécuter du PHP)
- `deflate` / `expires` — compression et cache pour réduire la bande passante

La configuration MPM Event est dimensionnée pour le Pi :

```apache
StartServers             2      # 2 processus au démarrage, pas la peine d'en lancer plus
MinSpareThreads         25      # Threads minimum disponibles pour répondre rapidement
MaxSpareThreads         75      # On ne garde pas trop de threads inutilisés en mémoire
MaxRequestWorkers      150      # Plafond de connexions simultanées
MaxConnectionsPerChild 1000     # Un processus traite 1000 requêtes puis redémarre proprement
```

On masque aussi la version d'Apache avec `ServerTokens Prod` — par défaut Apache annonce sa version exacte dans chaque réponse HTTP, ce qui donne des informations inutiles à un attaquant.

---

## 4. PHP et MariaDB

### PHP-FPM

PHP-FPM (FastCGI Process Manager) est le gestionnaire de processus PHP. Il tourne séparément d'Apache et communique avec lui via une socket Unix (`/run/php/php8.4-fpm.sock`). C'est plus performant que le mod_php classique car Apache n'a pas besoin d'embarquer PHP dans chaque processus.

Sur Trixie, la version disponible est PHP **8.4** (pas 8.3 comme sur Debian 12). Les extensions installées couvrent les besoins d'une application PHP standard et les prérequis de Zabbix : `mbstring`, `xml`, `gd`, `curl`, `ldap`, `snmp`, `intl`, `bcmath`.

Paramètres PHP importants pour Zabbix :

```ini
max_execution_time = 300    # Zabbix peut avoir des requêtes longues, 300s évite les timeouts
max_input_time     = 300    # Temps max pour parser les données reçues
memory_limit       = 256M   # Zabbix frontend est gourmand, 128M suffit rarement
post_max_size      = 32M    # Taille max des données POST (imports, fichiers)
expose_php         = Off    # Ne pas révéler la version PHP dans les headers HTTP
```

### MariaDB

MariaDB est la base de données. Elle stocke toutes les données de Zabbix (métriques, historique, configuration). On l'a tuné pour le Pi :

```ini
innodb_buffer_pool_size = 512M
# C'est le paramètre le plus important de MariaDB. Il définit la taille du cache
# en RAM pour les données et index InnoDB. Plus c'est grand, moins MariaDB lit
# sur le disque. Sur 4 Go de RAM avec cette stack, 512M est un bon compromis.

innodb_flush_log_at_trx_commit = 2
# Par défaut à 1 : MariaDB écrit et synchronise le log à chaque transaction.
# À 2 : il écrit à chaque transaction mais ne synchronise qu'une fois par seconde.
# Sur une SD card, ça réduit énormément les écritures au prix d'une perte de données
# maximale d'1 seconde en cas de crash — acceptable pour ce contexte.

innodb_flush_method = O_DIRECT
# Évite la double mise en cache (cache MariaDB + cache OS). MariaDB gère lui-même
# son cache via le buffer pool, pas besoin que l'OS garde une copie en plus.
```

### Création de la base Zabbix

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```
On crée une base en `utf8mb4` (UTF-8 complet, supporte les emojis et caractères spéciaux) avec la collation `utf8mb4_bin` (comparaison binaire, sensible à la casse — requis par Zabbix).

```sql
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'changeme';
```
On crée un utilisateur dédié à Zabbix. Le `@'localhost'` signifie que cet utilisateur ne peut se connecter que depuis la machine elle-même — impossible de se connecter à cette base depuis le réseau.

```sql
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
```
On donne tous les droits à cet utilisateur, mais uniquement sur la base `zabbix`. Il ne peut pas toucher aux autres bases (mysql, information_schema...).

```sql
FLUSH PRIVILEGES;
```
Force MariaDB à recharger la table des permissions en mémoire. Sans ça, les changements ne sont pas pris en compte immédiatement.

---

## 5. SSL

Comme le Pi n'a pas de nom de domaine public, on ne peut pas utiliser Let's Encrypt (qui nécessite de prouver qu'on contrôle un domaine via internet). On génère donc un certificat **auto-signé**.

```bash
openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
  -keyout /etc/ssl/raspserver/raspserver.key \
  -out /etc/ssl/raspserver/raspserver.crt
```

- `-x509` : génère un certificat auto-signé (pas une demande de signature à envoyer à une CA)
- `-nodes` : pas de passphrase sur la clé privée (nécessaire pour qu'Apache puisse démarrer automatiquement sans saisie manuelle)
- `-days 3650` : valable 10 ans
- `-newkey rsa:4096` : clé RSA 4096 bits (robuste)

Le navigateur affichera un avertissement "certificat non approuvé" — c'est normal, il n'est pas signé par une autorité reconnue. Le trafic est quand même chiffré.

On a désactivé TLS 1.0 et 1.1 (protocoles obsolètes avec des failles connues) et on ne garde que TLS 1.2 et 1.3.

---

## 6. ModSecurity

ModSecurity est un WAF (Web Application Firewall) — un pare-feu qui analyse le contenu des requêtes HTTP, pas juste les ports et IPs. Il peut détecter et bloquer des attaques applicatives comme les injections SQL, les XSS, ou les tentatives d'accès à des fichiers sensibles.

On l'a couplé aux règles **OWASP CRS** (Core Rule Set) version 4.7, qui est un ensemble de règles maintenu par la communauté OWASP et couvre les attaques les plus courantes.

ModSecurity fonctionne en deux modes :
- `DetectionOnly` : il détecte mais ne bloque pas (utile pour tester)
- `On` : il bloque les requêtes suspectes (mode actif, ce qu'on utilise)

On a ajouté des exclusions pour Zabbix car certaines requêtes légitimes du frontend Zabbix déclenchent des faux positifs (notamment les requêtes API avec des paramètres que ModSecurity interprète comme des injections).

---

## 7. Zabbix

### Compatibilité Trixie

Zabbix ne publie pas encore de paquets pour Debian 13 Trixie. On utilise le repo **Debian 12 Bookworm**, qui est compatible. Le seul problème rencontré : Zabbix compilé pour Bookworm dépend de `libldap-2.5`, alors que Trixie a `libldap-2.6`.

Solution : on ajoute temporairement le repo Bookworm avec un **pinning APT strict**. Le pinning permet de dire à APT "utilise ce repo uniquement pour ce paquet précis, ignore-le pour tout le reste". Ainsi on installe libldap-2.5 depuis Bookworm sans risquer qu'APT commence à tirer d'autres paquets Bookworm qui casseraient le système Trixie.

```
Package: *
Pin: release n=bookworm
Pin-Priority: -10          # Priorité négative = ignoré par défaut pour tout

Package: libldap-2.5-0
Pin: release n=bookworm
Pin-Priority: 500          # Sauf pour ce paquet précis
```

### Configuration serveur

Les paramètres de performance Zabbix sont dimensionnés pour le Pi :

```
StartPollers=5             # 5 processus qui interrogent les agents. Plus = plus de données
                           # collectées en parallèle, mais plus de RAM utilisée.
CacheSize=64M              # Cache de la configuration (hosts, items, triggers)
HistoryCacheSize=32M       # Buffer en RAM pour l'historique avant écriture en base
ValueCacheSize=32M         # Cache des dernières valeurs, évite des requêtes SQL répétées
```

Ces valeurs sont en dessous des recommandations pour un gros déploiement Zabbix, mais adaptées à un Pi qui fait aussi tourner Apache, PHP et MariaDB.

### Import du schéma

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix
```

Zabbix fournit son schéma SQL compressé. Cette commande le décompresse à la volée (`zcat`) et l'envoie directement à MariaDB sans créer de fichier intermédiaire. L'import prend 2-3 minutes sur le Pi.

---

## 8. Fail2ban et UFW

### UFW

UFW (Uncomplicated Firewall) est la surcouche simplifiée d'iptables. La politique de base : tout bloquer en entrée, tout autoriser en sortie. On ouvre uniquement ce qui est nécessaire :

- Port 22 (SSH) : accès admin
- Ports 80/443 (HTTP/HTTPS) : accès web
- Ports 10050/10051 (Zabbix) : uniquement depuis le réseau local `192.168.0.0/24`, pas depuis internet
- Interface Tailscale : tout autoriser sur cette interface (le VPN Tailscale gère sa propre sécurité)

### Fail2ban

Fail2ban surveille les logs et banne automatiquement les IPs qui échouent trop souvent à s'authentifier. Il agit en ajoutant des règles iptables temporaires.

Configuration SSH : 3 tentatives échouées = ban 24h. C'est volontairement strict car SSH est la porte d'entrée principale du serveur.

Configuration Apache : plusieurs jails couvrent les bots malveillants, les tentatives d'authentification HTTP, et les requêtes anormales.

---

## 9. Tuning 24/7

### tmpfs — logs en RAM

Une carte SD supporte un nombre limité de cycles d'écriture. Apache et Zabbix écrivent dans leurs logs en permanence, ce qui userait prématurément la SD card. On monte les dossiers de logs en RAM (tmpfs) :

```
tmpfs /var/log/apache2    tmpfs defaults,noatime,nosuid,size=50m
tmpfs /var/log/zabbix     tmpfs defaults,noatime,nosuid,size=50m
```

Conséquence : les logs sont perdus à chaque redémarrage. C'est le compromis accepté pour prolonger la durée de vie de la SD card.

### Watchdog

Un timer systemd vérifie toutes les 5 minutes que Apache, MariaDB, Zabbix et PHP-FPM tournent. Si l'un d'eux a crashé, il le redémarre automatiquement. Sur un serveur 24/7, c'est ce qui fait la différence entre un service qui reste down jusqu'à ce que quelqu'un le remarque, et un service qui redémarre seul en quelques minutes.

---

*RaspServer v1.0 — Mars 2026*
