# TP01 — DURCISSEMENT & DÉPLOIEMENT D’UNE ARCHITECTURE CONTENEURISÉE (DOCKER)
    
**Objectif :** Déployer une architecture Web + DB sécurisée en appliquant les bonnes pratiques DevSecOps.
**Environnement cible :** Debian 13 (Trixie) à jour
**Niveau :** Master 1 Cybersécurité - CODA Orléans
**Durée estimée :** 2h30

## 🎯 Objectifs pédagogiques

À la fin de ce TP, vous serez capable de :
- Installer Docker de manière sécurisée (chaîne de confiance)
- Créer une segmentation réseau stricte (frontend/backend)
- Construire une image Docker durcie (non-root, alpine, multi-stage)
- Scanner les vulnérabilités avec Trivy
- Appliquer le runtime hardening (read-only, capabilities, limites)
- Prouver l'isolation réseau par des tests concrets
- Gérer les secrets de manière sécurisée

## 📋 Prérequis

- Une machine Debian 13 (Trixie) avec accès internet
- Droits `sudo` sur la machine
- Connaissances de base en ligne de commande Linux

## 🏗️ Architecture cible

## 🔧 Déroulement du TP

### Étape 0 : Préparation de l'environnement

Avant de commencer, comprenez pourquoi ces étapes sont cruciales :

- **Dépôts à jour** : éviter d'installer des paquets vulnérables
- **Connectivité** : Docker et Trivy nécessitent des téléchargements
- **Clés GPG** : garantir l'authenticité des logiciels installés

```bash
# Vérifier la connectivité
ping -c 2 deb.debian.org
curl -I https://deb.debian.org

# Configurer les dépôts Debian
sudo tee /etc/apt/sources.list > /dev/null <<'EOF'
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
EOF

# Mettre à jour le système
sudo apt update && sudo apt upgrade -y
