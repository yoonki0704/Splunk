# AI 모델 다운로드 정보 확인 🔍

README와 설치 스크립트에 따르면 모델은 **HuggingFace**에서 다운로드합니다.

---

## README에서 확인된 모델 정보

README에 명시된 모델 목록:

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

> **총 10개 모델, 120GB 이상**

---

## 설치 스크립트에서 모델 다운로드 관련 파일 확인

```bash
# splunk-ai-operator 레포에서 모델 관련 스크립트 확인
find splunk-ai-operator -name "*.sh" | xargs grep -l \
  "huggingface\|download\|model" 2>/dev/null

# 모델 다운로드 스크립트 위치 확인
ls -la splunk-ai-operator/tools/artifacts_download_upload_scripts/

# 모델 설정 파일 확인
find splunk-ai-operator -name "model_artifacts_configs.yaml" 2>/dev/null
cat splunk-ai-operator/tools/cluster_setup/model_artifacts_configs.yaml \
  2>/dev/null || echo "파일 없음"
```

---

## 가장 쉬운 방법: yaml에서 modelStaging 활성화

```bash
vi splunk-ai-operator/tools/cluster_setup/my-cluster.yaml
```

```yaml
storage:
  modelStaging:
    enabled: true     # ✏️ false → true 로 변경
                      # 설치 시 자동으로 HuggingFace에서
                      # 다운로드 후 MinIO에 업로드
```

> ⚠️ **주의사항:**
> - 다운로드 용량: **120GB 이상**
> - Admin 서버 디스크: **250GB 이상** 여유 필요
> - 소요 시간: 인터넷 속도에 따라 **2~6시간**

---

## Admin 서버 디스크 확인

```bash
# 디스크 여유 공간 확인
df -h /

# 250GB 이상 여유 있어야 함
# 부족하면 EBS 볼륨 추가 필요
```

---

## 모델 다운로드 스크립트 직접 확인

```bash
# 다운로드 스크립트 내용 확인
cat splunk-ai-operator/tools/artifacts_download_upload_scripts/download_from_huggingface.sh \
  2>/dev/null | head -50

# 업로드 스크립트 확인
ls splunk-ai-operator/tools/artifacts_download_upload_scripts/
```

---

## 결과에 따른 두 가지 방법

### 방법 A: 자동 다운로드 (modelStaging: true)

```yaml
# my-cluster.yaml
storage:
  modelStaging:
    enabled: true
```

```bash
# 설치 실행 시 자동으로 다운로드 + MinIO 업로드
CONFIG_FILE=./my-cluster.yaml \
  ./k0s_cluster_with_stack.sh install
```

### 방법 B: 수동 다운로드 후 설치

```bash
# 모델만 먼저 다운로드
CONFIG_FILE=./my-cluster.yaml \
  ./k0s_cluster_with_stack.sh stage-artifacts

# 다운로드 완료 후 설치
CONFIG_FILE=./my-cluster.yaml \
  ./k0s_cluster_with_stack.sh install
```

---

우선 아래 명령어 결과를 알려주시면 정확한 다운로드 방법을 안내해 드릴게요!

```bash
# 1. 디스크 용량 확인
df -h /

# 2. 모델 관련 스크립트 확인
ls splunk-ai-operator/tools/artifacts_download_upload_scripts/ \
  2>/dev/null || echo "경로 없음"
```

😊
