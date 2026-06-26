# Admin 워크스테이션에서 실행
# 실제 IP로 변경해서 실행하세요

CONTROLLER="172.31.36.33"
CPU_WORKER="172.31.41.6"
GPU_WORKER_1="172.31.31.179"
GPU_WORKER_2="172.31.27.28"
SSH_KEY="/root/.ssh/id_rsa/yoonki-key.pem"
SSH_USER="ec2-user"
ALL_NODES="${CONTROLLER} ${CPU_WORKER} ${GPU_WORKER_1} ${GPU_WORKER_2}"

echo "========================================="
echo "1. 전체 노드 SSH + sudo 확인"
echo "========================================="
for NODE_IP in ${ALL_NODES}; do
  RESULT=$(ssh -i ${SSH_KEY} ${SSH_USER}@${NODE_IP} "sudo whoami" 2>&1)
  echo "${NODE_IP}: ${RESULT}"
done

echo ""
echo "========================================="
echo "2. 디스크 여유 공간 확인"
echo "========================================="
for NODE_IP in ${ALL_NODES}; do
  echo "--- ${NODE_IP} ---"
  ssh -i ${SSH_KEY} ${SSH_USER}@${NODE_IP} "df -h / | tail -1"
done

echo ""
echo "========================================="
echo "3. Python3 확인"
echo "========================================="
for NODE_IP in ${ALL_NODES}; do
  RESULT=$(ssh -i ${SSH_KEY} ${SSH_USER}@${NODE_IP} "python3 --version 2>&1")
  echo "${NODE_IP}: ${RESULT}"
done

echo ""
echo "========================================="
echo "4. MinIO 연결 확인"
echo "========================================="
curl -s -o /dev/null -w "MinIO 상태: %{http_code}\n" \
  http://172.31.50.45:9000/minio/health/live

echo ""
echo "========================================="
echo "5. Docker Registry 확인"
echo "========================================="
curl -s http://172.31.50.45:5000/v2/_catalog

echo ""
echo "========================================="
echo "6. yaml 파일 주요 설정 확인"
echo "========================================="
cd splunk-ai-operator/tools/cluster_setup
echo "--- 노드 IP ---"
grep -A 10 "existingIPs" my-cluster.yaml
echo "--- Storage 설정 ---"
grep -A 8 "objectStore" my-cluster.yaml
echo "--- Registry 설정 ---"
grep "registry:" my-cluster.yaml | head -5
