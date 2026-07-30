# Air-Gapped 환경 완전 준비 가이드 🔒

## 전체 준비 흐름

```
[1단계] 인터넷 연결된 "빌드 서버" 준비 (임시로 1대 필요)
    ↓
[2단계] Admin 워크스테이션 도구 패키지 만들기
    ↓
[3단계] k0s Air-gap 번들 생성 (prepare_airgap_bundle.sh)
    ↓
[4단계] 컨테이너 이미지 미러링 (Splunk AI Platform 이미지들)
    ↓
[5단계] AI 모델 파일 스테이징
    ↓
[6단계] 모든 파일을 물리적으로 고객사 환경에 전달
    ↓
[7단계] Air-gapped 환경에서 설치 실행
```

---

## 왜 "빌드 서버"가 하나 필요한가?

```
고객사 환경 = 완전 Air-gapped (인터넷 불가)
         ↓
다운로드는 인터넷 되는 곳에서 미리 해야 함
         ↓
빌드 서버 = 인터넷 되는 임시 EC2/VM 1대
  (설치 끝나면 삭제 가능, 고객사 네트워크와 무관)
```

---

## STEP 1: 빌드 서버 준비

인터넷이 되는 EC2 인스턴스(또는 어떤 서버든) 1대를 준비합니다.

```bash
# 빌드 서버 사양 권장
- OS: RHEL 9 (동일한 OS로 맞추는 것이 안전)
- 디스크: 500GB 이상 (모델 파일 120GB+ 포함)
- 인터넷: 반드시 필요
```

---

## STEP 2: Admin 워크스테이션 도구 패키지 만들기

Admin 서버(고객사에서 설치 스크립트를 실행할 서버) 자체도 인터넷이 안 되므로, 필요한 도구들을 미리 다운로드해서 패키지로 만듭니다.

### 빌드 서버에서 실행

```bash
mkdir -p /mnt/transfer/admin-tools
cd /mnt/transfer/admin-tools

echo "=== 1. kubectl 다운로드 ===" && \
KUBE_VERSION=$(curl -L -s https://dl.k8s.io/release/stable.txt) && \
curl -LO "https://dl.k8s.io/release/${KUBE_VERSION}/bin/linux/amd64/kubectl" && \
chmod +x kubectl

echo "=== 2. helm 다운로드 ===" && \
curl -fsSL -o helm.tar.gz \
  https://get.helm.sh/helm-v3.15.0-linux-amd64.tar.gz && \
tar -xzf helm.tar.gz && \
cp linux-amd64/helm . && \
rm -rf linux-amd64 helm.tar.gz

echo "=== 3. yq 다운로드 ===" && \
curl -fsSL -o yq \
  https://github.com/mikefarah/yq/releases/download/v4.44.1/yq_linux_amd64 && \
chmod +x yq

echo "=== 4. jq 다운로드 (RPM) ===" && \
mkdir -p rpms && \
sudo dnf download --destdir=rpms jq 2>/dev/null || \
sudo dnf install --downloadonly --downloaddir=rpms jq -y

echo "=== 5. git 다운로드 (RPM) ===" && \
sudo dnf install --downloadonly --downloaddir=rpms git -y

echo "=== 6. mc (MinIO client) 다운로드 ===" && \
curl -fsSL -o mc \
  https://dl.min.io/client/mc/release/linux-amd64/mc && \
chmod +x mc

echo "=== 7. sqlite3 (kine WAL 문제 대비) ===" && \
sudo dnf install --downloadonly --downloaddir=rpms sqlite -y

echo "=== 8. Docker/Registry 관련 (로컬 레지스트리용) ===" && \
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo \
sudo dnf makecache \
dnf repolist | grep docker \
cd /mnt/transfer/admin-tools \
sudo dnf install --downloadonly --downloaddir=rpms \
  docker-ce docker-ce-cli containerd.io docker-compose-plugin -y \
echo "docker repo 추가 필요 시 별도 진행"

# 압축
cd /mnt/transfer
tar -czf admin-tools-bundle.tar.gz admin-tools/
echo "Admin 도구 패키지 생성 완료: admin-tools-bundle.tar.gz"
ls -lh admin-tools-bundle.tar.gz
```

---

## STEP 3: k0s Air-gap 번들 생성 (핵심!)

이게 **가장 중요한 부분**입니다. 이전에 수동으로 했던 containerd 설정, GPU 드라이버 설치 등이 **자동화**되어 있습니다.

```bash
# 빌드 서버에서 실행
git clone https://github.com/splunk/splunk-ai-operator.git
cd splunk-ai-operator/tools/cluster_setup

# k0s 버전 확인 (문서 기준 v1.31+ 권장, 이전 사용 버전과 맞추려면 확인)
./prepare_airgap_bundle.sh \
  --output-dir /mnt/transfer \
  --k0s-version v1.31.2+k0s.0
```

**번들에 포함되는 내용 (문서 기준):**

```
binaries/    → k0s, yq
images/      → k0s-images.tar (Calico, CoreDNS 등)
             → addon-images.tar (cert-manager, prometheus, metallb, nvidia-device-plugin 등)
manifests/   → cert-manager, local-path-provisioner, nvidia-device-plugin
charts/      → kube-prometheus-stack, opentelemetry-operator, kuberay-operator, metallb
packages/    → EPEL, CUDA repo, nvidia-container-toolkit repo, PyYAML wheel
```

> ✅ **이 번들 덕분에 예전에 겪었던 "ImagePullBackOff (Calico, CoreDNS 등)" 문제가 원천적으로 해결됩니다.** 클러스터 자체 구성요소 이미지가 미리 포함되어 있어요.

```bash
# 생성 확인
ls -lh /mnt/transfer/airgap-bundle-*.tar.gz
```

---

## STEP 4: 컨테이너 이미지 미러링 (Splunk AI Platform 애플리케이션 이미지)

이전에 겪었던 **local Docker Registry의 containerd insecure 문제**를 완전히 새로운 방식으로 해결합니다.

### 4-1: 이미지 목록 확인

```bash
# ── Splunk ──────────────────────────────────────────────────────────────────
splunk/splunk:10.2.0
docker.io/splunk/splunk-operator:3.0.0

# ── Ray ─────────────────────────────────────────────────────────────────────
# Set images.ray.headImage and images.ray.workerImage in config to your mirrors
# (these are built internally — not on a public registry)

# ── Weaviate ─────────────────────────────────────────────────────────────────
docker.io/semitechnologies/weaviate:stable-v1.28-007846a

# ── KubeRay Operator ─────────────────────────────────────────────────────────
quay.io/kuberay/operator:v1.2.2

# ── OpenTelemetry ─────────────────────────────────────────────────────────────
docker.io/otel/opentelemetry-collector-contrib:0.122.1

# ── Fluent Bit ────────────────────────────────────────────────────────────────
docker.io/fluent/fluent-bit:1.9.6

# ── Nginx ────────────────────────────────────────────────────────────────────
docker.io/library/nginx:1.27-alpine

# ── MetalLB (installed by Helm chart — the chart handles its own images) ─────
# The metallb Helm chart pulls its own images; check the chart for the current
# image tags after running: helm show values metallb/metallb --version 0.14.8

# ── cert-manager (installed from manifest) ───────────────────────────────────
# cert-manager manifest embeds image references; check the manifest for exact tags:
# grep 'image:' manifests/cert-manager.yaml

# ── Prometheus stack (Helm chart — many images) ──────────────────────────────
# Run after pulling the chart: helm show values charts/kube-prometheus-stack-*.tgz
# to get the full image list.

# ── NOTE: Model weights (HuggingFace) ────────────────────────────────────────
# Model weights (~60 GB total) are NOT container images.
# Use tools/artifacts_download_upload_scripts/ to stage them separately.
# Models: gemma-4-31b-it, gpt-oss-20b, all-minilm-l6-v2, bi-encoder,
#         cross-encoder, e5-language-classifier, mbart-translator,
#         pii-classifier, uae-large, xlm-roberta-language-classifier
```

### 4-2: 이미지 다운로드 (빌드 서버에서, 이전에 준 tar 파일 포함)

```bash
# VOC 포탈에서 이미지 다운로드
https://download.splunk.com/products/ai_tier/beta/0.2/linux/ray-worker-gpu-build-preview.tar
https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-api-v2-build-preview.tar
https://download.splunk.com/products/ai_tier/beta/0.2/linux/ray-head-build-preview.tar
https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-data-loader-build-preview.tar
https://download.splunk.com/products/ai_tier/beta/0.2/linux/saia-api-build-preview.tar
https://download.splunk.com/products/ai_tier/beta/0.2/linux/Splunk_AI_Assistant_preview.tgz

# 공개 이미지들도 함께 저장 (Docker Hub 등 - 인터넷에서 미리 pull)
docker pull docker.io/splunk/splunk-ai-operator:0.2.0
docker pull docker.io/splunk/splunk:10.2-rhel9
docker pull docker.io/splunk/splunk-operator:3.0.0
docker pull docker.io/semitechnologies/weaviate:stable-v1.28-007846a
docker pull docker.io/fluent/fluent-bit:1.9.6
docker pull docker.io/otel/opentelemetry-collector-contrib:0.122.1
docker pull docker.io/library/nginx:1.27-alpine

docker save -o public-images-bundle.tar \
  docker.io/splunk/splunk-ai-operator:0.2.0 \
  docker.io/splunk/splunk:10.2-rhel9 \
  docker.io/splunk/splunk-operator:3.0.0 \
  docker.io/semitechnologies/weaviate:stable-v1.28-007846a \
  docker.io/fluent/fluent-bit:1.9.6 \
  docker.io/otel/opentelemetry-collector-contrib:0.122.1 \
  docker.io/library/nginx:1.27-alpine

ls -lh *.tar
```

---

## STEP 5: AI 모델 파일 스테이징

```bash
cd splunk-ai-operator/tools/artifacts_download_upload_scripts

# 모델 다운로드 (GPU 타입에 맞게)
./download_from_huggingface.sh --accelerator l40s

# 다운로드된 모델 파일 확인
du -sh ./model_artifacts/ 2>/dev/null || \
find . -name "*.safetensors" -o -name "*.bin" | \
  xargs du -ch 2>/dev/null | tail -1

# 압축 (전달용)
tar -czf /mnt/transfer/model-artifacts-bundle.tar.gz ./model_artifacts/
```

> 📌 **참고:** 모델은 나중에 고객사 환경의 MinIO에 직접 업로드해야 하므로, MinIO 서버가 준비되면 그때 업로드해도 됩니다. (지금은 파일만 확보)

---

## STEP 6: 전체 패키지 구성 확인

```bash
mkdir -p /mnt/transfer/FINAL_PACKAGE
cd /mnt/transfer

mv admin-tools-bundle.tar.gz FINAL_PACKAGE/
mv airgap-bundle-*.tar.gz FINAL_PACKAGE/
mv app-images/app-images-bundle.tar FINAL_PACKAGE/
mv app-images/public-images-bundle.tar FINAL_PACKAGE/
mv model-artifacts-bundle.tar.gz FINAL_PACKAGE/

# 설치 스크립트 및 설정 파일도 포함
cp -r splunk-ai-operator/tools/cluster_setup/*.sh FINAL_PACKAGE/
cp -r splunk-ai-operator/tools/cluster_setup/*.yaml FINAL_PACKAGE/
cp -r splunk-ai-operator/tools/artifacts_download_upload_scripts FINAL_PACKAGE/

echo "=== 최종 패키지 구성 ===" && \
ls -lh FINAL_PACKAGE/

echo "" && echo "=== 전체 크기 ===" && \
du -sh FINAL_PACKAGE/
```

**예상 최종 패키지 구성:**

```
FINAL_PACKAGE/
├── admin-tools-bundle.tar.gz       (kubectl, helm, yq, jq, git, mc)
├── airgap-bundle-<날짜>.tar.gz      (k0s 자체 + add-on 이미지)
├── app-images-bundle.tar           (ray, saia 이미지)
├── public-images-bundle.tar        (splunk, weaviate, nginx 등)
├── model-artifacts-bundle.tar.gz   (AI 모델 파일 120GB+)
├── k0s_cluster_with_stack.sh
├── install_from_airgap_bundle.sh
├── prepare_airgap_bundle.sh
├── k0s-cluster-config.yaml
└── artifacts_download_upload_scripts/
```

---

## STEP 7: 고객사 Air-gapped 환경으로 전달

USB, 물리 매체, 또는 사내 SCP 등으로 전체 `FINAL_PACKAGE` 디렉토리를 전달합니다.

```bash
# 최종 tar로 묶기 (선택사항 - 전달 편의)
tar -czf splunk-ai-airgap-complete.tar.gz FINAL_PACKAGE/
ls -lh splunk-ai-airgap-complete.tar.gz
```

---

## STEP 8: Air-gapped 환경(Admin 서버)에서 설치 준비

### 8-1: Admin 도구 설치

```bash
# 패키지 압축 해제
tar -xzf splunk-ai-airgap-complete.tar.gz
cd FINAL_PACKAGE

tar -xzf admin-tools-bundle.tar.gz
cd admin-tools

# 도구 설치
sudo cp kubectl helm yq mc /usr/local/bin/
sudo chmod +x /usr/local/bin/kubectl /usr/local/bin/helm \
  /usr/local/bin/yq /usr/local/bin/mc

# RPM 도구 설치 (jq, git, sqlite, docker)
sudo dnf install -y ./rpms/*.rpm 2>/dev/null || \
sudo rpm -ivh --nodeps ./rpms/*.rpm

# 확인
kubectl version --client
helm version
yq --version
jq --version
git --version
mc --version
```

### 8-2: Docker Registry 및 MinIO 설치 (이전과 동일하게 준비)

```bash
sudo systemctl enable docker
sudo systemctl start docker

# 로컬 Registry 실행
mkdir -p /data/registry
docker run -d \
  --name docker-registry \
  --restart always \
  -p 5000:5000 \
  -v /data/registry:/var/lib/registry \
  registry:2
```

### 8-3: 이미지 로드 및 Push

```bash
cd ../

# 이미지 로드
docker load -i app-images-bundle.tar
docker load -i public-images-bundle.tar

# 확인
docker images

# 로컬 Registry에 push (필요한 이미지들)
REGISTRY_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
docker tag <ray-head-image> ${REGISTRY_IP}:5000/ray/ray-head:build-953
docker push ${REGISTRY_IP}:5000/ray/ray-head:build-953
# ... 나머지 이미지도 동일하게
```

---

## STEP 9: my-cluster.yaml — registryInsecure 사용 (★핵심 개선점)

이전에 겪었던 모든 containerd 문제를 **`registryInsecure: true` 설정 하나로 해결**할 수 있습니다!

```yaml
images:
  registry: "172.31.51.179:5000"
  registryInsecure: true    # ★ 이 옵션 추가! 자동으로 v1/v2 감지 후 설정

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
```

```yaml
cluster:
  name: ai-cluster
  airgap: true    # ★ Air-gap 모드 활성화
  sshKeyPath: /root/.ssh/yoonki-key.pem
  sshUser: ec2-user
```

> ✅ **`registryInsecure: true`를 설정하면 스크립트가 자동으로:**
> - containerd v1/v2 버전 감지
> - v2인 경우 `/etc/k0s/containerd/certs.d/<registry>/hosts.toml` 자동 생성
> - k0s install 후, start 전에 자동 적용
> - Worker/Controller 모두에 일관되게 적용
>
> **이전에 수십 번 겪었던 nginx/containerd 수동 수정이 필요 없습니다!**

---

## STEP 10: MinIO 설치 및 모델 업로드

```bash
# 이전 STEP 3 가이드와 동일하게 MinIO 설치
sudo mkdir -p /data/minio
sudo wget https://... # 인터넷 안되므로 admin-tools에 포함 필요

# 미리 준비 필요! (빌드 서버에서 MinIO 바이너리도 다운로드해서 패키지에 포함)
```

> ⚠️ **놓친 부분:** MinIO 바이너리도 빌드 서버에서 미리 다운로드해서 패키지에 포함해야 합니다! 아래 STEP 2에 추가하겠습니다.

---

## 추가: STEP 2에 MinIO 바이너리 포함 (놓친 부분 보완)

```bash
# 빌드 서버에서 추가 실행
cd /mnt/transfer/admin-tools

echo "=== MinIO 서버 바이너리 ===" && \
curl -fsSL -o minio \
  https://dl.min.io/server/minio/release/linux-amd64/minio && \
chmod +x minio

echo "=== EPEL, sqlite, docker repo 관련 RPM 추가 ===" && \
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm 2>/dev/null && \
sudo dnf install --downloadonly --downloaddir=rpms \
  createrepo_c 2>/dev/null

# 다시 압축
cd /mnt/transfer
tar -czf admin-tools-bundle.tar.gz admin-tools/
```

---

## Air-gap 설치 실행 (Admin 서버, 모든 준비 완료 후)

```bash
cd FINAL_PACKAGE
chmod +x install_from_airgap_bundle.sh k0s_cluster_with_stack.sh

./install_from_airgap_bundle.sh \
  --bundle ./airgap-bundle-<timestamp>.tar.gz \
  --config ./my-cluster.yaml
```

---

## 📋 최종 체크리스트

```
□ 빌드 서버 준비 완료 (인터넷 연결)
□ Admin 워크스테이션 도구 패키지 생성 (kubectl, helm, yq, jq, git, mc, MinIO)
□ prepare_airgap_bundle.sh로 k0s 번들 생성
□ 애플리케이션 컨테이너 이미지 tar로 저장 (Ray, SAIA, Splunk 등)
□ AI 모델 파일 다운로드 및 압축
□ 전체 패키지 물리적 전달
□ Air-gapped Admin 서버에 도구 설치
□ Docker Registry 및 MinIO 설치
□ my-cluster.yaml에 registryInsecure: true, airgap: true 설정
□ install_from_airgap_bundle.sh 실행
```

---

## 💡 핵심 개선 포인트 정리

| 이전 방식 (수동, 문제 多) | 새 방식 (registryInsecure) |
|---|---|
| containerd v1/v2 직접 판별 | 스크립트가 자동 감지 |
| hosts.toml 직접 작성 | 스크립트가 자동 생성 |
| k0sworker 재시작 시 설정 사라짐 | k0s install→start 사이 자동 적용 |
| nginx ConfigMap 직접 수정 시도 | 애초에 필요 없음 (containerd 레벨 해결) |

이제 **Air-gap 패키지 준비부터 설치까지** 훨씬 안정적으로 진행하실 수 있을 거예요! 진행하시면서 막히는 부분 있으면 바로 알려주세요! 😊
