---
title: "Serveur DHCP Ubuntu - Clients Wifislax et Windows 7"
date: 2026-06-24
draft: false
tags: ["DHCP", "Ubuntu", "Reseau", "Linux", "Administration"]
categories: ["Labs"]
description: "Mise en place d'un serveur DHCP sous Ubuntu Server avec distribution automatique d'adresses IP a des clients Wifislax et Windows 7 en reseau Host-Only."
showToc: true
TocOpen: false
---

## Objectif du lab

Ce lab a pour but de mettre en place un **serveur DHCP** sous Ubuntu Server dans un environnement virtualise. Le serveur distribuera automatiquement des adresses IP aux machines clientes du reseau.

**Qu'est-ce que DHCP ?**

DHCP signifie **Dynamic Host Configuration Protocol**. C'est le protocole qui permet a un serveur de distribuer automatiquement des adresses IP aux machines qui se connectent au reseau. Sans DHCP, il faudrait configurer manuellement l'adresse IP de chaque machine — une tache fastidieuse et source d'erreurs dans un reseau de plusieurs dizaines de machines.

En entreprise, le serveur DHCP est l'un des premiers services configures par l'administrateur reseau. C'est un service critique : si le DHCP tombe, les nouvelles machines ne peuvent plus obtenir d'adresse IP et ne peuvent donc plus communiquer sur le reseau.

**Architecture du lab :**

| Machine | Systeme | Role | Adresse IP |
|---|---|---|---|
| Serveur | Ubuntu Server | Serveur DHCP | 192.168.56.10 (statique) |
| Routeur | VirtualBox Host-Only | Passerelle | 192.168.56.1 |
| Client 1 | Wifislax | Client DHCP | Dynamique |
| Client 2 | Windows 7 (VM) | Client DHCP | Dynamique |

Le reseau utilise est de type **Host-Only** (192.168.56.0/24), ce qui permet aux VMs de communiquer entre elles sans acces a Internet. C'est ideal pour un lab isole.

---

## Partie 01 - Configuration IP statique du serveur

### Pourquoi une IP statique sur le serveur DHCP ?

Le serveur DHCP doit avoir une **adresse IP fixe**. Si son IP changeait a chaque demarrage, les clients ne sauraient pas vers quelle adresse envoyer leurs requetes DHCP. C'est une regle fondamentale : tout serveur (DHCP, DNS, web, fichiers) doit avoir une IP statique.

### Identification de l'interface reseau

Avant de configurer l'IP, on identifie le nom de l'interface reseau :

```bash
ip a
```

L'interface utilisee dans ce lab est `enp0s3`. Ce nom peut varier selon la machine (eth0, ens33, enp0s8...).

### Configuration via Netplan

Ubuntu Server utilise **Netplan** pour la configuration reseau. Le fichier de configuration se trouve dans `/etc/netplan/`.

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

![](/images/dhcp-netplan.png)

Configuration appliquee :

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.56.10/24
      routes:
        - to: default
          via: 192.168.56.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

**Explication de chaque parametre :**

| Parametre | Valeur | Role |
|---|---|---|
| dhcp4: no | false | Desactive l'attribution DHCP pour ce serveur |
| addresses | 192.168.56.10/24 | IP statique du serveur + masque /24 |
| to: default | Route par defaut | Tout le trafic non local passe par la passerelle |
| via | 192.168.56.1 | Adresse de la passerelle (routeur VirtualBox) |
| nameservers | 8.8.8.8 | Serveur DNS Google utilise pour la resolution de noms |

### Application de la configuration

```bash
sudo netplan apply
```

Cette commande applique la nouvelle configuration reseau sans redemarrage. On verifie ensuite avec `ip a` que l'adresse 192.168.56.10 est bien configuree sur l'interface.

---

## Partie 02 - Installation du serveur DHCP

### Le service ISC DHCP Server

Le serveur DHCP utilise dans ce lab est **ISC DHCP Server** (Internet Systems Consortium). C'est l'implementation de reference du protocole DHCP sous Linux, utilisee dans de nombreuses entreprises et organisations.

### Installation

```bash
sudo apt update
sudo apt install isc-dhcp-server
```

Apres l'installation, le service est installe mais pas encore configure. Il est donc en etat `inactive` ou `failed` — c'est normal a ce stade.

```bash
systemctl status isc-dhcp-server
```

---

## Partie 03 - Configuration du service DHCP

### Le fichier principal dhcpd.conf

C'est le coeur de la configuration DHCP. Il definit la plage d'adresses a distribuer, la passerelle, le DNS et les durees de bail.

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

![](/images/dhcp-config.png)

Configuration appliquee :

```
authoritative;

subnet 192.168.56.0 netmask 255.255.255.0 {
  range 192.168.56.54 192.168.56.80;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  option domain-name "local.lab";
  option subnet-mask 255.255.255.0;
  option routers 192.168.56.1;
  option broadcast-address 192.168.56.255;
  default-lease-time 600;
  max-lease-time 7200;
}
```

**Explication de chaque parametre :**

| Parametre | Valeur | Role |
|---|---|---|
| authoritative | - | Ce serveur est la reference DHCP du reseau |
| subnet | 192.168.56.0/24 | Reseau sur lequel le DHCP distribue des IPs |
| range | .54 a .80 | Plage d'adresses attribuables aux clients |
| option routers | 192.168.56.1 | Passerelle par defaut fournie aux clients |
| option domain-name-servers | 8.8.8.8 | Serveur DNS fourni aux clients |
| default-lease-time | 600 | Duree de bail par defaut en secondes (10 min) |
| max-lease-time | 7200 | Duree maximale de validite d'une IP (2h) |

**Qu'est-ce qu'un bail DHCP ?** Quand un client obtient une IP via DHCP, cette attribution est temporaire. Le bail definit combien de temps le client peut garder cette IP. Avant l'expiration, le client renouvelle automatiquement son bail. En entreprise, on configure des baux plus longs (24h ou plus) pour eviter trop de trafic DHCP sur le reseau.

### Definition de l'interface d'ecoute

Il faut indiquer sur quelle interface le service DHCP doit ecouter les requetes des clients :

```bash
sudo nano /etc/default/isc-dhcp-server
```

![](/images/dhcp-interface.png)

On modifie la ligne :

```
INTERFACESv4="enp0s3"
```

Cela indique au service d'ecouter les requetes DHCP uniquement sur l'interface `enp0s3` (celle connectee au reseau Host-Only).

---

## Partie 04 - Demarrage et verification du service

### Demarrage du service

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

La commande `enable` garantit que le service demarrera automatiquement au prochain boot du serveur.

### Verification de l'etat

```bash
sudo systemctl status isc-dhcp-server
```

![](/images/dhcp-service-status.png)

Le service doit afficher `active (running)`. Si on voit `failed`, c'est generalement une erreur de syntaxe dans `dhcpd.conf` — on consulte les logs avec `journalctl -u isc-dhcp-server`.

| Etat | Signification | Action |
|---|---|---|
| active (running) | Service fonctionnel | Passer aux tests |
| inactive | Service arrete | sudo systemctl start isc-dhcp-server |
| failed | Erreur de configuration | Verifier dhcpd.conf avec journalctl |

---

## Partie 05 - Tests et validation

### Test avec Wifislax

Sur le client Wifislax (Linux), on demande une adresse IP au serveur DHCP :

```bash
dhclient
ip a
```

![](/images/dhcp-wifislax.png)

Le client a bien recu une adresse IP dans la plage definie (entre 192.168.56.54 et 192.168.56.80). On voit aussi la passerelle et le DNS correctement configures.

### Test avec Windows 7

Sur le client Windows 7, on utilise les commandes ipconfig :

```cmd
ipconfig /release
ipconfig /renew
ipconfig
```

![](/images/dhcp-windows7.png)

| Commande | Role |
|---|---|
| ipconfig /release | Libere l'adresse IP actuelle |
| ipconfig /renew | Demande une nouvelle adresse IP au serveur DHCP |
| ipconfig | Affiche la configuration IP actuelle |

Windows 7 a bien obtenu l'adresse 192.168.56.56, le masque 255.255.255.0, la passerelle 192.168.56.1 et le serveur DNS.

### Test de connectivite

On verifie que les machines peuvent communiquer entre elles. Depuis Windows 7, on ping le client Wifislax :

```cmd
ping 192.168.56.57
```

![](/images/dhcp-ping.png)

4 paquets envoyes, 4 recus, 0 perdus. La communication entre les deux clients via le reseau Host-Only fonctionne correctement.

---

## Partie 06 - Fonctionnement du protocole DHCP (cycle DORA)

Le protocole DHCP fonctionne selon un cycle en 4 etapes appele **DORA** :

| Etape | Message | Description |
|---|---|---|
| D - Discover | DHCPDISCOVER | Le client envoie un broadcast sur le reseau pour trouver un serveur DHCP |
| O - Offer | DHCPOFFER | Le serveur repond en proposant une adresse IP disponible |
| R - Request | DHCPREQUEST | Le client accepte l'offre et la confirme au serveur |
| A - Acknowledge | DHCPACK | Le serveur confirme l'attribution et le client peut utiliser l'IP |

**Schema du cycle DORA :**

```
Client                    Serveur DHCP
  |                            |
  |--- DHCPDISCOVER (broadcast) -->|
  |                            |
  |<-- DHCPOFFER (IP proposee) ----|
  |                            |
  |--- DHCPREQUEST (acceptation) -->|
  |                            |
  |<-- DHCPACK (confirmation) -----|
  |                            |
```

**Pourquoi le DHCPDISCOVER est un broadcast ?** Au moment du Discover, le client ne connait pas encore l'adresse IP du serveur DHCP. Il envoie donc le message a l'adresse 255.255.255.255 (broadcast), ce qui signifie "a toutes les machines du reseau". Seul le serveur DHCP repondra.

---

## Recap des commandes essentielles

**Cote serveur Ubuntu :**

| Commande | Usage |
|---|---|
| sudo netplan apply | Appliquer la configuration reseau |
| sudo systemctl restart isc-dhcp-server | Redemarrer le service DHCP |
| sudo systemctl status isc-dhcp-server | Verifier l'etat du service |
| sudo systemctl enable isc-dhcp-server | Activer le demarrage automatique |
| journalctl -u isc-dhcp-server | Consulter les logs du service |
| cat /var/lib/dhcp/dhcpd.leases | Voir les baux DHCP attribues |

**Cote client Windows :**

| Commande | Usage |
|---|---|
| ipconfig | Afficher la configuration IP |
| ipconfig /release | Liberer l'adresse IP |
| ipconfig /renew | Renouveler l'adresse IP |

**Cote client Linux :**

| Commande | Usage |
|---|---|
| dhclient | Demander une IP au serveur DHCP |
| ip a | Afficher les interfaces et adresses IP |

---

## Ce que j'aurais fait differemment

- Configurer des **reservations DHCP** pour attribuer toujours la meme IP a certaines machines (imprimantes, serveurs) via leur adresse MAC
- Mettre en place un **DHCP failover** avec deux serveurs DHCP pour la haute disponibilite
- Utiliser **Wireshark** pour capturer et analyser le cycle DORA en temps reel sur le reseau
- Tester avec plus de clients pour observer le comportement quand la plage d'adresses est epuisee

## Prochaine etape

Combiner ce serveur DHCP avec un serveur **DNS Bind9** pour avoir une resolution de noms interne sur le reseau local, et integrer le tout dans le homelab pfSense avec des VLANs.
