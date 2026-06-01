# 클라우드 가상화 기술

## 클라우드 보안 기술 실습

본 실습에서는 클라우드 네이티브 환경의 **CNAPP(Cloud-Native Application Protection Platform)** 5대 영역을 오픈소스 도구로 직접 구축하는 과정을 다룸

이론에서 학습한 **CSPM, CWPP, CIEM, KSPM, DSPM**의 5대 영역을 실제 도구로 구현함

클라우드 인프라 보안(CSPM/CIEM/DSPM)은 `LocalStack`으로 구성한 로컬 AWS 환경에서, 워크로드·클러스터 보안(CWPP/KSPM)은 `kubeadm` 기반의 실제 Kubernetes 클러스터(v1.29) 위에서 실습함

| 구분 | 사용 도구 | CNAPP 영역 |
| --- | --- | --- |
| 클라우드 설정 오류 점검 | Prowler | CSPM |
| 클라우드 권한 분석 | Steampipe | CIEM |
| 클라우드 데이터 보안 | Presidio | DSPM |
| 이미지/IaC 취약점 스캔 | Trivy | CWPP |
| 클러스터 보안 점검 | Kubescape | KSPM |
| 정책 강제 (Admission) | Kyverno, OPA Gatekeeper | KSPM / 거버넌스 |
| 런타임 격리 / 탐지 | Seccomp, AppArmor, Falco | CWPP / 런타임 |

---

## 1. 실습 환경 사전 준비

### 환경 요구사항

- Ubuntu 24.04 LTS
- 최소 RAM 8GB (SOLID CLOUD Large 이상)
- 디스크 여유 공간 20GB 이상
- 인터넷 연결 (apt 저장소 및 컨테이너 이미지 다운로드)
- 구성: 단일 노드 (Master 1대)

### 컨테이너 런타임(containerd) 설치

```bash
# 패키지 인덱스 갱신 및 필수 패키지 설치
sudo apt-get update
sudo apt-get install -y curl ca-certificates gnupg

# Docker GPG 키 등록
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker 저장소 등록
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# containerd 및 Docker 설치 (containerd: K8s 런타임, docker: LocalStack 실행용)
sudo apt-get update
sudo apt-get install -y containerd.io docker-ce docker-ce-cli

# containerd 설정 (SystemdCgroup 활성화)
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
sudo systemctl restart containerd

# Docker 서비스 시작 및 현재 사용자 권한 부여 (sudo 없이 docker 사용)
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```

### 참고

- `SystemdCgroup = true` 설정은 kubelet의 cgroup 드라이버(systemd)와 일치시키기 위해 필수임
- 이 설정이 누락되면 kubelet이 정상 기동되지 않음
- `containerd`는 Kubernetes의 컨테이너 런타임으로, `docker`는 이후 CSPM/CIEM/DSPM 실습에서 LocalStack 컨테이너를 실행하기 위해 함께 설치함

---

## 2. Kubernetes 설치 (v1.29)

### 저장소 등록 및 커널 설정

```bash
# Kubernetes apt 저장소 등록 (v1.29)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# IPv4 포워딩 활성화 및 swap 비활성화 (Kubernetes 필수 요건)
sudo sysctl -w net.ipv4.ip_forward=1
sudo swapoff -a

# bridge netfilter 활성화
sudo modprobe br_netfilter
sudo bash -c 'echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables'
```

### Kubernetes 구성 요소 설치

```bash
# kubelet, kubeadm, kubectl 설치
sudo apt-get install -y kubelet kubeadm kubectl

# 자동 업데이트 방지 (버전 고정)
sudo apt-mark hold kubelet kubeadm kubectl
```

### 설치 확인

```bash
# 버전 확인
kubeadm version
kubelet --version
kubectl version --client
```

![figure1](./images/figure1.png)

---

## 3. 클러스터 초기화 (Master 노드)

### config.yaml 작성

```bash
cd ~
sudo tee config.yaml > /dev/null <<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "10.244.0.0/16"      # Pod 네트워크 대역
  serviceSubnet: "10.96.0.0/16"   # Service 네트워크 대역
clusterName: "security-practice-cluster"
EOF
```

### 클러스터 초기화

```bash
# 클러스터 초기화
sudo kubeadm init --config=config.yaml --upload-certs | tee -a ~/k8s_init.log

# kubectl 접근 설정
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $USER:$USER $HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" | tee -a ~/.bashrc
export KUBECONFIG=$HOME/.kube/config
```

![figure2](./images/figure2.png)

### 스케줄링 제한 해제

```bash
# 단일 노드 환경에서는 Master 노드의 taint를 제거해야 Pod가 배포됨
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### 참고

- 기본적으로 Master 노드는 일반 워크로드가 스케줄링되지 않도록 taint가 설정되어 있음
- 단일 노드 실습에서는 이 제한을 제거해야 모든 Pod가 정상 배포됨

---

## 4. CNI 및 Helm 설치

### Cilium CNI 설치

```bash
# Cilium CLI 다운로드 및 설치
curl -sL --remote-name https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
sudo tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin
rm cilium-linux-amd64.tar.gz

# Cilium 설치
cilium install
```

### 클러스터 상태 확인

```bash
# Cilium 정상화 대기
cilium status --wait

# 노드 및 전체 Pod 상태 확인 (모두 Ready / Running 이어야 함)
kubectl get nodes
kubectl get pods -A
```

![figure3](./images/figure3.png)
![figure4](./images/figure4.png)

### Helm 설치

```bash
# Helm 3 설치
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
bash ./get_helm.sh
rm ./get_helm.sh

# 버전 확인
helm version
```

---

## 5. LocalStack 클라우드 환경 구축

CSPM, CIEM, DSPM 실습은 클라우드 인프라(AWS)를 점검 대상으로 함

실제 AWS 계정 대신 `LocalStack`으로 로컬에 가짜 AWS 환경을 구성하여 비용 없이 클라우드 보안을 실습함

### AWS CLI 도구 설치

```bash
# 실습용 가상환경 생성
python3 -m venv ~/cnapp-venv
source ~/cnapp-venv/bin/activate

# AWS CLI, awscli-local(엔드포인트 자동 지정 래퍼) 설치
pip install awscli awscli-local
```

### LocalStack 실행 (Docker)

```bash
# LocalStack 컨테이너 실행 (S3, IAM, STS 서비스)
docker run -d \
  --name localstack \
  -p 4566:4566 \
  -e SERVICES=s3,iam,sts \
  localstack/localstack:4.4.0

# 컨테이너가 Up 상태인지 확인
docker ps | grep localstack
```

### 가짜 AWS 자격증명 설정 및 동작 확인

```bash
# LocalStack은 자격증명 값을 검증하지 않으므로 임의 값 사용
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# S3 버킷 생성/조회로 정상 동작 확인
awslocal s3 mb s3://test-check
awslocal s3 ls

# 확인 후 테스트 버킷 삭제
awslocal s3 rb s3://test-check
```

![figure36](./images/figure36.png)

### 취약한 클라우드 리소스 생성

```bash
# 1) 퍼블릭 접근이 가능한 S3 버킷 (설정 오류)
awslocal s3 mb s3://public-data
awslocal s3api put-bucket-acl --bucket public-data --acl public-read

# 2) 암호화되지 않은 S3 버킷
awslocal s3 mb s3://unencrypted-data

# 3) 과도한 권한(전체 허용)을 가진 IAM 사용자
awslocal iam create-user --user-name test-admin
awslocal iam attach-user-policy --user-name test-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 4) 읽기 전용 권한만 가진 정상 사용자 (정상 - 최소 권한 준수)
awslocal iam create-user --user-name test-readonly
awslocal iam attach-user-policy --user-name test-readonly \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

![figure37](./images/figure37.png)

### 참고

- `LocalStack`은 AWS API를 로컬에서 모방하는 도구로, 인터넷·AWS 계정·비용 없이 클라우드 환경을 시뮬레이션함
- `awslocal`은 `aws` 명령에 `--endpoint-url=http://localhost:4566`을 자동으로 붙여주는 래퍼임
- 2026년 3월 이후 LocalStack 최신 버전은 인증 토큰을 요구하므로, 토큰 없이 사용 가능한 마지막 community 버전(`4.4.0`)을 Docker 이미지로 고정하여 실행함
- 무료(community) 기능으로 S3, IAM 등 핵심 서비스를 지원하므로 CSPM/CIEM/DSPM 실습에 충분함

---

## 6. CSPM - 클라우드 설정 오류 점검 (Prowler)

`CSPM(Cloud Security Posture Management)`은 클라우드 인프라의 설정 오류와 컴플라이언스 위반을 점검하는 영역

`Prowler`는 CIS 벤치마크 기반으로 클라우드 설정을 점검하는 오픈소스 CSPM 도구로, PDF에서 소개한 CloudSploit과 동일한 CSPM 영역을 담당함

### Prowler 설치 (별도 가상환경)

Prowler는 `awscli`와 다른 버전의 `botocore`를 요구하므로, 충돌을 피하기 위해 별도 가상환경에 설치함

```bash
# Prowler 전용 가상환경 생성 (cnapp-venv와 분리)
deactivate 2>/dev/null
python3 -m venv ~/prowler-venv
source ~/prowler-venv/bin/activate

# Prowler 설치 (의존성이 많아 수 분 소요됨)
pip install prowler
```

### LocalStack 대상 스캔

```bash
# 자격증명 및 LocalStack 엔드포인트 지정
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://localhost:4566

# S3 서비스 설정 점검 (LocalStack 대상)
prowler aws --service s3 --ignore-exit-code-3
```

![figure38](./images/figure38.png)

### 취약 항목 조치 후 재점검

스캔에서 탐지된 설정 오류를 조치하면 FAIL 개수가 감소함

조치는 `awslocal`로 수행하므로 `cnapp-venv`에서 실행함 (Prowler 스캔은 `prowler-venv`에서 실행)

```bash
# === awslocal 환경(cnapp-venv)에서 조치 ===
source ~/cnapp-venv/bin/activate

# 1) 버킷 버저닝 활성화 (실수로 인한 데이터 삭제/덮어쓰기 방지)
awslocal s3api put-bucket-versioning --bucket public-data \
  --versioning-configuration Status=Enabled
awslocal s3api put-bucket-versioning --bucket unencrypted-data \
  --versioning-configuration Status=Enabled

# 2) HTTPS 강제 정책 적용 (평문 전송 차단)
awslocal s3api put-bucket-policy --bucket public-data --policy '{
  "Version":"2012-10-17",
  "Statement":[{
    "Sid":"DenyInsecureTransport",
    "Effect":"Deny",
    "Principal":"*",
    "Action":"s3:*",
    "Resource":["arn:aws:s3:::public-data","arn:aws:s3:::public-data/*"],
    "Condition":{"Bool":{"aws:SecureTransport":"false"}}
  }]
}'
```

```bash
# === Prowler 환경(prowler-venv)에서 재점검 ===
source ~/prowler-venv/bin/activate
export AWS_ENDPOINT_URL=http://localhost:4566

# 다시 스캔하면 조치한 항목만큼 FAIL이 감소함
prowler aws --service s3 --ignore-exit-code-3
```

![figure39](./images/figure39.png)

### 참고

- Prowler는 점검 결과를 PASS/FAIL로 출력하며, FAIL은 보안 기준 위반(설정 오류)을 의미함
- 일부러 생성한 퍼블릭/미암호화 버킷이 다수의 FAIL로 탐지됨
- 결과 상단의 `AWS Account: 000000000000`은 LocalStack의 가짜 계정번호로, 실제 AWS가 아닌 LocalStack을 점검했음을 의미함
- Prowler는 `boto3` 기반이라 `AWS_ENDPOINT_URL` 환경변수를 자동 인식하므로, 별도의 엔드포인트 옵션 없이 LocalStack으로 요청이 전송됨
- 버저닝/HTTPS 정책 등을 조치한 뒤 재점검하면 FAIL이 감소하여 탐지 -> 조치 -> 개선의 CSPM 흐름을 확인할 수 있음
- 단, 일부 항목(Cross-Region Replication, MFA Delete, Object Lock 등)은 LocalStack 커뮤니티 버전에서 지원되지 않아 조치해도 FAIL로 남음

---

## 7. CIEM - 클라우드 권한 분석 (Steampipe)

`CIEM(Cloud Infrastructure Entitlement Management)`은 클라우드 자원에 대한 접근 권한(IAM)을 분석하여 과도하거나 미사용 중인 권한을 식별하는 영역

`Steampipe`는 클라우드 인프라를 SQL 테이블로 매핑하여 표준 SQL로 권한 및 설정을 분석하는 오픈소스 도구

### Steampipe 설치 및 AWS 플러그인 추가

```bash
# Steampipe 설치
sudo /bin/sh -c "$(curl -fsSL https://steampipe.io/install/steampipe.sh)"

# 버전 확인
steampipe --version

# AWS 플러그인 설치
steampipe plugin install aws
```

### LocalStack 연동 설정

```bash
# AWS 플러그인이 LocalStack을 바라보도록 설정
cat > ~/.steampipe/config/aws.spc <<'EOF'
connection "aws" {
  plugin              = "aws"
  access_key          = "test"
  secret_key          = "test"
  regions             = ["us-east-1"]
  endpoint_url        = "http://localhost:4566"
  s3_force_path_style = true
}
EOF
```

### SQL로 과도 권한 사용자 식별

```bash
# IAM 사용자 목록 조회 (연동 확인)
steampipe query "select name, arn from aws_iam_user;"

# AdministratorAccess(과도한 권한)가 부여된 사용자 식별
steampipe query "
select
  u.name as user_name,
  p.name as policy_name
from
  aws_iam_user as u,
  jsonb_array_elements(u.attached_policy_arns) as policy_arn
  join aws_iam_policy as p on p.arn = trim('\"' from policy_arn::text)
where
  p.name = 'AdministratorAccess';
"
```

![figure40](./images/figure40.png)

### 참고

- Steampipe는 클라우드 자원을 PostgreSQL 테이블로 매핑하여 친숙한 SQL로 권한 상태를 분석함
- CIEM의 핵심은 과도하거나 미사용 중인 권한을 식별하여 공격 표면(Attack Surface)을 줄이는 것임
- 앞서 생성한 `test-admin` 사용자가 `AdministratorAccess` 보유자(과도 권한)로 탐지됨
- `endpoint_url` 설정으로 Steampipe AWS 플러그인이 실제 AWS 대신 LocalStack을 점검함
- CloudQuery도 동일한 인프라를 SQL로 분석하는 CIEM 도구이나, 로그인 및 유료 플랜이 필요해져 동일 방식의 오픈소스 Steampipe로 실습함

---

## 8. DSPM - 클라우드 데이터 보안 점검

`DSPM(Data Security Posture Management)`은 클라우드에 흩어진 민감 데이터를 발견 및 분류하고 노출 위험을 평가하는 영역

완성형 DSPM은 대부분 상용 솔루션(AWS Macie, Cyera 등)이므로, 본 실습에서는 DSPM의 핵심 단계인 **데이터 분류**(Presidio)와 **노출 평가**(S3 ACL 점검)를 오픈소스로 조합하여 그 동작 원리를 실습함

### 민감 데이터 업로드

```bash
# 가상환경 활성화 상태 확인
source ~/cnapp-venv/bin/activate

# 개인정보(PII)가 포함된 테스트 파일 생성
cat > ~/customer-data.txt <<EOF
Hong Gildong, email: hong@example.com, card: 4111-1111-1111-1111
Kim Cheolsu, phone: 010-1234-5678, card: 5500-0000-0000-0004
EOF

# 퍼블릭 버킷에 민감 데이터 업로드 (위험 상황 재현)
awslocal s3 cp ~/customer-data.txt s3://public-data/
```

### Presidio를 활용한 데이터 분류

```bash
# Presidio(PII 탐지 엔진) 및 NLP 모델 설치
pip install presidio-analyzer
python -m spacy download en_core_web_lg

# S3에서 객체를 내려받아 PII 탐지·분류
mkdir -p ~/dspm-scan
awslocal s3 cp s3://public-data/customer-data.txt ~/dspm-scan/

# PII 탐지 스크립트 작성
cat > ~/dspm-scan.py <<'PYEOF'
from presidio_analyzer import AnalyzerEngine

analyzer = AnalyzerEngine()
text = open("/home/ubuntu/dspm-scan/customer-data.txt").read()
results = analyzer.analyze(text=text, language="en")

print("=== 탐지된 민감 데이터 (DSPM) ===")
for r in results:
    print(f"유형: {r.entity_type:18} 신뢰도: {r.score:.2f} 값: {text[r.start:r.end]}")
PYEOF

python ~/dspm-scan.py
```

![figure41](./images/figure41.png)

### 데이터 노출 위험 점검

```bash
# 해당 버킷이 퍼블릭으로 노출되어 있는지 확인 (CSPM + DSPM 결합)
awslocal s3api get-bucket-acl --bucket public-data
```

![figure42](./images/figure42.png)

### 참고

- Presidio는 신용카드(CREDIT_CARD), 이메일(EMAIL_ADDRESS), 전화번호(PHONE_NUMBER) 등 PII 유형을 자동으로 탐지 및 분류함
- Presidio는 DSPM 솔루션이 아니라 DSPM의 한 단계인 데이터 분류를 담당하는 엔진임
- DSPM의 핵심은 발견/분류/노출평가의 결합이며, 본 실습은 분류(Presidio) + 노출평가(S3 ACL)를 조합한 것임
- CSPM(버킷이 퍼블릭인가) + DSPM(그 안에 PII가 있는가)을 결합하면 실제 데이터 유출 위험을 정확히 식별 가능함

---
## 9. Trivy 설치 (CWPP)

`Trivy`는 컨테이너 이미지, 파일시스템, IaC 설정 파일의 취약점과 설정 오류를 통합 스캔하는 오픈소스 도구

### apt 저장소 등록 및 설치

```bash
# Trivy 공식 apt 저장소 등록
curl -fsSL https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list

# Trivy 설치
sudo apt-get update
sudo apt-get install -y trivy
```

### 설치 확인

```bash
# 버전 확인
trivy --version
```

---

## 10. Trivy 이미지 취약점 스캔

### 오래된 이미지 스캔

```bash
# 의도적으로 오래된 이미지 스캔 (다수의 CVE 발견됨)
trivy image python:3.4-alpine
```

![figure5](./images/figure5.png)

### 심각도 필터링 및 최신 이미지 비교

```bash
# CRITICAL, HIGH 등급만 필터링
trivy image --severity CRITICAL,HIGH python:3.4-alpine

# 최신 이미지와 비교 (취약점이 현저히 적음)
trivy image python:3.13-alpine
```

![figure6](./images/figure6.png)
![figure7](./images/figure7.png)

### 참고

- 스캔 결과는 패키지별 CVE, 심각도(Severity), 수정 버전(Fixed Version)을 표 형태로 출력함
- 동일 애플리케이션도 베이스 이미지 버전에 따라 취약점 수가 크게 달라짐

---

## 11. Trivy 파일시스템 및 IaC 스캔

### 시크릿 탐지

```bash
# 하드코딩된 시크릿이 포함된 테스트 파일 생성
mkdir -p ~/trivy-test
cat > ~/trivy-test/config.yaml <<EOF
database:
  host: db.internal
  password: "supersecret123"
aws:
  access_key_id: AKIAQYLPMN5HXYZ123AB
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYzAbCdEfGh12
EOF

cat > ~/trivy-test/secrets.sh <<EOF
export AWS_ACCESS_KEY_ID=AKIAQYLPMN5HXYZ123AB
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYzAbCdEfGh12
github_token=ghp_aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
EOF

# 파일시스템 스캔으로 하드코딩된 시크릿 탐지
trivy fs --scanners secret ~/trivy-test
```

![figure8](./images/figure8.png)
![figure9](./images/figure9.png)

### IaC 설정 오류 스캔

```bash
# 취약한 설정의 Dockerfile 생성
cat > ~/trivy-test/Dockerfile <<EOF
FROM ubuntu:latest

# root 사용자로 실행
USER root

# 패스워드를 환경변수에 평문 저장
ENV DB_PASSWORD=supersecret123

# SSH 포트 노출
EXPOSE 22
EOF

# IaC 설정 오류(Misconfiguration) 스캔
trivy config ~/trivy-test
```

![figure10](./images/figure10.png)
![figure11](./images/figure11.png)

### 참고

- `trivy fs --scanners secret`은 사전 정의된 정규식 룰로 `AKIA...`(AWS Access Key), `ghp_...`(GitHub Token) 등 고유 패턴이 명확한 자격 증명을 탐지함
- `password: "supersecret123"`처럼 룰에 등록되지 않은 임의 문자열이나, `AKIA...EXAMPLE` 같은 더미 키는 탐지되지 않음
- `trivy config`는 Dockerfile, Kubernetes YAML, Terraform 등 인프라 설정 파일의 보안 구성 오류를 탐지함
- 위 Dockerfile은 root 실행, 평문 시크릿(ENV), 미고정 베이스 이미지 태그(`:latest`) 등 다수의 취약점을 포함함

## 12. Trivy CI/CD 파이프라인 통합

### 테스트용 파일 구성

GitHub Repository를 생성하여 직접 테스트해보는 것을 추천함

Repository 루트에 아래 두 파일을 추가하고 push하면 GitHub Actions가 자동 실행됨

```dockerfile
# Dockerfile - 취약점이 다수 포함된 구버전 베이스 이미지
FROM python:3.4-alpine
CMD ["python", "--version"]
```

### GitHub Actions 워크플로우 작성

```bash
mkdir -p .github/workflows
cat > .github/workflows/trivy.yml <<EOF
name: Trivy Scan
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 취약한 이미지를 직접 빌드
      - name: Build Docker image
        run: docker build -t my-app:\${{ github.sha }} .

      # 빌드한 이미지를 Trivy로 스캔 (CRITICAL/HIGH 발견 시 실패)
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:\${{ github.sha }}'
          exit-code: '1'              # 취약점 발견 시 빌드 실패 처리
          severity: 'CRITICAL,HIGH'   # CRITICAL, HIGH 등급만 차단
EOF
```

push 후 Repository의 **Actions 탭**에서 워크플로우 실행 결과를 확인함

`python:3.4-alpine`은 CRITICAL/HIGH 취약점이 많아 빌드가 실패(빨간 X) 처리됨

![figure12](./images/figure12.png)

### 참고

- CI/CD는 명령의 종료 코드(exit code)로 빌드 성공/실패를 판단함
- 취약점이 없으면 `0`, `CRITICAL/HIGH`가 발견되면 `1`을 반환하여 파이프라인이 중단됨
- GitHub Actions의 `aquasecurity/trivy-action`도 내부적으로 동일한 `--exit-code` 옵션을 사용함
- 취약한 이미지가 운영에 도달하기 전 빌드 단계에서 차단하는 Shift-Left 보안을 구현함
- 베이스 이미지를 `python:3.13-alpine`으로 바꿔 push하면 취약점이 거의 없어 빌드가 성공(초록 체크)함
- push에 대한 스캔은 사후 검사이므로, 실패(빨간 X)하더라도 이미 push된 코드 자체를 되돌리지는 않음

---

## 13. Kubescape 설치 (KSPM)

`Kubescape`는 NSA-CISA, MITRE ATT&CK 등 산업 표준 프레임워크 기반으로 쿠버네티스 클러스터의 설정 오류와 컴플라이언스 준수 여부를 점검하는 KSPM 도구

### 설치 스크립트 실행

```bash
# 설치 스크립트 실행
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash

# PATH 적용 (현재 세션 + 영구 적용)
export PATH=$PATH:$HOME/.kubescape/bin
echo 'export PATH=$PATH:$HOME/.kubescape/bin' >> ~/.bashrc
```

### 설치 확인

```bash
# 버전 확인
kubescape version
```

![figure30](./images/figure30.png)

---

## 14. Kubescape 클러스터 보안 스캔

### 프레임워크 기반 스캔

```bash
# NSA 프레임워크 기반 스캔
kubescape scan framework nsa
```

![figure31](./images/figure31.png)

```bash
# MITRE ATT&CK 프레임워크 기반 스캔
kubescape scan framework mitre
```

![figure32](./images/figure32.png)

```bash
# 전체 프레임워크 스캔 및 요약 결과 확인
kubescape scan
```

![figure33](./images/figure33.png)

### 참고

- 스캔 완료 시 프레임워크별 Risk Score(%)와 실패한 컨트롤(Failed Controls) 개수를 요약 출력함
- Risk가 높은 항목(High Severity)부터 우선적으로 조치함

---

## 15. Kubescape 매니페스트 점검 및 개선

### 취약한 매니페스트 스캔

```bash
# 취약한 설정의 Pod 매니페스트 생성
cat > ~/vuln-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vuln-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    securityContext:
      privileged: true              # 과도한 권한 (위험)
      runAsUser: 0                  # root 실행 (위험)
EOF

# 특정 매니페스트만 스캔
kubescape scan ~/vuln-pod.yaml
```

![figure34](./images/figure34.png)

### 개선된 매니페스트 재스캔

```bash
# 개선된 매니페스트 생성
cat > ~/secure-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      privileged: false
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
EOF

# 재스캔하여 Risk 개수 감소 확인
kubescape scan ~/secure-pod.yaml
```

![figure35](./images/figure35.png)

---

## 16. Kyverno 설치 (정책 관리)

`Kyverno`는 별도 언어 없이 익숙한 YAML 문법으로 정책을 정의하는 Kubernetes 네이티브 정책 엔진

배포 요청을 가로채어 정책에 따라 허용/거부를 결정함 (Admission Control)

### Helm으로 설치

```bash
# Kyverno Helm 저장소 추가
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# Kyverno 설치
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

![figure13](./images/figure13.png)

### 설치 확인

```bash
# Pod가 Running 상태가 될 때까지 확인
kubectl get pods -n kyverno
```

---

## 17. Kyverno 정책 작성 및 적용

### 정책 1: 허용된 레지스트리 이미지 제한

```bash
# 승인된 레지스트리의 이미지만 배포되도록 강제하는 정책
cat > ~/policy-registry.yaml <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-approved-registries
spec:
  validationFailureAction: Enforce    # 위반 시 배포 거부
  rules:
  - name: validate-registries
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "승인된 레지스트리(registry.approved.com)의 이미지만 허용됩니다."
      pattern:
        spec:
          containers:
          - image: "registry.approved.com/*"
EOF

# 정책 적용
kubectl apply -f ~/policy-registry.yaml
```

### 정책 위반/정상 테스트

```bash
# 위반 테스트 - docker.io 이미지 배포 시도 (거부됨)
kubectl run test-nginx --image=nginx:latest

# 정상 테스트 - 허용된 레지스트리 이미지 (통과됨)
kubectl run test-ok --image=registry.approved.com/nginx:latest --dry-run=server

# 정책 삭제
kubectl delete clusterpolicy require-approved-registries
```

![figure14](./images/figure14.png)

### 참고

- `validationFailureAction: Enforce`는 정책 위반 시 배포를 차단함
- `Audit`으로 설정하면 차단 없이 위반 사항만 기록함

---

## 18. Kyverno 리소스 제한 정책

### 정책 2: 컨테이너 리소스 limits 강제

```bash
# CPU/메모리 limits가 없으면 배포를 거부하는 정책
cat > ~/policy-resources.yaml <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: validate-resources
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "모든 컨테이너는 CPU/메모리 limits를 정의해야 합니다."
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
EOF

# 정책 적용
kubectl apply -f ~/policy-resources.yaml
```

### 정책 위반/정상 테스트 (Enforce 모드)

```bash
# 위반 테스트 - limits 없는 Pod 생성 시도 (거부됨)
kubectl run no-limits --image=nginx:latest

# 정상 테스트 - limits를 정의한 Pod 생성 (통과됨)
cat > ~/ok-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ok-pod
spec:
  containers:
  - name: app
    image: nginx:latest
    resources:
      limits:
        memory: "128Mi"
        cpu: "250m"
EOF

kubectl apply -f ~/ok-pod.yaml
```

![figure15](./images/figure15.png)

### Audit 모드로 전환 후 리포트 확인

Enforce 모드는 위반 Pod를 즉시 거부하므로 클러스터에 남지 않아 PolicyReport에 기록되지 않음

위반 현황을 리포트로 확인하려면 Audit 모드로 전환함

```bash
# 정책을 Audit 모드로 변경 (위반을 거부하지 않고 기록만 함)
sed -i 's/Enforce/Audit/' ~/policy-resources.yaml
kubectl apply -f ~/policy-resources.yaml

# Audit 모드이므로 limits 없는 Pod가 생성됨 (거부되지 않음)
kubectl run no-limits-audit --image=nginx:latest

# 정책 평가 결과 확인 (OWNER별 PASS/FAIL 집계)
kubectl get ephemeralreports -A
```

![figure16](./images/figure16.png)

### 정책 및 리소스 정리

```bash
# 적용된 정책 목록 확인
kubectl get clusterpolicy

# 정책 및 테스트 Pod 삭제
kubectl delete clusterpolicy require-resource-limits
kubectl delete pod ok-pod no-limits-audit

# Kyverno 삭제
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno
```

### 참고

- **Enforce 모드**: 위반 리소스를 admission 단계에서 즉시 거부함 (사전 차단)
- **Audit 모드**: 위반을 허용하되 PolicyReport에 기록함 (현황 파악)
- Enforce는 거부 메시지로, Audit은 PolicyReport로 정책 동작을 확인하는 것이 각각 자연스러움
- PolicyReport는 admission 평가를 받은 리소스를 기록하므로, 정책이 적용된 상태에서 생성된 Pod만 집계됨

---

## 19. OPA Gatekeeper 설치 (정책 관리)

`OPA Gatekeeper`는 Rego 언어로 정책을 작성하는 범용 정책 엔진

Kyverno와 동일한 Admission Control 기능을 수행하지만, Kubernetes 외 환경에도 적용 가능한 범용성이 특징

### 매니페스트로 설치

```bash
# OPA Gatekeeper 설치
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.16.3/deploy/gatekeeper.yaml
```

### 설치 확인

```bash
# Pod 상태 확인
kubectl get pods -n gatekeeper-system
```

![figure17](./images/figure17.png)

### 참고

- Kyverno와 Gatekeeper를 동시에 활성화하면 정책이 충돌할 수 있음
- 한쪽 실습 후 `kubectl delete`로 정리한 뒤 다른 쪽을 진행하는 것을 권장함

---

## 20. OPA Gatekeeper 정책 작성 및 적용

### ConstraintTemplate 정의 (Rego)

```bash
# Rego 언어로 정책 로직을 정의하는 템플릿 생성
cat > ~/opa-template.yaml <<'EOF'
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: allowedrepos
spec:
  crd:
    spec:
      names:
        kind: AllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package allowedrepos

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not image_allowed(container.image)
        msg := sprintf("허용되지 않은 이미지 레지스트리: %v", [container.image])
      }

      image_allowed(image) {
        startswith(image, input.parameters.repos[_])
      }
EOF

# 템플릿 적용
kubectl apply -f ~/opa-template.yaml
```

### Constraint 적용 및 테스트

```bash
# 템플릿을 실제 정책으로 인스턴스화
cat > ~/opa-constraint.yaml <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRepos
metadata:
  name: prod-repo-is-required
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    repos:
    - "registry.approved.com/"
EOF

# 정책 적용
kubectl apply -f ~/opa-constraint.yaml

# 위반 테스트 - 허용되지 않은 레지스트리 (거부됨)
kubectl run opa-test --image=nginx:latest
```

![figure18](./images/figure18.png)

### 정책 및 Gatekeeper 정리

```bash
# Constraint 및 ConstraintTemplate 삭제
kubectl delete -f ~/opa-constraint.yaml
kubectl delete -f ~/opa-template.yaml

# Gatekeeper 전체 제거 (다음 실습에 영향을 주지 않도록)
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.16.3/deploy/gatekeeper.yaml
```

### 참고

- Constraint를 지워도 ConstraintTemplate이 남아있으면 `AllowedRepos` CRD가 유지되므로 둘 다 삭제함
- 이후 런타임 보안 실습에 영향을 주지 않도록 Gatekeeper도 함께 제거하는 것을 권장함
- **Kyverno**: K8s 전용, YAML 문법으로 진입장벽이 낮음
- **OPA Gatekeeper**: OPA(범용 정책 엔진)를 K8s에 특화시킨 형태로, 동일한 OPA/Rego를 마이크로서비스 API 인가, CI/CD 파이프라인 검증(conftest), IaC 설정 검사 등 K8s 외 환경에도 그대로 적용 가능함

---

## 21. Seccomp 시스템 콜 제한 (런타임 보호)

`Seccomp`는 불필요한 시스템 호출(System Call)을 제한하여 컨테이너의 공격 표면을 줄이는 커널 보안 기능

### Seccomp 프로파일 생성

```bash
# kubelet의 Seccomp 프로파일 디렉토리 생성
sudo mkdir -p /var/lib/kubelet/seccomp/profiles

# 특정 syscall을 차단하는 프로파일 생성
sudo tee /var/lib/kubelet/seccomp/profiles/restrict.json > /dev/null <<EOF
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["chmod", "chown", "mount", "umount2"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
EOF
```

### Seccomp 프로파일 적용

```bash
# Seccomp 프로파일을 적용한 Pod 생성
cat > ~/seccomp-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/restrict.json
  containers:
  - name: app
    image: alpine:latest
    command: ["sleep", "3600"]
EOF

# Pod 생성
kubectl apply -f ~/seccomp-pod.yaml
```

### 차단 동작 확인

```bash
# Pod 내부 접속
kubectl exec -it seccomp-pod -- sh

# 차단된 syscall 실행 시도 (Operation not permitted 발생)
chmod 777 /tmp
chown nobody /tmp

# 허용된 syscall은 정상 동작
ls -la /tmp

# 빠져나옴
exit
```

![figure19](./images/figure19.png)

```bash
# 테스트 Pod 삭제
kubectl delete pod seccomp-pod

# 노드에 배치한 Seccomp 프로파일 삭제
sudo rm /var/lib/kubelet/seccomp/profiles/restrict.json
```

### 참고

- Seccomp는 리눅스 커널에 내장된 기능으로, Kubernetes 없이 Docker(`--security-opt seccomp=`)나 일반 프로세스에도 동일하게 적용 가능함
- `defaultAction: SCMP_ACT_ALLOW`는 기본적으로 모든 syscall을 허용하되, 아래 명시한 syscall만 차단하는 방식임
- 반대로 `defaultAction: SCMP_ACT_ERRNO`로 두면 기본적으로 모든 syscall을 차단하고 명시한 것만 허용하는 화이트리스트 방식이 되며, 실무에서는 이 방식(또는 K8s의 `RuntimeDefault` 프로파일)이 더 안전하여 권장됨
- `SCMP_ACT_ERRNO`로 지정된 syscall은 호출 시 거부되고 오류를 반환함
- 차단 대상으로 지정한 syscall의 의미는 다음과 같음
  - `chmod` — 파일 권한 변경 (`chmod 777 file` 할 때 호출됨)
  - `chown` — 파일 소유자/그룹 변경 (`chown user file`)
  - `mount` — 파일시스템 마운트 (디스크나 디렉토리를 시스템에 붙임)
  - `umount2` — 마운트 해제 (`umount`의 syscall 버전)
- 이 syscall들은 정상적인 애플리케이션은 거의 사용하지 않지만, 권한 상승이나 컨테이너 탈출(escape) 공격에 악용될 수 있어 차단 대상으로 선정함

---

## 22. AppArmor 강제적 접근 제어 (런타임 보호)

`AppArmor`는 MAC(강제적 접근 제어) 정책으로 프로세스의 파일/네트워크 권한을 통제하는 커널 보안 모듈

Seccomp가 syscall 단위 제어라면, AppArmor는 파일 경로/권한 단위 제어임

### AppArmor 상태 확인

```bash
# AppArmor 활성 상태 확인 (Y가 출력되어야 함)
cat /sys/module/apparmor/parameters/enabled

# apparmor 유틸 설치
sudo apt-get install -y apparmor-utils
```

### AppArmor 프로파일 생성

```bash
# 특정 파일 쓰기를 차단하는 프로파일 생성
sudo tee /etc/apparmor.d/k8s-deny-write > /dev/null <<EOF
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,                          # 기본적으로 파일 접근 허용
  deny /etc/** w,                # /etc 하위 쓰기 차단
  deny /root/** w,               # /root 하위 쓰기 차단
}
EOF

# 프로파일 로드
sudo apparmor_parser -r -W /etc/apparmor.d/k8s-deny-write

# 로드된 프로파일 확인
sudo aa-status | grep k8s-deny-write
```

![figure20](./images/figure20.png)

### AppArmor 프로파일 적용

```bash
# K8s 1.29는 annotation 방식으로 프로파일 지정
cat > ~/apparmor-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write 
    # 여기의 kubernetes.io/app과 아래의 containers의 name이 일치해야 함
spec:
  containers:
  - name: app   # 컨테이너 이름
    image: alpine:latest
    command: ["sleep", "3600"]
EOF

# Pod 생성
kubectl apply -f ~/apparmor-pod.yaml
```

### 차단 동작 확인

```bash
# Pod 내부 접속
kubectl exec -it apparmor-pod -- sh

# 차단된 경로 쓰기 시도 (Permission denied 발생)
echo "test" > /etc/test.txt
echo "test" > /root/test.txt

# 허용된 경로 쓰기는 정상 동작
echo "test" > /tmp/test.txt
cat /tmp/test.txt

# 빠져나옴
exit
```

![figure21](./images/figure21.png)

```bash
# 테스트 Pod 삭제
kubectl delete pod apparmor-pod

# 노드에 로드한 AppArmor 프로파일 제거
sudo apparmor_parser -R /etc/apparmor.d/k8s-deny-write
sudo rm /etc/apparmor.d/k8s-deny-write
```

### 참고

- AppArmor는 Ubuntu에 기본 탑재/활성화되어 있으며, Kubernetes 없이 Docker(`--security-opt apparmor=`)나 일반 프로세스(`aa-exec`)에도 동일하게 적용 가능함
- Seccomp가 syscall 단위 제어라면, AppArmor는 **파일 경로별로 어떤 권한(읽기 r / 쓰기 w / 실행 x)을 허용/차단할지** 제어함
- `deny /etc/** w`는 `/etc` 하위 모든 파일에 대한 쓰기(w)만 차단한다는 의미이며, 읽기는 여전히 허용됨
- annotation 키의 `app` 부분은 프로파일을 적용할 대상 컨테이너 이름과 일치해야 함

---

## 23. Falco 설치 (런타임 탐지)

`Falco`는 eBPF로 시스템 콜을 실시간 분석하여 비정상적인 컨테이너 행위를 탐지하는 런타임 보안 도구

시그니처 기반이 아닌 행위 기반(Behavioral) 탐지로 제로 데이 위협까지 대응함

### Helm으로 설치

```bash
# Falco Helm 저장소 추가
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

# modern eBPF 드라이버로 Falco 설치
helm install falco falcosecurity/falco \
  --namespace falco --create-namespace \
  --set driver.kind=modern_ebpf \
  --set tty=true
```

![figure22](./images/figure22.png)

### 설치 확인

```bash
# DaemonSet Pod가 Running 상태가 될 때까지 확인
kubectl get pods -n falco
```

### 참고

- Falco는 DaemonSet으로 배포되어 노드에서 시스템 콜을 모니터링함
- `modern_ebpf` 드라이버는 커널 5.8+ 에서 동작하며 별도 커널 모듈 빌드가 불필요함

---

## 24. Falco 런타임 위협 탐지

### 쉘 실행 탐지

```bash
# 테스트용 Pod 생성
kubectl run falco-test --image=nginx:latest
```

```bash
# 터미널 1: Falco 로그 실시간 모니터링
kubectl logs -f -n falco -l app.kubernetes.io/name=falco
```

```bash
# 터미널 2: 컨테이너 내부 쉘 실행 (Falco가 탐지함)
kubectl exec -it falco-test -- bash
```

![figure23](./images/figure23.png)

### 민감 파일 접근 탐지

```bash
# 컨테이너 내부에서 민감 파일 읽기 시도 (Falco가 탐지함)
kubectl exec -it falco-test -- cat /etc/shadow
```

![figure24](./images/figure24.png)

### 참고

- 쉘 실행 시 `A shell was spawned in a container with an attached terminal` 알림이 Notice 레벨로 출력됨
- 민감 파일(`/etc/shadow`) 접근 시 `Sensitive file opened for reading by non-trusted program` 경고가 Warning 레벨로 출력됨
- 각 이벤트는 `user`, `process`, `command`, `container_name`, `k8s_pod_name` 등 상세 정보를 함께 기록하여 어떤 컨테이너에서 무슨 행위가 발생했는지 추적 가능함

---

## 25. Falco 커스텀 규칙 작성

### 사용자 정의 탐지 규칙 작성

```bash
# 커스텀 탐지 규칙 생성
cat > ~/falco-custom-rules.yaml <<EOF
- rule: MY CUSTOM etc write test
  desc: /etc 하위 쓰기 탐지
  condition: >
    open_write and container
    and fd.name startswith /etc
  output: >
    [CUSTOM-RULE-WORKS] etc write detected
    (file=%fd.name command=%proc.cmdline container=%container.name)
  priority: WARNING
  tags: [filesystem, container]
EOF
```

### 커스텀 규칙을 Falco에 적용

```bash
# Helm으로 커스텀 규칙 파일을 Falco에 주입하여 재배포
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --reuse-values \
  --set-file "customRules.custom-rules\.yaml=$HOME/falco-custom-rules.yaml"

# Falco Pod가 재시작되어 Running 상태가 될 때까지 대기
kubectl get pods -n falco -w
```

### 커스텀 규칙 동작 확인

```bash
# 터미널 1: Falco 로그 모니터링
kubectl logs -f -n falco -l app.kubernetes.io/name=falco
```

```bash
# 터미널 2: 컨테이너 내부에서 /etc 하위 파일 쓰기 시도
kubectl exec -it falco-test -- sh -c "echo test > /etc/custom-test.txt"
```

![figure25](./images/figure25.png)

### 참고

- ConfigMap 생성만으로는 규칙이 적용되지 않으며, Falco가 해당 규칙 파일을 읽도록 재배포해야 함
- `--set-file customRules.<파일명>=<경로>`는 로컬 규칙 파일을 Falco 설정에 주입하는 Helm 옵션임
- 커스텀 규칙 작성 시 핵심 필드
  - `condition`: 탐지 조건 (어떤 행위를 위협으로 볼 것인가)
  - `output`: 알림 메시지 (어떤 정보를 기록할 것인가)
  - `priority`: 심각도 (EMERGENCY ~ DEBUG)
- 터미널 2에서 `/etc` 하위 파일 쓰기 시 터미널 1에 `[CUSTOM-RULE-WORKS] etc write detected` 경고가 출력됨

---

## 26. Falcosidekick 알림 라우팅

### Rancher local-path-provisioner 설치

kubeadm 기본 클러스터에는 StorageClass가 없어 PVC가 바인딩되지 않음

Falcosidekick UI의 알림 저장용 Redis가 영구 볼륨을 요구하므로, 동적 볼륨 프로비저너를 먼저 설치함

```bash
# Rancher local-path-provisioner 설치 (단일 노드용 동적 볼륨 프로비저너)
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# 기본 StorageClass로 지정 (PVC가 자동으로 이 클래스를 쓰도록)
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

```bash
# provisioner Pod가 Running 인지
kubectl get pods -n local-path-storage

# StorageClass가 default로 잡혔는지 (이름 옆에 default 표시)
kubectl get storageclass
```

![figure26](./images/figure26.png)

### Falcosidekick 및 Web UI 활성화

```bash
# Falcosidekick과 Web UI를 활성화하여 재배포
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --reuse-values \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true

# Redis PVC가 Bound, 모든 Pod가 Running 되는지 확인
kubectl get pvc -n falco
kubectl get pods -n falco

# Falcosidekick Web UI 포트포워딩 (외부 접속 허용)
kubectl port-forward -n falco --address 0.0.0.0 svc/falco-falcosidekick-ui 2802:2802
```

브라우저에서 `http://<NODE_IP>:2802` 접속 (기본 계정: `admin / admin`)

```bash
# NODE_IP 확인
hostname -I | awk '{print $1}'
```

![figure27](./images/figure27.png)

### 알림 발생 및 대시보드 확인

```bash
# 컨테이너 내부에서 탐지 이벤트 발생 (쉘 실행, 민감 파일 접근 등)
kubectl exec -it falco-test -- sh -c "echo test > /etc/custom-test.txt"
kubectl exec -it falco-test -- cat /etc/shadow
```

![figure28](./images/figure28.png)
![figure29](./images/figure29.png)

### 참고

- `local-path` StorageClass는 `WaitForFirstConsumer` 모드이므로, PVC는 이를 사용하는 Pod가 스케줄될 때 바인딩됨
- SOLID CLOUD 등 외부 VM에서 실습하는 경우, 브라우저 접속을 위해 보안 그룹에서 2802 포트를 개방해야 함
- 탐지된 보안 알림을 대시보드에서 시각적으로 확인 가능하며, Redis 덕분에 과거 알림 이력도 조회됨
- 운영 환경에서는 Slack, Teams, Webhook 등으로 알림을 라우팅하여 침해 사고 대응(IR)을 자동화함

---

## Q & A

박찬욱  
cupark@dankook.ac.kr  

남재현  
namjh@dankook.ac.kr  

## Networked Systems and Security Lab (BoanLab) @ DKU
<img src="../images/boanlab_logo.svg" width="25%"/>