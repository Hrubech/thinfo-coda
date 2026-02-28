# MODULE RAR — APPROFONDISSEMENT DES ARCHITECTURES, SYSTEMES ET RESEAUX

**Objectif :** Déployer une architecture Web + DB sécurisée en appliquant les bonnes pratiques DevSecOps  
**Environnement cible :** 4 jours pour maîtriser la sécurité des architectures modernes  
**Niveau :** Master 1 Cybersécurité - CODA Orléans

## 🎯 OBJECTIF DU MODULE

À la fin des 4 jours, vous devez être capable de :
1. Expliquer les architectures modernes (monolithe, microservices, cloud-native, sécurité)
2. Comprendre docker (isolation, image, runtime, hardening)
3. Comprendre kubernetes (orchestration, objets, réseau, sécurité)
4. Manipuler les outils concrets (docker,k8s, trivy, ...)
5. Rélier la technique à la réalité (vision SOC/DevSecOps)
6. Travailler en équipe (projet de fin de session)

## 📋 PLAN DETAILLE SUR 4 JOURS

Les 4 jours de la formation
- Jour 1 : Architectures & Docker (fondations + hardening)
- Jour 2 : Kubernetes (orchestration + sécurité)
- Jour 3 : Data & Observabilité (SQL/NoSQL/Elastic)
- Jour 4 : Projet + restitution (Projet de fin de session)

## 🔧 PREREQUIS TECHNIQUES

Ce dont vous avez besoin pour ce module : 

**Matériel**
- PC avec droits admin
- 8 Go RAM minimum
- 10 Go espace disque libre
- Connexion internet stable  

**Logiciels**
- Git (Source Control Management) : https://git-scm.com/install/
- Vagrant (VM Management) : https://developer.hashicorp.com/vagrant/install
- VirtualBox (Virtualization software) : https://www.virtualbox.org/wiki/Downloads
- Termius (Modern SSH Client) : https://termius.com/download/windows  

**Mise en place de l'environnement de travail**
```bash
$  git clone https://github.com/Hrubech/thinfo-coda.git
$  cd thinfo-coda
$  vagrant up
$  vagrant ssh
```

## 📋 TRAVAUX PRATIQUES

- [TP01](tp01-docker-hardening) : Déploiement & durcissement d'une architecture conteneurisée (Docker)
- [TP02](tp02-kubernetes) : Kubernetes (Déploiement, sécurité de base, tests)
- [TP03](tp03-projet) : Data & Observabilité (SQL/NoSQL/Elastic)
- [TP03](tp03-projet) : Projet + restitution (Projet de fin de session)