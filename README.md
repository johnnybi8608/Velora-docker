# Velora-docker

[English](README.md) | [Español](README.es.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [Português](README.pt.md)

## Open Source Notice

This project is developed on top of [OpenIM Server](https://github.com/openimsdk/open-im-server).

- Upstream project: OpenIM Server
- Upstream license: Apache License 2.0
- Upstream repository: https://github.com/openimsdk/open-im-server

Thanks to the OpenIM team for their open-source contribution!

---

## Deployment Guide

Recommended server: Ubuntu 22.04 with at least 3.5 GB RAM.

## Step 1: Prepare Domains

- Primary domain: e.g., velora.velora.com
- Admin console domain: e.g., admin.velora.velora.com
- Calling service domain: e.g., livekit.velora.velora.com
- Calling service TURN domain: e.g., livekit-turn.velora.velora.com

If you deploy every component with this Docker stack, point all four domains to the same server IP.

Prepare an extra domain for your web site (the compass icon inside the chat list opens it). Use the format `explore.<primary-domain>`, for example `explore.velora.velora.com`.

---

## Step 2: Install Docker

### Update the system & install dependencies

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

### Add the Docker official repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install Docker Engine + buildx + compose plugin

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify

```bash
sudo docker --version
sudo docker compose version
```

---

## Step 3: Clone the Velora repository

```bash
git clone https://github.com/johnnybi8608/Velora-docker.git
cd Velora-docker
```

### Replace `127.0.0.1` in `.env` with the server public IP (for example `107.210.218.187`)

```bash
sed -i.bak 's|127\.0\.0\.1|107.210.218.187|g' .env
```

### Verify the replacement

```bash
grep -E 'MINIO_EXTERNAL_ADDRESS|GRAFANA_URL' .env
```

### Quick smoke test

```bash
docker compose up -d
```

Open `http://<server-ip>:11002` in a browser (note: HTTP, not HTTPS). You should see the admin welcome page but login will fail for now.

---

## Step 4: Deploy LiveKit

```bash
docker run --rm livekit/livekit-server generate-keys
```

The command prints an API Key and API Secret. Run the commands below and replace `YOUR_KEY` with the API Key, `YOUR_SECRET` with the API Secret, and `YOUR_DOMAIN` with the calling-service domain.

```bash
export LIVEKIT_API_KEY=YOUR_KEY
export LIVEKIT_API_SECRET=YOUR_SECRET
export LIVEKIT_DOMAIN=YOUR_DOMAIN
```

### Edit `docker-compose.yaml`

In the `livekit - command` section change every `127.0.0.1` to the real server IP.

```bash
nano docker-compose.yaml
```

### Update `chat-rpc-chat.yml` (replace `API_KEY` / `API_SECRET` with real values)

```bash
sudo docker exec openim-chat sh -lc 'sed -i "s/^  key: \".*\"$/  key: \"API_KEY\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc 'sed -i "s/^  secret: \".*\"$/  secret: \"API_SECRET\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### Append LiveKit settings to `.env`

```bash
cat <<EOF >> .env

# LiveKit
LIVEKIT_URL=wss://${LIVEKIT_DOMAIN}
LIVEKIT_API_KEY=${LIVEKIT_API_KEY}
LIVEKIT_API_SECRET=${LIVEKIT_API_SECRET}
EOF
```

### Update `livekit.yaml`

Replace the TURN domain placeholder with your own domain (for example `livekit-turn.yourdomain.com`).

```bash
sed -i 's/"livekit-turn.velora.velora.com"/"livekit-turn.yourdomain.yourdomain.com"/' livekit.yaml
```

Replace `YOUR_KEY` and `YOUR_SECRET` with the API credentials you generated earlier.

```bash
sed -i 's#  LK_API_KEY_REPLACE_ME_9f1c1f4b-3b6d-4a60-9b6a-8d2b4f6a6a77: LK_API_SECRET_REPLACE_ME_2a1e7b93-5b8f-4c6d-9a1e-77d2b0c41b12#  YOUR_KEY: YOUR_SECRET#' livekit.yaml
```

After running the command, ensure the end of `livekit.yaml` looks like:

```bash
keys:
  YOUR_KEY:YOUR_SECRET
```

### Validate LiveKit

```bash
curl -I http://127.0.0.1:7880
```

---

## Step 5: Configure Nginx

### Install Nginx / Certbot

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
```

### (Optional) stop unattended upgrades if apt is locked

```bash
sudo systemctl stop unattended-upgrades
```

### Stop Nginx and free port 80

```bash
sudo systemctl stop nginx
```

### Issue certificates (replace domains and email)

```bash
# Primary domain
sudo certbot certonly --standalone \
  -d velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Admin domain
sudo certbot certonly --standalone \
  -d admin.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# LiveKit domain
sudo certbot certonly --standalone \
  -d livekit.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# LiveKit TURN domain
sudo certbot certonly --standalone \
  -d livekit-turn.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email
```

### Configure Nginx

Copy everything from `###########` to `###########`, paste into the terminal, and replace the sample domains/cert paths with yours.

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
    gzip_disable "MSIE [1-6]\\.";

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
    gzip_disable "MSIE [1-6]\\.";

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

### Test the config

```bash
sudo nginx -t
```

### (Optional) remove the default config

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## Step 6: Configure Minio

### Start Docker

```bash
docker compose up -d
```

### Edit Minio configuration

Replace `https://velora.velora.com` with your primary domain.

```bash
docker compose exec openim-server sh -lc \
"sed -i 's#^internalAddress:.*#internalAddress: minio:9000#; s#^externalAddress:.*#externalAddress: https://velora.velora.com/im-minio-api#' /openim-server/config/minio.yml && grep -nE 'internalAddress|externalAddress' /openim-server/config/minio.yml"

sed -i 's#^MINIO_EXTERNAL_ADDRESS=.*#MINIO_EXTERNAL_ADDRESS="https://velora.velora.com/im-minio-api"#' .env
```

---

## Step 7: Start the system

### Restart Docker

```bash
docker compose down
docker compose up -d
```

### Start Nginx

```bash
sudo systemctl restart nginx
```

### Check Nginx status

```bash
sudo systemctl status nginx
```

At this point you should be able to visit `https://admin.velora.velora.com` and see the admin welcome page. The default admin username/password is `chatAdmin`; change it immediately after logging in.

## Important ⚠️⚠️⚠️

### If voice/video calls fail, double-check the key/secret inside `chat-rpc-chat.yml`

```bash
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### If the values are outdated, replace them and restart the chat service

```bash
docker compose restart openim-chat
```
