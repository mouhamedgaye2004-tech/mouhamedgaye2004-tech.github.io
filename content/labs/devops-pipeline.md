---
title: "Projet DevOps - Pipeline CI/CD complet"
date: 2026-06-23
draft: false
tags: ["Docker", "Jenkins", "SonarQube", "Prometheus", "Grafana", "CI/CD", "DevOps", "WSL"]
categories: ["Labs"]
description: "Mise en place d'un pipeline CI/CD complet avec Docker, Jenkins, SonarQube, Prometheus et Grafana sur WSL Ubuntu."
showToc: true
TocOpen: false
---

## Objectif du projet

Ce projet consiste a construire une chaine DevOps complete autour d'une application web de gestion de taches (Task Manager). L'idee est de reproduire ce qu'on trouve dans une vraie entreprise : chaque modification du code declenche automatiquement des tests, une analyse qualite, une conteneurisation Docker et un deploiement, le tout sans intervention manuelle.

**Qu'est-ce que le DevOps ?**

Le DevOps est une approche qui rapproche les equipes de developpement (Dev) et d'exploitation (Ops). L'objectif est de livrer du code plus rapidement, plus souvent et avec moins d'erreurs grace a l'automatisation. Le pipeline CI/CD est le coeur de cette approche.

| Terme | Definition |
|---|---|
| CI (Integration Continue) | Chaque modification de code est automatiquement testee et validee |
| CD (Deploiement Continu) | Le code valide est automatiquement deploye en production |
| Pipeline | Suite d'etapes automatisees de la modification du code jusqu'au deploiement |

**Environnement utilise :**

| Element | Detail |
|---|---|
| OS hote | Windows 11 + WSL2 Ubuntu 22.04 |
| Application | Task Manager en Node.js + Express + SQLite |
| Conteneurisation | Docker Engine |
| Orchestration CI/CD | Jenkins |
| Analyse qualite | SonarQube |
| Supervision | Prometheus + Grafana + Node Exporter |

---

## Partie 01 - Installation de l'environnement avec WSL

### Qu'est-ce que WSL ?

WSL signifie **Windows Subsystem for Linux**. C'est une fonctionnalite de Windows 11 qui permet de faire tourner un vrai noyau Linux directement dans Windows, sans avoir besoin d'une machine virtuelle classique.

**Pourquoi utiliser WSL plutot qu'une VM ?**

| Critere | WSL2 | Machine Virtuelle classique |
|---|---|---|
| Consommation RAM | Faible (partage avec Windows) | Elevee (RAM dediee) |
| Demarrage | Instantane | 30 secondes a 2 minutes |
| Integration fichiers | Acces direct aux fichiers Windows | Partage de dossiers a configurer |
| Performance Docker | Excellente | Correcte |
| Complexite setup | Faible | Elevee |

Pour un lab DevOps, WSL2 est ideal car Docker, Jenkins et tous les outils Linux fonctionnent nativement dedans.

### Installation de WSL

Dans PowerShell en administrateur :

```powershell
wsl --install
```

Cette commande installe automatiquement WSL2 avec Ubuntu 22.04. Un redemarrage est necessaire.

### Script d'installation automatique

Plutot que d'installer chaque outil un par un (ce qui prendrait des heures), un script shell installe tout automatiquement : Git, Node.js 18, Java 17 (requis par Jenkins), Docker Engine, Jenkins, SonarQube, Prometheus, Grafana et Node Exporter.

```bash
chmod +x install-devops.sh
./install-devops.sh
```

Une fois le script termine, on verifie que tous les services sont actifs :

```bash
docker ps
```

![](/images/docker-ps.png)

Tous les services sont accessibles depuis le navigateur Windows via localhost :

| Service | Port | Role dans le pipeline |
|---|---|---|
| Jenkins | 8080 | Orchestre et execute le pipeline CI/CD |
| SonarQube | 9000 | Analyse la qualite du code |
| Prometheus | 9090 | Collecte les metriques de supervision |
| Grafana | 3000 | Visualise les metriques via des dashboards |
| Node Exporter | 9100 | Expose les metriques systeme (CPU, RAM, disque) |

---

## Partie 02 - Git et collaboration

### Pourquoi Git est le point de depart de tout ?

Dans une chaine DevOps, **Git est le declencheur de tout**. Chaque fois qu'un developpeur fait un `git push`, Jenkins detecte le changement via un webhook et lance automatiquement le pipeline. Sans Git, pas d'automatisation possible.

La bonne pratique est de travailler avec une **branche par fonctionnalite** : chaque developpeur cree sa propre branche, travaille dessus, puis soumet une Pull Request pour merger sur la branche principale apres validation.

### Initialisation du depot

```bash
git init projet_devops
git add .
git commit -m first-commit
git push -u origin main
```

![](/images/github-repo.jpg)

### Structure du depot

```
projet_devops/
├── backend/          
├── frontend/         
├── package.json      
├── .gitignore        
├── README.md         
├── Dockerfile        
└── Jenkinsfile       
```

Le **Dockerfile** et le **Jenkinsfile** sont les deux fichiers cles :
- Le Dockerfile dit a Docker comment construire l'image de l'application
- Le Jenkinsfile dit a Jenkins quelles etapes executer et dans quel ordre

---

## Partie 03 - Docker et conteneurisation

### Pourquoi conteneuriser une application ?

**Le probleme classique sans Docker :** l'application fonctionne sur le PC du developpeur (Node.js 18, Ubuntu) mais ne fonctionne pas sur le serveur de production (Node.js 16, CentOS) a cause de differences de versions et de configuration.

**La solution Docker :** on empaquette l'application avec TOUT ce dont elle a besoin (runtime, dependances, configuration) dans une **image Docker**. Cette image est portable et fonctionnera de maniere identique partout : sur le PC du dev, sur le serveur de test, en production.

**Analogie :** Docker c'est comme une boite de conserve. Le contenu (l'application) est protege de l'environnement exterieur et peut etre transporte n'importe ou sans changer.

### Le Dockerfile explique ligne par ligne

```
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD npm start
```

| Instruction | Explication |
|---|---|
| FROM node:18-alpine | On part d'une image Linux legere (Alpine) avec Node.js 18 deja installe |
| WORKDIR /app | On definit le dossier de travail a l'interieur du conteneur |
| COPY package.json ./ | On copie les fichiers de dependances en premier (optimisation du cache) |
| RUN npm install | On installe les dependances Node.js dans le conteneur |
| COPY . . | On copie tout le code source dans le conteneur |
| EXPOSE 3000 | On indique que l'application ecoute sur le port 3000 |
| CMD npm start | Commande executee au demarrage du conteneur |

**Pourquoi copier package.json avant le reste du code ?** C'est une optimisation du cache Docker. Si on ne modifie que le code (pas les dependances), Docker reutilise le cache de `npm install` et le build est beaucoup plus rapide.

### Construction et publication

```bash
docker build -t devops-app .
docker push monusername/devops-app:latest
```

![](/images/devops-app.jpg)

L'application est accessible sur http://localhost:3000.

---

## Partie 04 - Pipeline Jenkins CI/CD

### Comment fonctionne Jenkins ?

Jenkins surveille le depot Git. Des qu'un developpeur pousse du code, Jenkins le detecte et execute automatiquement une serie d'etapes definies dans le **Jenkinsfile**. Si une etape echoue, le pipeline s'arrete et Jenkins notifie l'equipe. Le code defectueux ne peut jamais atteindre la production.

### Les 7 etapes du pipeline

![](/images/jenkins-pipeline.jpg)

| Etape | Duree | Ce qui se passe concretement |
|---|---|---|
| Checkout SCM | 2s | Jenkins telecharge le code depuis GitHub |
| Install | 4s | npm install telecharge les dependances Node.js |
| Compile | 1s | Verification de la syntaxe du code JavaScript |
| Test | 1s | Execution des tests unitaires automatises |
| SonarQube Analysis | 1min | Analyse statique de la qualite et securite du code |
| Docker Build | 5min | Construction de l'image Docker de l'application |
| Docker Push | 1s | Publication de l'image sur DockerHub |

**Pourquoi l'etape Docker Build prend 5 minutes ?** Parce que Docker doit telecharger l'image de base (node:18-alpine), installer les dependances et construire l'image. Les builds suivants sont plus rapides grace au cache.

### Le Jenkinsfile

Le Jenkinsfile est un fichier texte ecrit en Groovy (langage de Jenkins) qui definit toutes ces etapes. Il est versionne avec le code dans Git, ce qui signifie que l'historique du pipeline est trace avec l'historique du code.

---

## Partie 05 - SonarQube : analyse qualite du code

### Pourquoi analyser la qualite du code ?

Ecrire du code qui fonctionne ne suffit pas. Le code doit aussi etre **securise**, **maintenable** et **propre**. SonarQube analyse automatiquement le code source a chaque push pour detecter des problemes que les developpeurs n'auraient pas remarques.

**Les 4 categories analysees par SonarQube :**

| Categorie | Definition | Exemple |
|---|---|---|
| Bugs | Erreurs qui causeront un comportement inattendu | Division par zero, variable non initialisee |
| Vulnerabilites | Failles de securite exploitables | Injection SQL, mot de passe en dur dans le code |
| Code Smells | Code qui fonctionne mais est difficile a maintenir | Fonction de 500 lignes, variable nommee "x" |
| Duplications | Code copie-colle | Meme bloc de code present a 3 endroits differents |

### Resultats obtenus sur notre projet

![](/images/sonarqube-dashboard.jpg)

| Metrique | Resultat | Interpretation |
|---|---|---|
| Bugs | 1 | Une erreur potentielle a corriger |
| Vulnerabilites | 0 | Aucune faille de securite detectee |
| Code Smells | 0 | Code propre et maintenable |
| Duplication | 0% | Aucun code copie-colle |
| Quality Gate | Passed | Le code respecte les seuils de qualite definis |

Le **Quality Gate** est un ensemble de regles definies a l'avance (ex: "pas plus de 5 bugs", "couverture de tests superieure a 80%"). Si le code ne passe pas le Quality Gate, le pipeline bloque automatiquement le deploiement.

---

## Partie 06 - Prometheus et Grafana : supervision

### Pourquoi superviser ?

Sans supervision, on decouvre les problemes quand les utilisateurs se plaignent. Avec Prometheus et Grafana, on detecte les anomalies avant qu'elles causent une panne : une montee en charge du CPU, une memoire qui se remplit progressivement, une augmentation du temps de reponse.

### Architecture de supervision en 3 couches

| Outil | Role | Port |
|---|---|---|
| Node Exporter | Expose les metriques systeme en temps reel (CPU, RAM, disque, reseau) | 9100 |
| Prometheus | Collecte (scrape) ces metriques toutes les 15 secondes et les stocke | 9090 |
| Grafana | Se connecte a Prometheus et affiche les metriques sous forme de graphiques | 3000 |

### Dashboard Grafana

![](/images/grafana-dashboard.jpg)

Le dashboard montre en temps reel :
- **CPU** : pourcentage d'utilisation du processeur
- **RAM** : memoire utilisee vs disponible
- **Disque** : espace utilise par les conteneurs Docker
- **Reseau** : trafic entrant et sortant en octets par seconde

**En pratique en entreprise :** on configure des **alertes** qui envoient un email ou un message Slack quand un seuil est depasse (ex: CPU > 90% pendant plus de 5 minutes). L'equipe peut reagir avant que le serveur tombe.

---

## Commandes Docker essentielles

Ces commandes sont utilisees quotidiennement par un administrateur systemes :

| Commande | Usage | Exemple concret |
|---|---|---|
| docker images | Lister toutes les images locales | Voir quelles versions sont disponibles |
| docker ps -a | Lister tous les conteneurs | Voir ce qui tourne et ce qui est arrete |
| docker run -d app | Lancer un conteneur en arriere-plan | Demarrer l'application |
| docker stop id | Arreter proprement un conteneur | Arreter avant une mise a jour |
| docker rm id | Supprimer un conteneur arrete | Nettoyer les vieux conteneurs |
| docker build -t nom . | Construire une image | Apres modification du Dockerfile |
| docker push user/image | Publier sur DockerHub | Rendre l'image disponible |
| docker logs id | Voir les logs d'un conteneur | Debugger une erreur |
| docker exec -it id bash | Ouvrir un terminal dans un conteneur | Inspecter l'interieur |

---

## Ce que ce projet m'a appris

**Sur Docker :** Un conteneur partage le noyau Linux du systeme hote mais est completement isole (fichiers, processus, reseau). C'est ce qui le rend plus leger qu'une VM. Le Dockerfile doit etre optimise pour minimiser la taille de l'image et le nombre de couches.

**Sur Jenkins :** Le Jenkinsfile versionne avec le code est une excellente pratique car l'historique du pipeline est trace avec l'historique du code. Un pipeline bien configure empeche du code de mauvaise qualite d'atteindre la production.

**Sur SonarQube :** L'analyse statique ne remplace pas les tests unitaires, elle les complete. SonarQube detecte des patterns dangereux que les tests fonctionnels ne couvrent pas (failles de securite, mauvaises pratiques de code).

**Sur Prometheus/Grafana :** La supervision doit etre mise en place des le debut du projet, pas apres une panne. Un dashboard bien configure permet de voir l'impact d'un deploiement en temps reel et de detecter des regressions de performance.

## Prochaine etape

Enrichir ce pipeline avec :
- **Ansible** pour automatiser le deploiement sur plusieurs serveurs en parallele
- **Trivy** pour scanner les images Docker a la recherche de vulnerabilites connues
- **Webhook GitHub** pour declencher le pipeline automatiquement a chaque push sans intervention manuelle
