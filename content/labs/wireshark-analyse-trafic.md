---
title: "Analyse de trafic réseau avec Wireshark"
date: 2026-06-23
draft: false
tags: ["Wireshark", "Réseau", "TCP/IP", "DNS", "HTTP", "ARP", "ICMP"]
categories: ["Labs"]
description: "Capture et analyse de trafic DNS, HTTP/HTTPS, ARP et ICMP sur un réseau local avec Wireshark."
showToc: true
TocOpen: false
cover:
  alt: "Capture Wireshark du trafic réseau local"
---

## Objectif du lab

Ce lab a pour but d'analyser le trafic réseau en temps réel avec Wireshark afin de comprendre le fonctionnement concret des protocoles DNS, HTTP/HTTPS, ARP et ICMP. Chaque protocole est isolé avec un filtre d'affichage, observé et documenté avec des captures d'écran réelles.

**Environnement :**
| Élément | Détail |
|---|---|
| OS | Windows 11 |
| Outil | Wireshark 4.6.6 + Npcap |
| Interface capturée | Wi-Fi (réseau local) |
| Durée | 4h30 |

---

## Préparation — Premier coup d'oeil sur le trafic

Au démarrage de la capture sur l'interface Wi-Fi, Wireshark affiche en temps réel tous les paquets qui transitent par la carte réseau. Sans aucun filtre, le volume est impressionnant — des centaines de paquets par seconde provenant de dizaines de protocoles différents.

![Capture Wireshark globale sans filtre — trafic en temps réel](/images/wireshark/wireshark-capture-globale.png)

C'est pour ça que les filtres sont essentiels : ils permettent d'isoler exactement le protocole qu'on veut analyser sans se noyer dans le bruit de fond.

---

## Phase 1 — Analyse du trafic DNS

### Pourquoi DNS ?

DNS (Domain Name System) est le protocole qui traduit les noms de domaine en adresses IP. C'est le premier protocole sollicité lors de chaque navigation web. Sans DNS, il faudrait connaître l'adresse IP de chaque site par coeur.

### Filtre utilisé
```
dns
```

### Manipulations effectuées

Dans PowerShell, j'ai lancé des requêtes DNS manuelles pour générer du trafic contrôlé :

```powershell
nslookup google.com
nslookup github.com
nslookup nonexistant-xyz123.com
```

### Ce que j'ai observé

#### Échange DNS complet — Query et Response

Wireshark montre clairement les échanges en paires : une requête (**Standard query**) suivie d'une réponse (**Standard query response**). Les requêtes partent de mon PC (192.168.1.24) vers ma box (192.168.1.1) qui fait office de résolveur DNS.

![Liste des paquets DNS avec les échanges Query et Response visibles](/images/wireshark/dns-query-response.png)

En cliquant sur un paquet de réponse, le panneau de détails affiche toute la structure du paquet DNS couche par couche :

![Détails du paquet DNS sélectionné — structure complète visible](/images/wireshark/dns-response-detail.png)

On voit clairement :
- **Frame** : informations sur la trame physique
- **Ethernet II** : adresses MAC source et destination
- **Internet Protocol** : adresses IP (192.168.1.1 → 192.168.1.24)
- **UDP port 53** : DNS utilise UDP et non TCP, pour la rapidité
- **Domain Name System (response)** : la réponse DNS elle-même

#### Structure interne de la réponse DNS

En déroulant la section DNS, on accède aux métadonnées de la réponse :

![Structure interne du paquet DNS — Transaction ID, Flags, Answer RRs](/images/wireshark/dns-details-ttl.png)

- **Transaction ID** : identifiant unique qui permet d'associer une réponse à sa requête
- **Flags : Standard query response, No error** : la requête a abouti sans erreur
- **Answer RRs: 2** : la réponse contient 2 enregistrements

#### IP résolue et TTL dans les Answers

La section **Answers** contient le résultat concret de la résolution DNS :

![Section Answers déroulée — CNAME et adresse IP résolue visibles](/images/wireshark/dns-answers-ttl.png)

Pour `alive.github.com`, on voit une chaîne de résolution :
1. `alive.github.com` → CNAME vers `live.github.com`
2. `live.github.com` → IP **140.82.113.25**

En déroulant davantage, on accède au **TTL (Time To Live)** :

![TTL visible dans les détails de l'enregistrement DNS](/images/wireshark/dns-ttl.png)

Le TTL indique combien de secondes cette réponse peut être mise en cache. Un TTL de 60 signifie que le résolveur devra refaire la requête après 1 minute. C'est un mécanisme d'équilibrage entre fraîcheur des données et charge des serveurs DNS.

> **Conclusion DNS :** DNS fonctionne en UDP sur le port 53. Chaque navigation web commence par une résolution DNS. Le TTL contrôle la durée de vie du cache. Un TTL court = serveur DNS sollicité souvent. Un TTL long = risque de cache périmé.

---

## Phase 2 — HTTP vs HTTPS : la différence visible

### Pourquoi cette comparaison est importante ?

HTTP et HTTPS transportent les mêmes données, mais de manière radicalement différente en termes de sécurité. Wireshark permet de voir cette différence de façon concrète et irréfutable.

### Filtre HTTP — trafic en clair

```
http
```

Navigation sur `http://neverssl.com` (site volontairement non chiffré) :

![Trafic HTTP capturé — requêtes GET et réponses 200 OK visibles en clair](/images/wireshark/http-trafic-clair.png)

On voit immédiatement :
- `GET / HTTP/1.1` : mon navigateur demande la page d'accueil
- `HTTP/1.1 200 OK` : le serveur répond avec succès
- `HTTP/1.1 301 Moved Permanently` : une redirection détectée
- **Port 80** : HTTP utilise le port 80

#### Le flux TCP reconstitué — données en clair

En faisant clic droit sur un paquet HTTP → **Suivre → Flux TCP**, Wireshark reconstitue toute la conversation entre mon navigateur et le serveur :

![Flux TCP HTTP reconstitué — requête en rouge, réponse en bleu, tout est lisible](/images/wireshark/http-flux-tcp.png)

En **rouge** (ma requête) — visible par n'importe qui sur le réseau :
- `GET / HTTP/1.1`
- `Host: neverssl.com`
- `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)...` → mon OS et navigateur
- `Accept-Language: fr-FR` → ma langue

En **bleu** (réponse du serveur) :
- `HTTP/1.1 200 OK`
- `Server: Apache/2.4.66` → le serveur web utilisé
- `Content-Type: text/html; charset=UTF-8`

**Sur un réseau Wi-Fi public non sécurisé, n'importe quelle personne avec Wireshark verrait exactement ces informations.** Si le site avait demandé un mot de passe en HTTP, il serait visible en clair ici.

### Filtre TLS — trafic chiffré

```
tls
```

Navigation sur `https://github.com` :

![Trafic HTTPS capturé — tout est chiffré, seul "Application Data" est visible](/images/wireshark/https-trafic-chiffre.png)

La différence est immédiate :
- **Protocol : TLSv1.2 / TLSv1.3** au lieu de HTTP
- **Info : Application Data** — le contenu est illisible, complètement chiffré
- **Port 443** au lieu de 80
- Aucun GET, aucun Host, aucun User-Agent visible

#### Le SNI — ce qu'on peut quand même voir en HTTPS

Même en HTTPS, le **Client Hello TLS** (début du handshake) contient une information visible en clair : le **SNI (Server Name Indication)**. C'est le nom du site demandé, nécessaire pour que le serveur sache quel certificat présenter.

Filtre pour isoler les Client Hello :
```
tls.handshake.type == 1
```

![Extension server_name déroulée — SNI lh3.googleusercontent.com visible en clair](/images/wireshark/tls-client-hello-sni.png)

On voit **Server Name: lh3.googleusercontent.com** en clair, même si tout le reste est chiffré. Un observateur sur le réseau peut donc savoir **quels sites tu visites** même en HTTPS, mais pas **ce que tu y fais**.

| | HTTP | HTTPS |
|---|---|---|
| Contenu visible | ✅ Tout en clair | ❌ Chiffré |
| Port | 80 | 443 |
| User-Agent visible | ✅ Oui | ❌ Non |
| Site visité visible | ✅ Oui | ✅ Oui (SNI) |
| Protocole | HTTP/1.1 | TLSv1.2 / TLSv1.3 |

> **Conclusion HTTP/HTTPS :** HTTP expose tout le contenu en clair sur le réseau. HTTPS chiffre le contenu mais le nom du site reste visible via le SNI. C'est pourquoi HTTPS est obligatoire mais ne rend pas la navigation totalement anonyme.

---

## Phase 3 — ARP et ICMP : le fonctionnement réseau local

### ARP — Address Resolution Protocol

#### Pourquoi ARP existe ?

Sur un réseau local, les machines communiquent via des adresses MAC (physiques), pas des adresses IP. ARP est le protocole qui fait le lien entre les deux : quand mon PC veut envoyer un paquet à 192.168.1.1, il doit d'abord connaître l'adresse MAC de cette IP. ARP envoie un broadcast pour le demander à tout le réseau.

```
arp
```

![Requêtes ARP — broadcasts "Who has X? Tell Y" sur le réseau local](/images/wireshark/arp-who-has.png)

On voit les broadcasts ARP : `Who has 192.168.x.x? Tell 192.168.1.24` — mon PC cherche les adresses MAC des machines sur le réseau.

En cliquant sur un paquet ARP et en déroulant les détails :

![Détails d'un paquet ARP — adresses MAC et IP source/destination visibles](/images/wireshark/arp-details.png)

On voit la structure complète d'une requête ARP :
- **Sender MAC / Sender IP** : qui pose la question
- **Target MAC** : 00:00:00:00:00:00 (inconnu, c'est ce qu'on cherche)
- **Target IP** : l'IP dont on cherche la MAC

### ICMP — Internet Control Message Protocol (Ping)

#### Le ping en action

```
icmp
```

Ping vers le loopback (127.0.0.1) — mon PC se ping lui-même :

```powershell
ping 127.0.0.1
```

![Paquets ICMP Echo Request et Echo Reply — ping visible dans Wireshark](/images/wireshark/icmp-echo-request-reply.png)

Chaque ping génère une paire de paquets :
- **Echo (ping) request** (type 8) : envoyé par mon PC
- **Echo (ping) reply** (type 0) : réponse de la destination

En déroulant les détails d'un paquet ICMP :

![Détails d'un paquet ICMP — type, code, checksum et données visibles](/images/wireshark/icmp-details.png)

On voit :
- **Type : 8** (Echo Request) ou **Type : 0** (Echo Reply)
- **Code : 0**
- **Checksum** : vérification d'intégrité du paquet
- **Sequence number** : numéro de séquence pour associer request et reply

> **Note :** Le ping vers 8.8.8.8 et 192.168.1.1 était bloqué par le pare-feu Windows. C'est une configuration de sécurité courante en entreprise — les administrateurs bloquent ICMP pour éviter la reconnaissance réseau par des attaquants.

---

## Statistiques globales de la capture

### Résumé des protocoles

Le menu **Statistiques → Hiérarchie des protocoles** donne une vue globale de tous les protocoles détectés pendant la capture et leur proportion dans le trafic total :

![Hiérarchie des protocoles dans Wireshark — répartition du trafic par protocole](/images/wireshark/stats-protocoles.png)

### Conversations réseau

Le menu **Statistiques → Conversations** identifie toutes les paires source-destination qui ont communiqué, classées par volume de trafic :

![Tableau des conversations réseau — IP qui génèrent le plus de trafic](/images/wireshark/stats-conversations.png)

Ces statistiques sont très utiles en contexte de supervision : elles permettent d'identifier rapidement une machine qui génère un volume anormal de trafic, ce qui peut indiquer une infection malware ou une exfiltration de données.

---

## Filtres Wireshark essentiels à retenir

| Filtre | Usage |
|---|---|
| `dns` | Tout le trafic DNS |
| `http` | Trafic HTTP non chiffré |
| `tls` | Trafic HTTPS/TLS chiffré |
| `arp` | Résolution d'adresses MAC |
| `icmp` | Ping et messages d'erreur réseau |
| `tcp.flags.syn == 1` | Handshakes TCP (nouvelles connexions) |
| `tls.handshake.type == 1` | Client Hello TLS (SNI visible) |
| `ip.addr == x.x.x.x` | Filtrer par adresse IP |
| `dns and ip.dst == 8.8.8.8` | Requêtes DNS vers un serveur précis |
| `arp.duplicate-address-detected` | Conflits d'adresses IP |

---

## Ce que j'aurais fait différemment

- Faire les captures sur une VM dédiée avec du trafic plus contrôlé pour moins de bruit de fond
- Annoter directement les paquets clés dans Wireshark avec des commentaires (clic droit → Ajouter un commentaire)
- Simuler un ARP scan depuis Kali Linux pour observer à quoi ressemble une reconnaissance réseau en vrai
- Capturer un vrai échange DNS NXDOMAIN pour un domaine inexistant — le pare-feu a filtré mes requêtes nslookup

## Prochaine étape

Intégrer Wireshark dans le **homelab virtualisé** (pfSense + VLANs) pour analyser le trafic inter-VLANs et vérifier que les règles de pare-feu bloquent bien le trafic non autorisé entre segments réseau.