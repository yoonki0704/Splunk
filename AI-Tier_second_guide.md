# 처음부터 다시 — 완전한 설치 가이드 🚀

---

## 전체 진행 순서

```
[사전작업 1] 새 EC2 인스턴스 준비
      ↓
[사전작업 2] Admin 서버 환경 구성
      ↓
[사전작업 3] MinIO + Docker Registry 구성
      ↓
[사전작업 4] Registry HTTP 허용 설정
      ↓
[사전작업 5] yaml 파일 작성
      ↓
[설치 실행] validate → install
```

---

### 필요한 EC2 인스턴스 구성

DEPLOYMENT_GUIDE.md 기준으로 **최소 4대의 서버**가 필요합니다.
> OS : RHEL version 9

| 역할 | 수량 | 권장 인스턴스 | CPU | RAM | 디스크 |
|------|------|-------------|-----|-----|--------|
| Controller | 1대 | `m5.2xlarge` | 4코어 | 8GB | 150GB |
| CPU Worker | 1대+ | `m5.4xlarge` | 8코어 | 32GB | 250GB |
| GPU Worker | **2대 필수** | `g6e.12xlarge` | 48코어 | 384GB | 550GB |
| Admin (MinIO + Registry) | 1대 | `t3.xlarge` | - | - | **450GB** (모델 다운로드용) |

> ⚠️ **중요:** GPU Worker는 **반드시 2대** 필요합니다. 1대로는 AI 추론 스택이 동작하지 않아요.

### Security Group 설정

**클러스터 전용 Security Group 1개 생성:**

```
이름: splunk-ai-sg

Inbound Rules:
┌─────────────┬──────┬────────────────────────────────┐
│ Type        │ Port │ Source                         │
├─────────────┼──────┼────────────────────────────────┤
│ All traffic │ All  │ splunk-ai-sg (자기 자신)        │
│ SSH         │ 22   │ 내 IP (Admin PC)               │
└─────────────┴──────┴────────────────────────────────┘
```

> ✅ **모든 인스턴스(Admin 포함)를 이 Security Group 하나에 넣으세요.**

---

## 사전작업 2: Admin 서버 도구 설치

Admin 서버에서 실행:

```bash
# 기본 도구 설치
sudo apt-get update
sudo apt-get install -y git jq curl docker.io

# awscli 설치
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Docker 권한 설정
sudo usermod -aG docker ubuntu
newgrp docker

# kubectl 설치
KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
curl -LO "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# yq 설치
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
  -O /usr/local/bin/yq && chmod +x /usr/local/bin/yq

# mc (MinIO 클라이언트) 설치
wget https://dl.min.io/client/mc/release/linux-amd64/mc \
  -O /usr/local/bin/mc && chmod +x /usr/local/bin/mc

# 설치 확인
kubectl version --client && helm version && \
git --version && jq --version && yq --version
```

---

## 사전작업 3: Admin서버에 MinIO + Docker Registry 구성

```bash
# 데이터 디렉토리 생성
sudo mkdir -p /data/minio /data/registry
sudo chown -R ec2-user:ec2-user /data/minio /data/registry

# MinIO 설치
wget https://dl.min.io/server/minio/release/linux-amd64/minio \
  -O /usr/local/bin/minio && chmod +x /usr/local/bin/minio

# MinIO systemd 서비스 등록
sudo mkdir -p /etc/minio
sudo tee /etc/minio/minio.env > /dev/null <<EOF
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=MinioPassword123!
MINIO_VOLUMES=/data/minio
MINIO_OPTS="--address :9000 --console-address :9001"
EOF

sudo tee /etc/systemd/system/minio.service > /dev/null <<EOF
[Unit]
Description=MinIO Object Storage
After=network-online.target

[Service]
User=ubuntu
Group=ubuntu
EnvironmentFile=/etc/minio/minio.env
ExecStart=/usr/local/bin/minio server \$MINIO_VOLUMES \$MINIO_OPTS
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio

# Docker Registry 실행
docker run -d \
  --name docker-registry \
  --restart always \
  -p 5000:5000 \
  -v /data/registry:/var/lib/registry \
  registry:2

# MinIO 버킷 생성
mc alias set local http://localhost:9000 minioadmin MinioPassword123!
mc mb local/ai-platform-bucket

# 확인
echo "MinIO 상태:" && \
  curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:9000/minio/health/live
echo ""
echo "Registry 상태:" && \
  curl -s http://localhost:5000/v2/_catalog
```

---

## 사전작업 4: Admin 서버에서 Docker Registry HTTP 허용 설정

```bash
# Admin 서버에서 실행
REGISTRY_IP=172.31.51.179
echo "Registry IP: ${REGISTRY_IP}"

sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "insecure-registries": ["${REGISTRY_IP}:5000"]
}
EOF

sudo systemctl restart docker
docker start docker-registry

# 확인
curl -s http://${REGISTRY_IP}:5000/v2/_catalog
```
### 컨테이너 이미지 Download
```bash
wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/ray-worker-gpu-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-api-v2-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/ray-head-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-data-loader-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-api-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/Splunk_AI_Assistant_preview.tgz
```

### 컨테이너 이미지 Push

```bash
REGISTRY_IP=$172.31.51.179
REGISTRY="${REGISTRY_IP}:5000"

# /tmp의 tar 파일 로드 및 push
for TAR_FILE in /home/ec2-user/*.tar; do
  echo "=== 로드 중: ${TAR_FILE} ==="
  docker load -i ${TAR_FILE}
done

# 로드된 이미지 확인
docker images

# 이미지 태깅 및 push (실제 이미지명으로 수정)
ECR="658391232643.dkr.ecr.us-east-2.amazonaws.com/ml-platform"

declare -A IMAGE_MAP=(
  ["${ECR}/ray/ray-head:build-953"]="ray/ray-head:build-953"
  ["${ECR}/ray/ray-worker-gpu:build-953"]="ray/ray-worker-gpu:build-953"
  ["${ECR}/saia/saia-api:build-v2-main-c3b489d"]="saia/saia-api:build-v2-main-c3b489d"
  ["${ECR}/saia/saia-api-v2:build-v2-main-c3b489d"]="saia/saia-api-v2:build-v2-main-c3b489d"
  ["${ECR}/saia/saia-data-loader:build-v2-main-c3b489d"]="saia/saia-data-loader:build-v2-main-c3b489d"
)

for SRC in "${!IMAGE_MAP[@]}"; do
  DEST="${REGISTRY}/${IMAGE_MAP[$SRC]}"
  echo "=== Push: ${DEST} ==="
  docker tag "${SRC}" "${DEST}"
  docker push "${DEST}"
done

# Push 완료 확인
echo "=== Registry 이미지 목록 ==="
curl -s http://${REGISTRY}/v2/_catalog
```

### ★ 클러스터 노드 사전 설정 (가장 중요!)

**k0s 설치 전에** 모든 클러스터 노드에 Registry HTTP 허용 설정을 합니다:

```bash
SSH_KEY="/root/.ssh/yoonki-key.pem"
SSH_USER="ec2-user"
REGISTRY_IP=172.31.51.179

# 실제 노드 IP로 변경
CONTROLLER="172.31.43.176"
CPU_WORKER="172.31.39.105"
GPU_WORKER_1="172.31.31.249"
GPU_WORKER_2="172.31.23.195"
ALL_NODES="${CONTROLLER} ${CPU_WORKER} ${GPU_WORKER_1} ${GPU_WORKER_2}"

for NODE_IP in ${ALL_NODES}; do
  echo "========================================="
  echo "노드 사전 설정: ${NODE_IP}"
  echo "========================================="
  ssh -i ${SSH_KEY} ${SSH_USER}@${NODE_IP} "

    # 1. passwordless sudo 설정
    echo '${SSH_USER} ALL=(ALL) NOPASSWD:ALL' | \
      sudo tee /etc/sudoers.d/${SSH_USER}

    # 2. k0s 설정 디렉토리 생성
    sudo mkdir -p /etc/k0s
    sudo mkdir -p /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${REGISTRY_IP}:5000

    # 3. containerd.toml 설정 (k0s 전용)
    sudo tee /etc/k0s/containerd.toml > /dev/null << EOF
version = 2

[plugins]
  [plugins.\"io.containerd.grpc.v1.cri\"]
    [plugins.\"io.containerd.grpc.v1.cri\".registry]
      config_path = \"/var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts\"
EOF

    # 4. hosts.toml 설정 (HTTP 허용 핵심 설정)
    sudo tee /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${REGISTRY_IP}:5000/hosts.toml > /dev/null << EOF
server = \"http://${REGISTRY_IP}:5000\"

[host.\"http://${REGISTRY_IP}:5000\"]
  capabilities = [\"pull\", \"resolve\", \"push\"]
  skip_verify = true
EOF

    echo '--- containerd.toml 확인 ---'
    sudo cat /etc/k0s/containerd.toml

    echo '--- hosts.toml 확인 ---'
    sudo cat /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${REGISTRY_IP}:5000/hosts.toml

    echo '--- sudo 확인 ---'
    sudo whoami

    echo '완료: ${NODE_IP} ✅'
  "
  echo ""
done
```

---

## 사전작업 5: yaml 파일 작성

```bash
# 레포지토리 클론
git clone https://github.com/splunk/splunk-ai-operator.git
cd splunk-ai-operator/tools/cluster_setup
cp k0s-cluster-config.yaml my-cluster.yaml

# Registry IP 확인
REGISTRY_IP=172.31.51.179
echo "Registry IP: ${REGISTRY_IP}"
```

아래 내용으로 `my-cluster.yaml` 편집:

```yaml
cluster:
  name: ai-cluster
  region: ap-northeast-2
  sshKeyPath: /root/.ssh/yoonki-key.pem
  sshUser: ec2-user

nodes:
  controllers: 1
  cpuWorkers: 1
  gpuWorkers: 2
  existingIPs:
    controllers:
      - 172.31.43.176      # Controller IP
    workers:
      - 172.31.39.105      # CPU Worker IP
      - 172.31.31.249      # GPU Worker 1 IP
      - 172.31.23.195      # GPU Worker 2 IP

storage:
  storageClass: "local-path"
  vectorDbSize: "50Gi"
  minimumDiskSpace:
    controller: 90
    cpuWorker: 190
    gpuWorker: 480
  modelStaging:
    enabled: false
  objectStore:
    type: "minio"
    bucket: "ai-platform-bucket"
    endpoint: "http://172.31.51.179:9000"   # MinIO IP:9000
    auth:
      rootUser: "minioadmin"
      rootPassword: "MinioPassword123!"

images:
  registry: "172.31.51.179:5000"            # Registry IP:5000
  operator:
    image: "docker.io/splunk/splunk-ai-operator:0.2.0"
  splunk:
    image: "docker.io/splunk/splunk:10.2-rhel9"
    operatorImage: "docker.io/splunk/splunk-operator:3.0.0"
  ray:
    headImage: "172.31.51.179:5000/ray/ray-head:build-953"
    workerImage: "172.31.51.179:5000/ray/ray-worker-gpu:build-953"
  weaviate:
    image: "docker.io/semitechnologies/weaviate:stable-v1.28-007846a"
  saia:
    apiImage: "172.31.51.179:5000/saia/saia-api:build-v2-main-c3b489d"
    apiV2Image: "172.31.51.179:5000/saia/saia-api-v2:build-v2-main-c3b489d"
    dataLoaderImage: "172.31.51.179:5000/saia/saia-data-loader:build-v2-main-c3b489d"
  fluentBit:
    image: "docker.io/fluent/fluent-bit:1.9.6"
  otelCollector:
    image: "docker.io/otel/opentelemetry-collector-contrib:0.122.1"
  nginx:
    image: "docker.io/library/nginx:1.27-alpine"

operators:
  ray:
    version: "v1.2.2"
    modelVersion: "v0.3.14-36-g1549f5a"
    rayVersion: "2.53.0"
  certManager:
    installCRDs: true
  nvidia:
    devicePluginVersion: "v0.17.3"

kubernetes:
  namespace: ai-platform

files:
  splunkOperator: "./splunk-operator-cluster.yaml"
  aiPlatform: "./artifacts.yaml"

splunk:
  standaloneName: splunk-standalone

aiPlatform:
  name: "splunk-ai-stack"
  defaultAcceleratorType: "L40S"
  serviceTemplate:
    type: NodePort
    nodePort: 30080
  features:
    - name: "saia"
      version: "1.1.0"
  cpuScheduling:
    nodeSelector: {}
    tolerations: []
  gpuScheduling:
    nodeSelector: {}
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"

metallb:
  install: false

imagePullSecrets:
  autoCreateECR: false

ecr:
  account: ""
  region: us-east-2
```

---

## 설치 실행

```bash
cd splunk-ai-operator/tools/cluster_setup

# Step 1: 검증
CONFIG_FILE=./my_cluster.yaml \
  ./k0s_cluster_with_stack.sh validate

# Step 2: 설치 (검증 통과 후)
CONFIG_FILE=./my_cluster.yaml \
  ./k0s_cluster_with_stack.sh install
```

로그 모니터링 (다른 터미널):

```bash
tail -f splunk-ai-operator/tools/cluster_setup/logs/k0s-install-*.log
```

---

## 📋 최종 체크리스트

```
□ 모든 EC2 같은 Security Group에 포함
□ Admin 서버 도구 설치 완료
□ MinIO 실행 중 (포트 9000)
□ Docker Registry 실행 중 (포트 5000)
□ 모든 이미지 Registry에 Push 완료
□ 모든 클러스터 노드에 hosts.toml 설정 완료
□ 모든 클러스터 노드 passwordless sudo 설정
□ my-cluster.yaml 작성 완료
□ validate 통과
```

## K0S Cluster 상태 정보 확인 (Admin -> Controller)
```
ssh -i /root/.ssh/yoonki-key.pem ec2-user@172.31.43.176 "sudo k0s kubectl get nodes -A"
ssh -i /root/.ssh/yoonki-key.pem ec2-user@172.31.43.176 "sudo k0s kubectl get pods -A"
ssh -i /root/.ssh/yoonki-key.pem ec2-user@172.31.43.176 "sudo k0s kubectl get pods -n ai-platform -w"
```
