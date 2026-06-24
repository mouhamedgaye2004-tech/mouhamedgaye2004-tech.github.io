---
title: "Analyse de trafic reseau avec Wireshark"
date: 2026-06-23
draft: false
tags: ["Wireshark", "Reseau", "TCP/IP", "DNS", "HTTP", "ARP", "ICMP"]
categories: ["Labs"]
description: "Capture et analyse de trafic DNS, HTTP/HTTPS, ARP et ICMP sur un reseau local."
showToc: true
---

## Objectif

Analyser le trafic reseau avec Wireshark pour comprendre DNS, HTTP/HTTPS, ARP et ICMP.

Environnement : Windows 11, Wireshark 4.6.6, Npcap, interface Wi-Fi, duree 4h30.

---

## Preparation

![](/images/wireshark/wireshark-capture-globale.png)

Sans filtre, Wireshark affiche tout le trafic. Les filtres permettent d isoler un protocole specifique.

---

## Phase 1 - DNS

DNS traduit les noms de domaine en adresses IP. Filtre utilise : dns

Commandes PowerShell pour generer du trafic DNS :
nslookup google.com
nslookup github.com

![](/images/wireshark/dns-query-response.png)

On voit les echanges en paires : Standard query puis Standard query response.

![](/images/wireshark/dns-response-detail.png)

Couches visibles : Ethernet, IP, UDP port 53, DNS.

![](/images/wireshark/dns-details-ttl.png)

Transaction ID, Flags No error, Answer RRs 2.

![](/images/wireshark/dns-answers-ttl.png)

alive.github.com resolu en 140.82.113.25 via CNAME live.github.com.

![](/images/wireshark/dns-ttl.png)

Le TTL indique la duree de mise en cache de la reponse DNS.

---

## Phase 2 - HTTP vs HTTPS

Filtre http, navigation sur http://neverssl.com

![](/images/wireshark/http-trafic-clair.png)

On voit GET / HTTP/1.1, 200 OK, 301 Redirect, port 80.

Clic droit sur paquet HTTP, Suivre, Flux TCP :

![](/images/wireshark/http-flux-tcp.png)

Tout est visible en clair : Host, User-Agent, langue, reponse serveur Apache.
Sur un Wi-Fi public, n importe qui verrait ces donnees.

Filtre tls, navigation sur https://github.com

![](/images/wireshark/https-trafic-chiffre.png)

Contenu chiffre, TLSv1.3, port 443, rien de lisible.

Filtre tls.handshake.type == 1 pour voir le Client Hello :

![](/images/wireshark/tls-client-hello-sni.png)

Le SNI reste visible en clair : on sait quel site est visite meme en HTTPS.

---

## Phase 3 - ARP et ICMP

Filtre arp :

![](/images/wireshark/arp-who-has.png)

ARP resout les IP en MAC. Broadcasts Who has X Tell Y visibles.

![](/images/wireshark/arp-details.png)

Filtre icmp, ping 127.0.0.1 :

![](/images/wireshark/icmp-echo-request-reply.png)

Echo Request type 8 et Echo Reply type 0 par ping.

![](/images/wireshark/icmp-details.png)

---

## Statistiques

Menu Statistiques, Hierarchie des protocoles :

![](/images/wireshark/stats-protocoles.png)

Menu Statistiques, Conversations :

![](/images/wireshark/stats-conversations.png)

Permet d identifier les machines qui generent le plus de trafic.

---

## Filtres a retenir

dns - trafic DNS
http - HTTP non chiffre
tls - HTTPS chiffre
arp - resolution MAC
icmp - ping
tcp.flags.syn == 1 - handshakes TCP
tls.handshake.type == 1 - Client Hello
ip.addr == x.x.x.x - filtrer par IP

---

## Prochaine etape

Integrer Wireshark dans le homelab pfSense pour analyser le trafic inter-VLANs.
