# Velora-docker

[English](README.md) | [Español](README.es.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [Português](README.pt.md)

## Aviso de Código Abierto

Este proyecto se desarrolla sobre [OpenIM Server](https://github.com/openimsdk/open-im-server).

- Proyecto original: OpenIM Server
- Licencia original: Apache License 2.0
- Repositorio original: https://github.com/openimsdk/open-im-server

¡Gracias al equipo de OpenIM por su contribución!

---

## Guía de Despliegue

Servidor recomendado: Ubuntu 22.04 con al menos 3.5 GB de RAM.

## Paso 1: Preparar los dominios

- Dominio principal: p. ej., velora.velora.com
- Dominio del panel administrativo: p. ej., admin.velora.velora.com
- Dominio del servicio de llamadas: p. ej., livekit.velora.velora.com
- Dominio del servicio TURN de llamadas: p. ej., livekit-turn.velora.velora.com

Si todo se ejecuta dentro de este stack de Docker, apunta los cuatro dominios a la misma IP del servidor.

Prepara también un dominio adicional para tu sitio web (desde el icono de brújula en la app). Usa el formato `explore.<dominio-principal>`, por ejemplo `explore.velora.velora.com`.

---

## Paso 2: Instalar Docker

### Actualizar el sistema e instalar dependencias

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

### Añadir el repositorio oficial de Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Instalar Docker Engine + buildx + plugin de compose

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

## Paso 3: Clonar el repositorio de Velora

```bash
git clone https://github.com/johnnybi8608/Velora-docker.git
cd Velora-docker
```

### Sustituir `127.0.0.1` en `.env` por la IP pública del servidor (ej. `107.210.218.187`)

```bash
sed -i.bak 's|127\.0\.0\.1|107.210.218.187|g' .env
```

### Verificar el reemplazo

```bash
grep -E 'MINIO_EXTERNAL_ADDRESS|GRAFANA_URL' .env
```

### Prueba rápida

```bash
docker compose up -d
```

Abre `http://<ip-del-servidor>:11002` en el navegador (usa HTTP). Deberías ver la pantalla de bienvenida del panel, aunque todavía no podrás iniciar sesión.

---

## Paso 4: Desplegar LiveKit

```bash
docker run --rm livekit/livekit-server generate-keys
```

El comando devuelve una API Key y un API Secret. Ejecuta estas órdenes sustituyendo `YOUR_KEY`, `YOUR_SECRET` y `YOUR_DOMAIN` por tus propios valores.

```bash
export LIVEKIT_API_KEY=YOUR_KEY
export LIVEKIT_API_SECRET=YOUR_SECRET
export LIVEKIT_DOMAIN=YOUR_DOMAIN
```

### Editar `docker-compose.yaml`

En la sección `livekit - command` cambia cada `127.0.0.1` por la IP real del servidor.

```bash
nano docker-compose.yaml
```

### Actualizar `chat-rpc-chat.yml` (reemplaza `API_KEY` / `API_SECRET` por los reales)

```bash
sudo docker exec openim-chat sh -lc 'sed -i "s/^  key: \".*\"$/  key: \"API_KEY\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc 'sed -i "s/^  secret: \".*\"$/  secret: \"API_SECRET\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### Añadir la configuración de LiveKit a `.env`

```bash
cat <<EOF >> .env

# LiveKit
LIVEKIT_URL=wss://${LIVEKIT_DOMAIN}
LIVEKIT_API_KEY=${LIVEKIT_API_KEY}
LIVEKIT_API_SECRET=${LIVEKIT_API_SECRET}
EOF
```

### Actualizar `livekit.yaml`

Cambia el dominio TURN por el tuyo (ej. `livekit-turn.tudominio.com`).

```bash
sed -i 's/"livekit-turn.velora.velora.com"/"livekit-turn.yourdomain.yourdomain.com"/' livekit.yaml
```

Reemplaza `YOUR_KEY` y `YOUR_SECRET` por las credenciales generadas.

```bash
sed -i 's#  LK_API_KEY_REPLACE_ME_9f1c1f4b-3b6d-4a60-9b6a-8d2b4f6a6a77: LK_API_SECRET_REPLACE_ME_2a1e7b93-5b8f-4c6d-9a1e-77d2b0c41b12#  YOUR_KEY: YOUR_SECRET#' livekit.yaml
```

Ejecuta `cat livekit.yaml` y confirma que el final del archivo quede exactamente así:

```bash
keys:
  YOUR_KEY:YOUR_SECRET
```

### Validar LiveKit

```bash
curl -I http://127.0.0.1:7880
```

---

## Paso 5: Configurar Nginx

### Instalar Nginx / Certbot

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
```

### (Opcional) detener unattended-upgrades si apt está bloqueado

```bash
sudo systemctl stop unattended-upgrades
```

### Detener Nginx para liberar el puerto 80

```bash
sudo systemctl stop nginx
```

### Solicitar certificados (sustituye dominios y correo)

```bash
# Dominio principal
sudo certbot certonly --standalone \
  -d velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Dominio del panel
sudo certbot certonly --standalone \
  -d admin.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Dominio de LiveKit
sudo certbot certonly --standalone \
  -d livekit.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# Dominio TURN de LiveKit
sudo certbot certonly --standalone \
  -d livekit-turn.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email
```

### Configurar Nginx

Copia el bloque entre `###########` y `###########`, pégalo en la terminal y reemplaza los valores por tus dominios/rutas reales.

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

### Probar la configuración

```bash
sudo nginx -t
```

### (Opcional) eliminar el sitio por defecto

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## Paso 6: Configurar Minio

### Iniciar Docker

```bash
docker compose up -d
```

### Editar la configuración de Minio

Reemplaza `https://velora.velora.com` por tu dominio principal.

```bash
docker compose exec openim-server sh -lc \
"sed -i 's#^internalAddress:.*#internalAddress: minio:9000#; s#^externalAddress:.*#externalAddress: https://velora.velora.com/im-minio-api#' /openim-server/config/minio.yml && grep -nE 'internalAddress|externalAddress' /openim-server/config/minio.yml"

sed -i 's#^MINIO_EXTERNAL_ADDRESS=.*#MINIO_EXTERNAL_ADDRESS="https://velora.velora.com/im-minio-api"#' .env
```

---

## Paso 7: Iniciar el sistema

### Reiniciar Docker

```bash
docker compose down
docker compose up -d
```

### Iniciar Nginx

```bash
sudo systemctl restart nginx
```

### Revisar el estado de Nginx

```bash
sudo systemctl status nginx
```

Ahora deberías poder visitar `https://admin.velora.velora.com` y ver la pantalla de bienvenida. El usuario y contraseña por defecto es `chatAdmin`; cámbialos inmediatamente.

## Importante ⚠️⚠️⚠️

### Si las llamadas de voz/video fallan, revisa el key/secret dentro de `chat-rpc-chat.yml`

```bash
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### Si los valores no son los últimos, actualízalos y reinicia el servicio de chat

```bash
docker compose restart openim-chat
```
