# TP02 — KUBERNETES : DÉPLOIEMENT, SÉCURITÉ DE BASE, TESTS

**Niveau :** Master 1 Cybersécurité — CODA Orléans  
**Environnement cible :** Debian 13 (Trixie) à jour — Machine connectée à Internet  
**Durée visée :** 1 journée (ou gros bloc) — beaucoup de manip, peu de théorie lourde

## 🎯 OBJECTIF GLOBAL DU TP

Mettre en place un petit cluster Kubernetes local (avec KIND), puis déployer une mini-application
(front + backend) avec de bonnes pratiques simples :
- Namespaces (isolation logique)
- Déploiement (Deployment) + exposition (Service)
- Ingress HTTP (accès “comme en vrai”)
- ConfigMap / Secret (configuration)
- Ressources (requests/limits), probes (readiness/liveness)
- Sécurité de base : non-root, capabilities drop, pas d’élévation de privilèges
- RBAC : créer un accès “lecture seule” pour un service account
- Tests : vérifications concrètes + commandes d’observation

## 📋 PRÉREQUIS

- Debian 13 connecté à Internet
- Docker installé et fonctionnel (TP01)
- Outils : curl, sudo

## CONVENTION

- Les commandes sont copiées/collées dans un terminal.
- Tout se passe en local (lab), pas besoin de cloud.

## 🔧 DÉROULEMENT DU TP

### ÉTAPE 0 : PRÉ-CHECK INTERNET + DOCKER

```bash
# Vérifie la connexion Internet (DNS + réseau)
ping -c 2 deb.debian.org
# Attendu : réponses OK

# Vérifie que Docker fonctionne (nécessaire pour KIND)
sudo docker run --rm hello-world
# Attendu : message “Hello from Docker!”
```

### ÉTAPE 1 : INSTALLER kubectl (CLIENT KUBERNETES) — MÉTHODE OFFICIELLE

```bash
# Installe les paquets requis (TLS + GPG)
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Crée le dossier keyrings APT (bonne pratique)
sudo install -m 0755 -d /etc/apt/keyrings

# Ajoute la clé GPG officielle Kubernetes (keyring)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# Attendu : fichier /etc/apt/keyrings/kubernetes-apt-keyring.gpg créé

# Ajoute le dépôt Kubernetes (v1.30 stable)
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

# Installe kubectl
sudo apt update
sudo apt install -y kubectl
# Attendu : kubectl installé

# Vérifie la version kubectl
kubectl version --client --output=yaml
# Attendu : version client affichée
```

### ÉTAPE 2 : INSTALLER KIND (KUBERNETES IN DOCKER) — CLUSTER LOCAL RAPIDE

```bash
# Télécharge kind (binaire Linux amd64)
curl -Lo kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64

# Rend exécutable et place dans /usr/local/bin
chmod +x kind
sudo mv kind /usr/local/bin/kind
# Attendu : commande kind disponible

# Vérifie la version de kind
kind version
# Attendu : version affichée
```

### ÉTAPE 3 : CRÉER UN CLUSTER KIND (AVEC PORTS 80/443 POUR INGRESS)

```bash
# Crée un fichier de config kind avec mapping ports 80/443 vers le node
cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF
# Attendu : fichier kind-config.yaml créé

# Crée le cluster (nom : coda-rar)
kind create cluster --name coda-rar --config kind-config.yaml
# Attendu : cluster créé + contexte kubectl configuré automatiquement

# Vérifie le contexte kubectl actuel
kubectl config current-context
# Attendu : kind-coda-rar

# Vérifie le node
kubectl get nodes -o wide
# Attendu : 1 node Ready
```

### ÉTAPE 4 : INSTALLER UN INGRESS CONTROLLER (NGINX) POUR ACCÈS HTTP "COMME EN PROD"

```bash
# Déploie ingress-nginx (manifest officiel pour kind)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
# Attendu : création namespace ingress-nginx + pods

# Attend que l’ingress controller soit prêt
kubectl -n ingress-nginx get pods -w
# Attendu : ingress-nginx-controller en Running/Ready (CTRL+C quand OK)
```

### ÉTAPE 5 : CRÉER UN NAMESPACE DÉDIÉ AU TP (ISOLATION LOGIQUE)

```bash
# Crée le namespace du TP
kubectl create namespace coda-rar
# Attendu : namespace créé

# Vérifie
kubectl get ns | grep coda-rar
# Attendu : ligne coda-rar
```

### ÉTAPE 6 : DÉPLOYER LE BACKEND (API SIMPLE) + SERVICE (CLUSTERIP)

```bash
# Crée un fichier backend.yaml (Deployment + Service)
cat > backend.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: coda-rar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: hashicorp/http-echo:1.0
        args: ["-text=Backend OK (CODA RAR)", "-listen=:8080"]
        ports:
          - containerPort: 8080
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true # OK pour cette image (pas d'écriture disque)
          capabilities:
            drop: ["ALL"]
        resources:
          limits:
            cpu: "200m"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: coda-rar
spec:
  selector:
    app: backend
  ports:
    - port: 8080
  type: ClusterIP
EOF
# Attendu : fichier backend.yaml créé

# Applique le backend
kubectl apply -f backend.yaml
# Attendu : deployment + service créés

# Vérifier l'état du déploiement
kubectl get deployment -n coda-rar

# Vérifier que les Pods sont bien lancés
kubectl get pods -n coda-rar -l app=backend

# Voir le service et son adresse IP interne
kubectl get svc -n coda-rar backend-svc

#Explication architecture

[ UTILISATEUR ]
      |
      | (Requête HTTP : http://coda-rar.local)
      v
 [ INGRESS ] -------------------- (Routage Couche 7 : Nom d'hôte/Path)
      |
      v
 [ SERVICE ] -------------------- (IP Virtuelle Stable : ClusterIP)
      |
      | (Load Balancing Interne)
      v
============================================================
|  [ DÉPLOIEMENT ]  <-- (Gère la stratégie & les mises à jour)
|         |
|         v
|  [ REPLICA SET ]  <-- (Gère le nombre de copies : 2)
|         |
|    ------------
|    |          |
|    v          v
| [ POD 1 ]  [ POD 2 ] <-- (Conteneurs avec SecurityContext)
============================================================
      (Isolation via Namespace : coda-rar)
```

### ÉTAPE 7 : DÉPLOYER LE FRONTEND (NGINX DEMO) + SERVICE

```bash
# Crée un fichier frontend.yaml (Deployment + Service)
# Note : Nginx a besoin d'écrire dans /var/cache/nginx même en mode restreint.
# On utilise un volume "emptyDir" pour simuler le tmpfs du TP01.

# On déploie une application qui répond "FRONTEND-OK" sur le port 8080
cat > frontend.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: coda-rar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: hashicorp/http-echo
        args:
        - "-text=FRONTEND-OK (CODA-RAR)"
        - "-listen=:8080"
        ports:
        - containerPort: 8080
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: coda-rar
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
EOF
# Attendu : fichier frontend.yaml créé

# Applique le frontend
kubectl apply -f frontend.yaml
# Attendu : deployment + service créés

# Vérifier que les pods sont Running
kubectl get pods -n coda-rar -l app=frontend

# Tester en interne (doit répondre "FRONTEND-OK")
sudo kubectl run --rm -i --tty -n coda-rar debug-echo --image=alpine -- sh -c "apk add --no-cache curl >/dev/null && curl http://frontend-svc"

# Vérifier que les 2 réplicas sont bien "Ready" (Disponibilité)
kubectl get deployment -n coda-rar frontend
# Attendu : READY 2/2

# Lister les Pods et vérifier leur statut (Stabilité)
kubectl get pods -n coda-rar -l app=frontend
# Attendu : STATUS "Running" (Si "CrashLoopBackOff", vérifiez les volumes tmpfs)

# Vérifier que le durcissement Read-Only est bien actif (Sécurité)
kubectl get pod -n coda-rar -l app=frontend -o jsonpath='{.items[0].spec.containers[0].securityContext.readOnlyRootFilesystem}{"\n"}'
# Attendu : true

# Vérifier que les volumes vides (emptyDir) sont bien montés (Permissions)
# Cela permet à Nginx d'écrire dans /var/cache/nginx malgré le mode Read-Only.
kubectl describe pod -n coda-rar -l app=frontend | grep -A 5 "Mounts:"
# Attendu : /var/cache/nginx et /var/run listés

# Vérifier l'existence du service associé (Réseau)
kubectl get svc -n coda-rar frontend-svc
# Attendu : TYPE ClusterIP avec une adresse IP interne affectée
```

### ÉTAPE 8 : EXPOSITION VIA INGRESS

```bash
echo "127.0.0.1 coda-rar.local" | sudo tee -a /etc/hosts

cat > ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: coda-rar-ingress
  namespace: coda-rar
spec:
  ingressClassName: nginx
  rules:
  - host: coda-rar.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
EOF

kubectl apply -f ingress.yaml

# Test final
curl -s http://coda-rar.local | head
```

### ÉTAPE 9 : SÉCURITÉ RBAC (LECTURE SEULE)

```bash
kubectl -n coda-rar create serviceaccount viewer-sa

cat > rbac-viewer.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: viewer-role
  namespace: coda-rar
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f rbac-viewer.yaml

#lie le rôle viewer-role au compte de service viewer-sa au sein du namespace coda-rar, autorisant ainsi ce compte à effectuer les actions définies dans le rôle (lecture seule)

kubectl -n coda-rar create rolebinding viewer-binding \
  --role=viewer-role \
  --serviceaccount=coda-rar:viewer-sa

# Vérification des droits
kubectl -n coda-rar auth can-i list pods --as=system:serviceaccount:coda-rar:viewer-sa
# Attendu : yes
```

### ÉTAPE 10 : NETTOYAGE

```bash
# kind delete cluster --name coda-rar
```