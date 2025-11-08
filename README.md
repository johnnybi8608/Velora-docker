# Velora-docker

## 开源声明

本项目基于 [OpenIM Server](https://github.com/openimsdk/open-im-server) 进行开发和修改。

- 原项目: OpenIM Server
- 原项目许可证: Apache License 2.0
- 原项目地址: https://github.com/openimsdk/open-im-server

感谢 OpenIM 团队的开源贡献!

---

## 部署指南

推荐服务器: Ubuntu 22.04  内存不低于3.5G

## 第一步: 准备域名

- 主域名: 如 velora.velora.com
- 管理后台域名: 如 admin.velora.velora.com
- 通话服务域名: 如 livekit.velora.velora.com
- 通话服务 turn 域名: 如 livekit-turn.velora.velora.com

服务器 docker 已经涵盖了所有功能,如果全部功能都是用服务器 docker,上面四个域名可以全部指向服务器 IP。

另外准备一个域名,指向你的网站(可在 App 聊天页面顶部导航栏指南针图标访问),域名格式为 `explore.主域名`, 例如 `explore.velora.velora.com`。

---

## 第二步: 安装 docker

### 更新 & 依赖

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

### 添加 Docker 官方源

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 安装 Docker Engine + buildx + compose 插件

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 验证

```bash
sudo docker --version
sudo docker compose version
```

---

## 第三步: 拉取 Velora 仓库

```bash
git clone https://github.com/johnnybi8608/Velora-docker.git
cd Velora-docker
```

### 将 `.env` 中 `127.0.0.1` 替换为服务器公网 IP (如: `107.210.218.187`)

```bash
sed -i.bak 's|127\.0\.0\.1|107.210.218.187|g' .env
```

### 验证替换是否成功

```bash
grep -E 'MINIO_EXTERNAL_ADDRESS|GRAFANA_URL' .env
```

### 到这里可以测试一下

```bash
docker compose up -d
```

浏览器输入 `http://服务器IP:11002` (⚠️ 注意是 http 不是 https) 预期可以看到后台欢迎页面但是无法登录。

---

## 第四步: 部署 LiveKit

```bash
docker run --rm livekit/livekit-server generate-keys
```

上一条命令会输出 API Key、API Secret, 执行下面的命令, 将 `YOUR_KEY` 替换为上条命令输出的 API Key, `YOUR_SECRET` 替换为 API Secret, `YOUR_DOMAIN` 替换为上面准备的通话服务域名。

```bash
export LIVEKIT_API_KEY=YOUR_KEY
export LIVEKIT_API_SECRET=YOUR_SECRET
export LIVEKIT_DOMAIN=YOUR_DOMAIN
```

### 编辑 `docker-compose.yaml`

将文件中 `livekit - command` 部分的 `127.0.0.1` 替换为服务器真实 IP。

```bash
nano docker-compose.yaml
```

### 替换 `chat-rpc-chat.yml` 里的内容

```bash
sudo docker exec openim-chat sh -lc 'sed -i "s/^  key: \".*\"$/  key: \"${CHATENV_CHAT_RPC_CHAT_LIVEKIT_APIKEY}\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc 'sed -i "s/^  secret: \".*\"$/  secret: \"${CHATENV_CHAT_RPC_CHAT_LIVEKIT_APISECRET}\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### 编辑 `.env` 文件,加入 LiveKit 配置

```bash
cat <<EOF >> .env

# LiveKit
LIVEKIT_URL=wss://${LIVEKIT_DOMAIN}
LIVEKIT_API_KEY=${LIVEKIT_API_KEY}
LIVEKIT_API_SECRET=${LIVEKIT_API_SECRET}
EOF
```

### 将 LiveKit 配置加入 `livekit.yaml`

将下面命令中的 `livekit-turn.yourdomain.yourdomain.com` 替换为上面准备的通话服务 turn 域名。

```bash
sed -i 's/"livekit-turn.velora.velora.com"/"livekit-turn.yourdomain.yourdomain.com"/' livekit.yaml
```

将下面命令中的 `YOUR_KEY` 替换为上面准备的 API Key, `YOUR_SECRET` 替换为 API Secret。

```bash
sed -i 's#  LK_API_KEY_REPLACE_ME_9f1c1f4b-3b6d-4a60-9b6a-8d2b4f6a6a77: LK_API_SECRET_REPLACE_ME_2a1e7b93-5b8f-4c6d-9a1e-77d2b0c41b12#  YOUR_KEY: YOUR_SECRET#' livekit.yaml
```

### 验证 LiveKit

```bash
curl -I http://127.0.0.1:7880
```

---

## 第五步: 配置 Nginx

### 安装 Nginx / Certbot

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
```

### 如果报apt锁可以先停掉自动升级服务 (可选)
```bash
sudo systemctl stop unattended-upgrades
```

### 停掉 Nginx, 放行 80 端口

```bash
sudo systemctl stop nginx
```

### 申请证书 (注意替换域名和邮箱)

```bash
# 主域名
sudo certbot certonly --standalone \
  -d velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# 管理后台域名
sudo certbot certonly --standalone \
  -d admin.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# 通话服务域名
sudo certbot certonly --standalone \
  -d livekit.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# 通话服务 turn 域名
sudo certbot certonly --standalone \
  -d livekit-turn.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email
```

### 配置 Nginx `conf` 文件

下条命令从 `##########` 开始,一直到 `###########` 结束,全部复制粘贴到终端执行。注意细心替换配置文件中的域名、证书路径等。

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

### 测试 Nginx 配置是否正确

```bash
sudo nginx -t
```

### 删除默认配置 (可选)

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## 第六步: 配置 Minio

### 启动 docker

```bash
docker compose up -d
```

### 修改 Minio 配置

注意将命令中的域名 `https://velora.velora.com` 替换为上面准备的主域名。

```bash
docker compose exec openim-server sh -lc \
"sed -i 's#^internalAddress:.*#internalAddress: minio:9000#; s#^externalAddress:.*#externalAddress: https://velora.velora.com/im-minio-api#' /openim-server/config/minio.yml && grep -nE 'internalAddress|externalAddress' /openim-server/config/minio.yml"

sed -i 's#^MINIO_EXTERNAL_ADDRESS=.*#MINIO_EXTERNAL_ADDRESS="https://velora.velora.com/im-minio-api"#' .env
```

---

## 第七步: 启动系统

### 重新启动 docker

```bash
docker compose down
docker compose up -d
```

### 启动 Nginx

```bash
sudo systemctl restart nginx
```

### 查看 Nginx 状态

```bash
sudo systemctl status nginx
```

到这里, 浏览器输入 `https://admin.velora.velora.com` 预期可以看到后台欢迎页面。默认管理员账号密码均为 `chatAdmin`, 登录后请立即修改密码。
