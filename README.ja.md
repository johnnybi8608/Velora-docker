# Velora-docker

[English](README.md) | [Español](README.es.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md) | [Português](README.pt.md)

## オープンソース告知

本プロジェクトは [OpenIM Server](https://github.com/openimsdk/open-im-server) をベースに開発・改造しています。

- 元プロジェクト: OpenIM Server
- ライセンス: Apache License 2.0
- リポジトリ: https://github.com/openimsdk/open-im-server

OpenIM チームの貢献に感謝します。

---

## デプロイ手順

推奨サーバー: Ubuntu 22.04 / メモリ 3.5 GB 以上。

## 手順 1: ドメインの準備

- メインドメイン: 例 `velora.velora.com`
- 管理画面ドメイン: 例 `admin.velora.velora.com`
- 通話サービスドメイン: 例 `livekit.velora.velora.com`
- 通話 TURN ドメイン: 例 `livekit-turn.velora.velora.com`

すべてをこの Docker スタックで動かす場合、4 つのドメインは同じサーバー IP を指して構いません。

アプリ上部のコンパスから開く Web 用に `explore.<メインドメイン>` 形式の追加ドメインも用意してください (例 `explore.velora.velora.com`)。

---

## 手順 2: Docker をインストール

### システム更新と依存パッケージ

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

### Docker 公式リポジトリを追加

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Docker Engine + buildx + compose プラグインをインストール

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 動作確認

```bash
sudo docker --version
sudo docker compose version
```

---

## 手順 3: Velora リポジトリを取得

```bash
git clone https://github.com/johnnybi8608/Velora-docker.git
cd Velora-docker
```

### `.env` 内の `127.0.0.1` をサーバーのグローバル IP (例 `107.210.218.187`) に置換

```bash
sed -i.bak 's|127\.0\.0\.1|107.210.218.187|g' .env
```

### 置換を確認

```bash
grep -E 'MINIO_EXTERNAL_ADDRESS|GRAFANA_URL' .env
```

### 動作確認

```bash
docker compose up -d
```

ブラウザで `http://<サーバーIP>:11002` (HTTP) を開き、管理画面のウェルカムページが表示されることを確認します (まだログイン不可)。

---

## 手順 4: LiveKit をデプロイ

```bash
docker run --rm livekit/livekit-server generate-keys
```

API Key / Secret が表示されるので、以下のコマンドで `YOUR_KEY` `YOUR_SECRET` `YOUR_DOMAIN` を実値に置き換えて設定します。

```bash
export LIVEKIT_API_KEY=YOUR_KEY
export LIVEKIT_API_SECRET=YOUR_SECRET
export LIVEKIT_DOMAIN=YOUR_DOMAIN
```

### `docker-compose.yaml` を編集

`livekit - command` セクションの `127.0.0.1` をすべて実サーバー IP に変更します。

```bash
nano docker-compose.yaml
```

### `chat-rpc-chat.yml` を更新 (実際の `API_KEY` / `API_SECRET` を入力)

```bash
sudo docker exec openim-chat sh -lc 'sed -i "s/^  key: \".*\"$/  key: \"API_KEY\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc 'sed -i "s/^  secret: \".*\"$/  secret: \"API_SECRET\"/" /openim-chat/config/chat-rpc-chat.yml'
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### `.env` に LiveKit 設定を追記

```bash
cat <<EOF >> .env

# LiveKit
LIVEKIT_URL=wss://${LIVEKIT_DOMAIN}
LIVEKIT_API_KEY=${LIVEKIT_API_KEY}
LIVEKIT_API_SECRET=${LIVEKIT_API_SECRET}
EOF
```

### `livekit.yaml` を更新

TURN ドメインを自分のドメイン (例 `livekit-turn.yourdomain.com`) に置き換えます。

```bash
sed -i 's/"livekit-turn.velora.velora.com"/"livekit-turn.yourdomain.yourdomain.com"/' livekit.yaml
```

生成した `YOUR_KEY` / `YOUR_SECRET` を設定します。

```bash
sed -i 's#  LK_API_KEY_REPLACE_ME_9f1c1f4b-3b6d-4a60-9b6a-8d2b4f6a6a77: LK_API_SECRET_REPLACE_ME_2a1e7b93-5b8f-4c6d-9a1e-77d2b0c41b12#  YOUR_KEY: YOUR_SECRET#' livekit.yaml
```

`cat livekit.yaml` を実行し、ファイル末尾が次の形になっているか確認してください。

```bash
keys:
  YOUR_KEY:YOUR_SECRET
```

### LiveKit を確認

```bash
curl -I http://127.0.0.1:7880
```

---

## 手順 5: Nginx を設定

### Nginx / Certbot をインストール

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
```

### (必要に応じて) unattended-upgrades を停止

```bash
sudo systemctl stop unattended-upgrades
```

### ポート 80 を空けるため Nginx を停止

```bash
sudo systemctl stop nginx
```

### 証明書を取得 (ドメイン・メールを置き換え)

```bash
# メインドメイン
sudo certbot certonly --standalone \
  -d velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# 管理ドメイン
sudo certbot certonly --standalone \
  -d admin.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# LiveKit ドメイン
sudo certbot certonly --standalone \
  -d livekit.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email

# LiveKit TURN ドメイン
sudo certbot certonly --standalone \
  -d livekit-turn.velora.velora.com \
  -m your@email.com --agree-tos --no-eff-email
```

### Nginx 設定を投入

`###########` から `###########` までをターミナルに貼り付け、ドメインと証明書パスを実環境に合わせてください。

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

### 設定をテスト

```bash
sudo nginx -t
```

### (任意) デフォルト設定を削除

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## 手順 6: Minio を設定

### Docker を起動

```bash
docker compose up -d
```

### Minio 設定を修正

`https://velora.velora.com` を自分のメインドメインに置換します。

```bash
docker compose exec openim-server sh -lc \
"sed -i 's#^internalAddress:.*#internalAddress: minio:9000#; s#^externalAddress:.*#externalAddress: https://velora.velora.com/im-minio-api#' /openim-server/config/minio.yml && grep -nE 'internalAddress|externalAddress' /openim-server/config/minio.yml"

sed -i 's#^MINIO_EXTERNAL_ADDRESS=.*#MINIO_EXTERNAL_ADDRESS="https://velora.velora.com/im-minio-api"#' .env
```

---

## 手順 7: システムを起動

### Docker を再起動

```bash
docker compose down
docker compose up -d
```

### Nginx を起動

```bash
sudo systemctl restart nginx
```

### Nginx の状態を確認

```bash
sudo systemctl status nginx
```

`https://admin.velora.velora.com` にアクセスし、ウェルカムページが表示されることを確認します。初期の管理者アカウントは `chatAdmin` なので、ログイン後すぐパスワードを変更してください。

## 注意 ⚠️⚠️⚠️

### 通話が接続できない場合は `chat-rpc-chat.yml` の key/secret を確認

```bash
sudo docker exec openim-chat sh -lc "grep -n 'key\\|secret' /openim-chat/config/chat-rpc-chat.yml"
```

### 値が最新でない場合は更新後にチャットサービスを再起動

```bash
docker compose restart openim-chat
```
