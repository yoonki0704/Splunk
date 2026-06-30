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
