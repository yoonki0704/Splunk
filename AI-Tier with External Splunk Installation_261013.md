# AI-Tier + 외부 Splunk 설치 가이드 (GA 1.0)

데이터가 있는 **외부 Splunk**에 AI Assistant / Toolkit을 붙이고, 추론은 AWS k0s AI-Tier가 하는 랩 가이드입니다.  
클러스터 안에 Splunk를 올리지 않습니다.

---

## 한 줄 구조

```
노트북 브라우저
  → Splunk Web  (HTTP 공인 :8000)
  → SAIA        (HTTP 공인 :30080)     ← Chrome이 직접 침. Splunk는 프록시가 아님

AI-Tier 파드
  → Splunk 관리 (HTTPS 사설 :8089)     ← JWT 검증 (JWKS)

외부 Splunk 호스트
  → SAIA :30080 / SLIM :30081          ← 온보딩·Toolkit
```

**공인 Splunk Web + 사설 SAIA IP 는 쓰지 마세요.** Chrome이 막습니다. curl은 되고 GUI만 실패합니다.

| 쓰는 것 | 쓰지 않는 것 |
|---------|----------------|
| Splunkbase 앱 9666, Assistant **2.3.0**, Toolkit **>= 6.1.0** | 베타 VOC / preview.tgz |
| `tools/ai-tier-cluster-setup` | `tools/cluster_setup` |
| `trustedIssuers` + JWT `:8089` | `splunk.external.endpoint`, HEC `:8088` |
| SAIA URL `http://<CPU공인>:30080` | `127.0.0.1`, `/tenant/...` 를 URL에 붙이기 |

버전: Splunk Enterprise **10.2**, AI-Tier / SAIA / Ray 이미지 **v1.0**, k0s **v1.36**.

---

## 1. 머신

전부 **RHEL 9.8**, SSH `ec2-user`. GPU는 L40S 또는 H100 중 하나만, **2대**.

| 역할 | 수량 | 권장 | 비고 |
|------|------|------|------|
| Admin | 1 | t3.xlarge+, 디스크 250GB+ | 설치기 + MinIO + HTTP Registry. **클러스터 노드 아님** |
| Controller | 1 | m5.2xlarge, `/var/lib/k0s` 100GB+ | **EBS**. instance store 금지 |
| CPU worker | 1 | m5.4xlarge, 200GB+ | SAIA / SLIM |
| GPU worker | 2 | g6e.12xlarge (L40S), 500GB+ | |
| 외부 Splunk | 1 | Enterprise 10.2 | 데이터가 있는 그 박스 |

같은 VPC, **같은 SG**. 인바운드 Source = VPC CIDR.

필수 포트: **22, 6443, 2380, 10250, 8132, 179, 4789/UDP, 30000-32767, 9000, 9001, 5000**.  
외부 Splunk SG: **8089** 를 클러스터에서 허용. Splunk Web **8000** 은 노트북에서.  
노트북에서 Assistant를 쓰려면 CPU의 **30080** 을 노트북 공인 IP에서도 엽니다 (또는 Admin SOCKS).

22만 열면 설치가 중간에 죽습니다.

---

## 2. Admin 서버 — 처음부터

Admin은 **설치기**입니다. 클러스터 노드가 아닙니다. kubectl을 여기서 쓰려면 나중에 kubeconfig가 필요합니다.  
도구·MinIO·Registry가 이미 있으면 맨 아래 **확인만** 하고 3장으로 가세요. `clean-all` 하지 마세요.

### 2.1 도구

```bash
sudo dnf install -y ca-certificates curl git jq openssh-clients tar wget tmux python3 python3-pyyaml

curl -fsSLO "https://dl.k8s.io/release/v1.36.1/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm -f kubectl

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

sudo curl -fsSL https://github.com/mikefarah/yq/releases/download/v4.44.1/yq_linux_amd64 \
  -o /usr/local/bin/yq && sudo chmod +x /usr/local/bin/yq

curl -fsSL https://github.com/google/go-containerregistry/releases/download/v0.21.9/go-containerregistry_Linux_x86_64.tar.gz \
  -o /tmp/crane.tar.gz
tar -xzf /tmp/crane.tar.gz -C /tmp crane
sudo install -o root -g root -m 0755 /tmp/crane /usr/local/bin/crane
rm -f /tmp/crane.tar.gz /tmp/crane

export PATH="/usr/local/bin:$PATH"
grep -q '/usr/local/bin' ~/.bashrc || echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
echo 'export PATH="/usr/local/bin:$PATH"' | sudo tee /etc/profile.d/usr-local-bin.sh
```

RHEL은 `/usr/local/bin` 이 PATH에 없어서 `kubectl: command not found` 가 납니다. `sudo kubectl` 은 `sudo /usr/local/bin/kubectl` 로 실행하세요.

```bash
kubectl version --client && helm version && yq --version && crane version && git --version
```

GPU 노드가 RHEL이면 Admin에 `sudo dnf install -y createrepo_c` (오프라인 NVIDIA 패키지용).  
SSH 키: `chmod 600 /path/to/private-key`.

클러스터 노드마다 (한 번):

```bash
ssh -i /path/to/private-key ec2-user@<NODE_사설> \
  'hostname; sudo -n true && echo sudo_ok; python3 --version'
# sudo 가 비밀번호를 물으면 그 노드에서:
# echo 'ec2-user ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/ec2-user
# sudo chmod 440 /etc/sudoers.d/ec2-user
```

GPU 드라이버는 직접 깔지 않습니다. `install` 이 넣습니다.

### 2.2 MinIO (AIStor)

예전 `dl.min.io/server/minio/...` 는 **HTTP 410** 입니다. AIStor 경로를 씁니다.

```bash
sudo mkdir -p /data/minio /etc/minio
sudo chown -R "$USER:$USER" /data/minio

sudo curl -fsSL https://dl.min.io/aistor/minio/release/linux-amd64/minio -o /usr/local/bin/minio
sudo curl -fsSL https://dl.min.io/aistor/mc/release/linux-amd64/mc -o /usr/local/bin/mc
sudo chmod +x /usr/local/bin/minio /usr/local/bin/mc
```

`/etc/minio/minio.env` (실제 비밀번호. Git에 넣지 말 것):

```bash
sudo tee /etc/minio/minio.env > /dev/null << 'EOF'
MINIO_ROOT_USER=<MINIO_ROOT_USER>
MINIO_ROOT_PASSWORD=<MINIO_ROOT_PASSWORD>
MINIO_VOLUMES=/data/minio
MINIO_OPTS=--address :9000 --console-address :9001
EOF
sudo chmod 600 /etc/minio/minio.env
```

```bash
sudo tee /etc/systemd/system/minio.service > /dev/null << 'EOF'
[Unit]
Description=MinIO
After=network-online.target
[Service]
User=root
EnvironmentFile=/etc/minio/minio.env
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES $MINIO_OPTS
Restart=always
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now minio
curl -s -o /dev/null -w "minio:%{http_code}\n" http://127.0.0.1:9000/minio/health/live
```

health 200만으로는 S3가 안 열립니다. **AIStor Free 라이선스**가 필요합니다. [Pricing](https://www.min.io/pricing) Free Get Started → `minio.license` 를 Admin으로 복사.

```bash
export PATH="/usr/local/bin:$PATH"
mc alias set local http://127.0.0.1:9000 "<MINIO_ROOT_USER>" "<MINIO_ROOT_PASSWORD>"
mc license register local --license /path/to/minio.license
mc mb local/ai-platform-bucket
```

S3 API = **9000**, 콘솔 = **9001** (`http://<ADMIN_사설>:9001`). 버킷 이름은 yaml과 같고 **소문자**.

### 2.3 HTTP Registry + 이미지 미러

노드가 Docker Hub를 못 가는 이 가이드의 기본입니다. Registry 디스크 **+30GB**.

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo mkdir -p /etc/docker /data/registry
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{ "insecure-registries": ["<ADMIN_사설>:5000"] }
EOF
sudo systemctl enable --now docker
sudo systemctl restart docker

docker run -d --name docker-registry --restart always \
  -p 5000:5000 -v /data/registry:/var/lib/registry registry:2
```

```bash
export PATH="/usr/local/bin:$PATH"
REGISTRY="<ADMIN_사설>:5000"
for image in \
  docker.io/splunk/ai-tier-saia-data-loader:v1.0 \
  docker.io/splunk/ai-tier-saia-api-v2:v1.0 \
  docker.io/splunk/ai-tier-saia-api:v1.0 \
  docker.io/splunk/ai-tier-ray-head:v1.0 \
  docker.io/splunk/ai-tier-ray-worker:v1.0 \
  docker.io/splunk/splunk-ai-operator:v1.0 \
  docker.io/splunk/ai-tier-slim-service:v1.0 \
  docker.io/splunk/splunk:10.2-rhel9 \
  docker.io/splunk/splunk-operator:3.0.0 \
  docker.io/semitechnologies/weaviate:stable-v1.28-007846a \
  docker.io/otel/opentelemetry-collector-contrib:0.122.1 \
  docker.io/fluent/fluent-bit:1.9.6 \
  docker.io/library/nginx:1.27-alpine; do
    crane copy "${image}" "${REGISTRY}/${image#docker.io/}" --insecure
done
curl -s "http://${REGISTRY}/v2/splunk/splunk-operator/tags/list"
```

**`3.0.0` 이 없으면** Operator ImagePullBackOff 입니다. yaml 이미지 경로를 전부 `splunk/` 로 바꾸지 마세요. weaviate·nginx는 원래 경로 그대로입니다.

모델(약 120GB)은 yaml 작성 후:

```bash
CONFIG_FILE=./my-cluster-config.yaml ./k0s_cluster_with_stack.sh stage-artifacts
```

버킷에 이미 있으면 yaml `modelStaging.enabled: false`.

### 2.4 이미 Admin을 쓰는 경우

```bash
which kubectl helm yq crane mc
curl -s -o /dev/null -w "minio:%{http_code}\n" http://127.0.0.1:9000/minio/health/live
curl -s "http://<ADMIN_사설>:5000/v2/splunk/splunk-operator/tags/list"
mc ls local/ai-platform-bucket/model_artifacts/ | head
```

통과하면 2.1–2.3을 다시 하지 마세요. 노드만 교체할 때는 kubeconfig를 `.old` 로 옮기고 yaml IP만 바꾸고 `useExisting: never`.

---

## 3. yaml

```bash
git clone -b <branch> --single-branch https://github.com/splunk/splunk-ai-operator.git
cd splunk-ai-operator/tools/ai-tier-cluster-setup
cp k0s-airgapped-config.yaml my-cluster-config.yaml
```

핵심만. 나머지는 템플릿 유지.

```yaml
cluster:
  name: ai-cluster
  airgap: true
  useExisting: never          # 새 노드 첫 설치. 재실행만 auto
  sshKeyPath: /path/to/private-key
  sshUser: ec2-user

nodes:
  controllers: 1
  cpuWorkers: 1
  gpuWorkers: 2
  existingIPs:
    controllers: ["<CONTROLLER_사설>"]
    workers:
      - "<CPU_사설>"          # CPU 먼저
      - "<GPU1_사설>"
      - "<GPU2_사설>"

storage:
  storageClass: "local-path"
  modelStaging:
    enabled: false            # 첫 설치면 true 이거나 아래 stage-artifacts. 이미 버킷에 있으면 false
  objectStore:
    type: minio
    bucket: ai-platform-bucket
    endpoint: "http://<ADMIN_사설>:9000"
    auth:
      rootUser: "<MINIO_USER>"
      rootPassword: "<MINIO_PASSWORD>"

images:
  registry: "<ADMIN_사설>:5000"
  registryInsecure: true
  splunk:
    operatorImage: "splunk/splunk-operator:3.0.0"   # 필수. docker.io/ 금지

splunk:
  enabled: false              # 내장 Splunk 안 올림
  trustedIssuers:
    - "https://<SPLUNK_사설>:8089"   # 컨트롤러 IP 아님. 127.0.0.1 아님

aiPlatform:
  defaultAcceleratorType: "L40S"     # 또는 H100. 하나만
  serviceTemplate:
    type: NodePort
    nodePort: 30080
    slimNodePort: 30081
  features:
    - { name: saia, version: "1.1.0" }
    - { name: slim, version: "1.0.0" }
```

`splunk.external.endpoint` 는 만들지 마세요.  
`trustedIssuers` 는 공식 템플릿에 **주석**으로만 있습니다. 직접 추가합니다.

이미지 경로는 템플릿의 상대 경로를 유지하세요. 전부 `splunk/` 로 바꾸지 마세요 (weaviate, nginx). 템플릿의 ray/saia/slim 상대 경로도 그대로 둡니다.

첫 설치에서 모델이 버킷에 없으면 yaml을 저장한 뒤:

```bash
CONFIG_FILE=./my-cluster-config.yaml ./k0s_cluster_with_stack.sh stage-artifacts
```

`paste-` / `CHANGE THIS` / `10.0.0.5` 가 yaml에 남아 있으면 이 명령이 거절합니다. L40S↔H100 을 바꾸면 다시 stage 합니다.

---

## 4. 설치

노드: passwordless sudo, `python3-pyyaml`. GPU 드라이버는 install이 넣습니다.

```bash
export PATH="/usr/local/bin:$PATH"
tmux new -s splunk-ai-install
CONFIG_FILE=./my-cluster-config.yaml ./k0s_cluster_with_stack.sh validate
CONFIG_FILE=./my-cluster-config.yaml ./k0s_cluster_with_stack.sh install
```

노드가 Ready로 찍히면 **옆 창에서** 바로:

**HTTP 레지스트리 hosts.toml** (조인 60초 넘으면 설치기가 건너뜀)

Admin에서 컨트롤러는 `k0scontroller`, 워커는 `k0sworker` 로 `/etc/k0s/containerd/certs.d/<ADMIN_사설>:5000/hosts.toml` 을 넣고 서비스를 restart.  
(`server = "http://<ADMIN_사설>:5000"`, `skip_verify = true`)

**local-path** (컨트롤러가 GitHub에 못 나가면 StorageClass가 없음)

```bash
export KUBECONFIG=~/.kube/k0s-ai-cluster
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Operator가 `1/1 Running` 인지 보고, 실패 재개만 `useExisting: auto`.

Admin에서 kubectl 하면 `localhost:8080` 이 나옵니다. **설치 실패가 아닙니다.**

```bash
ls /root/.kube/k0s-* /home/ec2-user/.kube/k0s-*
export KUBECONFIG=/root/.kube/k0s-ai-cluster
kubectl get nodes -o wide
kubectl get pods -n ai-platform | grep -iE 'saia|nginx'
```

모델이 버킷에 있으면 설치는 약 30–60분. GPU 워커 `Ready 0/1` + `Readiness probe failed: success` 는 Serve 프록시 체크입니다. `kubectl describe rayservice` 에 Gemma / GptOss **HEALTHY** 면 됩니다.

---

## 5. 외부 Splunk (여기가 연동)

### JWT — 없으면 Assistant가 403

`btool authentication list oauth2_settings` 가 비면 stanza가 없는 것입니다.

```ini
# /opt/splunk/etc/system/local/authentication.conf
[oauth2_settings]
issuer_uri = https://<SPLUNK_사설>:8089
certFile = $SPLUNK_HOME/etc/auth/server.pem
sslPassword = <server.conf sslConfig 값을 푼 것>
```

Web 공인 IP를 issuer에 넣지 마세요. 파드는 사설 `:8089` 로 키를 가져옵니다.

```bash
SPLUNK_OWNER=$(stat -c '%U' /opt/splunk)
sudo -H -u "$SPLUNK_OWNER" /opt/splunk/bin/splunk stop
sudo -H -u "$SPLUNK_OWNER" /opt/splunk/bin/splunk start --answer-yes --accept-license
/opt/splunk/bin/splunk btool authentication list oauth2_settings
```

클러스터에서 `https://<SPLUNK_사설>:8089/services/authorization/tokens-keys` 가 **200**.

yaml `trustedIssuers` 와 **한 글자도 같게**. 이미 설치했다면:

```bash
kubectl get aiplatform -n ai-platform \
  -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.spec.splunkConfiguration.trustedIssuers}{"\n"}{end}'
# 리스트는 .items[0] 없이 {.spec} 만 치면 항상 비어 보임
```

비었거나 틀리면 AIPlatform을 패치하고 yaml도 고칩니다. ConfigMap을 직접 고치지 마세요.

### 앱

Assistant **2.3.0** `/setup` → **AI tier** (Cloud Connect 아님).  
Toolkit >= 6.1.0 → Connections → Splunk AI tier  
`http://<CPU_공인_또는_사설>:30081/tenant/slim-api/v1alpha1`

### Assistant URL — 브라우저가 여는 것과 같게

GUI는 `saia_sok_url` 만 넣는 경우가 많습니다. **둘 다** 필요합니다.

```ini
# .../Splunk_AI_Assistant_Cloud/local/splunkaiassistant.conf
[saia_sok_configurations]
saia_sok_enabled = true
saia_sok_url = http://<CPU_공인>:30080
saia_endpoint = http://<CPU_공인>:30080
```

노트북이 VPC 밖이면 **CPU Elastic IP:30080** + SG에 노트북 IP. Stop/Start 하면 일반 Public IP는 바뀝니다. EIP를 쓰세요.  
`/tenant/...` 와 `/health` 는 URL에 붙이지 마세요.

소유 유저로 reload/restart 후 Assistant **로그아웃**.

### 브라우저 (curl 200 ≠ GUI)

| Splunk Web | SAIA URL | 결과 |
|------------|----------|------|
| `http://<SPLUNK_공인>:8000` | `http://<CPU_공인>:30080` | OK |
| `http://<SPLUNK_공인>:8000` | `http://172.31.x.x:30080` | Chrome이 막음 (Private Network Access) |
| `https://...:8000` | `http://...:30080` | mixed-content |

Splunk를 공인으로 열면 SAIA도 공인 HTTP. 사설 SAIA를 쓰려면 Splunk Web도 `ssh -L` 로 **localhost** 로 엽니다.

확인 (노트북):

```bash
# GET / 는 404가 정상. 헬스는 /health
curl -4 -sS -o /dev/null -w '%{http_code}\n' "http://<CPU_공인>:30080/health"   # 200
curl -4 -sS -D - -o /dev/null -X OPTIONS \
  "http://<CPU_공인>:30080/tenant/saia-api-v2/v2alpha1/metadata" \
  -H 'Access-Control-Request-Method: GET' \
  -H 'Access-Control-Request-Headers: authorization,splunk-client,x-requested-with,x-stack-url' \
  -H 'Origin: http://<SPLUNK_공인>:8000'   # 204

# JWT 없이 GET metadata → 403 은 정상
```

첫 질문은 `안녕` 또는 SPL 생성만. 「모든 index」는 Context 수집이라 오래 걸립니다.

---

## 6. EC2 Stop/Start

재설치하지 마세요. **Admin MinIO·Registry → k0scontroller → k0sworker → nvidia-smi → Ray.**  
사설 IP는 유지, Public IP만 바뀜 (EIP 제외). `/var/lib/k0s` 는 EBS.

```bash
sudo systemctl start minio
docker start docker-registry
sudo systemctl enable --now k0scontroller   # 컨트롤러
sudo systemctl enable --now k0sworker       # 워커
```

---

## 막히면

| 증상 | 원인 / 할 일 |
|------|----------------|
| `localhost:8080 connection refused` | `export KUBECONFIG=/root/.kube/k0s-ai-cluster` |
| `containerd.toml` 60s / cert-manager webhook | SG **6443 / 8132 / 4789 UDP** |
| Operator Pending PVC | local-path default SC |
| Operator ImagePullBackOff HTTPS | hosts.toml + `operatorImage` short path |
| `btool oauth2_settings` 빈 출력 | authentication.conf 에 stanza 추가 |
| GET metadata **403** (CORS 헤더는 있음) | curl에 JWT 없으면 정상. GUI도 403이면 issuer |
| GUI만 실패, curl health 200 | Splunk 공인 + SAIA 사설 금지. `saia_endpoint` 누락. Network Status |
| OPTIONS 204, GET 401 | `trustedIssuers` = `issuer_uri` = `https://<SPLUNK_사설>:8089` |

공식: [external-splunk-integration.md](https://github.com/splunk/splunk-ai-operator/blob/main/docs/ai-tier-docs/external-splunk-integration.md) · [k0s-quick-reference.md](https://github.com/splunk/splunk-ai-operator/blob/main/docs/ai-tier-docs/k0s-quick-reference.md)
