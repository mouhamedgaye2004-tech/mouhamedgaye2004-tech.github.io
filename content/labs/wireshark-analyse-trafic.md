---
title: "Analyse de trafic reseau avec Wireshark"
date: 2026-06-23
draft: false
tags: ["Wireshark", "Reseau", "TCP/IP", "DNS", "HTTP", "ARP", "ICMP"]
categories: ["Labs"]
description: "Capture et analyse de trafic DNS, HTTP/HTTPS, ARP et ICMP sur un reseau local avec Wireshark."
showToc: true
TocOpen: false
---

## Objectif du lab

Ce lab a pour but de comprendre comment les machines communiquent sur un reseau en capturant et analysant le trafic en temps reel avec Wireshark.

Wireshark est un **analyseur de paquets reseau** (ou sniffer). Il intercepte tous les paquets qui transitent par la carte reseau et les affiche de facon lisible. C'est l'outil numero 1 des administrateurs reseau et des analystes en securite pour diagnostiquer des problemes ou detecter des comportements suspects.

**Environnement utilise :**

| Element | Detail |
|---|---|
| OS | Windows 11 |
| Outil principal | Wireshark 4.6.6 |
| Pilote de capture | Npcap |
| Interface reseau | Wi-Fi |
| Duree du lab | 4h30 |

---

## Preparation de l'environnement

### Pourquoi Npcap est necessaire ?

Wireshark seul ne peut pas capturer de paquets. Il a besoin d'un pilote systeme appele **Npcap** qui s'intercale entre la carte reseau et Windows pour intercepter les paquets avant qu'ils soient traites par le systeme.

Sans Npcap, Wireshark affiche le message "Les interfaces locales ne sont pas disponibles".

### Premier demarrage

Au lancement, Wireshark affiche la liste de toutes les interfaces reseau disponibles. On peut voir en temps reel quelles interfaces ont du trafic grace aux courbes animees.

![](/images/wireshark/wireshark-capture-globale.png)

On selectionne l'interface **Wi-Fi** (celle qui a une courbe qui bouge) et on double-clique pour demarrer la capture. Des centaines de paquets apparaissent immediatement. C'est tout le trafic reseau de notre machine en temps reel.

Sans filtre, c'est illisible. C'est pourquoi Wireshark propose des **filtres d'affichage** qui permettent d'isoler exactement le protocole qu'on veut analyser.

---

## Phase 1 - Analyse du protocole DNS

### Qu'est-ce que DNS ?

DNS signifie **Domain Name System**. C'est le protocole qui traduit les noms de domaine lisibles par les humains (comme google.com) en adresses IP utilisables par les machines (comme 142.250.75.46).

Sans DNS, il faudrait connaitre l'adresse IP de chaque site web par coeur. DNS fonctionne en **UDP sur le port 53**.

### Filtre utilise

Dans la barre de filtre Wireshark, on tape :

```
dns
```

### Manipulations effectuees

Dans PowerShell, on lance des requetes DNS manuelles pour generer du trafic controle :

```powershell
nslookup google.com
nslookup github.com
nslookup nonexistant-xyz123.com
```

### Ce que j'ai observe

#### Les echanges DNS en paires

Wireshark affiche les requetes et reponses DNS. Chaque echange se fait en deux paquets :
- **Standard query** : mon PC demande l'IP d'un domaine
- **Standard query response** : le serveur DNS repond avec l'IP

![](/images/wireshark/dns-query-response.png)

On voit clairement dans la colonne Info :
- Les requetes partent de mon IP (192.168.1.24) vers ma box (192.168.1.1)
- Les reponses reviennent de ma box vers mon PC

| Colonne | Signification |
|---|---|
| No. | Numero du paquet dans la capture |
| Time | Temps ecoule depuis le debut de la capture |
| Source | IP qui envoie le paquet |
| Destination | IP qui recoit le paquet |
| Protocol | Protocole utilise (DNS, HTTP, TLS...) |
| Info | Resume du contenu du paquet |

#### La structure interne d'un paquet DNS

En cliquant sur un paquet, le panneau du bas affiche toutes les couches du modele OSI :

![](/images/wireshark/dns-response-detail.png)

On voit les 5 couches empilees :
1. **Frame** : informations sur la trame physique (taille en octets)
2. **Ethernet II** : adresses MAC source et destination
3. **Internet Protocol** : adresses IP (192.168.1.1 vers 192.168.1.24)
4. **UDP port 53** : DNS utilise UDP et non TCP car c'est plus rapide pour des petites requetes
5. **Domain Name System** : le contenu de la reponse DNS

#### Le detail de la reponse DNS

En deroulant la section Domain Name System :

![](/images/wireshark/dns-details-ttl.png)

| Champ | Valeur | Signification |
|---|---|---|
| Transaction ID | 0x4afe | Identifiant unique qui associe la requete a sa reponse |
| Flags | Standard query response, No error | La requete a abouti sans erreur |
| Questions | 1 | Une seule question posee |
| Answer RRs | 2 | La reponse contient 2 enregistrements |

#### L'IP resolue et le TTL

La section **Answers** contient le resultat concret :

![](/images/wireshark/dns-answers-ttl.png)

Pour alive.github.com, on voit une chaine de resolution :
1. alive.github.com -> CNAME vers live.github.com (alias)
2. live.github.com -> IP **140.82.113.25** (adresse finale)

En deroulant encore plus, on voit le **TTL (Time To Live)** :

![](/images/wireshark/dns-ttl.png)

Le TTL indique combien de secondes cette reponse peut etre mise en cache. Un TTL de 60 signifie que le systeme doit refaire la requete DNS apres 1 minute. Un TTL de 3600 signifie qu'il peut reutiliser la reponse pendant 1 heure sans recontacter le serveur DNS.

**Pourquoi c'est important en securite ?** Un attaquant qui fait du DNS poisoning va modifier les enregistrements DNS pour rediriger les utilisateurs vers un faux site. Avec Wireshark, on peut detecter des reponses DNS anormales (IP inconnue, TTL bizarre).

---

## Phase 2 - HTTP vs HTTPS : la difference visible

### Pourquoi cette comparaison est importante ?

HTTP et HTTPS transportent les memes donnees (pages web, formulaires, cookies) mais de maniere radicalement differente. Wireshark permet de voir cette difference de facon concrete.

### HTTP en clair - filtre http

On tape dans Wireshark :

```
http
```

Puis on navigue sur **http://neverssl.com** (un site volontairement non chiffre, utilise pour tester les portails captifs).

![](/images/wireshark/http-trafic-clair.png)

On voit immediatement plusieurs types de paquets :

| Type de paquet | Signification |
|---|---|
| GET / HTTP/1.1 | Le navigateur demande la page d'accueil |
| HTTP/1.1 200 OK | Le serveur repond avec succes et envoie la page |
| HTTP/1.1 301 Moved Permanently | Le serveur redirige vers une autre URL |
| GET /favicon.ico | Le navigateur demande l'icone du site |

#### Le flux TCP reconstitue

En faisant **clic droit sur un paquet HTTP -> Suivre -> Flux TCP**, Wireshark reconstitue toute la conversation entre le navigateur et le serveur :

![](/images/wireshark/http-flux-tcp.png)

En **rouge** (ce que mon navigateur envoie) :
- `GET / HTTP/1.1` : methode et chemin demandes
- `Host: neverssl.com` : nom du site
- `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)` : mon OS et navigateur
- `Accept-Language: fr-FR` : ma langue

En **bleu** (ce que le serveur repond) :
- `HTTP/1.1 200 OK` : succes
- `Server: Apache/2.4.66` : le logiciel serveur utilise
- `Content-Type: text/html` : type de contenu envoye

**Conclusion securite :** Sur un Wi-Fi public non securise, n'importe quelle personne avec Wireshark peut voir exactement ces informations. Si le site demandait un mot de passe en HTTP, il serait visible en clair ici.

### HTTPS chiffre - filtre tls

On change le filtre :

```
tls
```

Puis on navigue sur **https://github.com**.

![](/images/wireshark/https-trafic-chiffre.png)

La difference est immediate :

| | HTTP | HTTPS |
|---|---|---|
| Protocole visible | HTTP/1.1 | TLSv1.2 ou TLSv1.3 |
| Port utilise | 80 | 443 |
| Contenu lisible | Oui, tout en clair | Non, chiffre |
| Headers visibles | Oui (Host, User-Agent...) | Non |
| Info affichee | GET, 200 OK, contenu | Application Data uniquement |

#### Le SNI - ce qu'on peut quand meme voir en HTTPS

Meme en HTTPS, il existe une information visible en clair : le **SNI (Server Name Indication)**. C'est le nom du site demande, transmis au debut de la connexion TLS pour que le serveur sache quel certificat presenter.

On filtre les Client Hello TLS :

```
tls.handshake.type == 1
```

![](/images/wireshark/tls-client-hello-sni.png)

On voit **Server Name: lh3.googleusercontent.com** en clair. Un observateur sur le reseau peut donc savoir **quels sites tu visites** meme en HTTPS, mais pas **ce que tu y fais**.

---

## Phase 3 - ARP et ICMP : fonctionnement reseau local

### ARP - Address Resolution Protocol

#### Qu'est-ce qu'ARP ?

Sur un reseau local, les machines communiquent via des **adresses MAC** (physiques, gravees dans la carte reseau), pas des adresses IP. ARP est le protocole qui fait le lien entre les deux.

Quand mon PC veut envoyer un paquet a 192.168.1.1 (ma box), il doit d'abord connaitre son adresse MAC. Il envoie un **broadcast ARP** (message a tout le reseau) : "Qui a l'IP 192.168.1.1 ? Donnez-moi son adresse MAC."

Filtre utilise :

```
arp
```

![](/images/wireshark/arp-who-has.png)

On voit les broadcasts ARP avec le message "Who has X.X.X.X? Tell 192.168.1.24". Ce sont des requetes de mon PC pour trouver les adresses MAC des machines sur le reseau.

#### La structure d'un paquet ARP

![](/images/wireshark/arp-details.png)

| Champ | Valeur | Signification |
|---|---|---|
| Sender MAC | Adresse MAC de mon PC | Qui pose la question |
| Sender IP | 192.168.1.24 | Mon IP |
| Target MAC | 00:00:00:00:00:00 | Inconnu, c'est ce qu'on cherche |
| Target IP | 192.168.1.1 | L'IP dont on cherche la MAC |

**En securite :** Un **ARP scan** genere des dizaines de requetes ARP en rafale vers toutes les IPs d'un reseau. C'est la premiere etape d'une reconnaissance reseau par un attaquant. Wireshark permet de le detecter facilement.

### ICMP - Internet Control Message Protocol (Ping)

#### Qu'est-ce qu'ICMP ?

ICMP est le protocole utilise par la commande **ping**. Il permet de verifier si une machine est joignable sur le reseau et de mesurer le temps de reponse.

Filtre utilise :

```
icmp
```

On lance un ping depuis PowerShell :

```powershell
ping 127.0.0.1
```

(127.0.0.1 est le loopback, notre PC se ping lui-meme. Le ping vers 8.8.8.8 etait bloque par le pare-feu Windows, configuration courante en entreprise pour eviter la reconnaissance reseau.)

![](/images/wireshark/icmp-echo-request-reply.png)

Chaque ping genere une paire de paquets :
- **Echo Request (type 8)** : mon PC envoie une demande
- **Echo Reply (type 0)** : la destination repond

#### Details d'un paquet ICMP

![](/images/wireshark/icmp-details.png)

| Champ | Valeur | Signification |
|---|---|---|
| Type | 8 (Request) ou 0 (Reply) | Type de message ICMP |
| Code | 0 | Sous-type (0 = standard) |
| Checksum | Valeur hex | Verification d'integrite du paquet |
| Sequence number | 1, 2, 3... | Permet d'associer chaque request a son reply |

---

## Statistiques globales de la capture

### Hierarchie des protocoles

Le menu **Statistiques -> Hierarchie des protocoles** donne une vue globale de tous les protocoles detectes et leur proportion dans le trafic total :

![](/images/wireshark/stats-protocoles.png)

Cela permet de voir rapidement si un protocole est sur-represente (signe possible d'une activite anormale).

### Conversations reseau

Le menu **Statistiques -> Conversations** identifie toutes les paires qui ont communique, classees par volume de trafic :

![](/images/wireshark/stats-conversations.png)

En entreprise, ces statistiques permettent d'identifier une machine qui genere un volume anormal de trafic, signe possible d'une infection malware, d'un scan de port ou d'une exfiltration de donnees.

---

## Recap des filtres Wireshark a retenir

| Filtre | Usage | Exemple d'utilisation |
|---|---|---|
| dns | Tout le trafic DNS | Verifier les resolutions de noms |
| http | Trafic HTTP non chiffre | Analyser les requetes web en clair |
| tls | Trafic HTTPS chiffre | Voir les connexions securisees |
| arp | Resolution adresses MAC | Detecter un ARP scan |
| icmp | Ping et messages d'erreur | Tester la connectivite |
| tcp.flags.syn == 1 | Nouvelles connexions TCP | Detecter des scans de port |
| tls.handshake.type == 1 | Client Hello TLS | Voir les SNI (sites visites) |
| ip.addr == x.x.x.x | Filtrer par IP | Isoler le trafic d'une machine |
| ip.dst == x.x.x.x | Filtrer par destination | Voir ce qu'une machine recoit |
| dns and ip.dst == 8.8.8.8 | DNS vers serveur precis | Verifier quel DNS est utilise |

---

## Ce que j'aurais fait differemment

- Faire les captures sur une VM dediee pour avoir moins de bruit de fond (moins de protocoles parasites)
- Annoter directement les paquets importants dans Wireshark avec des commentaires (clic droit -> Ajouter un commentaire au paquet)
- Simuler un vrai ARP scan depuis Kali Linux (avec nmap -sn) pour voir a quoi ressemble une reconnaissance reseau en conditions reelles
- Capturer une reponse DNS NXDOMAIN pour un domaine inexistant (le pare-feu a filtre mes requetes nslookup vers les serveurs externes)

## Prochaine etape

Integrer Wireshark dans le **homelab pfSense** pour analyser le trafic inter-VLANs et verifier que les regles de pare-feu bloquent correctement le trafic non autorise entre les segments reseau.
