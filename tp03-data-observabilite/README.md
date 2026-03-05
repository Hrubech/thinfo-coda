# TP03 — DATA ET OBSERVABILITE

**Objectif :** Déployer des bases de données sécurisées (PostgreSQL, MongoDB) et une stack d'observabilité (Elasticsearch, Kibana, Filebeat) avec toutes les bonnes pratiques de hardening.  
**Niveau :** Master 1 Cybersécurité - CODA Orléans  
**Durée estimée :** 3h

## 🎯 OBJECTIF GLOBAL DU TP

À la fin de ce TP, vous serez capable de :
- Déployer PostgreSQL et MongoDB de façon sécurisée (moindre privilège, isolation réseau)
- Configurer Elasticsearch et Kibana pour la centralisation des logs
- Collecter des logs avec Filebeat et les envoyer à Elasticsearch
- Effectuer des requêtes de corrélation simples (logs + métriques)
- Appliquer les principes de hardening aux bases de données

## 📋 PRÉREQUIS

- Docker installé sur Debian 13
- Accès internet
- Droits `sudo`

## 🏗️ ARCHITECHTURE CIBLE

```bash
┌───────────────────────────────────────────────────────────────┐
│                         RÉSEAUX DOCKER                        │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  backend-db (interne, seulement DB)                           │
│  ├── postgres-db                                              │
│  └── mongodb                                                  │
│                                                               │
│  backend-app (pour les applis)                                │
│  ├── postgres-db (connectée)                                  │
│  ├── mongodb (connectée)                                      │
│  └── (future web-app)                                         │
│                                                               │
│  elastic (réseau dédié observabilité)                         │
│  ├── elasticsearch                                            │
│  ├── kibana                                                   │
│  └── filebeat                                                 │
│                                                               │
│  frontend (public)                                            │
│  └── (future reverse proxy)                                   │
└───────────────────────────────────────────────────────────────┘
```

### ÉTAPE 0 : PREPARATION

```bash
# Créer l'arborescence de travail
mkdir -p ~/tp03-data-observabilite/{secrets,postgres,mongodb,filebeat}
cd ~/tp03-data-observabilite

# Créer les réseaux
docker network create backend-db --internal   # DB uniquement, pas d'accès extérieur
docker network create backend-app             # Pour les applis
docker network create elastic                 # Pour Elastic Stack
docker network create frontend                # Pour exposition publique
```

### ÉTAPE 1 : POSTGRESQL SECURISE

```bash
# Fichier de secret
echo "SuperSecretPostgres2026!" > secrets/postgres_password.txt
chmod 600 secrets/postgres_password.txt

# Lancer PostgreSQL avec les bonnes capacités
docker run -d --name postgres-db \
  --network backend-db \
  -v $(pwd)/secrets:/run/secrets:ro \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/postgres_password.txt \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  --restart=unless-stopped \
  --memory="256m" \
  --cpus="0.5" \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --cap-add CHOWN \
  --cap-add DAC_OVERRIDE \
  --cap-add FOWNER \
  --cap-add SETGID \
  --cap-add SETUID \
  postgres:16-alpine

# Vérification
docker logs postgres-db --tail 10
# Attendu : "database system is ready to accept connections"

# Connecter PostgreSQL au réseau applicatif
docker network connect backend-app postgres-db
```

### ÉTAPE 2 : MONGODB SECURISE

```bash
# Fichier de secret
echo "MongoSuperSecret2026!" > secrets/mongo_password.txt
chmod 600 secrets/mongo_password.txt

# Configuration MongoDB
cat > mongodb/mongod.conf <<'EOF'
security:
  authorization: enabled
net:
  bindIp: 0.0.0.0
  port: 27017
  tls:
    mode: disabled
storage:
  dbPath: /data/db
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
EOF

# Lancer MongoDB
docker run -d --name mongodb \
  --network backend-db \
  -v $(pwd)/mongodb/mongod.conf:/etc/mongod.conf:ro \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD_FILE=/run/secrets/mongo_password.txt \
  -e MONGO_INITDB_DATABASE=appdb \
  -v $(pwd)/secrets:/run/secrets:ro \
  --memory="256m" \
  --cpus="0.5" \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --cap-add CHOWN \
  --cap-add DAC_OVERRIDE \
  --cap-add FOWNER \
  --cap-add SETGID \
  --cap-add SETUID \
  mongo:4.4.18 mongod -f /etc/mongod.conf

# Vérification
docker logs mongodb --tail 10
# Attendu : "Waiting for connections" ou "Listening on 0.0.0.0"

# Connecter MongoDB au réseau applicatif
docker network connect backend-app mongodb
```

### ÉTAPE 3 : ELASTIC STACK

```bash
# Lancer Elasticsearch
docker run -d --name elasticsearch \
  --network elastic \
  -p 9200:9200 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  --memory="1g" \
  --cpus="1.0" \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0

# Vérification
curl -s http://localhost:9200 | jq .version || echo "✅ Elasticsearch OK"

# Lancer Kibana
docker run -d --name kibana \
  --network elastic \
  -p 5601:5601 \
  -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
  --memory="512m" \
  --cpus="0.5" \
  docker.elastic.co/kibana/kibana:8.11.0

# Créer un index et des données de test
# Créer l'index
curl -X PUT "localhost:9200/logs-app" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "service": { "type": "keyword" },
      "user": { "type": "keyword" },
      "ip": { "type": "keyword" },
      "status_code": { "type": "integer" }
    }
  }
}'

# Insérer des logs de test (10 entrées)
curl -X POST "localhost:9200/logs-app/_bulk" -H 'Content-Type: application/json' --data-binary @- <<'EOF'
{"index":{}}
{"@timestamp":"2026-03-05T10:00:00Z","level":"INFO","message":"User login successful","service":"web-app","user":"alice","ip":"192.168.1.100","status_code":200}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:01Z","level":"INFO","message":"User login successful","service":"web-app","user":"bob","ip":"192.168.1.101","status_code":200}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:02Z","level":"WARN","message":"Failed login attempt","service":"web-app","user":"admin","ip":"185.142.53.1","status_code":401}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:03Z","level":"WARN","message":"Failed login attempt","service":"web-app","user":"admin","ip":"185.142.53.1","status_code":401}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:04Z","level":"WARN","message":"Failed login attempt","service":"web-app","user":"admin","ip":"185.142.53.1","status_code":401}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:05Z","level":"ERROR","message":"Database connection timeout","service":"web-app","user":"-","ip":"","status_code":500}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:06Z","level":"CRITICAL","message":"Database connection pool exhausted","service":"web-app","user":"-","ip":"","status_code":503}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:07Z","level":"INFO","message":"User login successful","service":"web-app","user":"charlie","ip":"192.168.1.102","status_code":200}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:08Z","level":"WARN","message":"Slow query detected","service":"postgres-db","user":"appuser","ip":"","status_code":-1}
{"index":{}}
{"@timestamp":"2026-03-05T10:00:09Z","level":"INFO","message":"User login successful","service":"web-app","user":"david","ip":"192.168.1.103","status_code":200}
EOF

# Vérifier le nombre de documents
curl -s "localhost:9200/logs-app/_count" | grep -o '"count":[0-9]*' | cut -d':' -f2
# Attendu : 10
```

### ÉTAPE 4 : FILEBEAT (COLLECTE DE LOGS)

```bash
# Configuration Filebeat
cat > filebeat/filebeat.yml <<'EOF'
filebeat.inputs:
- type: log
  paths:
    - '/var/log/*.log'
  fields:
    service: system

output.elasticsearch:
  hosts: ['elasticsearch:9200']
  indices:
    - index: "filebeat-system-%{+yyyy.MM.dd}"

logging.level: error
EOF

# Permissions strictes (indispensable !)
# Filebeat exige : propriétaire root ET permissions 644
sudo chown root:root filebeat/filebeat.yml
sudo chmod 644 filebeat/filebeat.yml
ls -la filebeat/filebeat.yml
# Attendu : -rw-r--r-- 1 root root ...

# Lancer Filebeat
docker run -d --name filebeat \
  --network elastic \
  -v /var/log:/var/log:ro \
  -v $(pwd)/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro \
  --user root \
  docker.elastic.co/beats/filebeat:8.11.0 filebeat -e

# Vérifier les logs
docker logs filebeat --tail 10
# Attendu : "No errors" ou "Connected to Elasticsearch"

# Test connexion avec elasticsearch
docker exec filebeat filebeat test output
```

### ÉTAPE 5 : TESTS DE CONNECTIVITE ET ISOLATION

```bash
# Script de test complet
cat > test-all.sh <<'EOF'
#!/bin/bash
echo "🔍 TEST DE TOUS LES SERVICES"
echo "============================="

# Test PostgreSQL
echo -n "PostgreSQL: "
docker run --rm --network backend-app mongo:4.4.18 \
  mongo --host mongodb --port 27017 -u admin -p MongoSuperSecret2026! \
  --quiet --eval "db.runCommand({ping:1}).ok" 2>/dev/null | grep -q "1" && echo "✅ OK" || echo "❌ ÉCHEC"

# Test MongoDB
echo -n "MongoDB: "
docker run --rm --network backend-app mongo:4.4.18 \
  mongo --host mongodb --port 27017 -u admin -p MongoSuperSecret2026! \
  --quiet --eval "db.runCommand({ping:1}).ok" 2>/dev/null | grep -q "1" && echo "✅ OK" || echo "❌ ÉCHEC"

# Test Elasticsearch
echo -n "Elasticsearch: "
docker run --rm --network elastic alpine:3.20 sh -c "
  apk add --no-cache curl >/dev/null 2>&1
  curl -s http://elasticsearch:9200 2>/dev/null | grep -q 'cluster_name' && echo '✅ OK' || echo '❌ ÉCHEC'
"

# Test Kibana
echo -n "Kibana: "
curl -s -I http://localhost:5601 2>/dev/null | grep -q "HTTP/1.1 302" && echo "✅ OK" || echo "❌ ÉCHEC"

# Test Filebeat
echo -n "Filebeat: "
docker ps | grep -q filebeat && echo "✅ OK" || echo "❌ ÉCHEC"

echo ""
echo "🌐 Services disponibles :"
echo "  - Kibana: http://localhost:5601"
echo "  - Elasticsearch: http://localhost:9200"
EOF

chmod +x test-all.sh
./test-all.sh

# Test d'isolation réseau
# Depuis frontend (doit échouer)
echo "Test depuis frontend (doit échouer):"
docker run --rm --network frontend alpine:3.20 sh -c "
  nc -zv postgres-db 5432 2>&1 || echo '✅ PostgreSQL inaccessible'
  nc -zv mongodb 27017 2>&1 || echo '✅ MongoDB inaccessible'
"

# Depuis backend-app (doit réussir)
echo "Test depuis backend-app (doit réussir):"
docker run --rm --network backend-app mongo:4.4.18 \
  mongo --host mongodb --port 27017 -u admin -p MongoSuperSecret2026! \
  --eval "db.adminCommand('listDatabases')" | grep -q "ok" && echo "✅ MongoDB accessible"

docker run --rm --network backend-app alpine:3.20 sh -c "
  apk add --no-cache postgresql-client >/dev/null 2>&1
  pg_isready -h postgres-db -p 5432 -U appuser | grep -q 'accepting' && echo '✅ PostgreSQL accessible'
"
```

### ÉTAPE 6 : REQUETES ET CORRELATION

```bash
# Rechercher les échecs de connexion
curl -s "localhost:9200/logs-app/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "message": "Failed login"
    }
  }
}' | grep -A 5 -B 5 "185.142.53.1"

# Agrégation par IP (détection brute force)
curl -s "localhost:9200/logs-app/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "ip_addresses": {
      "terms": {
        "field": "ip.keyword",
        "size": 5
      }
    }
  }
}' | grep -A 10 "buckets"

# Corrélation temporelle
curl -s "localhost:9200/logs-app/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "message": "Database" }}
      ],
      "filter": {
        "range": {
          "@timestamp": {
            "gte": "2026-03-05T10:00:00Z",
            "lte": "2026-03-05T10:01:00Z"
          }
        }
      }
    }
  }
}' | grep -A 5 "message"
```

### ÉTAPE 7 : NETTOYAGE

```bash
# Arrêter tous les conteneurs
docker rm -f postgres-db mongodb elasticsearch kibana filebeat

# Supprimer les réseaux
docker network rm backend-db backend-app elastic frontend

# Nettoyer les fichiers (optionnel)
# rm -rf ~/tp03-data-observabilite
```