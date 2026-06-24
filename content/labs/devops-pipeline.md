---
title: "Projet DevOps - Pipeline CI/CD complet"
date: 2026-06-23
draft: false
tags: ["Docker", "Jenkins", "SonarQube", "Prometheus", "Grafana", "CI/CD", "DevOps"]
categories: ["Labs"]
description: "Pipeline CI/CD avec Docker, Jenkins, SonarQube, Prometheus et Grafana sur WSL."
showToc: true
---

## Objectif

Construire une chaine DevOps complete autour d une application Task Manager en Node.js.
Chaque push sur GitHub declenche automatiquement tests, analyse qualite, build Docker et deploiement.

Environnement : Windows 11 + WSL2 Ubuntu 22.04, Docker, Jenkins, SonarQube, Prometheus, Grafana.

---

## Partie 01 - WSL et Ubuntu

WSL permet de faire tourner Linux dans Windows sans VM classique. Plus leger et mieux integre.

Installation dans PowerShell administrateur : wsl --install

Un script shell installe ensuite Git, Node.js 18, Java 17, Docker, Jenkins, SonarQube, Prometheus, Grafana.

chmod +x install-devops.sh puis ./install-devops.sh

![](/images/docker-ps.png)

Services disponibles sur localhost : Jenkins port 8080, SonarQube port 9000, Prometheus port 9090, Grafana port 3000, Node Exporter port 9100.

---

## Partie 02 - Git et collaboration

Git est le point de depart de tout pipeline DevOps. Chaque push declenche Jenkins via webhook.

git init projet_devops
git add .
git commit -m first commit
git push -u origin main

![](/images/github-repo.jpg)

Structure du depot : backend, frontend, package.json, Dockerfile, Jenkinsfile.

---

## Partie 03 - Docker

Sans Docker, l application fonctionne chez le developpeur mais pas en production a cause des differences de versions. Docker empackette tout dans une image portable.

Dockerfile utilise :
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD npm start

docker build -t devops-app .
docker push monusername/devops-app:latest

![](/images/devops-app.jpg)

---

## Partie 04 - Pipeline Jenkins

Le Jenkinsfile definit 7 etapes executees automatiquement a chaque push :
Checkout SCM 2s, Install 4s, Compile 1s, Test 1s, SonarQube Analysis 1min, Docker Build 5min, Docker Push 1s.

Si une etape echoue, le pipeline s arrete. Le code defectueux ne peut pas atteindre la production.

![](/images/jenkins-pipeline.jpg)

---

## Partie 05 - SonarQube

SonarQube analyse le code pour detecter bugs, vulnerabilites, code smells et duplications.

Resultats obtenus : Bugs 1, Vulnerabilites 0, Code Smells 0, Duplication 0%, Statut Passed.

![](/images/sonarqube-dashboard.jpg)

Le Quality Gate bloque le deploiement si le code ne respecte pas les seuils definis.

---

## Partie 06 - Prometheus et Grafana

Node Exporter port 9100 expose les metriques systeme.
Prometheus port 9090 collecte ces metriques toutes les 15 secondes.
Grafana port 3000 visualise via des dashboards personnalisables.

![](/images/grafana-dashboard.jpg)

Le dashboard montre CPU, RAM, disque et reseau en temps reel.

---

## Commandes Docker essentielles

docker images - lister les images
docker ps -a - lister les conteneurs
docker run -d -p 3000 app - lancer un conteneur
docker stop - arreter un conteneur
docker build -t nom . - construire une image
docker push user/image - publier sur DockerHub
docker logs - voir les logs
docker exec bash - terminal dans un conteneur

---

## Ce que j ai appris

Docker : un conteneur partage le noyau Linux mais est isole. Plus leger qu une VM.
Jenkins : le Jenkinsfile versionne avec le code est une bonne pratique DevOps.
SonarQube : l analyse statique complete les tests unitaires.
Prometheus/Grafana : la supervision doit etre mise en place des le debut du projet.

## Prochaine etape

Ajouter Ansible pour le deploiement automatise, Trivy pour scanner les images Docker, et un webhook GitHub pour le declenchement instantane.
