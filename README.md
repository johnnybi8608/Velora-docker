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

# 运行
docker compose up -d
