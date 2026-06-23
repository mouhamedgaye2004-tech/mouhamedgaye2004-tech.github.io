---
title: "Projet DevOps — Pipeline CI/CD complet"
date: 2026-06-23
draft: false
tags: ["Docker", "Jenkins", "SonarQube", "Prometheus", "Grafana", "CI/CD", "DevOps", "WSL"]
categories: ["Labs"]
description: "Mise en place dun pipeline CI/CD complet avec Docker, Jenkins, SonarQube, Prometheus et Grafana sur WSL/Ubuntu."
showToc: true
---

## Objectif du projet

Mettre en place une chaine DevOps complete autour dune application web de gestion de taches (Task Manager).

**Environnement :**
- OS : Windows 11 + WSL2 (Ubuntu 22.04)
- Outils : Docker, Jenkins, SonarQube, Prometheus, Grafana
- Application : Task Manager (Node.js + Express + SQLite)

---

## Partie 01 - Installation de lenvironnement (WSL + Ubuntu)

WSL permet de faire tourner Linux directement sur Windows sans machine virtuelle classique.

Installation dans PowerShell administrateur : wsl --install

Un script shell installe ensuite automatiquement tous les outils : Git, Node.js 18, Java 17, Docker Engine, Jenkins, SonarQube, Prometheus, Grafana et Node Exporter.

chmod +x install-devops.sh puis ./install-devops.sh

![Containers Docker actifs](images/docker-ps.png)

| Service | Port |
|---|---|
| Jenkins | 8080 |
| SonarQube | 9000 |
| Prometheus | 9090 |
| Grafana | 3000 |
| Node Exporter | 9100 |

---

## Partie 02 - Git & Collaboration

Travail en binome avec une branche par developpeur.

git init projet_devops puis git add . puis git commit -m "first commit" puis git push -u origin main

![Repo GitHub avec Dockerfile et Jenkinsfile](images/github-repo.png.jpg)

Structure du repo : backend/ frontend/ package.json .gitignore README.md Dockerfile Jenkinsfile

---

## Partie 03 - Docker

Conteneurisation de lapplication avec un Dockerfile.

FROM node:18-alpine / WORKDIR /app / COPY package*.json ./ / RUN npm install / COPY . . / EXPOSE 3000 / CMD ["npm", "start"]

docker build -t devops-app . puis docker push monusername/devops-app:latest

![Application DevOps Notes en ligne](images/devops-app.png.jpg)

---

## Partie 04 - Pipeline Jenkins

Pipeline Groovy automatise en 9 etapes.

![Pipeline Jenkins - toutes les etapes vertes](images/jenkins-pipeline.png.jpg)

| Etape | Duree |
|---|---|
| Checkout SCM | 2s |
| Install | 4s |
| Compile | 1s |
| Test | 1s |
| SonarQube Analysis | 1min |
| Docker Build | 5min |
| Docker Push | 1s |

---

## Partie 05 - SonarQube

Analyse statique du code lancee automatiquement via Jenkins.

![Dashboard SonarQube](images/sonarqube-dashboard.png.jpg)

| Metrique | Resultat |
|---|---|
| Bugs | 1 |
| Vulnerabilites | 0 |
| Code Smells | 0 |
| Duplication | 0.0% |
| Statut | Passed |

---

## Partie 06 - Prometheus et Grafana

Supervision de lapplication et des containers Docker.

![Dashboard Grafana](images/grafana-dashboard.png.jpg)

- Node Exporter (9100) expose les metriques systeme
- Prometheus (9090) scrape et stocke les metriques
- Grafana (3000) visualise via des dashboards

---

## Commandes Docker essentielles

| Commande | Usage |
|---|---|
| docker images | Lister les images locales |
| docker ps -a | Lister tous les containers |
| docker run | Lancer un container |
| docker stop | Arreter un container |
| docker rm | Supprimer un container arrete |
| docker rmi | Supprimer une image |
| docker build | Construire une image |
| docker push | Deposer sur DockerHub |
| docker login | Se connecter a DockerHub |

---

## Ce que jai appris

- Un Dockerfile doit toujours commencer par FROM
- Jenkins orchestre toutes les etapes du pipeline automatiquement
- SonarQube detecte bugs, vulnerabilites et code smells
- Prometheus scrape et normalise les metriques
- Grafana visualise via des dashboards personnalisables
- Node Exporter expose les metriques CPU et RAM

## Prochaine etape

Integrer ce pipeline dans un projet personnel avec declenchement automatique via webhook GitHub.
