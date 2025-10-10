# 更新 & 依赖
sudo apt-get update

sudo apt-get install -y ca-certificates curl gnupg lsb-release

# 添加 Docker 官方源
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine + buildx + compose 插件
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 验证
sudo docker --version

sudo docker compose version


git clone https://github.com/johnnybi8608/Velora-docker.git

cd Velora-docker

sed -i.bak 's|192\.168\.1\.138|172.245.67.138|g' .env

# 验证
grep -E 'MINIO_EXTERNAL_ADDRESS|GRAFANA_URL' .env

1) 生成 LiveKit Key/Secret
docker run --rm livekit/livekit-server generate-keys
# 记下输出的 API Key / Secret

2) 替换 livekit.yaml 里的占位符（键值对格式）
Linux：
sed -i 's|LK_API_KEY_REPLACE_ME_9f1c1f4b-3b6d-4a60-9b6a-8d2b4f6a6a77|AP_YOUR_KEY|g' livekit.yaml
sed -i 's|LK_API_SECRET_REPLACE_ME_2a1e7b93-5b8f-4c6d-9a1e-77d2b0c41b12|YOUR_SECRET|g' livekit.yaml

把 AP_YOUR_KEY / YOUR_SECRET 换成上一步生成的值。

echo 'LIVEKIT_URL=wss://livekit.your-domain.com' >> .env
echo 'LIVEKIT_API_KEY=AP_YOUR_KEY'               >> .env
echo 'LIVEKIT_API_SECRET=YOUR_SECRET'            >> .env

docker compose up -d

# 健康检查
curl -I http://127.0.0.1:7880     # 返回 200/401/404 都行
nc -vz 127.0.0.1 7881             # succeeded 即 OK
docker logs --tail=40 livekit     # 看到 "starting LiveKit server" 即 OK