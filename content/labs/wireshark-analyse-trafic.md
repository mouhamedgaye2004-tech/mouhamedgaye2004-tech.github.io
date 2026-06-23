---
title: "Analyse de trafic réseau avec Wireshark"
date: 2026-06-23
draft: false
tags: ["Wireshark", "Réseau", "TCP/IP", "DNS", "HTTP", "ARP", "ICMP"]
categories: ["Labs"]
description: "Capture et analyse de trafic DNS, HTTP/HTTPS et ARP sur un réseau local."
showToc: true
---

## Objectif du lab

Analyser le trafic réseau en temps réel avec Wireshark pour comprendre le fonctionnement des protocoles DNS, HTTP/HTTPS, ARP et ICMP.

**Environnement :**
- OS : Windows 11
- Outil : Wireshark 4.x
- Interface capturée : Wi-Fi / Ethernet

---

## Phase 1 — Analyse du trafic DNS

### Filtre utilisé dans Wireshark
Tape dans la barre de filtre Wireshark : dns

### Manipulations effectuées
Dans PowerShell tape :
nslookup google.com
nslookup github.com
nslookup nonexistant-xyz123.com

### Ce que jai observé

#### Échange DNS complet (Query / Response)

<!-- SCREENSHOT : requête DNS Query type A + réponse avec IP résolue -->

- Source : mon IP locale
- Destination : 8.8.8.8 (DNS Google)
- Type : A (IPv4)
- TTL : durée de mise en cache (ex: 300s = 5 min)

#### Réponse NXDOMAIN (domaine inexistant)

<!-- SCREENSHOT : réponse DNS avec code NXDOMAIN -->

Quand le domaine nexiste pas, le serveur répond NXDOMAIN (Non-Existent Domain).

> Conclusion : DNS fonctionne en UDP port 53. Sans lui, il faudrait connaître lIP de chaque site par coeur.

---

## Phase 2 — HTTP vs HTTPS

### Filtres utilisés dans Wireshark
http
tls

### Manipulations effectuées
- Navigation sur http://neverssl.com (HTTP non chiffré)
- Navigation sur https://github.com (HTTPS chiffré)
- Clic droit sur un paquet HTTP > Suivre > Flux TCP

### Ce que jai observé

#### Handshake TCP (SYN / SYN-ACK / ACK)
Filtre : tcp.flags.syn == 1

<!-- SCREENSHOT : les 3 paquets SYN / SYN-ACK / ACK -->

1. SYN : le client demande une connexion
2. SYN-ACK : le serveur accepte
3. ACK : le client confirme

#### HTTP — contenu visible en clair

<!-- SCREENSHOT : flux TCP avec requête GET et headers HTTP visibles -->

En HTTP, tout est lisible : méthode GET, Host, User-Agent, cookies.
Sur un Wi-Fi public, nimporte qui avec Wireshark peut lire ces données.

#### HTTPS — contenu chiffré

<!-- SCREENSHOT : paquet TLS Client Hello avec champ SNI -->

Le contenu est chiffré. Mais le SNI (Server Name Indication) reste visible dans le Client Hello.

> Conclusion : HTTP expose tout en clair. HTTPS est obligatoire.

---

## Phase 3 — ARP et ICMP

### Filtres utilisés dans Wireshark
arp
icmp

### Manipulations effectuées
Dans PowerShell tape :
ping 8.8.8.8
ping 192.168.1.254
arp -a

### Ce que jai observé

#### Requêtes ARP — résolution MAC

<!-- SCREENSHOT : requêtes ARP Who has X Tell Y -->

ARP résout les adresses IP en adresses MAC sur le réseau local.

#### ICMP — Echo Request / Echo Reply

<!-- SCREENSHOT : paquets ICMP Echo Request et Echo Reply -->

- Echo Request type 8 : envoyé par mon PC
- Echo Reply type 0 : répondu par la destination

#### Ping vers IP inexistante

<!-- SCREENSHOT : requêtes ARP répétées sans réponse -->

Wireshark montre des requêtes ARP répétées sans réponse.

---

## Filtres essentiels à retenir

dns → tout le trafic DNS
http → trafic HTTP non chiffré
tls → trafic HTTPS
arp → résolution adresses MAC
icmp → ping et erreurs réseau
tcp.flags.syn == 1 → handshakes TCP
ip.addr == x.x.x.x → filtrer par IP

---

## Ce que jaurais fait différemment

- Annoter les paquets directement dans Wireshark
- Faire les captures sur une VM pour moins de bruit de fond
- Simuler un ARP scan depuis Kali Linux

## Prochaine étape

Intégrer Wireshark dans le homelab pfSense pour analyser le trafic inter-VLANs.
