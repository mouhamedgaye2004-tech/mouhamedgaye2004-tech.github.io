---
title: "Mission CyberSec - Cryptographie et PKI avec OpenSSL"
date: 2026-06-24
draft: false
tags: ["OpenSSL", "Cryptographie", "PKI", "RSA", "AES", "Cybersecurite", "Certificats"]
categories: ["Labs"]
description: "Mise en oeuvre de la cryptographie avec OpenSSL : chiffrement symetrique AES, asymetrique RSA, signatures numeriques et creation d'une autorite de certification."
showToc: true
TocOpen: false
---

## Objectif du lab

Ce lab met en oeuvre les concepts fondamentaux de la **cryptographie appliquee** avec OpenSSL. Le scenario place les etudiants en tant que consultants de l'agence CyberGuard Senegal, missiones pour securiser les communications d'une entreprise Fintech (TechSN Solutions).

**Qu'est-ce qu'OpenSSL ?**

OpenSSL est la boite a outils cryptographique de reference, utilisee dans quasiment tous les systemes Linux et serveurs web. Elle permet de generer des cles, chiffrer des fichiers, creer des certificats SSL/TLS et mettre en place une infrastructure a cles publiques (PKI). C'est l'outil qu'on retrouve derriere chaque cadenas HTTPS dans un navigateur.

**Objectifs de la mission :**

| Phase | Objectif |
|---|---|
| Phase 0 | Inventaire et prise en main d'OpenSSL |
| Phase 1 | Evaluation des vulnerabilites (Base64) |
| Phase 2 | Chiffrement symetrique AES-256 |
| Phase 3 | Cles asymetriques RSA et protocole hybride |
| Phase 4 | Signatures numeriques |
| Phase 5 | Creation d'une Autorite de Certification (PKI) |

---

## Phase 0 - Inventaire et prise en main d'OpenSSL

### Verification de l'environnement

Avant de commencer, on verifie la version d'OpenSSL installee et les algorithmes disponibles :

```bash
openssl version -a
openssl list -commands
openssl list -ciphers
openssl list -digests
```

![](/images/openssl/openssl-01.png)

Cette etape confirme la disponibilite des algorithmes necessaires et la compatibilite de l'environnement.

### Tests d'entropie et performances

```bash
openssl rand -hex 16
openssl speed aes-256-cbc
openssl speed rsa2048
```

![](/images/openssl/openssl-02.png)

**Qu'est-ce que l'entropie ?** En cryptographie, l'entropie designe le caractere aleatoire d'une donnee. Une cle cryptographique doit etre generee avec une entropie maximale pour etre imprevisible. `openssl rand` puise dans les sources d'aleatoire du systeme (`/dev/urandom`) pour generer des donnees vraiment aleatoires.

**Observation sur les performances :** RSA est beaucoup plus lent qu'AES. Plus la taille de la cle RSA augmente, plus les operations sont lentes. C'est pourquoi en pratique on utilise un **protocole hybride** : RSA pour echanger une cle AES, puis AES pour chiffrer les donnees.

---

## Phase 1 - Evaluation des vulnerabilites

### Base64 : faux sentiment de securite

**Base64 n'est pas un chiffrement.** C'est simplement un encodage qui transforme des donnees binaires en texte ASCII. N'importe qui peut decoder du Base64 en quelques secondes.

```bash
echo "URGENT: Mot de passe temporaire: Admin2024!" > secret.txt
openssl enc -base64 -in secret.txt -out message_encode.txt
openssl enc -base64 -d -in message_encode.txt -out message_decode.txt
```

![](/images/openssl/openssl-05.png)

**Resultat :** le message est immediatement recuperable. Base64 augmente meme la taille des donnees d'environ 33% (4 caracteres pour chaque groupe de 3 octets).

**Formule :** Taille encodee = 4 x ceil(n / 3). Pour 100 octets : 4 x 34 = 136 octets.

**Probleme frequent en securite :** des developpeurs stockent des mots de passe ou tokens en Base64 en pensant les "cacher". C'est une erreur grave — Base64 ne protege rien.

---

## Phase 2 - Chiffrement symetrique AES-256

### Qu'est-ce que le chiffrement symetrique ?

Dans le chiffrement **symetrique**, la meme cle sert a chiffrer ET a dechiffrer. C'est rapide et efficace pour de gros volumes de donnees. L'algorithme utilise ici est **AES-256-CBC** :
- **AES** : Advanced Encryption Standard, standard mondial depuis 2001
- **256** : taille de la cle en bits (le plus securise)
- **CBC** : Cipher Block Chaining, mode de chiffrement par blocs

### Mise en place du canal chiffre

```bash
openssl enc -aes-256-cbc -salt -pbkdf2 -in document_classifie.txt -out document_chiffre.enc -k "OperationCoffreFort2024!"
```

![](/images/openssl/openssl-06.png)

**Explication des options :**

| Option | Role |
|---|---|
| -aes-256-cbc | Algorithme de chiffrement utilise |
| -salt | Ajoute un sel aleatoire pour contrer les rainbow tables |
| -pbkdf2 | Derive la cle depuis le mot de passe de facon securisee |
| -in | Fichier d'entree (texte clair) |
| -out | Fichier de sortie (chiffre) |
| -k | Mot de passe utilise pour generer la cle |

### Dechiffrement

```bash
openssl enc -aes-256-cbc -d -pbkdf2 -in document_chiffre.enc -out document_dechiffre.txt -k "OperationCoffreFort2024!"
```

**Observation :** le fichier chiffre est legerement plus volumineux que l'original a cause du padding (alignement sur des blocs de 16 octets) et des metadonnees (sel, IV).

**Limite du chiffrement symetrique :** comment transmettre la cle de facon securisee a l'autre partie ? C'est le probleme de la distribution de cles, resolu par la cryptographie asymetrique.

---

## Phase 3 - Cles asymetriques RSA et protocole hybride

### Qu'est-ce que la cryptographie asymetrique ?

La cryptographie **asymetrique** utilise une **paire de cles** :
- **Cle publique** : peut etre partagee avec tout le monde. Sert a chiffrer.
- **Cle privee** : gardee secrete. Sert a dechiffrer.

Ce qui est chiffre avec la cle publique ne peut etre dechiffre qu'avec la cle privee correspondante. Cela resout le probleme de distribution de cles.

### Generation des paires de cles RSA

```bash
openssl genrsa -out agent_dakar.pem 2048
openssl genrsa -des3 -out agent_saint_louis.pem 2048
openssl rsa -in agent_dakar.pem -pubout -out identite_publique_dakar.pem
```

![](/images/openssl/openssl-09.png)

| Taille de cle | Securite | Usage recommande |
|---|---|---|
| 1024 bits | Insuffisant | Obsolete, a eviter |
| 2048 bits | Correct | Standard actuel |
| 4096 bits | Tres eleve | Autorites de certification |

**L'option -des3** protege la cle privee par un mot de passe. Sans cette option, la cle privee est stockee en clair sur le disque — risque si le fichier est vole.

### Protocole hybride

Le protocole hybride combine les avantages des deux approches :
1. On chiffre le document avec **AES** (rapide pour de gros fichiers)
2. On chiffre la cle AES avec **RSA** (securise pour de petites donnees)

```bash
openssl enc -aes-256-cbc -salt -in plan_strategic.txt -out plan_chiffre.enc -k "CleTemporaire2024"
echo "CleTemporaire2024" > cle_aes.txt
openssl rsautl -encrypt -pubin -inkey identite_publique_binome.pem -in cle_aes.txt -out cle_chiffree.enc
```

![](/images/openssl/openssl-11.png)

**Le destinataire recoit deux fichiers :**
- `plan_chiffre.enc` : le document chiffre avec AES
- `cle_chiffree.enc` : la cle AES chiffree avec RSA

Il dechiffre d'abord la cle AES avec sa cle privee RSA, puis utilise cette cle AES pour dechiffrer le document. C'est exactement ainsi que fonctionne HTTPS.

---

## Phase 4 - Signatures numeriques

### Qu'est-ce qu'une signature numerique ?

Une signature numerique garantit deux choses :
- **Authenticite** : le document vient bien de l'expediteur pretendu
- **Integrite** : le document n'a pas ete modifie depuis sa signature

On **signe avec la cle privee** (seul le proprietaire peut signer) et on **verifie avec la cle publique** (tout le monde peut verifier).

### Creation d'une signature

```bash
openssl dgst -sha256 -sign agent_dakar.pem -out signature.sig document.txt
```

![](/images/openssl/openssl-12.png)

### Verification de la signature

```bash
openssl dgst -sha256 -verify identite_publique_dakar.pem -signature signature.sig document.txt
```

![](/images/openssl/openssl-14.png)

**Test de falsification :** si on modifie meme un seul caractere du document apres signature, la verification echoue immediatement. C'est l'**effet avalanche** des fonctions de hachage SHA-256 : le moindre changement produit un hash completement different.

| Scenario | Resultat de la verification |
|---|---|
| Document original + signature correcte | Verification OK |
| Document modifie + meme signature | Verification ECHEC |
| Mauvaise cle publique | Verification ECHEC |

---

## Phase 5 - Creation d'une Autorite de Certification (PKI)

### Qu'est-ce qu'une PKI ?

Une **PKI (Public Key Infrastructure)** est un systeme qui permet de verifier l'identite des entites sur un reseau. Le coeur d'une PKI est l'**Autorite de Certification (AC)** qui signe les certificats des utilisateurs et serveurs.

Quand votre navigateur voit un cadenas HTTPS, c'est parce que le serveur possede un certificat signe par une AC de confiance (Let's Encrypt, DigiCert, etc.).

### Creation de l'AC racine

```bash
openssl genrsa -out autorite_senegal.key 4096
openssl req -new -x509 -key autorite_senegal.key -out certificat_senegal.crt -days 3650
```

![](/images/openssl/openssl-15.png)

La cle de 4096 bits est plus longue car l'AC racine est le maillon le plus critique de la chaine de confiance.

### Signature d'un certificat client

Le processus de signature d'un certificat se fait en 3 etapes :

**1. L'entite genere une CSR (Certificate Signing Request) :**
```bash
openssl req -new -key techsn.key -out demande_techsn.csr
```

**2. L'AC signe la CSR et emet le certificat :**
```bash
openssl x509 -req -in demande_techsn.csr -CA certificat_senegal.crt -CAkey autorite_senegal.key -CAcreateserial -out certificat_techsn.crt -days 365
```

![](/images/openssl/openssl-17.png)

**3. Verification de la chaine de confiance :**
```bash
openssl verify -CAfile certificat_senegal.crt certificat_techsn.crt
```

![](/images/openssl/openssl-18.png)

### Risques et bonnes pratiques PKI

| Risque | Consequence | Protection |
|---|---|---|
| Compromission de la cle AC racine | Rupture totale de confiance | Stocker la cle dans un HSM |
| Certificat expire | Service inaccessible | Surveiller les dates d'expiration |
| Algorithme obsolete | Vulnerabilite aux attaques | Utiliser RSA 4096 ou ECDSA |
| Pas de revocation | Certificat compromis reste valide | Mettre en place CRL ou OCSP |

---

## Bonnes pratiques retenues

Suite a ce lab, voici les recommandations a appliquer en environnement de production :

1. **Ne jamais utiliser Base64 comme protection** - c'est de l'encodage, pas du chiffrement
2. **Toujours utiliser -pbkdf2** pour deriver les cles depuis des mots de passe
3. **Preferer AES-GCM** a AES-CBC pour avoir l'integrite en plus de la confidentialite
4. **Proteger les cles privees** avec un mot de passe et des permissions strictes (chmod 600)
5. **Stocker les cles AC** dans un HSM (Hardware Security Module)
6. **Mettre en place la revocation** (CRL ou OCSP) pour invalider les certificats compromis
7. **Surveiller Certificate Transparency** pour detecter les certificats emis frauduleusement

---

## Recap des commandes OpenSSL essentielles

| Commande | Usage |
|---|---|
| openssl version -a | Verifier la version et les options |
| openssl rand -hex 16 | Generer des donnees aleatoires |
| openssl enc -aes-256-cbc | Chiffrer avec AES-256 |
| openssl enc -d | Dechiffrer |
| openssl genrsa -out cle.pem 2048 | Generer une cle privee RSA |
| openssl rsa -pubout | Extraire la cle publique |
| openssl dgst -sha256 -sign | Signer un document |
| openssl dgst -sha256 -verify | Verifier une signature |
| openssl req -new -x509 | Creer un certificat auto-signe |
| openssl verify -CAfile | Verifier une chaine de certificats |

---

## Ce que ce lab m'a appris

Ce lab m'a permis de comprendre concrètement les mecanismes qui securisent les communications sur Internet. HTTPS, les VPNs, les emails signes, les mises a jour logicielles — tous reposent sur ces memes principes : AES pour chiffrer les donnees, RSA pour echanger les cles, SHA-256 pour verifier l'integrite, et une PKI pour authentifier les identites.

La lecon la plus importante : la securite ne repose pas sur l'obscurite (cacher un algorithme) mais sur des **standards cryptographiques robustes** et une **gestion rigoureuse des cles**.

## Prochaine etape

Approfondir avec la mise en place d'un serveur HTTPS complet avec certificats Let's Encrypt, et explorer les attaques courantes sur les implementations cryptographiques (padding oracle, timing attacks).
