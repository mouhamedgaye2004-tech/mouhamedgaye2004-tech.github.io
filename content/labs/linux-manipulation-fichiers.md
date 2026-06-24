---
title: "Manipulation de fichiers sous Linux - Filtres et commandes essentielles"
date: 2026-06-24
draft: false
tags: ["Linux", "Bash", "Administration", "Terminal", "Vim"]
categories: ["Labs"]
description: "Maitrise des commandes de filtres Linux : cat, grep, sort, cut, tr, find, tar, gzip et editeur Vim."
showToc: true
TocOpen: false
---

## Objectif du lab

Ce lab a pour but de maitriser les **commandes de filtres et d'edition de texte sous Linux**. Ces commandes sont utilisees quotidiennement par tout administrateur systeme pour analyser des logs, rechercher des informations, transformer des donnees et gerer des fichiers depuis le terminal.

**Pourquoi ces commandes sont importantes ?**

Dans un environnement Linux professionnel, l'interface graphique n'est pas toujours disponible (serveurs headless, acces SSH distant). L'administrateur doit etre capable de tout faire en ligne de commande : lire des fichiers de configuration, chercher des erreurs dans des logs, extraire des donnees, compresser des archives. Ces outils sont la base de tout travail en administration systeme.

**Environnement utilise :**

| Element | Detail |
|---|---|
| OS | Ubuntu Server |
| Interface | Terminal (ligne de commande) |
| Etablissement | ISM Dakar - Licence 2 Cybersecurite |

---

## Partie 01 - Affichage de fichiers

### La commande cat

`cat` (concatenate) est la commande la plus basique pour afficher le contenu d'un fichier texte. Elle affiche tout le contenu d'un coup, du debut a la fin.

```bash
cat nom_fichier
cat fichier1 fichier2
```

![](/images/linux/linux-01.png)

**Usages courants en administration :**
- Lire un fichier de configuration : `cat /etc/passwd`
- Concatener deux fichiers : `cat fichier1 fichier2 > fusion.txt`
- Creer un fichier rapidement : `cat > nouveau_fichier.txt`

### La commande tac

`tac` est l'inverse de `cat` — elle affiche le fichier en partant de la derniere ligne. Tres utile pour lire les logs recents en premier sans avoir a scroller jusqu'a la fin.

```bash
tac nom_fichier
```

![](/images/linux/linux-02.png)

### La commande less

`less` permet de lire un fichier avec navigation interactive. Contrairement a `cat`, elle n'affiche pas tout d'un coup mais page par page.

```bash
less nom_fichier
```

![](/images/linux/linux-03.png)

**Navigation dans less :**

| Touche | Action |
|---|---|
| Espace ou Page Down | Page suivante |
| b ou Page Up | Page precedente |
| /motif | Rechercher un mot |
| n | Occurrence suivante |
| q | Quitter |

`less` est indispensable pour lire de gros fichiers de logs sans les charger entierement en memoire.

### Les commandes head et tail

`head` affiche les premieres lignes d'un fichier, `tail` affiche les dernieres.

```bash
head -n 10 fichier.txt
tail -n 20 fichier.txt
tail -f /var/log/syslog
```

![](/images/linux/linux-06.jpeg)

**L'option -f de tail** est particulierement utile en administration : elle affiche les nouvelles lignes en temps reel au fur et a mesure qu'elles sont ajoutees au fichier. C'est la commande de reference pour surveiller un log en direct lors d'une intervention.

---

## Partie 02 - Decompte avec wc

`wc` (word count) compte les lignes, mots et caracteres d'un fichier.

```bash
wc fichier.txt
wc -l fichier.txt
wc -w fichier.txt
wc -c fichier.txt
```

![](/images/linux/linux-08.png)

| Option | Compte |
|---|---|
| -l | Nombre de lignes |
| -w | Nombre de mots |
| -c | Nombre de caracteres (octets) |

**Exemple pratique :** compter le nombre d'utilisateurs sur le systeme :
```bash
wc -l /etc/passwd
```

---

## Partie 03 - Tri avec sort

`sort` trie les lignes d'un fichier texte par ordre alphabetique ou numerique.

```bash
sort fichier.txt
sort -r fichier.txt
sort -n fichier.txt
sort -t ":" -k3 -n /etc/passwd
```

![](/images/linux/linux-09.png)

| Option | Effet |
|---|---|
| -r | Tri inverse (Z -> A) |
| -n | Tri numerique |
| -t | Definit le separateur de champ |
| -k | Specifie la colonne de tri |

**Exemple avance :** trier le fichier /etc/passwd par UID (3e colonne) :
```bash
sort -t ":" -k3 -n /etc/passwd
```

---

## Partie 04 - Recherche avec grep

`grep` est l'une des commandes les plus puissantes de Linux. Elle recherche les lignes contenant un motif dans un fichier.

```bash
grep "motif" fichier.txt
grep -n "erreur" /var/log/syslog
grep -i "warning" fichier.log
grep -v "commentaire" fichier.conf
grep -r "motif" /etc/
```

![](/images/linux/linux-10.jpeg)

| Option | Effet |
|---|---|
| -n | Affiche les numeros de lignes |
| -i | Ignore la casse (majuscules/minuscules) |
| -v | Inverse la selection (exclut le motif) |
| -c | Affiche le nombre de lignes correspondantes |
| -l | Affiche uniquement le nom des fichiers |
| -r | Recherche recursive dans les sous-dossiers |

**Exemples pratiques en administration :**
```bash
grep "Failed password" /var/log/auth.log
grep -i "error" /var/log/syslog | tail -20
grep -v "^#" /etc/ssh/sshd_config
```

---

## Partie 05 - Selection de colonnes avec cut

`cut` extrait des colonnes ou des champs specifiques d'un fichier texte.

```bash
cut -d ":" -f1 /etc/passwd
cut -d ":" -f1,3 /etc/passwd
cut -c1-10 fichier.txt
```

![](/images/linux/linux-12.png)

| Option | Effet |
|---|---|
| -d | Definit le delimiteur de champ |
| -f | Selectionne les champs (colonnes) |
| -c | Selectionne par position de caracteres |

**Exemple pratique :** extraire uniquement les noms d'utilisateurs du fichier passwd :
```bash
cut -d ":" -f1 /etc/passwd
```

---

## Partie 06 - Remplacement avec tr

`tr` (translate) remplace ou supprime des caracteres dans un flux de texte.

```bash
echo "hello" | tr "a-z" "A-Z"
tr "a" "A" < fichier.txt
tr -d "\n" < fichier.txt
```

![](/images/linux/linux-13.jpeg)

| Option | Effet |
|---|---|
| tr "a-z" "A-Z" | Convertit en majuscules |
| tr -d "caractere" | Supprime un caractere |
| tr -s " " | Compresse les espaces multiples |

---

## Partie 07 - Recherche, compression et archivage

### La commande find

`find` recherche des fichiers et dossiers selon des criteres (nom, taille, date, permissions...).

```bash
find /etc -name "*.conf"
find / -name "passwd" -type f
find /home -size +10M
find /var/log -mtime -7
```

### Compression avec gzip

`gzip` compresse des fichiers pour reduire leur taille.

```bash
gzip fichier.txt
gzip -d fichier.txt.gz
gunzip fichier.txt.gz
```

![](/images/linux/linux-16.png)

### Archivage avec tar

`tar` cree des archives (regroupement de fichiers) avec ou sans compression.

```bash
tar -cvf archive.tar dossier/
tar -czvf archive.tar.gz dossier/
tar -xzvf archive.tar.gz
tar -tvf archive.tar
```

![](/images/linux/linux-17.jpeg)

| Option | Effet |
|---|---|
| -c | Creer une archive |
| -x | Extraire une archive |
| -v | Mode verbose (affiche les fichiers traites) |
| -f | Specifier le nom du fichier archive |
| -z | Compression gzip |
| -t | Lister le contenu sans extraire |

**Combinaison tar + gzip :** la commande `tar -czvf archive.tar.gz dossier/` cree une archive compressee en une seule etape. C'est la methode standard pour sauvegarder des dossiers sous Linux.

---

## Partie 08 - Editeur de texte Vim

### Pourquoi Vim ?

Vim est l'editeur de texte de reference en administration Linux. Il est disponible sur pratiquement tous les systemes Unix/Linux, meme les plus minimalistes. Maitriser Vim est indispensable pour modifier des fichiers de configuration sur un serveur distant en SSH.

### Installation

```bash
sudo apt install vim -y
select-editor
```

![](/images/linux/linux-19.png)

### Verification de l'installation

```bash
vim --version
```

![](/images/linux/linux-21.png)

### Vimtutor - le tutoriel integre

`vimtutor` est un tutoriel interactif integre dans Vim. Il prend environ 30 minutes et couvre toutes les bases.

```bash
vimtutor
```

![](/images/linux/linux-22.png)

### Les modes de Vim

Vim fonctionne avec plusieurs modes — c'est la principale difference avec les editeurs classiques :

| Mode | Acces | Usage |
|---|---|---|
| Normal | Echap | Navigation, copier/coller, supprimer |
| Insertion | i, a, o | Ecrire du texte |
| Visuel | v | Selectionner du texte |
| Commande | : | Sauvegarder, quitter, rechercher |

### Commandes Vim essentielles

| Commande | Action |
|---|---|
| i | Passer en mode insertion |
| Echap | Revenir en mode normal |
| :w | Sauvegarder |
| :q | Quitter |
| :wq | Sauvegarder et quitter |
| :q! | Quitter sans sauvegarder |
| dd | Supprimer une ligne |
| yy | Copier une ligne |
| p | Coller |
| /motif | Rechercher |
| u | Annuler |

---

## Recap des commandes essentielles

| Commande | Usage principal |
|---|---|
| cat | Afficher le contenu d'un fichier |
| tac | Afficher a l'envers (utile pour les logs) |
| less | Lire un fichier avec navigation |
| head -n X | Afficher les X premieres lignes |
| tail -f | Suivre un log en temps reel |
| wc -l | Compter les lignes |
| sort | Trier les lignes |
| grep | Rechercher un motif |
| cut -d -f | Extraire des colonnes |
| tr | Remplacer des caracteres |
| find | Rechercher des fichiers |
| gzip / gunzip | Compresser / decompresser |
| tar -czvf | Creer une archive compressee |
| tar -xzvf | Extraire une archive |

---

## Ce que ce lab m'a appris

Ces commandes semblent simples individuellement, mais leur vraie puissance vient des **pipes** (`|`) qui permettent de les chainer. Par exemple :

```bash
grep "Failed" /var/log/auth.log | cut -d " " -f11 | sort | uniq -c | sort -rn | head -10
```

Cette commande unique extrait les 10 IPs qui ont le plus tente de se connecter en echec sur le serveur SSH — utile pour detecter une attaque par force brute.

## Prochaine etape

Approfondir avec les **expressions regulieres** dans grep et sed, et automatiser des taches repetitives avec des **scripts Bash** combinant ces commandes.
