SSH_KEY="/root/.ssh/yoonki-key.pem"
SSH_USER="ec2-user"
REGISTRY_IP="172.31.50.45"   # Admin 서버 IP

# 모든 워커 노드에 즉시 적용
for NODE_IP in \
  172.31.32.23 \   # CPU Worker
  172.31.24.22 \   # GPU Worker 1
  172.31.31.251; do # GPU Worker 2

  echo "=== 설정 적용: ${NODE_IP} ==="
  ssh -i ${SSH_KEY} ${SSH_USER}@${NODE_IP} "

    # k0s가 실제로 사용하는 hosts 경로 확인
    sudo find /var/lib/k0s -name 'hosts.toml' 2>/dev/null
    sudo find /var/lib/k0s -type d -name '${REGISTRY_IP}*' 2>/dev/null

    # hosts.toml 강제 재생성
    sudo mkdir -p /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${REGISTRY_IP}:5000

    sudo tee /var/lib/k0s/containerd/io.containerd.grpc.v1.cri/hosts/${REGISTRY_IP}:5000/hosts.toml > /dev/null <<EOF
server = \"http://172.31.57.34:5000\"

[host.\"http://172.31.57.34:5000\"]
  capabilities = [\"pull\", \"resolve\", \"push\"]
  skip_verify = true
EOF

    # containerd.toml 설정 확인 및 수정
    sudo cat /etc/k0s/containerd.toml

    # k0sworker 재시작 (containerd도 같이 재시작됨)
    sudo systemctl restart k0sworker
    sleep 15

    # 소켓 생성 확인
    ls -la /run/k0s/containerd.sock 2>/dev/null && \
      echo '소켓 OK ✅' || echo '소켓 없음 ❌'

    # Pull 즉시 테스트
    sudo k0s ctr images pull \
      --plain-http \
      172.31.57.34:5000/ray/ray-head:build-953 2>&1 | tail -3
  "
  echo ""
done
