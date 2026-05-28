# 🏗️ Architecture Réseau Sécurisée — Expliquée Simplement

> Concevoir un réseau qui résiste aux attaques : DMZ, segmentation, VLAN, firewall, reverse proxy, VPN, IDS/IPS.

![Sécurité](https://img.shields.io/badge/Domaine-Architecture%20Réseau-blue)
![Niveau](https://img.shields.io/badge/Niveau-Débutant%20→%20Expert-orange)
![Standards](https://img.shields.io/badge/Standards-ISO%2027001%20·%20NIST-green)

---

## 📖 Pourquoi une architecture réseau sécurisée ?

Un réseau mal conçu est la première cause de compromission en entreprise. Le principe fondamental est simple :

```
Un attaquant qui pénètre dans une zone
ne doit JAMAIS pouvoir atteindre directement les données critiques.
```

La défense en profondeur (*Defense in Depth*) consiste à multiplier les couches de sécurité, de sorte que contourner l'une n'ouvre pas l'accès à tout.

```
Internet → Pare-feu périmétrique → DMZ → Pare-feu interne → LAN → Données
              (couche 1)          (couche 2)   (couche 3)   (couche 4)
```

---

## 🌐 DMZ — Zone Démilitarisée

### Définition

La **DMZ** (DeMilitarized Zone) est un segment réseau intermédiaire entre Internet et le réseau interne. Elle héberge les services accessibles depuis l'extérieur, tout en les isolant des ressources internes critiques.

### Principe

```
Internet ──► [Firewall externe] ──► DMZ ──► [Firewall interne] ──► LAN interne
                                     │
                              Serveurs exposés :
                              - Serveur web
                              - Serveur mail
                              - Reverse proxy
                              - Serveur DNS public
                              - Serveur VPN (point d'entrée)
```

### Règles de flux DMZ

| Source | Destination | Port | Action | Justification |
|--------|-------------|------|--------|---------------|
| Internet | DMZ (Web) | 80, 443 | ✅ Autorisé | Accès web public |
| Internet | DMZ (Mail) | 25, 465, 587 | ✅ Autorisé | Réception mail |
| Internet | LAN interne | Tout | ❌ Bloqué | Jamais d'accès direct |
| DMZ | LAN interne | 3306 (SQL) | ⚠️ Restreint | Seulement si nécessaire |
| LAN interne | DMZ | Tout | ✅ Autorisé | Administration |
| DMZ | Internet | Tout | ❌ Bloqué | Évite les rebonds |

### Bonnes pratiques DMZ

- **Double firewall** : firewall différents (éditeurs différents) pour éviter qu'une CVE compromette les deux
- **Pas de données sensibles en DMZ** : bases de données client, fichiers RH — jamais en DMZ
- **Logs centralisés** : tout trafic DMZ est loggé et envoyé au SIEM
- **Segmenter la DMZ elle-même** : le serveur web ne doit pas parler au serveur mail

---

## 🔲 Segmentation réseau

### Définition

La **segmentation** consiste à diviser un réseau en zones distinctes avec des politiques de filtrage entre elles. En cas de compromission d'une zone, l'attaquant est contenu.

### Modèle de segmentation typique en entreprise

```
┌─────────────────────────────────────────────────────────────┐
│                        RÉSEAU ENTREPRISE                    │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │   DMZ    │  │ SERVERS  │  │  USERS   │  │  ADMIN   │  │
│  │          │  │          │  │          │  │          │  │
│  │ Web      │  │ DB       │  │ Postes   │  │ DC       │  │
│  │ Mail     │  │ App      │  │ WiFi     │  │ PKI      │  │
│  │ DNS pub  │  │ NAS      │  │ BYOD     │  │ SIEM     │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
│       │              │              │              │        │
│  ─────┴──────────────┴──────────────┴──────────────┴─────  │
│                    CORE SWITCH / FIREWALL                   │
└─────────────────────────────────────────────────────────────┘
```

### Zones recommandées

| Zone | Contenu | Niveau de confiance |
|------|---------|-------------------|
| **DMZ** | Serveurs exposés (web, mail, proxy) | Non fiable |
| **Servers** | Applications internes, bases de données | Élevé |
| **Users** | Postes utilisateurs, WiFi entreprise | Moyen |
| **Admin** | Contrôleurs de domaine, PKI, SIEM | Très élevé |
| **IoT** | Caméras, imprimantes, capteurs | Non fiable |
| **Quarantine** | Machines suspectes, invités | Zéro confiance |

### Principe du moindre privilege réseau

```
Règle d'or : Deny All par défaut, puis autoriser explicitement ce qui est nécessaire.

Mauvais : autoriser tout le LAN à parler au DC
Bon     : seuls les serveurs d'authentification et les admins parlent au DC
```

---

## 🏷️ VLAN — Virtual LAN

### Définition

Un **VLAN** (Virtual Local Area Network) est une segmentation logique d'un réseau physique. Des machines sur le même switch physique peuvent être dans des segments réseau complètement isolés.

### Comment ça marche ?

```
Switch physique unique
├── Port 1-10  → VLAN 10 (Utilisateurs)    192.168.10.0/24
├── Port 11-20 → VLAN 20 (Serveurs)        192.168.20.0/24
├── Port 21-25 → VLAN 30 (Administration)  192.168.30.0/24
├── Port 26-28 → VLAN 40 (IoT)             192.168.40.0/24
└── Port 29-30 → VLAN 99 (Trunk / Uplink)  — tous les VLANs
```

### Types de ports VLAN

| Type | Rôle | Usage |
|------|------|-------|
| **Access port** | Appartient à un seul VLAN | Connexion d'un poste utilisateur |
| **Trunk port** | Transporte tous les VLANs (tag 802.1Q) | Liaison switch-switch, switch-firewall |
| **Native VLAN** | VLAN par défaut (non taggé) | Risque — à changer du VLAN 1 par défaut |

### Configuration VLAN (Cisco IOS)

```bash
# Création des VLANs
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 30
 name ADMIN

# Port access (poste utilisateur)
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10

# Port trunk (vers le firewall)
interface GigabitEthernet0/24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 switchport trunk native vlan 99   # Changer le native VLAN !
```

### Attaque VLAN Hopping — et comment s'en protéger

```
Attaque : un attaquant en VLAN 10 envoie des trames double-taggées
          pour atteindre le VLAN 20 (serveurs).

Protection :
  1. Changer le Native VLAN (≠ VLAN 1)
  2. Désactiver le DTP (Dynamic Trunking Protocol) sur les ports access
  3. Éteindre les ports inutilisés → VLAN de quarantaine
```

```bash
# Désactiver DTP sur les ports utilisateurs
interface GigabitEthernet0/1
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
```

---

## 🔥 Firewall

### Définition

Le **firewall** filtre le trafic réseau selon des règles prédéfinies. Il analyse les paquets et décide de les laisser passer ou de les bloquer.

### Les 3 générations de firewalls

| Génération | Type | Analyse | Exemple |
|-----------|------|---------|---------|
| 1ère | Stateless (filtrage de paquets) | IP source/dest, port | iptables basique |
| 2ème | Stateful | État de la connexion (TCP SYN/ACK) | pfSense, iptables stateful |
| 3ème | NGFW (Next-Gen) | Applications, utilisateurs, contenu, SSL inspection | Palo Alto, Fortinet, Checkpoint |

### Architecture firewall typique (double firewall)

```
Internet
    │
[Firewall 1 — Périmétrique]   ← Filtre le trafic Internet brut
    │                           (Cisco, Fortinet, pfSense)
   DMZ
    │
[Firewall 2 — Interne]        ← Protège le LAN de la DMZ
    │                           (éditeur DIFFÉRENT du FW1)
  LAN interne
```

### Règles firewall — bonnes pratiques

```
Ordre des règles (traitement de haut en bas, première correspondance gagne) :

1. [ALLOW] Trafic établi (ESTABLISHED, RELATED)
2. [ALLOW] Services autorisés explicitement
3. [LOG]   Trafic suspect (optionnel, avant la règle finale)
4. [DENY]  Tout le reste  ← Implicit deny — jamais oublier !
```

### Exemple de ruleset

```bash
# iptables — Règles firewall Linux
# Politique par défaut : tout bloquer
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP

# Autoriser les connexions établies
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Autoriser SSH depuis le réseau admin uniquement
iptables -A INPUT -s 192.168.30.0/24 -p tcp --dport 22 -j ACCEPT

# Autoriser HTTPS entrant (serveur web)
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Loguer les paquets bloqués avant DROP
iptables -A INPUT -j LOG --log-prefix "FW-DROP: " --log-level 4
iptables -A INPUT -j DROP
```

---

## 🔄 Reverse Proxy

### Définition

Le **reverse proxy** est un serveur intermédiaire placé devant les serveurs web internes. Les clients ne contactent jamais directement les serveurs d'application — ils parlent au proxy, qui relaie la requête.

```
Client Internet
      │
      ▼
[Reverse Proxy — nginx/HAProxy]   ← Point d'entrée unique
      │
      ├──► Serveur App 1 (192.168.20.10)
      ├──► Serveur App 2 (192.168.20.11)
      └──► Serveur App 3 (192.168.20.12)
```

### Avantages sécurité

| Fonction | Bénéfice |
|----------|----------|
| **Masquage des serveurs** | Les IPs internes ne sont jamais exposées |
| **Terminaison TLS** | Le certificat SSL est géré centralement |
| **WAF intégré** | Filtrage des requêtes HTTP malveillantes |
| **Load balancing** | Répartition de charge + disponibilité |
| **DDoS mitigation** | Rate limiting, connexions maximum |
| **Cache** | Réduction de la charge sur les backends |

### Configuration nginx — Reverse Proxy sécurisé

```nginx
server {
    listen 443 ssl http2;
    server_name app.entreprise.com;

    # TLS — certificat et configuration durcie
    ssl_certificate     /etc/ssl/app.crt;
    ssl_certificate_key /etc/ssl/app.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    # Headers de sécurité
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Content-Security-Policy "default-src 'self'";

    # Rate limiting — 10 requêtes/seconde par IP
    limit_req zone=api burst=20 nodelay;

    # Proxy vers le backend (réseau interne — IP non exposée)
    location / {
        proxy_pass         http://192.168.20.10:8080;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_hide_header  X-Powered-By;   # Masquer la techno backend
    }
}
```

---

## 🔐 VPN — Virtual Private Network

### Définition

Un **VPN** crée un tunnel chiffré entre deux points sur un réseau non fiable (Internet). Il permet l'accès distant sécurisé au réseau interne, ou l'interconnexion de sites géographiquement distants.

### Types de VPN

| Type | Usage | Protocoles |
|------|-------|-----------|
| **Remote Access VPN** | Employé depuis chez lui → réseau entreprise | OpenVPN, WireGuard, SSL/TLS |
| **Site-to-Site VPN** | Siège ↔ Agence Paris ↔ Agence Lyon | IPsec, GRE over IPsec |
| **Zero Trust VPN** | Accès granulaire par application | Tailscale, Cloudflare Access |

### Architecture VPN dans le réseau

```
Employé à distance                    Réseau Entreprise
┌──────────────┐                     ┌─────────────────┐
│  Laptop      │                     │  VPN Gateway    │
│              │──── Tunnel chiffré ─│  (DMZ)          │
│  WireGuard / │     (Internet)      │                 │
│  OpenVPN     │                     │  ┌───────────┐  │
└──────────────┘                     │  │ LAN intern│  │
                                     │  └───────────┘  │
                                     └─────────────────┘
```

### Comparaison des protocoles VPN

| Protocole | Vitesse | Sécurité | Complexité | Usage recommandé |
|-----------|---------|----------|------------|-----------------|
| **WireGuard** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Faible | Modern — recommandé |
| **OpenVPN** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Moyenne | Compatible, éprouvé |
| **IPsec/IKEv2** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Élevée | Site-to-site enterprise |
| **L2TP/IPsec** | ⭐⭐⭐ | ⭐⭐⭐ | Moyenne | Legacy — éviter si possible |
| **PPTP** | ⭐⭐⭐⭐ | ⭐ | Faible | ❌ Obsolète — ne pas utiliser |

### Configuration WireGuard (exemple)

```ini
# /etc/wireguard/wg0.conf — Serveur VPN

[Interface]
Address    = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

# Client 1 — Alice (télétravail)
[Peer]
PublicKey  = <ALICE_PUBLIC_KEY>
AllowedIPs = 10.8.0.2/32

# Client 2 — Bob (télétravail)
[Peer]
PublicKey  = <BOB_PUBLIC_KEY>
AllowedIPs = 10.8.0.3/32
```

### Split Tunneling vs Full Tunneling

```
Full Tunneling : TOUT le trafic passe par le VPN entreprise
                 ✅ Sécurisé  ❌ Lent (tout transite par le siège)

Split Tunneling : Seul le trafic entreprise passe par VPN
                  ✅ Rapide   ⚠️ Risque si le poste est compromis
```

---

## 🚨 IDS / IPS — Détection et Prévention d'Intrusion

### Définition

| Sigle | Nom complet | Action |
|-------|-------------|--------|
| **IDS** | Intrusion Detection System | Détecte et **alerte** — passif |
| **IPS** | Intrusion Prevention System | Détecte et **bloque** — actif |

### Positionnement dans le réseau

```
Internet
    │
[Firewall périmétrique]
    │
   DMZ  ←──── [IDS/IPS — mode inline ou SPAN]
    │               Surveille le trafic DMZ ↔ LAN
[Firewall interne]
    │
  LAN interne  ←── [IDS/IPS interne — détection de mouvement latéral]
```

### Types de signatures IDS/IPS

| Type | Mécanisme | Avantage | Inconvénient |
|------|-----------|----------|--------------|
| **Signature-based** | Correspondance avec une base de patterns connus | Précis, peu de FP | Inefficace contre 0-day |
| **Anomaly-based** | Détection des déviations par rapport à la baseline | Détecte les 0-day | Beaucoup de faux positifs |
| **Stateful Protocol** | Vérifie la conformité des protocoles | Robuste | Limité aux protocoles connus |

### Outils IDS/IPS open source

| Outil | Type | Points forts |
|-------|------|-------------|
| **Snort** | IDS/IPS | Standard de l'industrie, règles Talos |
| **Suricata** | IDS/IPS | Multi-thread, plus performant que Snort |
| **Zeek (ex-Bro)** | IDS | Analyse comportementale, scripts puissants |
| **OSSEC** | HIDS (Host) | Analyse les logs, intégrité des fichiers |
| **Wazuh** | HIDS + SIEM | OSSEC modernisé, interface web |

### Exemple de règle Suricata

```
# Détection d'un scan de ports (Nmap)
alert tcp any any -> $HOME_NET any (
    msg:"SCAN Nmap SYN Scan détecté";
    flags:S;
    threshold: type threshold, track by_src, count 20, seconds 1;
    classtype:attempted-recon;
    sid:1000001;
    rev:1;
)

# Détection de Pass-the-Hash (MITRE T1550.002)
alert smb $EXTERNAL_NET any -> $HOME_NET 445 (
    msg:"LATERAL MOVEMENT Pass-the-Hash SMB";
    content:"|4e 54 4c 4d|";
    content:"|00 00 00 00 00 00 00 00 20 00 00 00|";
    classtype:attempted-user;
    sid:1000002;
    rev:1;
)
```

---

## 🗺️ Architecture complète — Schéma de référence

```
                           INTERNET
                              │
                    ┌─────────┴─────────┐
                    │  FIREWALL EXT.    │  ← Périmètre, anti-DDoS
                    │  (Fortinet/PF)    │
                    └─────────┬─────────┘
                              │
              ┌───────────────┴───────────────┐
              │             DMZ               │
              │  ┌─────────┐  ┌───────────┐  │
              │  │ Reverse │  │  Serveur  │  │
              │  │  Proxy  │  │   Mail    │  │
              │  │ (nginx) │  │ (Postfix) │  │
              │  └─────────┘  └───────────┘  │
              │  ┌─────────┐  ┌───────────┐  │
              │  │   DNS   │  │  VPN GW   │  │
              │  │ public  │  │(WireGuard)│  │
              │  └─────────┘  └───────────┘  │
              │         IDS/IPS               │
              └───────────────┬───────────────┘
                              │
                    ┌─────────┴─────────┐
                    │  FIREWALL INT.    │  ← Éditeur différent !
                    │  (Checkpoint)     │
                    └─────────┬─────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
   ┌──────┴──────┐   ┌────────┴──────┐   ┌───────┴──────┐
   │  VLAN 10   │   │    VLAN 20    │   │   VLAN 30    │
   │  USERS     │   │   SERVERS     │   │    ADMIN     │
   │            │   │               │   │              │
   │ Postes     │   │ App servers   │   │ DC / AD      │
   │ WiFi corp  │   │ Databases     │   │ SIEM         │
   │            │   │ NAS/backup    │   │ PAM          │
   └────────────┘   └───────────────┘   └──────────────┘
          │
   ┌──────┴──────┐
   │  VLAN 40   │
   │    IoT     │
   │            │
   │ Caméras    │
   │ Imprimantes│
   │ Capteurs   │
   └────────────┘
```

---

## 📊 Matrice de communication entre zones

| Source ↓ / Dest → | Internet | DMZ | VLAN Users | VLAN Servers | VLAN Admin | VLAN IoT |
|-------------------|----------|-----|-----------|--------------|------------|----------|
| **Internet** | — | ✅ 80/443 | ❌ | ❌ | ❌ | ❌ |
| **DMZ** | ❌ | — | ❌ | ⚠️ restreint | ❌ | ❌ |
| **VLAN Users** | ✅ | ✅ | — | ⚠️ restreint | ❌ | ❌ |
| **VLAN Servers** | ❌ | ⚠️ | ❌ | — | ✅ | ❌ |
| **VLAN Admin** | ❌ | ✅ | ✅ | ✅ | — | ✅ |
| **VLAN IoT** | ❌ | ❌ | ❌ | ❌ | ❌ | — |

> ✅ Autorisé · ⚠️ Restreint (ports/IPs spécifiques) · ❌ Bloqué par défaut

---

## ✅ Checklist — Architecture réseau sécurisée

### Périmètre
- [ ] Firewall périmétrique avec règle "Deny All" par défaut
- [ ] DMZ créée et isolée du LAN interne
- [ ] Double firewall (éditeurs différents)
- [ ] Reverse proxy devant tous les serveurs web exposés
- [ ] Certificats TLS valides, TLS 1.2+ seulement

### Segmentation
- [ ] VLANs créés par zone fonctionnelle
- [ ] Native VLAN changé (≠ VLAN 1)
- [ ] DTP désactivé sur les ports access
- [ ] Ports inutilisés désactivés ou en VLAN quarantaine
- [ ] ACLs inter-VLAN sur le firewall interne

### Accès distant
- [ ] VPN déployé (WireGuard ou OpenVPN)
- [ ] Authentification MFA sur le VPN
- [ ] Split vs Full tunneling défini selon politique
- [ ] Accès administrateur uniquement via jump host

### Détection
- [ ] IDS/IPS déployé en coupure sur le trafic DMZ
- [ ] IDS/IPS interne pour le mouvement latéral
- [ ] SIEM avec corrélation des événements firewall + IDS
- [ ] Alertes configurées pour les scans de ports, brute force, anomalies

---

## 🔗 Ressources utiles

| Ressource | Description |
|-----------|-------------|
| [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) | Guides de durcissement réseau |
| [NIST SP 800-41](https://csrc.nist.gov/publications/detail/sp/800-41/rev-1/final) | Guide firewalls et politique réseau |
| [pfSense docs](https://docs.netgate.com/pfsense/en/latest/) | Firewall open source |
| [Suricata rules](https://rules.emergingthreats.net/) | Règles IDS Emerging Threats |
| [Tailscale](https://tailscale.com/) | VPN Zero Trust moderne |

---

*Made for educational cybersecurity — Architecture réseau défensive*
