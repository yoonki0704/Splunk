## 전체 구성도

```
[Admin 서버 - RHEL]
  - kubectl, helm, git, jq, yq 설치
  - Docker + Local Registry (포트 5000)
  - MinIO (포트 9000)
  - AI 모델 파일 업로드
        │
        │ SSH
        ▼
[클러스터 노드들]
  - Controller  1대
  - CPU Worker  1대
  - GPU Worker  2대
```

---

## STEP 1: Admin 서버 (RHEL) 기본 도구 설치

```bash
# RHEL에서 실행

# 1. 기본 도구 설치
sudo yum install -y git jq curl wget

# 2. kubectl 설치
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

sudo yum install -y kubectl
kubectl version --client

# 3. helm 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
echo $PATH
export PATH=$PATH:/usr/local/bin
helm version

# 4. yq 설치
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
  -O /usr/local/bin/yq
chmod +x /usr/local/bin/yq
yq --version

# 5. AWS CLI 설치
sudo yum install -y awscli
aws --version

# 6. 전체 확인
echo "=== 설치 확인 ===" && \
kubectl version --client && \
helm version && \
git --version && \
jq --version && \
yq --version && \
aws --version
```

---

## STEP 2: Docker 설치 (RHEL)

```bash
# RHEL에서 Docker 설치
sudo yum install -y yum-utils
sudo yum-config-manager \
  --add-repo \
  https://download.docker.com/linux/rhel/docker-ce.repo

sudo yum install -y docker-ce docker-ce-cli containerd.io

# Docker 시작
sudo systemctl enable docker
sudo systemctl start docker

# 현재 유저 docker 그룹 추가
sudo usermod -aG docker $USER
newgrp docker

# 확인
docker --version
sudo systemctl status docker | grep Active
```

---

## STEP 3: MinIO 설치

```bash
# 데이터 저장 경로 생성
sudo mkdir -p /data/minio
sudo chown -R $USER:$USER /data/minio

# MinIO 바이너리 설치
sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio \
  -O /usr/local/bin/minio
sudo chmod +x /usr/local/bin/minio

# MinIO 클라이언트 설치
sudo wget https://dl.min.io/client/mc/release/linux-amd64/mc \
  -O /usr/local/bin/mc
sudo chmod +x /usr/local/bin/mc

# MinIO systemd 서비스 등록
sudo mkdir -p /etc/minio
sudo tee /etc/minio/minio.env > /dev/null << 'EOF'
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=MinioPassword123!
MINIO_VOLUMES=/data/minio
MINIO_OPTS=--address :9000 --console-address :9001
EOF

sudo tee /etc/systemd/system/minio.service > /dev/null << 'EOF'
[Unit]
Description=MinIO Object Storage
After=network-online.target

[Service]
User=root
EnvironmentFile=/etc/minio/minio.env
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES $MINIO_OPTS
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable minio
sudo systemctl start minio

# 확인
sleep 3
sudo systemctl status minio | grep Active
curl -s -o /dev/null -w "MinIO 상태: %{http_code}\n" \
  http://localhost:9000/minio/health/live
```

---

## STEP 4: MinIO 버킷 및 폴더 생성

```bash
# MinIO 연결 설정
mc alias set local http://localhost:9000 \
  minioadmin MinioPassword123!

# 버킷 생성
mc mb local/ai-platform-bucket

# 필수 폴더 구조 생성
mc mb local/ai-platform-bucket/model_artifacts
mc mb local/ai-platform-bucket/artifacts
mc mb local/ai-platform-bucket/conversations
mc mb local/ai-platform-bucket/config
mc mb local/ai-platform-bucket/storage_queue

# 확인
mc ls local/ai-platform-bucket
```

---

## STEP 5: AI 모델 MinIO에 업로드

> AI 모델 다운로드 정보 확인 🔍

README에 명시된 모델 목록(**HuggingFace**에서 다운로드):

| 모델 ID | 용도 |
|---------|------|
| `gemma-4-31b-it` | 기본 LLM (채팅, SPL 생성) |
| `gpt-oss-20b` | 보조 LLM |
| `all-minilm-l6-v2` | 문장 임베딩 / 시맨틱 검색 |
| `bi-encoder` | BGE 인코더 |
| `cross-encoder` | MS MARCO 크로스 인코더 |
| `e5-language-classifier` | 다국어 언어 감지 |
| `mbart-translator` | 다국어 번역 |
| `pii-classifier` | 개인정보 감지 |
| `uae-large` | 임베딩 모델 |
| `xlm-roberta-language-classifier` | 언어 분류기 |

**총 10개 모델, 120GB 이상**

> AI 모델을 다운로드하기 전에 yaml파일(my_cluster.yaml)에 object storage 경로 수정 필요
```bash
# AI 모델 자동 다운로드를 false로 설정
  modelStaging:
    enabled: false

  objectStore:
    type: "minio"                                # aws | s3compat | minio | seaweedfs (external only for non-aws)
    bucket: "ai-platform-bucket-minio-us-east-2"
    endpoint: "http://172.31.51.179:9000"              # CHANGE THIS: MinIO/SeaweedFS/S3 API endpoint
    auth:
      rootUser: "minioadmin"          # CHANGE THIS — AWS_ACCESS_KEY_ID (AKIA…) or MinIO root user
      rootPassword: "MinioPassword123!"  # CHANGE THIS — AWS secret OR MinIO root password; NEVER commit real keys
```


> AI 모델을 수동으로 다운로드하는 방법

```bash
CONFIG_FILE=./my-cluster.yaml \
  ./k0s_cluster_with_stack.sh stage-artifacts
```

> 모델 파일들이 다운로드 되는 경로

```bash
cd /home/ec2-user/splunk-ai-operator/tools/artifacts_download_upload_scripts/model_artifacts/
```

### 모델 파일이 폴더로 있는 경우

```bash
model_path=home/ec2-user/splunk-ai-operator/tools/artifacts_download_upload_scripts/model_artifacts

# 각 모델 폴더를 MinIO에 업로드
for model_dir in /$model_path/gemma-4-31b-it \
                 /$model_path/gpt-oss-20b \
                 /$model_path/all-minilm-l6-v2 \
                 /$model_path/bi-encoder \
                 /$model_path/cross-encoder \
                 /$model_path/e5-language-classifier \
                 /$model_path/mbart-translator \
                 /$model_path/pii-classifier \
                 /$model_path/uae-large \
                 /$model_path/xlm-roberta-language-classifier; do

  if [ -d "${model_dir}" ]; then
    model_name=$(basename ${model_dir})
    echo "=== 업로드 중: ${model_name} ==="
    mc cp --recursive ${model_dir}/ \
      local/ai-platform-bucket/model_artifacts/${model_name}/
    echo "완료: ${model_name} ✅"
  else
    echo "폴더 없음: ${model_dir}"
  fi
done

# 업로드 확인
mc ls local/ai-platform-bucket/model_artifacts/
```

### 모델 파일이 어디 있는지 모를 경우

```bash
# 모델 관련 파일 찾기
find /tmp -name "*.bin" -o \
         -name "*.safetensors" -o \
         -name "*.pt" \
         -o -name "config.json" 2>/dev/null | head -20

# 또는 설치 스크립트로 자동 다운로드 (modelStaging 활성화)
# yaml에서 modelStaging.enabled: true 로 변경하면 자동 다운로드
```

---

## STEP 6: Docker Registry 설치

```bash
# Registry 데이터 경로 생성
sudo mkdir -p /data/registry

# Admin 서버 Private IP 확인
ADMIN_IP=$172.31.51.179
echo "Admin IP: ${ADMIN_IP}"

# daemon.json 설정 (HTTP Registry 허용)
sudo tee /etc/docker/daemon.json > /dev/null << EOF
{
  "insecure-registries": ["${ADMIN_IP}:5000"]
}
EOF

# Docker 재시작
sudo systemctl restart docker

# Docker Registry 실행
docker run -d \
  --name docker-registry \
  --restart always \
  -p 5000:5000 \
  -v /data/registry:/var/lib/registry \
  registry:2

# 확인
sleep 3
curl -s http://${ADMIN_IP}:5000/v2/_catalog
echo "Registry 정상 ✅"
```

---

## STEP 7: 컨테이너 이미지 Registry에 Download 및 Push

### 컨테이너 이미지 Download
```bash
wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/ray-worker-gpu-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-api-v2-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/ray-head-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-data-loader-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-api-build-preview.tar

wget https://download.splunk.com/products/ai_tier/beta/0.2/linux/Splunk_AI_Assistant_preview.tgz
```

```bash
ADMIN_IP=$172.31.51.179
REGISTRY="${ADMIN_IP}:5000"
ECR="658391232643.dkr.ecr.us-east-2.amazonaws.com/ml-platform"

echo "=== /tmp의 tar 파일 로드 ==="
for TAR_FILE in /home/ec2-user/*.tar; do
  [ -f "${TAR_FILE}" ] || continue
  echo "로드 중: ${TAR_FILE}"
  docker load -i ${TAR_FILE}
done

echo "=== 로드된 이미지 목록 ==="
docker images

echo "=== 이미지 태깅 및 Push ==="
declare -A IMAGES=(
  ["${ECR}/ray/ray-head:build-953"]="ray/ray-head:build-953"
  ["${ECR}/ray/ray-worker-gpu:build-953"]="ray/ray-worker-gpu:build-953"
  ["${ECR}/saia/saia-api:build-v2-main-c3b489d"]="saia/saia-api:build-v2-main-c3b489d"
  ["${ECR}/saia/saia-api-v2:build-v2-main-c3b489d"]="saia/saia-api-v2:build-v2-main-c3b489d"
  ["${ECR}/saia/saia-data-loader:build-v2-main-c3b489d"]="saia/saia-data-loader:build-v2-main-c3b489d"
)

for SRC in "${!IMAGES[@]}"; do
  DEST="${REGISTRY}/${IMAGES[$SRC]}"
  echo "Push: ${DEST}"
  docker tag "${SRC}" "${DEST}" 2>/dev/null || \
    echo "태깅 실패: ${SRC}"
  docker push "${DEST}" && \
    echo "완료 ✅" || \
    echo "Push 실패 ❌"
done

echo "=== Registry 최종 확인 ==="
curl -s http://${REGISTRY}/v2/_catalog
```

---

## STEP 8: 클러스터 노드 사전 준비

```bash
SSH_KEY="/root/.ssh/yoonki-key.pem"
SSH_USER="ec2-user"
ADMIN_IP=$(curl -s \
  http://169.254.169.254/latest/meta-data/local-ipv4)

CONTROLLER="172.31.xx.xx"    # 실제 IP 입력
CPU_WORKER="172.31.xx.xx"    # 실제 IP 입력
GPU_WORKER_1="172.31.xx.xx"  # 실제 IP 입력
GPU_WORKER_2="172.31.xx.xx"  # 실제 IP 입력

ALL_NODES="${CONTROLLER} ${CPU_WORKER} ${GPU_WORKER_1} ${GPU_WORKER_2}"

echo "=== SSH 접속 테스트 ==="
for NODE_IP in ${ALL_NODES}; do
  ssh -i ${SSH_KEY} \
    -o StrictHostKeyChecking=no \
    -o ConnectTimeout=5 \
    ${SSH_USER}@${NODE_IP} \
    "echo '${NODE_IP}: OK'" 2>&1
done

echo ""
echo "=== 노드 사전 설정 적용 ==="
for NODE_IP in ${ALL_NODES}; do
  echo "--- 설정 중: ${NODE_IP} ---"
  ssh -i ${SSH_KEY} \
    -o StrictHostKeyChecking=no \
    ${SSH_USER}@${NODE_IP} "

    # passwordless sudo
    echo '${SSH_USER} ALL=(ALL) NOPASSWD:ALL' | \
      sudo tee /etc/sudoers.d/${SSH_USER}

    # k0s 설정 디렉토리 생성
    sudo mkdir -p /etc/k0s
    sudo mkdir -p /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${ADMIN_IP}:5000

    # containerd.toml 설정
    sudo tee /etc/k0s/containerd.toml > /dev/null << 'EOF'
version = 2

[plugins]
  [plugins.\"io.containerd.grpc.v1.cri\"]
    [plugins.\"io.containerd.grpc.v1.cri\".registry]
      config_path = \"/var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts\"
EOF

    # hosts.toml 설정
    sudo tee /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${ADMIN_IP}:5000/hosts.toml > /dev/null << 'EOF'
server = \"http://${ADMIN_IP}:5000\"

[host.\"http://${ADMIN_IP}:5000\"]
  capabilities = [\"pull\", \"resolve\", \"push\"]
  skip_verify = true
EOF

    # systemd drop-in 설정 (k0s 재시작 후에도 유지)
    sudo mkdir -p /etc/systemd/system/k0sworker.service.d
    sudo tee /etc/systemd/system/k0sworker.service.d/registry.conf \
      > /dev/null << 'EOF'
[Service]
ExecStartPost=/bin/bash -c 'sleep 10 && mkdir -p /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${ADMIN_IP}:5000 && tee /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${ADMIN_IP}:5000/hosts.toml > /dev/null << TOML\nserver = \"http://${ADMIN_IP}:5000\"\n\n[host.\"http://${ADMIN_IP}:5000\"]\n  capabilities = [\"pull\", \"resolve\", \"push\"]\n  skip_verify = true\nTOML'
EOF

    sudo systemctl daemon-reload
    echo '완료: ${NODE_IP} ✅'
  "
done
```

---

## STEP 9: my-cluster.yaml 작성

```bash
cd splunk-ai-operator/tools/cluster_setup
cp k0s-cluster-config.yaml my-cluster.yaml

ADMIN_IP=$(curl -s \
  http://169.254.169.254/latest/meta-data/local-ipv4)
echo "Admin IP: ${ADMIN_IP}"
```

**아래 내용으로 편집:**

```yaml
cluster:
  name: ai-cluster
  region: us-east-2
  sshKeyPath: /root/.ssh/yoonki-key.pem
  sshUser: ec2-user

nodes:
  controllers: 1
  cpuWorkers: 1
  gpuWorkers: 2
  existingIPs:
    controllers:
      - 172.31.xx.xx      # Controller IP
    workers:
      - 172.31.xx.xx      # CPU Worker IP
      - 172.31.xx.xx      # GPU Worker 1 IP
      - 172.31.xx.xx      # GPU Worker 2 IP

storage:
  storageClass: "local-path"
  vectorDbSize: "50Gi"
  minimumDiskSpace:
    controller: 90
    cpuWorker: 190
    gpuWorker: 480
  modelStaging:
    enabled: false        # 모델이 이미 MinIO에 있으면 false
                          # 모델이 없으면 true로 변경
  objectStore:
    type: "minio"
    bucket: "ai-platform-bucket"
    endpoint: "http://<ADMIN_IP>:9000"
    auth:
      rootUser: "minioadmin"
      rootPassword: "MinioPassword123!"

images:
  registry: "<ADMIN_IP>:5000"
  operator:
    image: "docker.io/splunk/splunk-ai-operator:0.2.0"
  splunk:
    image: "docker.io/splunk/splunk:10.2-rhel9"
    operatorImage: "docker.io/splunk/splunk-operator:3.0.0"
  ray:
    headImage: "<ADMIN_IP>:5000/ray/ray-head:build-953"
    workerImage: "<ADMIN_IP>:5000/ray/ray-worker-gpu:build-953"
  weaviate:
    image: "docker.io/semitechnologies/weaviate:stable-v1.28-007846a"
  saia:
    apiImage: "<ADMIN_IP>:5000/saia/saia-api:build-v2-main-c3b489d"
    apiV2Image: "<ADMIN_IP>:5000/saia/saia-api-v2:build-v2-main-c3b489d"
    dataLoaderImage: "<ADMIN_IP>:5000/saia/saia-data-loader:build-v2-main-c3b489d"
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

## STEP 10: 설치 실행

```bash
cd splunk-ai-operator/tools/cluster_setup

# 검증
CONFIG_FILE=./my-cluster.yaml \
  ./k0s_cluster_with_stack.sh validate

# 검증 통과 후 설치
CONFIG_FILE=./my-cluster.yaml \
  ./k0s_cluster_with_stack.sh install
```

---

## 📋 최종 체크리스트

```
□ STEP 1: 기본 도구 설치 완료
□ STEP 2: Docker 설치 완료
□ STEP 3: MinIO 설치 및 실행 중
□ STEP 4: MinIO 버킷 생성 완료
□ STEP 5: ★ AI 모델 MinIO 업로드 완료
□ STEP 6: Docker Registry 실행 중
□ STEP 7: 이미지 Push 완료
□ STEP 8: 클러스터 노드 사전 설정 완료
□ STEP 9: my-cluster.yaml 작성 완료
□ STEP 10: validate 통과 후 설치 실행
```

---

## ❓ 모델 파일 확인 먼저

시작 전에 알려주세요:

```bash
# /tmp에 있는 파일 목록
ls -lh /tmp/
```

**모델 파일이 어떤 형태로 있는지에 따라 STEP 5 방법이 달라집니다!** 😊
