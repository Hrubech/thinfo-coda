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
        args:
          - "-text=Backend OK (CODA RAR)"
          - "-listen=:8080"
        ports:
          - containerPort: 8080
        # Sécurité simple : non-root + pas d'escalade + FS read-only + drop capabilities
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        # Ressources : éviter qu’un pod consomme tout
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        # Probes : Kubernetes vérifie l’état du conteneur
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
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
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
EOF
# Attendu : fichier backend.yaml créé

# Applique le backend
kubectl apply -f backend.yaml
# Attendu : deployment + service créés

# Vérifie les pods
kubectl -n coda-rar get pods -o wide
# Attendu : 2 pods backend Running

# Vérifie le service
kubectl -n coda-rar get svc
# Attendu : backend-svc en ClusterIP
```

### ÉTAPE 7 : DÉPLOYER LE FRONTEND (NGINX DEMO) + SERVICE

```bash
# Crée un fichier frontend.yaml (Deployment + Service)
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
        image: nginxdemos/hello:plain-text
        ports:
          - containerPort: 80
        securityContext:
          runAsNonRoot: true
          runAsUser: 101
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
          capabilities:
            drop: ["ALL"]
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
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
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
EOF
# Attendu : fichier frontend.yaml créé

# Applique le frontend
kubectl apply -f frontend.yaml
# Attendu : deployment + service créés

# Vérifie
kubectl -n coda-rar get pods
kubectl -n coda-rar get svc
# Attendu : frontend pods Running + frontend-svc ClusterIP
```

### ÉTAPE 8 : TESTS INTERNES (CLUSTER) : PORT-FORWARD + CURL

```bash
# Ouvre un tunnel local vers le service backend (port 18080 local)
kubectl -n coda-rar port-forward svc/backend-svc 18080:8080
# Attendu : “Forwarding from 127.0.0.1:18080 …”
# (LAISSE CE TERMINAL OUVERT)

# Dans un 2e terminal : teste le backend
curl -s http://127.0.0.1:18080
# Attendu : “Backend OK (CODA RAR)”

# Stoppe le port-forward (CTRL+C) quand terminé
# Attendu : retour au prompt
```

### ÉTAPE 9 : EXPOSER EN HTTP AVEC INGRESS (ACCÈS “COMME EN VRAI”)

```bash
# Ajoute une entrée DNS locale sur la machine (Linux) pour pointer vers localhost
# (On simule un nom de domaine)
echo "127.0.0.1 coda-rar.local" | sudo tee -a /etc/hosts
# Attendu : la ligne est ajoutée

# Crée un Ingress pour le frontend
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
# Attendu : fichier ingress.yaml créé

# Applique l’ingress
kubectl apply -f ingress.yaml
# Attendu : ingress créé

# Vérifie
kubectl -n coda-rar get ingress
# Attendu : host coda-rar.local visible

# Teste via le host (ingress)
curl -s http://coda-rar.local | head
# Attendu : réponse texte de nginxdemos/hello
```

### ÉTAPE 10 : CONFIGURATION : CONFIGMAP + SECRET (CONCEPTS + CAS CONCRET)

```bash
# Crée un ConfigMap (config non sensible)
kubectl -n coda-rar create configmap app-config --from-literal=ENV=dev --from-literal=APP_NAME="CODA-RAR"
# Attendu : configmap créé

# Crée un Secret (donnée sensible) — exemple token
kubectl -n coda-rar create secret generic app-secret --from-literal=API_TOKEN="coda-token-123!"
# Attendu : secret créé

# Vérifie la présence
kubectl -n coda-rar get configmap app-config -o yaml | head -n 30
# Attendu : clés ENV / APP_NAME visibles

kubectl -n coda-rar get secret app-secret -o yaml | head -n 30
# Attendu : data en base64 (pas en clair)

# Injecte ConfigMap + Secret dans le backend (en variables d’environnement)
kubectl -n coda-rar set env deployment/backend --from=configmap/app-config
kubectl -n coda-rar set env deployment/backend --from=secret/app-secret
# Attendu : deployment mis à jour (rolling update)

# Vérifie que de nouveaux pods ont été recréés
kubectl -n coda-rar get pods -l app=backend
# Attendu : pods redémarrés (nouveaux IDs)

# Affiche les variables d’environnement dans un pod backend
POD_BACKEND=$(kubectl -n coda-rar get pod -l app=backend -o jsonpath='{.items[0].metadata.name}')
kubectl -n coda-rar exec -it "$POD_BACKEND" -- sh -c 'env | grep -E "ENV=|APP_NAME=|API_TOKEN="'
# Attendu : variables présentes (attention : API_TOKEN affiché => discussion sur la gestion des secrets)
```

### ÉTAPE 11 : OBSERVABILITÉ SIMPLE : LOGS, DESCRIBE, EVENTS

```bash
# Affiche les logs du backend
kubectl -n coda-rar logs -l app=backend --tail=50
# Attendu : logs du serveur http-echo

# Décrit le deployment (utile pour debug)
kubectl -n coda-rar describe deployment backend | head -n 60
# Attendu : image, replicas, events, probes, resources

# Affiche les events du namespace (diagnostic)
kubectl -n coda-rar get events --sort-by=.metadata.creationTimestamp | tail -n 20
# Attendu : événements récents (pull image, started container, etc.)
```

### ÉTAPE 12 : RBAC SIMPLE : CRÉER UN ACCÈS "LECTURE SEULE" AU NAMESPACE

```bash
# Crée un ServiceAccount “viewer-sa”
kubectl -n coda-rar create serviceaccount viewer-sa
# Attendu : serviceaccount créé

# Crée un Role lecture seule (pods, services, ingress)
cat > rbac-viewer.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: viewer-role
  namespace: coda-rar
rules:
- apiGroups: [""]
  resources: ["pods","services","configmaps"]
  verbs: ["get","list","watch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","list","watch"]
EOF
# Attendu : fichier rbac-viewer.yaml créé

# Applique le Role
kubectl apply -f rbac-viewer.yaml
# Attendu : role créé

# Lie le Role au ServiceAccount
kubectl -n coda-rar create rolebinding viewer-binding --role=viewer-role --serviceaccount=coda-rar:viewer-sa
# Attendu : rolebinding créé

# Test “can-i” : vérifie ce que viewer-sa a le droit de faire
kubectl -n coda-rar auth can-i list pods --as=system:serviceaccount:coda-rar:viewer-sa
# Attendu : yes

kubectl -n coda-rar auth can-i delete pods --as=system:serviceaccount:coda-rar:viewer-sa
# Attendu : no
```

### ÉTAPE 13 : COMPORTEMENT KUBERNETES : AUTO-HEALING + SCALING + ROLLING UPDATE

```bash
# Supprime un pod frontend manuellement (K8s doit le recréer)
POD_FRONT=$(kubectl -n coda-rar get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl -n coda-rar delete pod "$POD_FRONT"
# Attendu : pod supprimé puis recréé automatiquement

# Observe la recréation
kubectl -n coda-rar get pods -l app=frontend -w
# Attendu : nouveau pod apparaît (CTRL+C quand OK)

# Scale : passe frontend à 4 replicas
kubectl -n coda-rar scale deployment/frontend --replicas=4
# Attendu : 4 pods frontend

kubectl -n coda-rar get pods -l app=frontend
# Attendu : 4 pods Running

# Rolling update : change l’image frontend (déclenche remplacement progressif)
kubectl -n coda-rar set image deployment/frontend frontend=nginxdemos/hello:plain-text
# Attendu : rollout (peut être instant, même image => pas de changement réel, mais commande pédagogique)

# Vérifie l’état du rollout
kubectl -n coda-rar rollout status deployment/frontend
# Attendu : “successfully rolled out”
```

### ÉTAPE 14 : MINI-EXERCICES POUR OCCUPER ET RENFORCER (À FAIRE PAR BINÔMES)

```bash
# Exercice A (10–15 min) : changer la page / message backend
# - modifier args du backend (texte), puis vérifier avec port-forward

# Exercice B (15–20 min) : durcir encore plus
# - ajouter readOnlyRootFilesystem sur frontend si possible
# - vérifier que le pod démarre toujours
# (Si ça casse, expliquer pourquoi : l’app a parfois besoin d’écrire quelque part.)

# Exercice C (15–20 min) : requests/limits
# - mettre un limit trop bas (ex: memory=32Mi) et observer OOMKilled
# - puis corriger à 128Mi et vérifier stabilité

# Exercice D (15–20 min) : probes
# - casser readinessProbe (mauvais port) et observer que le pod est “Not Ready”
# - corriger et vérifier “Ready”

# Exercice E (10–15 min) : RBAC
# - étendre viewer-role pour autoriser “get deployments”
# - tester à nouveau “kubectl auth can-i …”
```

### ÉTAPE 15 : NETTOYAGE (FIN DE TP)

```bash
# Supprime toutes les ressources du namespace
kubectl delete namespace coda-rar
# Attendu : namespace supprimé + ressources supprimées

# Supprime le cluster kind
kind delete cluster --name coda-rar
# Attendu : cluster supprimé

# (Option) retire l’entrée /etc/hosts (si besoin)
# sudo sed -i '/coda-rar\.local/d' /etc/hosts
```