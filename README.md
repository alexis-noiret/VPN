# 🔒 Déploiement OpenVPN — Kompta Plus Bien (KPB)

<div align="center">

![OpenVPN](https://img.shields.io/badge/OpenVPN-2.6.14-EA7E20?style=for-the-badge&logo=openvpn&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-12_Bookworm-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![VMware](https://img.shields.io/badge/VMware-Workstation-607078?style=for-the-badge&logo=vmware&logoColor=white)
![easy-rsa](https://img.shields.io/badge/easy--rsa-PKI-065A82?style=for-the-badge&logo=openssl&logoColor=white)
![AES](https://img.shields.io/badge/AES--256--GCM-Chiffrement-22C55E?style=for-the-badge&logo=letsencrypt&logoColor=white)

*Projet Sécurité Réseau — La Plateforme*

</div>

---

## 📋 Contexte

**Kompta Plus Bien (KPB)** est un cabinet comptable basé à Marignane employant une trentaine de personnes. Dans le cadre du télétravail, l'entreprise a besoin d'une connexion sécurisée permettant à ses 30 salariés d'accéder aux documents de comptabilité, factures internes et dossiers partagés.

La solution retenue : **OpenVPN** sur infrastructure Debian, testé sur maquette avec deux machines virtuelles.

---

## 🏗️ Architecture

```
┌─────────────────────────────┐         ┌─────────────────────────────┐
│        VM1 — SERVEUR        │         │        VM2 — CLIENT         │
│                             │         │                             │
│  Debian 12 Bookworm         │◄───────►│  Debian 12 Bookworm         │
│  OpenVPN 2.6.14             │  Tunnel │  OpenVPN Client             │
│  IP: 172.16.228.133         │  VPN    │  IP: 172.16.228.136         │
│  tun0: 10.8.0.1             │AES-256  │  tun0: 10.8.0.6             │
│  Port UDP 1194              │         │  Watchdog cron              │
│  easy-rsa PKI               │         │  Auto-restart               │
└─────────────────────────────┘         └─────────────────────────────┘
         │
         │  PKI (easy-rsa)
         │  ├── CA : KPB-CA
         │  ├── server.crt / server.key
         │  ├── client1.crt / client1.key
         │  ├── dh.pem (2048 bits)
         │  └── ta.key (tls-auth)
```

---

## ✅ Ce qui a été réalisé

| Élément | Statut | Détail |
|---------|--------|--------|
| Installation OpenVPN | ✅ | v2.6.14 sur Debian 12 |
| PKI complète easy-rsa | ✅ | CA KPB-CA + certs serveur & client |
| Chiffrement AES-256-GCM | ✅ | Tunnel sécurisé opérationnel |
| Authentification TLS mutuelle | ✅ | tls-auth + certificats X.509 |
| Diffie-Hellman 2048 bits | ✅ | Échange de clés sécurisé |
| Tunnel tun0 opérationnel | ✅ | 0% packet loss, ~1ms latence |
| Forwarding IP | ✅ | net.ipv4.ip_forward=1 |
| Démarrage automatique | ✅ | systemctl enable openvpn@server/client |
| Watchdog résilience | ✅ | Script cron toutes les 5 minutes |
| Transfert certificats SCP | ✅ | Via SSH entre les deux VMs |
| 2FA Google Authenticator (PAM) | ⚠️ | PAM configuré et fonctionnel en local, intégration OpenVPN non finalisée |

---

## 📁 Structure du projet

```
vpn-kpb/
├── README.md
├── server/
│   ├── server.conf          # Configuration serveur OpenVPN
│   └── pam.d/openvpn        # Configuration PAM (2FA)
├── client/
│   └── client.conf          # Configuration client OpenVPN
├── scripts/
│   └── vpn-watchdog.sh      # Script de résilience
└── pki/
    └── (généré via easy-rsa, non versionné)
```

---

## 🚀 Installation

### Prérequis

- 2 VMs Debian 12
- VMware Workstation (ou autre hyperviseur)
- Réseau commun entre les VMs

### 1. Installer OpenVPN + easy-rsa (serveur)

```bash
apt update && apt install -y openvpn easy-rsa
```

### 2. Initialiser la PKI

```bash
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass          # Common Name : KPB-CA
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey secret ~/openvpn-ca/pki/ta.key
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

### 3. Copier les fichiers

```bash
cp ~/openvpn-ca/pki/ca.crt /etc/openvpn/
cp ~/openvpn-ca/pki/issued/server.crt /etc/openvpn/
cp ~/openvpn-ca/pki/private/server.key /etc/openvpn/
cp ~/openvpn-ca/pki/dh.pem /etc/openvpn/
cp ~/openvpn-ca/pki/ta.key /etc/openvpn/
mkdir -p /var/log/openvpn
```

### 4. Configuration serveur (`/etc/openvpn/server.conf`)

```
port 1194
proto udp
dev tun

ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem
tls-auth /etc/openvpn/ta.key 0

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt

push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"

keepalive 10 120
cipher AES-256-CBC
persist-key
persist-tun

status /var/log/openvpn/openvpn-status.log
log /var/log/openvpn/openvpn.log
verb 3
```

### 5. Activer le forwarding IP et démarrer

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
systemctl start openvpn@server
systemctl enable openvpn@server
```

### 6. Transférer les certificats vers le client

```bash
scp ~/openvpn-ca/pki/ca.crt root@<IP_CLIENT>:/etc/openvpn/
scp ~/openvpn-ca/pki/issued/client1.crt root@<IP_CLIENT>:/etc/openvpn/
scp ~/openvpn-ca/pki/private/client1.key root@<IP_CLIENT>:/etc/openvpn/
scp ~/openvpn-ca/pki/ta.key root@<IP_CLIENT>:/etc/openvpn/
```

### 7. Configuration client (`/etc/openvpn/client.conf`)

```
client
dev tun
proto udp

remote <IP_SERVEUR> 1194

resolv-retry infinite
nobind

ca /etc/openvpn/ca.crt
cert /etc/openvpn/client1.crt
key /etc/openvpn/client1.key
tls-auth /etc/openvpn/ta.key 1
remote-cert-tls server

cipher AES-256-CBC
verb 3
```

### 8. Démarrer le client

```bash
systemctl start openvpn@client
systemctl enable openvpn@client
```

---

## 🛡️ Résilience

### Watchdog (`/usr/local/bin/vpn-watchdog.sh`)

```bash
#!/bin/bash
if ! ip link show tun0 > /dev/null 2>&1; then
    echo "$(date): tun0 down, restarting OpenVPN..." >> /var/log/vpn-watchdog.log
    systemctl restart openvpn@client
fi
```

```bash
chmod +x /usr/local/bin/vpn-watchdog.sh
crontab -e
# Ajouter :
*/5 * * * * /usr/local/bin/vpn-watchdog.sh
```

---

## 🔐 Bonus — 2FA Google Authenticator (partiel)

L'authentification à deux facteurs a été partiellement configurée :

```bash
apt install -y libpam-google-authenticator
google-authenticator -t -d -f -r 3 -R 30 -w 3
```

Le module PAM a été testé et fonctionne en local (`pamtester openvpn root authenticate` → `successfully authenticated`).

```
# /etc/pam.d/openvpn
auth required pam_google_authenticator.so secret=/root/.google_authenticator user=root no_increment_hotp
```

> ⚠️ L'intégration complète avec le plugin OpenVPN (`openvpn-plugin-auth-pam.so`) n'a pas pu être finalisée en raison d'un problème de permissions entre le daemon OpenVPN et le fichier `.google_authenticator`. PAM fonctionne, l'intégration VPN est en cours.

---

## 🔍 Vérification

```bash
# Côté client — vérifier le tunnel
ip a | grep tun0
# → inet 10.8.0.6 peer 10.8.0.5/32 scope global tun0

ping 10.8.0.1 -c 3
# → 3 packets transmitted, 3 received, 0% packet loss
```

---

## 🎯 Compétences visées

- ➔ Administrer et sécuriser les infrastructures systèmes
- ➔ Administrer et sécuriser les infrastructures réseaux
- ➔ Participer à l'élaboration et à la mise en œuvre de la politique de sécurité

---

## 🧰 Stack technique

| Technologie | Usage |
|-------------|-------|
| OpenVPN 2.6.14 | Serveur/Client VPN |
| easy-rsa | Gestion PKI & certificats |
| AES-256-GCM | Chiffrement du tunnel |
| TLS 1.3 | Authentification mutuelle |
| Diffie-Hellman 2048 | Échange de clés |
| PAM + Google Authenticator | 2FA (partiel) |
| systemd | Gestion des services |
| cron | Watchdog résilience |

---

<div align="center">
  <sub>Projet réalisé dans le cadre de la formation Full Stack & Cybersécurité — <strong>La Plateforme</strong></sub>
</div>
