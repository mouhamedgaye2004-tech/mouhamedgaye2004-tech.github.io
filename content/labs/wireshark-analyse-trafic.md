$content = @'
---
title: "Analyse de trafic reseau avec Wireshark"
date: 2026-06-23
draft: false
tags: ["Wireshark", "Reseau", "TCP/IP", "DNS", "HTTP", "ARP", "ICMP"]
categories: ["Labs"]
description: "Capture et analyse de trafic DNS, HTTP/HTTPS, ARP et ICMP sur un reseau local."
showToc: true
TocOpen: false
---

## Objectif du lab

Ce lab analyse le trafic reseau en temps reel avec Wireshark pour comprendre les protocoles DNS, HTTP/HTTPS, ARP et ICMP.

**Environnement :**
| Element | Detail |
|---|---|
| OS | Windows 11 |
| Outil | Wireshark 4.6.6 + Npcap |
| Interface | Wi-Fi |
| Duree | 4h30 |

---

## Preparation

Au demarrage, Wireshark affiche tous les paquets en temps reel. Sans filtre, le volume est enorme. Les filtres permettent d'isoler exactement le protocole voulu.

![Capture Wireshark globale sans filtre](/images/wireshark/wireshark-capture-globale.png)

---

## Phase 1 - Analyse DNS

DNS traduit les noms de domaine en adresses IP. C'est le premier protocole sollicite lors de chaque navigation web.

Filtre utilise : dns

Manipulations dans PowerShell :
nslookup google.com
nslookup github.com
nslookup nonexistant-xyz123.com

Wireshark montre les echanges en paires : requete (Standard query) puis reponse (Standard query response).

![Liste des paquets DNS avec echanges Query et Response](/images/wireshark/dns-query-response.png)

En cliquant sur un paquet de reponse, on voit les couches reseau : Ethernet, IP, UDP port 53, DNS.

![Details du paquet DNS selectionne](/images/wireshark/dns-response-detail.png)

Structure interne de la reponse DNS :

![Structure interne du paquet DNS](/images/wireshark/dns-details-ttl.png)

- Transaction ID : identifiant unique pour associer requete et reponse
- Flags : Standard query response, No error
- Answer RRs 2 : la reponse contient 2 enregistrements

Section Answers avec IP resolue et TTL :

![Section Answers - CNAME et IP resolue](/images/wireshark/dns-answers-ttl.png)

Pour alive.github.com : CNAME vers live.github.com puis IP 140.82.113.25

![TTL visible dans les details DNS](/images/wireshark/dns-ttl.png)

Le TTL indique combien de secondes la reponse peut etre mise en cache. DNS fonctionne en UDP port 53.

---

## Phase 2 - HTTP vs HTTPS

### HTTP en clair

Filtre : http
Navigation sur http://neverssl.com

![Trafic HTTP capture - requetes GET et reponses 200 OK](/images/wireshark/http-trafic-clair.png)

On voit : GET / HTTP/1.1, HTTP/1.1 200 OK, HTTP/1.1 301 Moved Permanently, port 80.

Clic droit sur un paquet HTTP puis Suivre puis Flux TCP :

![Flux TCP HTTP reconstitue - tout est lisible](/images/wireshark/http-flux-tcp.png)

En rouge (ma requete) visible par n'importe qui sur le reseau : Host neverssl.com, User-Agent Mozilla Windows NT 10.0, Accept-Language fr-FR.
En bleu (reponse serveur) : HTTP/1.1 200 OK, Server Apache/2.4.66.

Sur un Wi-Fi public, n'importe qui avec Wireshark verrait ces donnees en clair.

### HTTPS chiffre

Filtre : tls
Navigation sur https://github.com

![Trafic HTTPS capture - tout est chiffre](/images/wireshark/https-trafic-chiffre.png)

Difference immediate : TLSv1.2/1.3, Application Data illisible, port 443. Aucun header visible.

Filtre pour isoler le Client Hello : tls.handshake.type == 1

![SNI visible dans le Client Hello TLS](/images/wireshark/tls-client-hello-sni.png)

Server Name lh3.googleusercontent.com visible en clair dans le SNI. On peut savoir quels sites sont visites meme en HTTPS.

| | HTTP | HTTPS |
|---|---|---|
| Contenu visible | Oui | Non |
| Port | 80 | 443 |
| Site visite visible | Oui | Oui via SNI |

---

## Phase 3 - ARP et ICMP

### ARP

Filtre : arp

![Requetes ARP broadcasts sur le reseau local](/images/wireshark/arp-who-has.png)

ARP resout les IP en adresses MAC. On voit les broadcasts Who has 192.168.x.x Tell 192.168.1.24

![Details d'un paquet ARP](/images/wireshark/arp-details.png)

### ICMP

Filtre : icmp
Ping vers 127.0.0.1 (loopback)

![Paquets ICMP Echo Request et Echo Reply](/images/wireshark/icmp-echo-request-reply.png)

Chaque ping genere une paire : Echo Request type 8 et Echo Reply type 0.

![Details d'un paquet ICMP](/images/wireshark/icmp-details.png)

Le ping vers 8.8.8.8 etait bloque par le pare-feu Windows. C'est une configuration courante en entreprise pour eviter la reconnaissance reseau.

---

## Statistiques globales

Menu Statistiques puis Hierarchie des protocoles :

![Hierarchie des protocoles Wireshark](/images/wireshark/stats-protocoles.png)

Menu Statistiques puis Conversations :

![Tableau des conversations reseau](/images/wireshark/stats-conversations.png)

Ces statistiques permettent d'identifier une machine qui genere un volume anormal de trafic, signe possible d'une infection ou d'une exfiltration de donnees.

---

## Filtres essentiels a retenir

| Filtre | Usage |
|---|---|
| dns | Tout le trafic DNS |
| http | Trafic HTTP non chiffre |
| tls | Trafic HTTPS/TLS |
| arp | Resolution adresses MAC |
| icmp | Ping et erreurs reseau |
| tcp.flags.syn == 1 | Handshakes TCP |
| tls.handshake.type == 1 | Client Hello TLS |
| ip.addr == x.x.x.x | Filtrer par IP |

---

## Ce que j'aurais fait differemment

- Annoter les paquets directement dans Wireshark
- Faire les captures sur une VM pour moins de bruit de fond
- Simuler un ARP scan depuis Kali Linux pour observer une reconnaissance reseau

## Prochaine etape

Integrer Wireshark dans le homelab pfSense pour analyser le trafic inter-VLANs et verifier que les regles de pare-feu bloquent bien le trafic non autorise.
'@
[System.IO.File]::WriteAllText("C:\Users\mouha\Documents\tech-blog\content\labs\wireshark-analyse-trafic.md", $content, [System.Text.Encoding]::UTF8)
git add .
git commit -m "fix: réécriture article Wireshark sans caracteres speciaux"
git push