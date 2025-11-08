# Velora-docker

[English](README.md) | [Español](README.es.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [Português](README.pt.md)

## Aviso de Código Aberto

Este projeto é desenvolvido com base no [OpenIM Server](https://github.com/openimsdk/open-im-server).

- Projeto original: OpenIM Server
- Licença original: Apache License 2.0
- Repositório original: https://github.com/openimsdk/open-im-server

Obrigado à equipe do OpenIM pela contribuição!

---

## Guia de Implantação

Servidor recomendado: Ubuntu 22.04 com pelo menos 3,5 GB de RAM.

## Passo 1: Preparar domínios

- Domínio principal: ex. velora.velora.com
- Domínio do painel administrativo: ex. admin.velora.velora.com
- Domínio do serviço de chamadas: ex. livekit.velora.velora.com
- Domínio TURN do serviço de chamadas: ex. livekit-turn.velora.velora.com

Se tudo rodar neste stack Docker, aponte os quatro domínios para o mesmo IP do servidor.

Reserve também um domínio extra para o site acessado pelo app (ícone de bússola). Use `explore.<domínio-principal>`, por exemplo `explore.velora.velora.com`.

---

## Passo 2: Instalar Docker

### Atualizar o sistema e instalar dependências

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

### Adicionar o repositório oficial do Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Instalar Docker Engine + buildx + plugin compose

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verificar

```bash
sudo docker --version
sudo docker compose version
```

---

## Passo 3: Clonar o repositório Velora

```bash
git clone https://github.com/johnnybi8608/Velora-docker.git
cd Velora-docker
```

### Trocar `127.0.0.1` no `.env` pelo IP público (ex. `107.210.218.187`)

```bash
sed -i.bak 's|127\.0\.0\.1|107.210.218.187|g' .env
```

### Conferir

```bash
grep -E 'MINIO_EXTERNAL_ADDRESS|GRAFANA_URL' .env
```

### Teste rápido

```bash
docker compose up -d
```

Abra `http://<ip-do-servidor>:11002` (HTTP). A tela inicial do painel deve aparecer, mas o login ainda falhará.

---

## Passo 4: Implantar o LiveKit

```bash
docker run --rm livekit/livekit-server generate-keys
```

O comando retorna API Key/Secret. Execute as instruções abaixo substituindo `YOUR_KEY`, `YOUR_SECRET` e `YOUR_DOMAIN` pelos seus valores.

```bash
export LIVEKIT_API_KEY=YOUR_KEY
export LIVEKIT_API_SECRET=YOUR_SECRET
export LIVEKIT_DOMAIN=YOUR_DOMAIN
```

### Editar `docker-compose.yaml`

Na seção `livekit - command`, troque cada `127.0.0.1` pelo IP real do servidor.

```bash
nano docker-compose.yaml
```

### Atualizar `chat-rpc-chat.yml` (use os valores reais de `API_KEY` / `API_SECRET`)

```bash
sudo docker exec openim-chat sh -lc 'sed -i "s/^  key: ".*"$/  key: "API_KEY"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc 'sed -i "s/^  secret: ".*"$/  secret: "API_SECRET"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc "grep -n 'key\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### Adicionar configurações do LiveKit ao `.env`

```bash
cat <<EOF >> .env

# LiveKit
LIVEKIT_URL=wss://${LIVEKIT_DOMAIN}
LIVEKIT_API_KEY=${LIVEKIT_API_KEY}
LIVEKIT_API_SECRET=${LIVEKIT_API_SECRET}
EOF
```

### Atualizar `livekit.yaml`

Substitua o domínio TURN pelo seu (ex. `livekit-turn.seudominio.com`).

```bash
sed -i 's/"livekit-turn.velora.velora.com"/"livekit-turn.yourdomain.yourdomain.com"/' livekit.yaml
```

Troque `YOUR_KEY` / `YOUR_SECRET` pelas credenciais geradas.

```bash
sed -i 's#  LK_API_KEY_REPLACE_ME_9f1c1f4b-3b6d-4a60-9b6a-8d2b4f6a6a77: LK_API_SECRET_REPLACE_ME_2a1e7b93-5b8f-4c6d-9a1e-77d2b0c41b12#  YOUR_KEY: YOUR_SECRET#' livekit.yaml
```

### Validar o LiveKit

```bash
curl -I http://127.0.0.1:7880
```

---

## Passo 5: Configurar o Nginx

### Instalar Nginx / Certbot

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
```

### (Opcional) parar unattended-upgrades se o apt estiver travado

```bash
sudo systemctl stop unattended-upgrades
```

### Parar o Nginx para liberar a porta 80

```bash
sudo systemctl stop nginx
```

### Solicitar certificados (ajuste domínios e e-mail)

```bash
# Domínio principal
sudo certbot certonly --standalone \
  -d velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Domínio do painel
sudo certbot certonly --standalone \
  -d admin.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Domínio do LiveKit
sudo certbot certonly --standalone \
  -d livekit.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Domínio TURN do LiveKit
sudo certbot certonly --standalone \
  -d livekit-turn.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email
```

### Configurar o Nginx

Copie o bloco entre `###########` e `###########`, cole no terminal e substitua pelos seus valores.

```
###########
sudo tee /etc/nginx/conf.d/velora.conf > /dev/null <<'EOF'
# Upstream definitions
upstream msg_gateway      { server 127.0.0.1:10001; }
upstream im_api           { server 127.0.0.1:10002; }
upstream im_chat_api      { server 127.0.0.1:10008; }
upstream im_admin_api     { server 127.0.0.1:10009; }
upstream minio_s3         { server 127.0.0.1:10005; }
upstream pc_web           { server 127.0.0.1:11001; }
upstream openim_admin     { server 127.0.0.1:11002; }

# Main web (velora.velora.com)
server {
    listen 443 ssl http2;
    server_name velora.velora.com;

    ssl_certificate     /etc/letsencrypt/live/velora.velora.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/velora.velora.com/privkey.pem;

    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_comp_level 2;
    gzip_types text/plain application/javascript text/css application/xml application/wasm;
    gzip_disable "MSIE [1-6]\.";

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://pc_web/;
    }

    location /msg_gateway {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://msg_gateway/;
    }

    location ^~/api/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Request-Api $scheme://$host/api;
        proxy_pass http://im_api/;
    }

    location ^~/chat/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://im_chat_api/;
    }

    location ^~/im-minio-api/ {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        chunked_transfer_encoding off;
        proxy_pass http://minio_s3/;
    }
}

# Admin web (admin.velora.velora.com)
server {
    listen 443 ssl http2;
    server_name admin.velora.velora.com;

    ssl_certificate     /etc/letsencrypt/live/admin.velora.velora.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.velora.velora.com/privkey.pem;

    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_comp_level 2;
    gzip_types text/plain application/javascript text/css application/xml application/wasm;
    gzip_disable "MSIE [1-6]\.";

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://openim_admin/;
    }

    location /msg_gateway {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://msg_gateway/;
    }

    location ^~/api/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Request-Api $scheme://$host/api;
        proxy_pass http://im_api/;
    }

    location ^~/chat/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://im_chat_api/;
    }

    location ^~/complete_admin/ {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_pass http://im_admin_api/;
    }
}

# HTTP -> HTTPS redirect
server {
    listen 80;
    server_name velora.velora.com admin.velora.velora.com livekit.velora.velora.com;
    return 301 https://$host$request_uri;
}

# LiveKit (livekit.velora.velora.com)
server {
    listen 443 ssl http2;
    server_name livekit.velora.velora.com;

    ssl_certificate     /etc/letsencrypt/live/livekit.velora.velora.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/livekit.velora.velora.com/privkey.pem;

    ssl_session_timeout 1d;
    ssl_session_cache shared:LiveKitSSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:7880;
    }
}
EOF
###########
```

### Testar a configuração

```bash
sudo nginx -t
```

### (Opcional) remover o site padrão

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## Passo 6: Configurar o Minio

### Iniciar Docker

```bash
docker compose up -d
```

### Ajustar o Minio

Substitua `https://velora.velora.com` pelo seu domínio principal.

```bash
docker compose exec openim-server sh -lc "sed -i 's#^internalAddress:.*#internalAddress: minio:9000#; s#^externalAddress:.*#externalAddress: https://velora.velora.com/im-minio-api#' /openim-server/config/minio.yml && grep -nE 'internalAddress|externalAddress' /openim-server/config/minio.yml"

sed -i 's#^MINIO_EXTERNAL_ADDRESS=.*#MINIO_EXTERNAL_ADDRESS="https://velora.velora.com/im-minio-api"#' .env
```

---

## Passo 7: Ligar o sistema

### Reiniciar Docker

```bash
docker compose down
docker compose up -d
```

### Reiniciar Nginx

```bash
sudo systemctl restart nginx
```

### Verificar status do Nginx

```bash
sudo systemctl status nginx
```

Agora acesse `https://admin.velora.velora.com` e faça login (usuário e senha padrão `chatAdmin`; altere imediatamente).

## Importante ⚠️⚠️⚠️

### Se chamadas de voz/vídeo falharem, confira o key/secret em `chat-rpc-chat.yml`

```bash
sudo docker exec openim-chat sh -lc "grep -n 'key\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### Se não estiverem atualizados, substitua e reinicie o serviço de chat

```bash
docker compose restart openim-chat
```
