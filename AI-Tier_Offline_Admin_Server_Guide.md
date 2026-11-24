# AWS 실습: Windows Server에서 AI-Tier 구성요소 전체 다운로드 → 오프라인 Admin 서버 설치

> 구성: 인터넷이 되는 **Windows Server(EC2)** 1대 + 인터넷이 차단된 **Rocky Linux 9.6 Admin 서버(EC2)**. 두 서버 사이에는 SSH와 SCP만 허용합니다.
> 다운로드 항목별 상세 URL, Admin 단계 설명, 설정 파일 수정, 트러블슈팅은 함께 전달한 `AI-Tier_Offline_Kit_Windows_Rocky9.6_H100.md`를 참고하세요.
> 이 문서에는 키, 비밀번호, 실제 IP를 적지 않습니다. `<...>`는 현장 값으로 바꿉니다.

---

## 0. 전체 흐름

```
[Windows Server EC2]  인터넷 O, C: 300 GB
  Get-AITierOffline.ps1  →  C:\ai-offline  (약 30 GB, Rocky ISO와 모델 제외)
        │  scp.exe (VPC 내부, 22/tcp)
        ▼
[Admin EC2: Rocky 9.6]  인터넷 X, 루트 볼륨 300 GB
  /opt/ai-offline/ai-offline-admin.sh all
        │  SSH
        ▼
[k0s 노드 EC2: Rocky 9.6]  인터넷 X  (CPU 워커, H100 GPU 워커)
```

---

## 1. AWS 리소스 준비

### 1-1. Windows Server (다운로드용)

| 항목 | 권장 |
|---|---|
| AMI | Windows Server 2022 또는 2025 (2019도 가능). **2016은 쓰지 않습니다.** curl.exe와 tar.exe가 없습니다 |
| 인스턴스 | m6i.xlarge 이상. 네트워크 대역폭이 클수록 빠름 |
| 디스크 | 루트 C: 기본값(30 GB)은 부족합니다. **루트를 300 GB로 만들거나**, 데이터용 EBS 200 GB를 추가합니다 |
| 네트워크 | 퍼블릭 서브넷 또는 NAT를 통한 인터넷 아웃바운드. RDP(3389)는 내 IP에서만 허용 |

디스크를 하나(예: 300 GB)로 만들었다면 C:만 써도 됩니다. EC2Launch가 첫 부팅 때 C: 파티션을 볼륨 크기까지 늘려 주는데, 늘어나지 않았으면 직접 확장합니다 (관리자 PowerShell):

```powershell
Get-PSDrive C    # Free가 250 GB 이상이면 OK
Resize-Partition -DriveLetter C -Size (Get-PartitionSupportedSize -DriveLetter C).SizeMax
```

선택 사항: 실습 환경이라면 Defender 실시간 검사에서 다운로드 폴더를 제외하면 100 GB 쓰기가 훨씬 빨라집니다.

```powershell
Add-MpPreference -ExclusionPath 'C:\ai-offline'
```

필요한 도구가 있는지 확인합니다.

```powershell
curl.exe --version; tar.exe --version; Get-Command scp.exe
```

`scp.exe`가 없으면 설치합니다. 이 서버는 인터넷이 되므로 바로 설치됩니다.

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

### 1-2. Admin 서버 (오프라인)

| 항목 | 권장 |
|---|---|
| AMI | Rocky Linux 9.6 공식 AMI. EC2 콘솔 AMI 검색에서 `Rocky-9-EC2-Base-9.6`을 찾습니다. 게시자가 Rocky Enterprise Software Foundation인지 확인하세요. **9.7 이상이나 "latest" AMI는 쓰지 않습니다** |
| 인스턴스 | m6i.2xlarge 이상 (8 vCPU, 32 GB) |
| 디스크 | **루트 EBS gp3 300 GB** (키트 30 GB + 모델 63 GB + SeaweedFS 모델 63 GB + 레지스트리 26 GB + 번들과 여유). 루트 파티션은 첫 부팅 때 cloud-init이 자동으로 확장합니다 |
| 기본 사용자 | `rocky` |
| 인터넷 차단 | 프라이빗 서브넷에 두고 라우트 테이블에 IGW와 NAT 경로를 넣지 않습니다. 또는 보안그룹 아웃바운드를 VPC CIDR로만 제한합니다 |
| 보안그룹 인바운드 | 22/tcp (Windows Server의 프라이빗 IP에서), 8333/tcp와 5000/tcp (k0s 노드 보안그룹에서) |

> **설치 전에 `dnf update`를 하지 마세요.** 인터넷을 막기 전에라도 업데이트하면 키트의 Rocky 9.6 GA RPM과 버전이 달라집니다.

### 1-3. k0s 노드 (컨트롤러, CPU 워커, GPU 워커)

- Admin과 **같은 Rocky 9.6 AMI**로 만들고 업데이트하지 않습니다. 이렇게 하면 노드의 패키지 버전이 Admin과 같아지므로, Admin의 rpm 목록을 노드 정보로 그대로 쓸 수 있습니다.
- H100 GPU 워커는 `p5.48xlarge`(H100 8장)입니다. 비용이 크므로 필요한 시간만 띄우세요. 설정 파일의 `aiPlatform.defaultAcceleratorType`은 `"H100"`입니다.
- 노드 보안그룹: 노드끼리 모든 트래픽 허용, Admin에서 22/tcp 허용.

---

## 2. 노드 정보 수집 (Windows Server에서 실행)

NVIDIA 드라이버를 노드에서 빌드하려면 kernel-devel, glibc-devel 등이 노드와 정확히 같은 버전이어야 합니다. Windows Server에서 SSH로 직접 수집합니다. GPU 노드가 아직 없으면 같은 AMI로 만든 Admin에서 수집합니다.

```powershell
# EC2 키 파일 권한 정리 (Windows OpenSSH는 권한이 넓으면 키를 거부함)
icacls C:\keys\<key>.pem /inheritance:r /grant:r "$($env:USERNAME):R"

mkdir C:\ai-kit -Force; cd C:\ai-kit
ssh.exe -i C:\keys\<key>.pem rocky@<gpu-node-or-admin-private-ip> "uname -r"
ssh.exe -i C:\keys\<key>.pem rocky@<gpu-node-or-admin-private-ip> "rpm -qa --qf '%{NAME} %{VERSION}-%{RELEASE} %{ARCH} %{SOURCERPM}\n' | sort" > gpu-node1-rpms.txt
```

> PowerShell 5.1의 `>`는 UTF-16으로 저장합니다. 스크립트는 그대로 읽을 수 있으므로 변환하지 않아도 됩니다.

`uname -r` 결과가 `5.14.0-570.17.1.el9_6.x86_64`(9.6 GA)이면 키트의 `rpms\rocky`에 이미 맞는 커널 패키지가 있습니다.

---

## 3. 다운로드 (Windows Server)

### 3-1. 스크립트 복사 (MacBook → Windows Server)

**방법 A (권장): Windows App(구 Microsoft Remote Desktop)의 폴더 리디렉션**
1. MacBook에서 두 파일을 한 폴더(예: `~/ai-kit`)에 둡니다.
2. Windows App에서 해당 PC를 우클릭하고 **Edit**를 엽니다. **Folders** 탭에서 **Redirect folders**를 체크하고 `+`로 `~/ai-kit`을 추가한 뒤 저장합니다.
3. 접속해 있었다면 끊고 다시 접속합니다.
4. Windows 탐색기 **내 PC**에 `ai-kit on <Mac이름>`이 보입니다. 주소창에 `\\tsclient\ai-kit`을 입력해도 됩니다. 두 파일을 `C:\ai-kit`으로 복사합니다.

**방법 B: S3 presigned URL** (RDP 리디렉션이 막혀 있을 때)
```bash
# MacBook
aws s3 cp ~/ai-kit/ s3://<bucket>/ai-kit/ --recursive
aws s3 presign s3://<bucket>/ai-kit/Get-AITierOffline.ps1 --expires-in 3600
aws s3 presign s3://<bucket>/ai-kit/ai-offline-admin.sh --expires-in 3600
```
```powershell
# Windows Server
mkdir C:\ai-kit -Force; cd C:\ai-kit
curl.exe -fL -o Get-AITierOffline.ps1 "<presigned-url-1>"
curl.exe -fL -o ai-offline-admin.sh  "<presigned-url-2>"
```

**방법 C: 내용 붙여넣기.** 메모장을 열어 각 파일 내용을 붙여넣고 같은 이름으로 저장합니다(인코딩은 UTF-8). 두 파일 모두 ASCII 문자만 쓰므로 인코딩 문제가 없습니다. `ai-offline-admin.sh`가 CRLF로 저장되어도 다운로드 스크립트가 LF로 바꿔서 키트에 넣습니다.

복사 후 파일이 손상되지 않았는지 확인합니다. MacBook에서 `shasum -a 256 *`로 구한 값과 Windows Server에서 `Get-FileHash C:\ai-kit\*`로 구한 값이 같아야 합니다. 방법 C로 만든 파일은 줄바꿈이 달라져 해시가 다를 수 있는데, 정상입니다.

### 3-2. 실행

```powershell
cd C:\ai-kit
Unblock-File .\Get-AITierOffline.ps1
powershell -ExecutionPolicy Bypass -File .\Get-AITierOffline.ps1 `
  -Out C:\ai-offline `
  -GpuKernel <uname -r 결과> `
  -GpuNodeRpmList .\gpu-node1-rpms.txt
```

- 출력은 리다이렉션하지 말고 콘솔에 그대로 둡니다.
- 중단되면 같은 명령을 다시 실행해 이어받습니다.
- **Rocky ISO와 LLM 모델은 기본값으로 받지 않습니다.** Rocky는 VM에 이미 설치되어 있으므로 부족한 RPM 317개(약 0.7 GB)만 받고, 모델은 고객 Windows PC에 받아 둔 것을 씁니다. 대신 모델 검증용 `models-manifest.tsv`를 만듭니다.
- **AWS 실습에는 고객 모델이 없으므로** 모델 흐름까지 시험하려면 위 명령에 `-WithModels`를 붙입니다 (+63 GB). 일부만 받으려면 `-WithModels -ModelIds all-minilm-l6-v2,cross-encoder`처럼 지정합니다. ISO가 필요하면 `-WithIso`를 씁니다.
- 소요 시간: EC2에서는 대부분 수십 분 안에 끝납니다. 가장 큰 항목은 AI-Tier 이미지(26 GB)입니다. 마지막 단계에서 SHA256SUMS를 만드느라 해시를 계산하는 시간이 추가됩니다.

### 3-3. 받는 항목 요약

| 폴더 | 내용 | 크기 |
|---|---|---|
| `bin\` | kubectl 1.36.1, helm 3.22.1, crane 0.21.9, yq 4.44.1, mc, SeaweedFS 4.55, k0s v1.36.1+k0s.0 (모두 Linux용) | 0.8 GB |
| `installer\` | splunk-ai-operator `release/v1.0` 소스 tar.gz | 2 MB |
| `bundle\manifests\`, `bundle\charts\` | cert-manager, local-path, nvidia-device-plugin, kube-prometheus-stack, opentelemetry-operator, kuberay-operator, metallb | 2 MB |
| `bundle\images\` | k0s 시스템 이미지 13개, add-on 이미지 24개 | 1.9 GB |
| `images-extra\` | registry:2 | 10 MB |
| `images-aitier\` | AI-Tier 플랫폼 이미지 13개 + `images.txt` | 26 GB |
| `rpms\rocky\` | 최소 설치 Rocky 9.6에 없는 Admin 패키지, NVIDIA 빌드 의존성, createrepo_c (317개 + `bootstrap.txt`) | 0.7 GB |
| `rpms\extras\` | NVIDIA 드라이버 610.57.04, container-toolkit, dkms(EPEL), Docker CE, 노드 버전에 맞춘 Rocky RPM | 0.6 GB |
| 루트 | `ai-offline-admin.sh`, `models-manifest.tsv`, `SHA256SUMS`, `kit-info.txt` | |
| `models\`, `iso\` | `-WithModels`, `-WithIso`를 썼을 때만 | 63 GB, 12 GB |

---

## 4. Admin 서버로 전송 (SCP)

Admin에서 받을 자리를 먼저 확인합니다. 루트 볼륨이 300 GB로 확장되었는지 봅니다.

```bash
df -h /
```

Windows Server에서 전송합니다. `_windows` 폴더도 30 MB 정도라 같이 보내도 됩니다.

```powershell
scp.exe -i C:\keys\<key>.pem -r C:\ai-offline rocky@<admin-private-ip>:/home/rocky/
```

Admin에서 위치를 옮기고 검증합니다.

```bash
sudo mv /home/rocky/ai-offline /opt/ai-offline
sudo chown -R rocky:rocky /opt/ai-offline
cd /opt/ai-offline && sha256sum -c SHA256SUMS --quiet && echo "ALL OK"
```

고객 환경에서는 고객이 받아 둔 모델 폴더를 `/opt/ai-offline/models/` 아래에 모델마다 하나씩 넣습니다. 폴더 이름은 모델 ID(`gpt-oss-20b`), Hugging Face 저장소 이름(`UAE-Large-V1`), HF 캐시 이름(`models--openai--gpt-oss-20b`) 중 무엇이든 됩니다. 자세한 규칙은 메인 가이드 6-0절을 참고하세요. `sha256sum -c`는 `installer` 단계 전에 실행합니다 (그 단계에서 모델 폴더가 installer 쪽으로 이동합니다).

전송이 중간에 끊기면 `sha256sum -c`에서 실패한 파일만 다시 보내면 됩니다. 예를 들어 `scp.exe -i ... C:\ai-offline\models\gpt-oss-20b\model-00000-of-00002.safetensors rocky@<ip>:/opt/ai-offline/models/gpt-oss-20b/`처럼 보냅니다. WinSCP의 "동기화" 기능을 써도 됩니다.

---

## 5. Admin 설치 (오프라인)

```bash
cd /opt/ai-offline
chmod +x ai-offline-admin.sh

export S3_ACCESS_KEY=<access-key>
read -rsp 'S3 secret: ' S3_SECRET_KEY; echo; export S3_SECRET_KEY
export GPU_KERNELS=<gpu-node-uname-r>
export NODES="rocky@<controller-ip> rocky@<cpu-worker-ip> rocky@<gpu-worker-ip>"   # Admin에서 노드로 SSH 가능할 때

./ai-offline-admin.sh all 2>&1 | tee /opt/ai-offline/admin-setup.log
```

`all` 순서: `repos`(rocky·extras 로컬 repo) → `packages` → `tools` → `installer`(모델 폴더 이동과 검증) → `objectstore`(SeaweedFS :8333) → `models`(업로드와 10개 검증) → `registry`(registry:2 :5000) → `pushimages`(AI-Tier 이미지 13개) → `bundle`(air-gap 번들과 NVIDIA closure) → `nodeprep`(노드 python3-pyyaml) → `verify`

`NODES` 단계와 installer는 Admin에서 노드로 SSH 키 접속이 되어야 합니다. EC2 키를 Admin의 `~/.ssh/`에 두고(권한 600), 설정 파일의 `cluster.sshKeyPath`와 `cluster.sshUser: rocky`를 맞춥니다.

---

## 6. 설정 파일과 설치

```bash
cd /opt/ai-offline/splunk-ai-operator/tools/ai-tier-cluster-setup
cp k0s-cluster-config.yaml my-cluster-config.yaml && chmod 600 my-cluster-config.yaml
sed -i -E '/[Ii]mage: "docker\.io\//s#"docker\.io/#"<admin-private-ip>:5000/#' my-cluster-config.yaml
vi my-cluster-config.yaml
```

| 키 | 값 |
|---|---|
| `cluster.airgap` | `true` |
| `cluster.sshUser` / `cluster.sshKeyPath` | `rocky` / `~/.ssh/<key>.pem` |
| `nodes.*` | 노드 개수와 프라이빗 IP |
| `storage.modelStaging.enabled` | `false` |
| `storage.objectStore.type` / `endpoint` / `bucket` | `seaweedfs` / `http://<admin-private-ip>:8333` / `ai-platform-bucket` |
| `storage.objectStore.auth.rootUser` / `rootPassword` | 5장에서 입력한 S3 키 |
| `images.registry` / `images.registryInsecure` | `<admin-private-ip>:5000` / `true` |
| `aiPlatform.defaultAcceleratorType` | `"H100"` |
| Splunk 연동 (외부 Splunk, JWT, 8089) | 기존 가이드와 동일 |

```bash
export AIRGAP_BUNDLE_DIR=/opt/ai-offline/airgap-bundle/airgap-bundle-offline
source "${AIRGAP_BUNDLE_DIR}/airgap-env.sh"
CONFIG_FILE=./my-cluster-config.yaml ./k0s_cluster_with_stack.sh validate
CONFIG_FILE=./my-cluster-config.yaml ./k0s_cluster_with_stack.sh install
```

---

## 7. AWS 실습 체크리스트

- [ ] Windows Server 2022 또는 2025, C: 여유 60 GB 이상 (`-WithModels`면 130 GB 이상), `curl.exe`, `tar.exe`, `scp.exe` 확인
- [ ] Admin과 노드는 같은 Rocky 9.6 AMI이고 `dnf update`를 하지 않음
- [ ] Admin 루트 볼륨 300 GB, 인터넷 경로 없음, 22/8333/5000 보안그룹 설정
- [ ] `uname -r`와 rpm 목록을 수집해 `-GpuKernel`, `-GpuNodeRpmList`로 전달
- [ ] 다운로드 완료 후 `C:\ai-offline\SHA256SUMS` 생성 확인
- [ ] Admin에서 `sha256sum -c SHA256SUMS` 통과
- [ ] `ai-offline-admin.sh all` 끝에 `Bundle verified` 출력
- [ ] `curl -s http://127.0.0.1:5000/v2/_catalog`에 저장소 13개
- [ ] `installer` 단계에서 `models: 10 OK, 0 bad, 0 missing`
- [ ] `models` 단계에서 10개 모델 모두 업로드 확인
