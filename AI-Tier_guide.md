# Splunk AI Platform (AI Tier) — 초보자 완전 설치 가이드 🚀

첨부해 주신 파일들을 기반으로 처음부터 차근차근 설명할게요. 전체 흐름을 먼저 이해하고, 단계별로 따라하면 됩니다.

---

## 📌 전체 설치 흐름 한눈에 보기

```
[STEP 1] 내 환경 파악 및 EC2 인스턴스 준비
    ↓
[STEP 2] Admin 워크스테이션 도구 설치 (kubectl, helm 등)
    ↓
[STEP 3] GitHub에서 설치 스크립트 다운로드
    ↓
[STEP 4] my-cluster.yaml 설정 파일 편집 (★ 핵심)
    ↓
[STEP 5] AWS S3 및 ECR 준비
    ↓
[STEP 6] 설치 실행
    ↓
[STEP 7] 설치 검증
    ↓
[STEP 8] Splunk AI Assistant 앱 설치
```

---

## STEP 1: 내 환경 파악 및 EC2 인스턴스 준비

### 필요한 EC2 인스턴스 구성

DEPLOYMENT_GUIDE.md 기준으로 **최소 4대의 서버**가 필요합니다.

| 역할 | 수량 | 권장 인스턴스 | CPU | RAM | 디스크 |
|------|------|-------------|-----|-----|--------|
| Controller | 1대 | `m5.xlarge` | 4코어 | 8GB | 100GB |
| CPU Worker | 1대+ | `m5.2xlarge` | 8코어 | 32GB | 200GB |
| GPU Worker | **2대 필수** | `g6e.12xlarge` | 48코어 | 384GB | 500GB |
| Admin 워크스테이션 | 1대 | `t3.medium` | - | - | **250GB** (모델 다운로드용) |

> ⚠️ **중요:** GPU Worker는 **반드시 2대** 필요합니다. 1대로는 AI 추론 스택이 동작하지 않아요.

> ⚠️ **OS 주의:** 클러스터 노드(Controller, Worker)는 **RHEL 9** 사용을 권장합니다. Admin 워크스테이션은 Ubuntu 사용 가능합니다.

### EC2 보안 그룹 설정

모든 클러스터 노드들이 같은 Security Group에 있어야 하고, 아래 포트를 허용해야 합니다:

```
# 같은 Security Group 내 모든 트래픽 허용 (가장 간단한 방법)
- Inbound: All traffic, Source: [자기 Security Group ID]

# 외부 접속용 (Admin 워크스테이션에서 SSH)
- Port 22 (TCP): Admin 워크스테이션 IP
- Port 30080 (TCP): SAIA 외부 접속용 (NodePort)
- Port 8000 (TCP): Splunk Web 접속용
```

### SSH 키 설정

```bash
# Admin 워크스테이션에서 SSH 키 생성
ssh-keygen -t rsa -b 4096 -f ~/.ssh/splunk-ai-key -N ""

# 각 클러스터 노드에 SSH 키 배포 (EC2는 보통 이미 설정되어 있음)
# EC2 키페어를 사용한다면 해당 .pem 파일 경로를 나중에 yaml에 입력

# 키 권한 설정
chmod 600 ~/.ssh/splunk-ai-key

# 접속 테스트
ssh -i ~/.ssh/splunk-ai-key ec2-user@<controller-IP>
```

### passwordless sudo 설정 (모든 클러스터 노드에서)

```bash
# 각 노드에 SSH 접속 후 실행
ssh -i ~/.ssh/splunk-ai-key ec2-user@<각-노드-IP>

# sudoers 파일 편집
sudo visudo

# 아래 줄 추가 (ec2-user 자리에 실제 사용자명 입력)
ec2-user ALL=(ALL) NOPASSWD:ALL
```

---

## STEP 2: Admin 워크스테이션 도구 설치

이전 대화에서 진행한 내용입니다. 아직 안 되셨다면:

```bash
# 1. 기본 도구
sudo apt-get update
sudo apt-get install -y git jq curl

# 2. kubectl (바이너리 직접 설치)
KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt)
curl -LO "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# 3. helm (공식 스크립트)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 4. yq
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
  -O /usr/local/bin/yq && chmod +x /usr/local/bin/yq

# 5. AWS CLI (ECR 사용을 위해 필요)
sudo apt-get install -y awscli

# 설치 확인
kubectl version --client && helm version && git --version && jq --version && yq --version
```

---

## STEP 3: GitHub에서 설치 스크립트 다운로드

```bash
# Splunk AI Operator 레포지토리 클론
git clone https://github.com/splunk/splunk-ai-operator.git
cd splunk-ai-operator/tools/cluster_setup

# 파일 목록 확인
ls -la
# 주요 파일:
# k0s_cluster_with_stack.sh  ← 메인 설치 스크립트
# k0s-cluster-config.yaml    ← 설정 파일 템플릿
# artifacts.yaml              ← AI Platform 매니페스트
# splunk-operator-cluster.yaml ← Splunk Operator 매니페스트

# 내 설정 파일 복사
cp k0s-cluster-config.yaml my-cluster.yaml
```

---

## STEP 4: my-cluster.yaml 설정 파일 편집 ⭐ 핵심

이게 가장 중요한 단계입니다. 공유해 주신 `k0s-cluster-config.yaml`을 기반으로 **CHANGE THIS** 항목들을 하나씩 설명할게요.

```bash
vi my-cluster.yaml
```

### 📝 섹션별 상세 편집 가이드

---

#### 섹션 1: cluster (클러스터 기본 설정)

```yaml
cluster:
  name: ai-cluster          # ✏️ 원하는 클러스터 이름으로 변경 (영문+하이픈만 사용)
                            # 예: my-ai-cluster, splunk-ai-prod

  region: us-east-2         # ✏️ 내 AWS 리전으로 변경
                            # 서울: ap-northeast-2
                            # 버지니아: us-east-1

  sshKeyPath: ~/.ssh/splunk-ai-key   # ✏️ 내 SSH 키 경로로 변경
                                      # EC2 키페어 사용 시: ~/.ssh/my-keypair.pem

  sshUser: ec2-user         # ✏️ SSH 접속 유저명
                            # RHEL/Amazon Linux: ec2-user
                            # Ubuntu: ubuntu
                            # root 사용 시: root
```

---

#### 섹션 2: nodes (노드 IP 주소 설정)

> 이 부분이 제일 중요합니다! EC2 인스턴스의 **Private IP**를 입력해야 합니다.

```bash
# 각 EC2의 Private IP 확인 방법
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,Tags[?Key==`Name`].Value]' --output table
```

```yaml
nodes:
  controllers: 1
  cpuWorkers: 1             # CPU Worker 수
  gpuWorkers: 2             # GPU Worker 수 (반드시 2)

  existingIPs:
    controllers:
      - 10.0.0.1            # ✏️ Controller EC2의 실제 Private IP
                            # 예: 172.31.10.100

    workers:
      - 10.0.0.2            # ✏️ CPU Worker의 Private IP (첫 번째 = CPU)
                            # 예: 172.31.10.101

      - 10.0.0.3            # ✏️ GPU Worker 1의 Private IP
                            # 예: 172.31.10.102

      - 10.0.0.4            # ✏️ GPU Worker 2의 Private IP
                            # 예: 172.31.10.103
```

> 💡 **순서 중요:** workers 목록에서 위에서부터 `cpuWorkers` 수만큼이 CPU Worker, 나머지가 GPU Worker로 자동 분류됩니다.

---

#### 섹션 3: storage (저장소 설정)

AWS S3를 사용하는 경우:

```bash
# 먼저 S3 버킷 생성
aws s3 mb s3://my-splunk-ai-bucket --region ap-northeast-2

# IAM 사용자의 Access Key 확인
aws sts get-caller-identity
```

```yaml
storage:
  storageClass: "local-path"    # 그대로 유지
  vectorDbSize: "50Gi"          # 그대로 유지 (필요시 늘림)

  modelStaging:
    enabled: false              # ✏️ 처음엔 false로 설정
                                # 모델이 아직 없으면 true로 바꾸면 자동 다운로드
                                # (단, 120GB+ 다운로드 시간 필요)

  objectStore:
    type: "aws"                 # ✏️ AWS S3 사용 시 "aws"로 변경
                                # MinIO 사용 시: "minio"

    bucket: "my-splunk-ai-bucket"  # ✏️ 위에서 만든 S3 버킷 이름

    endpoint: ""                # ✏️ AWS S3는 비워둠 (type이 "aws"일 때)
                                # MinIO 사용 시: "http://10.0.0.5:9000"

    auth:
      rootUser: "AKIAXXXXXXXXXXXXXXXX"      # ✏️ AWS Access Key ID
      rootPassword: "xxxxxxxxxxxxxxxxxxxxxxxx"  # ✏️ AWS Secret Access Key
```

> ⚠️ **보안 주의:** Access Key는 절대 Git에 올리지 마세요!

---

#### 섹션 4: images (컨테이너 이미지 설정)

ECR(Elastic Container Registry)에서 이미지를 가져오는 경우:

```bash
# 내 AWS 계정 ID 확인
aws sts get-caller-identity --query Account --output text
# 예: 123456789012

# ECR 레지스트리 주소 형식
# [계정ID].dkr.ecr.[리전].amazonaws.com
# 예: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com
```

```yaml
images:
  registry: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com"
  # ✏️ 위 형식으로 내 ECR 주소 입력

  operator:
    image: "docker.io/splunk/splunk-ai-operator:0.2.0"
    # ✏️ ECR에 이미지를 push한 경우:
    # "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/splunk-ai-operator:0.2.0"

  splunk:
    image: "docker.io/splunk/splunk:10.2-rhel9"
    operatorImage: "docker.io/splunk/splunk-operator:3.0.0"

  ray:
    headImage: "splunk/ray-head-build-preview:latest"
    workerImage: "splunk/ray-worker-gpu-build-preview:latest"
    # ✏️ ECR에 있는 이미지 경로로 변경 필요

  weaviate:
    image: "docker.io/semitechnologies/weaviate:stable-v1.28-007846a"
    # Docker Hub에서 직접 pull 가능하면 그대로 유지

  saia:
    apiImage: "splunk/saia-api-build-preview:latest"
    apiV2Image: "splunk/saia-api-v2-build-preview:latest"
    dataLoaderImage: "splunk/saia-data-loader-build-preview:latest"
    # ✏️ ECR에 있는 이미지 경로로 변경 필요
```

---

#### 섹션 5: imagePullSecrets & ecr (ECR 인증 설정)

```yaml
imagePullSecrets:
  secrets:
    - ecr-registry-secret     # 그대로 유지
  autoCreateECR: true         # ✏️ ECR 사용 시 true (자동으로 ECR 로그인 처리)

ecr:
  account: "123456789012"     # ✏️ 내 AWS 계정 ID
  region: ap-northeast-2      # ✏️ 내 AWS 리전
```

---

#### 섹션 6: aiPlatform (AI 플랫폼 설정)

```yaml
aiPlatform:
  name: "splunk-ai-stack"           # 원하는 이름으로 변경 가능
  defaultAcceleratorType: "L40S"    # 그대로 유지 (필수값)

  serviceTemplate:
    type: NodePort                  # AWS EC2에서는 NodePort 사용
    nodePort: 30080                 # SAIA 접속 포트 (보안그룹에서 열어야 함)

  features:
    - name: "saia"
      version: "1.1.0"              # 그대로 유지
```

---

### ✅ 최종 편집된 my-cluster.yaml 전체 예시

아래는 AWS 환경 기준 실제 사용 예시입니다 (값은 예시이므로 본인 환경에 맞게 변경):

```yaml
cluster:
  name: my-ai-cluster
  region: ap-northeast-2
  sshKeyPath: ~/.ssh/splunk-ai-key
  sshUser: ec2-user

nodes:
  controllers: 1
  cpuWorkers: 1
  gpuWorkers: 2
  existingIPs:
    controllers:
      - 172.31.10.100        # ← 내 Controller EC2 Private IP
    workers:
      - 172.31.10.101        # ← CPU Worker Private IP
      - 172.31.10.102        # ← GPU Worker 1 Private IP
      - 172.31.10.103        # ← GPU Worker 2 Private IP

storage:
  storageClass: "local-path"
  vectorDbSize: "50Gi"
  modelStaging:
    enabled: false           # 나중에 true로 변경하여 모델 다운로드
  objectStore:
    type: "aws"
    bucket: "my-splunk-ai-bucket"
    endpoint: ""
    auth:
      rootUser: "AKIAXXXXXXXXXXXXXXXX"
      rootPassword: "xxxxxxxxxxxxxxxxxxxxxxxx"

images:
  registry: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com"
  operator:
    image: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/splunk-ai-operator:0.2.0"
  splunk:
    image: "docker.io/splunk/splunk:10.2-rhel9"
    operatorImage: "docker.io/splunk/splunk-operator:3.0.0"
  ray:
    headImage: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/ray-head-build-preview:latest"
    workerImage: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/ray-worker-gpu-build-preview:latest"
  weaviate:
    image: "docker.io/semitechnologies/weaviate:stable-v1.28-007846a"
  saia:
    apiImage: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/saia-api-build-preview:latest"
    apiV2Image: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/saia-api-v2-build-preview:latest"
    dataLoaderImage: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/saia-data-loader-build-preview:latest"
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
  workerGroupConfig:
    imageRegistry: ""
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
  secrets:
    - ecr-registry-secret
  autoCreateECR: true

ecr:
  account: "123456789012"
  region: ap-northeast-2
```

---

## STEP 5: AWS S3 및 ECR 준비

### S3 버킷 생성

```bash
# 버킷 생성
aws s3 mb s3://my-splunk-ai-bucket --region ap-northeast-2

# 버킷 접근 확인
aws s3 ls s3://my-splunk-ai-bucket
```

### ECR 이미지 저장소 준비

Splunk로부터 제공받은 컨테이너 이미지들을 ECR에 올려야 합니다:

```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 | \
  docker login --username AWS \
  --password-stdin 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com

# ECR 저장소 생성 (각 이미지마다)
aws ecr create-repository --repository-name splunk-ai-operator --region ap-northeast-2
aws ecr create-repository --repository-name ray-head-build-preview --region ap-northeast-2
aws ecr create-repository --repository-name ray-worker-gpu-build-preview --region ap-northeast-2
aws ecr create-repository --repository-name saia-api-build-preview --region ap-northeast-2
aws ecr create-repository --repository-name saia-api-v2-build-preview --region ap-northeast-2
aws ecr create-repository --repository-name saia-data-loader-build-preview --region ap-northeast-2

# 이미지 태깅 및 push (Splunk에서 받은 이미지를 ECR에 업로드)
docker tag splunk/splunk-ai-operator:0.2.0 \
  123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/splunk-ai-operator:0.2.0
docker push 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/splunk-ai-operator:0.2.0
# ... 나머지 이미지들도 동일하게 반복
```

---

## STEP 6: 설치 실행

### 설치 전 설정 검증

```bash
cd splunk-ai-operator/tools/cluster_setup

# 설정 파일 검증 (실제 변경 없이 체크만)
CONFIG_FILE=./my-cluster.yaml ./k0s_cluster_with_stack.sh validate
```

모든 항목에 ✅ 표시가 나오면 진행합니다.

### 설치 실행

```bash
# 설치 시작
CONFIG_FILE=./my-cluster.yaml ./k0s_cluster_with_stack.sh install
```

> ⏱️ **예상 소요 시간:**
> - 모델 스테이징 없이 (`modelStaging.enabled: false`): 약 30~60분
> - 모델 스테이징 포함 (`modelStaging.enabled: true`): 약 3~7시간

### 설치 로그 모니터링 (다른 터미널에서)

```bash
tail -f splunk-ai-operator/tools/cluster_setup/logs/k0s-install-*.log
```

---

## STEP 7: 설치 검증

```bash
# kubeconfig 설정
export KUBECONFIG=~/.kube/k0s-my-ai-cluster

# 노드 상태 확인 (모두 Ready여야 함)
kubectl get nodes

# 전체 Pod 상태 확인 (모두 Running 또는 Completed여야 함)
kubectl get pods --all-namespaces

# AI Platform 상태 확인 (Ready여야 함)
kubectl get aiplatform -n ai-platform

# GPU 인식 확인
kubectl get nodes -l splunk.ai/workload-type=gpu -o yaml | grep nvidia.com/gpu
```

**정상 상태 예시:**
```
NAME              STATUS   ROLES    AGE
controller        Ready    master   15m
cpu-worker-1      Ready    <none>   12m
gpu-worker-1      Ready    <none>   12m
gpu-worker-2      Ready    <none>   12m
```

---

## STEP 8: Splunk AI Assistant 앱 설치

```bash
# Splunk 접속 비밀번호 확인
kubectl get secret splunk-standalone-secret -n ai-platform \
  -o jsonpath='{.data.password}' | base64 --decode && echo

# Splunk Web 포트 포워딩 (로컬 접속 시)
kubectl port-forward -n ai-platform svc/splunk-standalone-service 8000:8000
```

브라우저에서 `http://localhost:8000` 접속 → admin으로 로그인 → **앱 관리** → **파일에서 앱 설치** → `Splunk_AI_Assistant_Cloud.tgz` 업로드

---

## ❓ 지금 당장 필요한 정보 확인 리스트

시작하기 전에 아래 정보를 먼저 수집해 두세요:

```
□ Controller EC2 Private IP: _______________
□ CPU Worker EC2 Private IP: _______________
□ GPU Worker 1 EC2 Private IP: _______________
□ GPU Worker 2 EC2 Private IP: _______________
□ SSH 키 파일 경로: _______________
□ SSH 유저명 (ec2-user/ubuntu): _______________
□ AWS 계정 ID: _______________
□ AWS 리전: _______________
□ S3 버킷 이름: _______________
□ AWS Access Key ID: _______________
□ AWS Secret Access Key: _______________
□ ECR 레지스트리 주소: _______________
```

---

## 🚦 어디서부터 시작할까요?

현재 어느 단계에 계신지 알려주시면 그 단계부터 바로 도와드릴게요!

예를 들어:
- "EC2 인스턴스는 만들었는데 IP를 어떻게 확인하는지 모르겠어요"
- "ECR에 이미지를 올리는 방법을 모르겠어요"
- "yaml 파일을 편집했는데 맞는지 확인해 주세요"

질문이 있으시면 언제든지 물어보세요! 😊
