---
title: "Projet DevOps — Pipeline CI/CD complet"
date: 2026-06-23
draft: false
tags: ["Docker", "Jenkins", "SonarQube", "Prometheus", "Grafana", "CI/CD", "DevOps", "WSL"]
categories: ["Labs"]
description: "Mise en place d'un pipeline CI/CD complet avec Docker, Jenkins, SonarQube, Prometheus et Grafana sur WSL/Ubuntu."
showToc: true
TocOpen: false
cover:
  alt: "Pipeline CI/CD DevOps complet"
---

## Objectif du projet

Ce projet consiste à construire une chaîne DevOps complète autour d'une application web de gestion de tâches (Task Manager). L'idée est de reproduire ce qu'on trouve dans une vraie entreprise : chaque modification du code déclenche automatiquement des tests, une analyse qualité, une conteneurisation Docker et un déploiement — le tout sans intervention manuelle.

**Environnement :**
| Élément | Détail |
|---|---|
| OS hôte | Windows 11 + WSL2 (Ubuntu 22.04) |
| Application | Task Manager (Node.js + Express + SQLite) |
| Conteneurisation | Docker Engine |
| CI/CD | Jenkins |
| Qualité de code | SonarQube |
| Supervision | Prometheus + Grafana + Node Exporter |

---

## Partie 01 — Installation de l'environnement (WSL + Ubuntu)

### Pourquoi WSL ?

WSL (Windows Subsystem for Linux) permet de faire tourner un vrai noyau Linux directement dans Windows, sans avoir besoin d'une machine virtuelle classique. C'est plus léger qu'une VM et parfaitement intégré : on peut accéder aux fichiers Windows depuis Linux et inversement.

Pour un lab DevOps, WSL2 est idéal car Docker, Jenkins et tous les outils Linux fonctionnent nativement dedans, alors qu'ils sont plus compliqués à installer directement sur Windows.

### Installation

Dans PowerShell en administrateur :

```powershell
wsl --install
```

Cette commande installe automatiquement WSL2 avec Ubuntu. Un redémarrage est nécessaire.

### Script d'installation automatique

Plutôt que d'installer chaque outil un par un, un script shell installe tout en une seule commande : Git, Node.js 18, Java 17 (requis par Jenkins), Docker Engine, Jenkins, SonarQube, Prometheus, Grafana et Node Exporter.

```bash
chmod +x install-devops.sh
./install-devops.sh
```

Une fois le script terminé, on vérifie que tous les conteneurs sont actifs :

```bash
docker ps
```

![Conteneurs Docker actifs avec tous les services](/images/docker-ps.png)

Tous les services sont accessibles depuis le navigateur Windows via `localhost` :

| Service | Port | Rôle |
|---|---|---|
| Jenkins | 8080 | Orchestration du pipeline CI/CD |
| SonarQube | 9000 | Analyse qualité du code |
| Prometheus | 9090 | Collecte des métriques |
| Grafana | 3000 | Visualisation des métriques |
| Node Exporter | 9100 | Exposition des métriques système |

---

## Partie 02 — Git et collaboration

### Pourquoi Git est central dans un pipeline DevOps

Dans une chaîne DevOps, Git est le point de départ de tout. Chaque `git push` déclenche le pipeline Jenkins via un webhook. Sans Git, il n'y a pas d'automatisation possible.

Le projet est organisé avec une branche par développeur. Chaque fonctionnalité est développée sur une branche dédiée puis mergée sur `main` après validation.

### Initialisation du dépôt

```bash
git init projet_devops
git add .
git commit -m "first commit"
git push -u origin main
```

![Dépôt GitHub avec Dockerfile et Jenkinsfile visibles](/images/github-repo.jpg)

### Structure du dépôt

```
projet_devops/
├── backend/          # Code serveur Node.js
├── frontend/         # Interface utilisateur
├── package.json      # Dépendances Node.js
├── .gitignore        # Fichiers exclus du versioning
├── README.md         # Documentation du projet
├── Dockerfile        # Instructions de conteneurisation
└── Jenkinsfile       # Définition du pipeline CI/CD
```

Le `Dockerfile` et le `Jenkinsfile` à la racine du projet sont les deux fichiers clés qui permettent à Jenkins de savoir comment construire et déployer l'application automatiquement.

---

## Partie 03 — Docker et conteneurisation

### Pourquoi conteneuriser ?

Sans Docker, l'application fonctionne sur le PC du développeur mais peut ne pas fonctionner sur le serveur de production à cause de différences de versions (Node.js, dépendances, OS). Docker résout ce problème : on empaquète l'application avec tout ce dont elle a besoin dans une image portable qui fonctionnera partout de manière identique.

### Le Dockerfile

```dockerfile
FROM node:18-alpine
# On part d'une image Linux légère (Alpine) avec Node.js 18 préinstallé

WORKDIR /app
# On définit le dossier de travail dans le conteneur

COPY package*.json ./
# On copie d'abord les fichiers de dépendances

RUN npm install
# On installe les dépendances (cette étape est mise en cache par Docker)

COPY . .
# On copie le reste du code source

EXPOSE 3000
# On indique que l'application écoute sur le port 3000

CMD ["npm", "start"]
# Commande exécutée au démarrage du conteneur
```

### Construction et publication de l'image

```bash
# Construire l'image localement
docker build -t devops-app .

# Publier l'image sur DockerHub pour la rendre accessible
docker push monusername/devops-app:latest
```

![Application Task Manager déployée et accessible dans le navigateur](/images/devops-app.jpg)

L'application est accessible sur `http://localhost:3000` depuis le navigateur Windows.

---

## Partie 04 — Pipeline Jenkins (CI/CD)

### Ce qu'est un pipeline CI/CD

CI/CD signifie **Intégration Continue / Déploiement Continu**. L'idée : dès qu'un développeur pousse du code sur GitHub, Jenkins détecte le changement et exécute automatiquement une série d'étapes — tests, analyse, build Docker, déploiement. Zéro intervention manuelle.

Le pipeline est défini dans un fichier `Jenkinsfile` écrit en Groovy, versionné avec le code. Chaque étape s'appelle un **stage**.

### Les 7 étapes du pipeline

![Pipeline Jenkins avec toutes les étapes au vert](/images/jenkins-pipeline.jpg)

| Étape | Durée | Ce qui se passe |
|---|---|---|
| Checkout SCM | 2s | Jenkins récupère le code depuis GitHub |
| Install | 4s | `npm install` installe les dépendances |
| Compile | 1s | Vérification de la syntaxe du code |
| Test | 1s | Exécution des tests unitaires |
| SonarQube Analysis | ~1min | Analyse qualité du code (bugs, failles) |
| Docker Build | ~5min | Construction de l'image Docker |
| Docker Push | 1s | Publication de l'image sur DockerHub |

Si une étape échoue, le pipeline s'arrête et Jenkins notifie l'équipe. Le code défectueux ne peut pas atteindre la production.

---

## Partie 05 — SonarQube : analyse qualité du code

### Pourquoi analyser la qualité du code ?

SonarQube analyse automatiquement le code source pour détecter :
- **Bugs** : erreurs qui peuvent provoquer un comportement inattendu
- **Vulnérabilités** : failles de sécurité exploitables
- **Code Smells** : code qui fonctionne mais qui est difficile à maintenir
- **Duplications** : portions de code copiées-collées

Cette analyse est déclenchée automatiquement par Jenkins à chaque push — pas besoin de la lancer manuellement.

### Résultats obtenus

![Dashboard SonarQube avec les métriques du projet](/images/sonarqube-dashboard.jpg)

| Métrique | Résultat | Signification |
|---|---|---|
| Bugs | 1 | 1 erreur potentielle détectée dans le code |
| Vulnérabilités | 0 | Aucune faille de sécurité |
| Code Smells | 0 | Code propre et maintenable |
| Duplication | 0.0% | Aucune duplication de code |
| Statut global | ✅ Passed | Le code passe le Quality Gate |

Le **Quality Gate** est un seuil de qualité défini à l'avance. Si le code ne passe pas ce seuil, le pipeline bloque le déploiement automatiquement.

---

## Partie 06 — Prometheus et Grafana : supervision

### Architecture de supervision

La supervision fonctionne en 3 couches :

1. **Node Exporter** (port 9100) — tourne sur le serveur et expose en temps réel les métriques système : utilisation CPU, RAM, espace disque, trafic réseau
2. **Prometheus** (port 9090) — scrape (collecte) ces métriques toutes les 15 secondes et les stocke dans sa base de données time-series
3. **Grafana** (port 3000) — se connecte à Prometheus et affiche les métriques sous forme de graphiques et tableaux de bord

### Dashboard Grafana

![Dashboard Grafana avec métriques CPU, RAM et réseau en temps réel](/images/grafana-dashboard.jpg)

Le dashboard montre en temps réel :
- **CPU** : pourcentage d'utilisation du processeur
- **RAM** : mémoire utilisée vs disponible
- **Disque** : espace utilisé par les conteneurs Docker
- **Réseau** : trafic entrant et sortant

En entreprise, ce type de supervision permet de détecter une surcharge avant qu'elle provoque une panne, et de créer des alertes automatiques (email, Slack) quand un seuil est dépassé.

---

## Commandes Docker essentielles

Ces commandes sont utilisées quotidiennement par un administrateur systèmes :

| Commande | Usage |
|---|---|
| `docker images` | Lister toutes les images locales |
| `docker ps -a` | Lister tous les conteneurs (actifs et arrêtés) |
| `docker run -d -p 3000:3000 app` | Lancer un conteneur en arrière-plan |
| `docker stop <id>` | Arrêter proprement un conteneur |
| `docker rm <id>` | Supprimer un conteneur arrêté |
| `docker rmi <image>` | Supprimer une image locale |
| `docker build -t nom .` | Construire une image depuis un Dockerfile |
| `docker push user/image` | Publier une image sur DockerHub |
| `docker logs <id>` | Voir les logs d'un conteneur |
| `docker exec -it <id> bash` | Ouvrir un terminal dans un conteneur actif |

---

## Ce que ce projet m'a appris

**Sur Docker :** Un conteneur est isolé du système hôte mais partage son noyau Linux — c'est ce qui le rend plus léger qu'une VM. Le Dockerfile doit être optimisé pour minimiser le nombre de couches et la taille de l'image finale.

**Sur Jenkins :** Le Jenkinsfile versionné avec le code est une bonne pratique — l'historique du pipeline est tracé avec le code. Jenkins peut être déclenché par un webhook GitHub pour un déclenchement instantané à chaque push.

**Sur SonarQube :** L'analyse statique ne remplace pas les tests unitaires — elle les complète. SonarQube détecte des patterns de code dangereux que les tests ne couvrent pas forcément.

**Sur Prometheus/Grafana :** La supervision doit être mise en place dès le début d'un projet, pas après une panne. Un dashboard bien configuré permet de voir l'impact d'un déploiement en temps réel.

## Prochaine étape

Enrichir ce pipeline avec :
- **Ansible** pour automatiser le déploiement sur plusieurs serveurs
- **Trivy** pour scanner les images Docker à la recherche de vulnérabilités
- **Webhook GitHub** pour déclencher le pipeline automatiquement à chaque push