---
title: "Projet DevOps — Pipeline CI/CD complet"
date: 2026-06-23
draft: false
tags: ["Docker", "Jenkins", "SonarQube", "Prometheus", "Grafana", "CI/CD", "DevOps", "WSL"]
categories: ["Labs"]
description: "Mise en place d'un pipeline CI/CD complet avec Docker, Jenkins, SonarQube, Prometheus et Grafana sur WSL/Ubuntu."
showToc: true
TocOpen: false
---

## Objectif du projet

Ce projet consiste a construire une chaine DevOps complete autour d'une application web de gestion de taches (Task Manager). L'idee est de reproduire ce qu'on trouve dans une vraie entreprise : chaque modification du code declenche automatiquement des tests, une analyse qualite, une conteneurisation Docker et un deploiement, le tout sans intervention manuelle.

**Environnement :**
| Element | Detail |
|---|---|
| OS hote | Windows 11 + WSL2 (Ubuntu 22.04) |
| Application | Task Manager (Node.js + Express + SQLite) |
| Conteneurisation | Docker Engine |
| CI/CD | Jenkins |
| Qualite de code | SonarQube |
| Supervision | Prometheus + Grafana + Node Exporter |

---

## Partie 01 - Installation de l'environnement (WSL + Ubuntu)

### Pourquoi WSL ?

WSL (Windows Subsystem for Linux) permet de faire tourner un vrai noyau Linux directement dans Windows, sans avoir besoin d'une machine virtuelle classique. C'est plus leger qu'une VM et parfaitement integre : on peut acceder aux fichiers Windows depuis Linux et inversement.

Pour un lab DevOps, WSL2 est ideal car Docker, Jenkins et tous les outils Linux fonctionnent nativement dedans, alors qu'ils sont plus compliques a installer directement sur Windows.

### Installation

Dans PowerShell en administrateur :

```powershell
wsl --install
```

Cette commande installe automatiquement WSL2 avec Ubuntu. Un redemarrage est necessaire.

### Script d'installation automatique

Plutot que d'installer chaque outil un par un, un script shell installe tout en une seule commande : Git, Node.js 18, Java 17 (requis par Jenkins), Docker Engine, Jenkins, SonarQube, Prometheus, Grafana et Node Exporter.

```bash
chmod +x install-devops.sh
./install-devops.sh
```

Une fois le script termine, on verifie que tous les conteneurs sont actifs :

```bash
docker ps
```

![](/images/docker-ps.png)

Tous les services sont accessibles depuis le navigateur Windows via localhost :

| Service | Port | Role |
|---|---|---|
| Jenkins | 8080 | Orchestration du pipeline CI/CD |
| SonarQube | 9000 | Analyse qualite du code |
| Prometheus | 9090 | Collecte des metriques |
| Grafana | 3000 | Visualisation des metriques |
| Node Exporter | 9100 | Exposition des metriques systeme |

---

## Partie 02 - Git et collaboration

### Pourquoi Git est central dans un pipeline DevOps

Dans une chaine DevOps, Git est le point de depart de tout. Chaque git push declenche le pipeline Jenkins via un webhook. Sans Git, il n'y a pas d'automatisation possible.

Le projet est organise avec une branche par developpeur. Chaque fonctionnalite est developpee sur une branche dedicee puis mergee sur main apres validation.

### Initialisation du depot

```bash
git init projet_devops
git add .
git commit -m "first commit"
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

Le Dockerfile et le Jenkinsfile a la racine du projet sont les deux fichiers cles qui permettent a Jenkins de savoir comment construire et deployer l'application automatiquement.

---

## Partie 03 - Docker et conteneurisation

### Pourquoi conteneuriser ?

Sans Docker, l'application fonctionne sur le PC du developpeur mais peut ne pas fonctionner sur le serveur de production a cause de differences de versions. Docker resout ce probleme : on empaquette l'application avec tout ce dont elle a besoin dans une image portable qui fonctionnera partout de maniere identique.

### Le Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

Explication ligne par ligne :
- **FROM node:18-alpine** : image Linux legere avec Node.js 18 preinstalle
- **WORKDIR /app** : dossier de travail dans le conteneur
- **COPY package.json ./** : copie des fichiers de dependances en premier
- **RUN npm install** : installation des dependances (mise en cache par Docker)
- **COPY . .** : copie du reste du code source
- **EXPOSE 3000** : l'application ecoute sur le port 3000
- **CMD** : commande executee au demarrage du conteneur

### Construction et publication de l'image

```bash
docker build -t devops-app .
docker push monusername/devops-app:latest
```

![](/images/devops-app.jpg)

---

## Partie 04 - Pipeline Jenkins (CI/CD)

### Ce qu'est un pipeline CI/CD

CI/CD signifie Integration Continue / Deploiement Continu. Des qu'un developpeur pousse du code sur GitHub, Jenkins detecte le changement et execute automatiquement une serie d'etapes : tests, analyse, build Docker, deploiement. Zero intervention manuelle.

Le pipeline est defini dans un fichier Jenkinsfile ecrit en Groovy, versionne avec le code. Chaque etape s'appelle un stage.

### Les 7 etapes du pipeline

![](/images/jenkins-pipeline.jpg)

| Etape | Duree | Ce qui se passe |
|---|---|---|
| Checkout SCM | 2s | Jenkins recupere le code depuis GitHub |
| Install | 4s | npm install installe les dependances |
| Compile | 1s | Verification de la syntaxe du code |
| Test | 1s | Execution des tests unitaires |
| SonarQube Analysis | 1min | Analyse qualite du code |
| Docker Build | 5min | Construction de l'image Docker |
| Docker Push | 1s | Publication de l'image sur DockerHub |

Si une etape echoue, le pipeline s'arrete et Jenkins notifie l'equipe. Le code defectueux ne peut pas atteindre la production.

---

## Partie 05 - SonarQube : analyse qualite du code

### Pourquoi analyser la qualite du code ?

SonarQube analyse automatiquement le code source pour detecter :
- **Bugs** : erreurs qui peuvent provoquer un comportement inattendu
- **Vulnerabilites** : failles de securite exploitables
- **Code Smells** : code difficile a maintenir
- **Duplications** : portions de code copiees-collees

### Resultats obtenus

![](/images/sonarqube-dashboard.jpg)

| Metrique | Resultat | Signification |
|---|---|---|
| Bugs | 1 | 1 erreur potentielle detectee |
| Vulnerabilites | 0 | Aucune faille de securite |
| Code Smells | 0 | Code propre et maintenable |
| Duplication | 0.0% | Aucune duplication |
| Statut global | Passed | Le code passe le Quality Gate |

Le **Quality Gate** est un seuil de qualite defini a l'avance. Si le code ne passe pas ce seuil, le pipeline bloque le deploiement automatiquement.

---

## Partie 06 - Prometheus et Grafana : supervision

### Architecture de supervision

La supervision fonctionne en 3 couches :

1. **Node Exporter** (port 9100) : expose les metriques systeme en temps reel
2. **Prometheus** (port 9090) : collecte ces metriques toutes les 15 secondes
3. **Grafana** (port 3000) : visualise les metriques sous forme de dashboards

### Dashboard Grafana

![](/images/grafana-dashboard.jpg)

Le dashboard montre en temps reel :
- **CPU** : pourcentage d'utilisation du processeur
- **RAM** : memoire utilisee vs disponible
- **Disque** : espace utilise par les conteneurs Docker
- **Reseau** : trafic entrant et sortant

---

## Commandes Docker essentielles

| Commande | Usage |
|---|---|
| `docker images` | Lister toutes les images locales |
| `docker ps -a` | Lister tous les conteneurs |
| `docker run -d -p 3000:3000 app` | Lancer un conteneur en arriere-plan |
| `docker stop` | Arreter un conteneur |
| `docker rm` | Supprimer un conteneur arrete |
| `docker build -t nom .` | Construire une image depuis un Dockerfile |
| `docker push user/image` | Publier une image sur DockerHub |
| `docker logs` | Voir les logs d'un conteneur |
| `docker exec -it bash` | Ouvrir un terminal dans un conteneur |

---

## Ce que ce projet m'a appris

**Sur Docker :** Un conteneur est isole du systeme hote mais partage son noyau Linux. Le Dockerfile doit etre optimise pour minimiser le nombre de couches et la taille de l'image finale.

**Sur Jenkins :** Le Jenkinsfile versionne avec le code est une bonne pratique. Jenkins peut etre declenche par un webhook GitHub pour un declenchement instantane a chaque push.

**Sur SonarQube :** L'analyse statique ne remplace pas les tests unitaires. SonarQube detecte des patterns de code dangereux que les tests ne couvrent pas.

**Sur Prometheus/Grafana :** La supervision doit etre mise en place des le debut d'un projet. Un dashboard bien configure permet de voir l'impact d'un deploiement en temps reel.

## Prochaine etape

Enrichir ce pipeline avec Ansible pour automatiser le deploiement sur plusieurs serveurs, Trivy pour scanner les images Docker, et un webhook GitHub pour le declenchement automatique.
