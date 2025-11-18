# 🧩 Rocket.Chat + MongoDB Externo Seguro (Ubuntu 22.04)

---

## 🧰 1. Instalar MongoDB 8.0 (repositório oficial)

```bash
curl -fsSL https://pgp.mongodb.com/server-8.0.asc |   sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt update
sudo apt install -y mongodb-org
```

Habilite e inicie o serviço:
```bash
sudo systemctl enable mongod
sudo systemctl start mongod
```

---

## 🔑 2. Gerar chave de replicação segura (`/etc/mongod.key`)

```bash
openssl rand -base64 756 | sudo tee /etc/mongod.key
sudo chown mongodb:mongodb /etc/mongod.key
sudo chmod 600 /etc/mongod.key
```

---

## ⚙️ 3. Configurar o MongoDB

Arquivo `/etc/mongod.conf`:

```yaml
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  bindIp: 0.0.0.0
  port: 27017

replication:
  replSetName: rs0

#security:
#  keyFile: /etc/mongod.key
#  authorization: enabled

setParameter:
  enableLocalhostAuthBypass: false


# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: false
```

Reinicie:
```bash
sudo systemctl restart mongod
```

---

## 🧩 4. Inicializar o ReplicaSet

```bash
mongosh
```

Inicialize com o loopback:

```javascript
rs.initiate({
  _id: "rs0",
  members: [ { _id: 0, host: "127.0.0.1:27017" } ]
})
```

Depois altere para o IP real do servidor:

```javascript
cfg = rs.conf()
cfg.members[0].host = "172.16.0.75:27017"
rs.reconfig(cfg, { force: true })
```

Verifique:
```javascript
rs.status()
```

---

## 👤 5. Criar usuário administrador root

```bash
mongosh
```

```javascript
use admin
db.createUser({
  user: "admin",
  pwd: "Key(mudar)",
  roles: [ { role: "root", db: "admin" } ]
})
```
---

## ⚙️  6. Descomentar segurança

Arquivo `/etc/mongod.conf`:

```yaml

security:
  File: /etc/mongod.
  authorization: enabled
```

Reinicie:
```bash
sudo systemctl restart mongod
```

---

## 🔐 7. Testar login seguro

```bash
mongosh -u admin -p 'Key(mudar)' --authenticationDatabase admin
```

---

## 💬 8. Rocket.Chat (com Mongo externo e Oplog)

### 👤 Criar usuário Rocket.Chat no Mongo
```bash
mongosh -u admin -p 'Key(mudar)' --authenticationDatabase admin
use admin
db.createUser({
  user: "rocketchat",
  pwd: "Key(mudar)",
  roles: [
    { role: "readWrite", db: "rocketchat" },
    { role: "readWrite", db: "local" },
    { role: "dbAdmin", db: "rocketchat" }
  ]
})
```

---

## 🔐 9. Testar login seguro

```bash
mongosh "mongodb://rocketchat:Key(mudar)@172.16.0.75:27017/rocketchat?replicaSet=rs0&authSource=admin"
```

---

### 🐳 docker-compose.yml
```yaml
services:
  rocketchat:
    image: rocketchat/rocket.chat:latest
    container_name: rocketchat
    restart: unless-stopped
    environment:
      - PORT=3000
      - ROOT_URL=https://chat.meudominio.com.br
      - MONGO_URL=mongodb://rocketchat:Key(mudar)@172.16.0.75:27017/rocketchat?replicaSet=rs0&authSource=admin
      - MONGO_OPLOG_URL=mongodb://rocketchat:Key(mudar)@172.16.0.75:27017/local?replicaSet=rs0&authSource=admin
      - DEPLOY_METHOD=docker
      - Accounts_UseDNSDomainCheck=false
    logging:
     options:
      max-size: 10m
    ports:
      - 3005:3000
```

Verifique logs:
```bash
docker logs -f rocketchat
```

---
