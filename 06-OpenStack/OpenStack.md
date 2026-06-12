# 클라우드 가상화 기술

## 06. 오픈스택 설치 실습

> 본 실습은 `Ubuntu 24.04` 환경을 기준으로 합니다.

OpenStack을 패키지 기반으로 직접 설치하여 단일 노드 클라우드 환경을 구축합니다. 가상 머신 1대를 준비해 단일 노드 환경에 OpenStack 패키지를 차례대로 설치하며, 각 핵심 컴포넌트가 어떻게 연동되는지 직접 확인합니다.

- 가상 머신 1대를 준비하여 단일 노드 환경에 OpenStack 패키지 설치
- OpenStack 릴리스 : `2024.1 Caracal` (Ubuntu 24.04 기본 제공)
- 설치 대상 : `Keystone` → `Glance` → `Placement` → `Nova` → `Neutron` → `Horizon` → `Skyline`

---

## 1. VM 사양 및 사전 준비

OpenStack을 올릴 토대가 될 가상 머신을 마련하는 단계입니다. `SOLID CLOUD` 환경에서 가상 머신 1대를 준비하고, 외부와 격리된 전용 네트워크를 만든 뒤 **포트 포워딩 규칙**까지 구성하여 외부에서 접속 가능한 단일 노드 환경을 갖춥니다.

### 1-1. VM 사양

| 항목 | 권장 사양 |
|------|----------|
| **OS** | Ubuntu 24.04 LTS (Noble) |
| **vCPU** | 8 core (`XLarge`) |
| **RAM** | 16 GB |
| **Disk** | 100 GB (`Override root disk offering → Custom → 100GB`) |
| **Network** | Isolated Network 사용 |

### 1-2. VPN 프로파일 발급

```bash
# 아래 주소에서 OVPN 프로파일 다운로드
dku.kloud.zone/vpn
```

![figure1](./images/figure1.png)

### 1-3. Isolated Network 생성

SOLID CLOUD의 `Network → Guest Networks` 메뉴에서 `Add Network` 클릭

![figure2](./images/figure2.png)

네트워크 이름(예: `isolated-[식별자]`) 입력, `Network Offering`에 `Source Nat service enabled` 확인 후 생성

![figure3](./images/figure3.png)

### 1-4. VM 생성

`Compute Offering`에서 `XLarge` 선택, `Override root disk offering`에서 `Custom` 선택 후 `100GB`로 설정, `Network`에서 생성한 Isolated Network 선택

![figure4](./images/figure4.png)
![figure5](./images/figure5.png)
![figure6](./images/figure6.png)

### 1-5. Egress Rule 설정

VM 생성 후 `Network → Guest Networks` 메뉴에서 생성한 네트워크 클릭

![figure7](./images/figure7.png)

`Egress rules` 탭에서 `Source/Destination CIDR`을 `0.0.0.0/0`으로 설정하고, `Protocol`을 `ALL`로 설정하여 모든 외부 통신 허용

![figure8](./images/figure8.png)

### 1-6. Firewall 규칙 설정

`Public IP addresses` 탭에서 IP 클릭

![figure9](./images/figure9.png)

`Firewall` 탭에서 `Source CIDR`을 `10.0.0.0/16`으로 설정 후 `TCP` 프로토콜에 대해 `22, 80, 443, 9999` 포트 허용

![figure10](./images/figure10.png)
![figure11](./images/figure11.png)

### 1-7. 포트 포워딩 설정

`Port forwarding` 탭에서 `Private Port`와 `Public Port`를 같게 설정하고, `Protocol`을 `TCP`로 설정하여 `22, 80, 443, 9999` 포트 각각에 대해 포트 포워딩 규칙 추가 (생성한 VM에 적용)

![figure12](./images/figure12.png)
![figure13](./images/figure13.png)
![figure14](./images/figure14.png)

### 1-8. SSH 접속 테스트

`Public IP` 주소로 SSH 접속 확인

![figure15](./images/figure15.png)

### 참고
- 가상 머신 위에서 다시 가상 머신을 실행하므로 **XLarge 사양 필수**
- 기존 `10.0.0.0/16` 대역이 아닌 **격리된 네트워크 사용**
- 격리 네트워크는 SOLID CLOUD의 `Isolated Network` 기능으로 생성
- **기존에 사용하던 `10.0.0.0/16` 대역과 `Isolated Network`(`192.168.0.0/24`)를 헷갈리지 않도록 주의**
- 포트별 용도
  - `22` : SSH 접속
  - `80` : Horizon 대시보드
  - `443` : HTTPS (선택)
  - `9999` : Skyline 대시보드

---

## 2. 호스트 네임 및 hosts 설정

`OpenStack 서비스 간 통신`을 위해 호스트 네임과 `/etc/hosts` 파일 설정

```bash
# 호스트네임 변경
sudo hostnamectl set-hostname openstack

# 호스트네임 반영
exec bash
```

```bash
# /etc/hosts 파일 편집
sudo vi /etc/hosts

# /etc/hosts에 추가 (xx.xx.xx.xx 부분을 현재 VM의 실제 IP로 변경)
127.0.0.1 localhost
xx.xx.xx.xx openstack
xx.xx.xx.xx controller
```
![figure16](./images/figure16.png)
![figure17](./images/figure17.png)

### 2-3. 호스트네임 적용 확인

```bash
# 호스트네임 확인
hostname

# /etc/hosts 매핑 확인 (자기 자신으로 ping)
ping -c 2 openstack
ping -c 2 controller
```

![figure18](./images/figure18.png)

### 참고
- `hostnamectl set-hostname` 명령은 시스템 호스트네임을 **즉시 변경**하지만 현재 셸 프롬프트에는 반영되지 않음
- `exec bash`로 새 셸을 시작하거나 SSH 재접속 시 프롬프트가 `username@openstack:~$` 형태로 표시됨
- **재부팅 불필요** — `/etc/hostname`이 영속적으로 변경되어 다음 부팅에도 유지됨
- OpenStack 서비스 간 통신에서 `openstack`, `controller` 호스트명을 사용
- IP 주소가 변경되면 `/etc/hosts`를 다시 수정하고 모든 OpenStack 서비스 재시작 필요

---

## 3. 방화벽 해제 및 패키지 업데이트

OpenStack 서비스들이 사용하는 다수의 포트가 로컬 방화벽에 막히지 않도록 `ufw`를 비활성화하고, 설치할 패키지의 최신 정보를 받아오기 위해 패키지 인덱스를 갱신합니다.

```bash
# 방화벽 비활성화
sudo ufw disable

# 패키지 인덱스 업데이트
sudo apt-get update
```

### 참고
- Ubuntu 24.04는 OpenStack `2024.1 Caracal`을 **기본 저장소에 포함**하므로 별도 UCA 추가 불필요
- 다른 릴리스(Epoxy, Dalmatian 등) 사용 시 `sudo add-apt-repository cloud-archive:<릴리스명>` 으로 추가

---

## 4. Chrony 설치 (시간 동기화)

`Chrony`는 시스템 시간을 정확하게 유지하기 위한 시간 동기화 도구

```bash
# chrony 설치
sudo apt install chrony -y

# 시간 동기화 서버 설정
echo "server openstack iburst" | sudo tee -a /etc/chrony/chrony.conf > /dev/null

# chrony 서비스 활성화
sudo systemctl enable --now chrony
```

![figure19](./images/figure19.png)

### 참고
- OpenStack은 **토큰 만료, 인증서 검증, 분산 트랜잭션**에서 시간 동기화가 필수
- 다중 노드 환경에서는 모든 노드 간 시간이 일치해야 함

---

## 5. MariaDB 설치 및 설정

`MariaDB`는 OpenStack의 모든 서비스가 메타데이터 저장에 사용하는 관계형 데이터베이스

```bash
# MariaDB 및 pymysql 설치
sudo apt install mariadb-server python3-pymysql -y
```

```bash
# OpenStack용 MariaDB 설정 파일 작성
sudo vi /etc/mysql/mariadb.conf.d/99-openstack.cnf
```

```ini
# /etc/mysql/mariadb.conf.d/99-openstack.cnf
[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

```bash
# MariaDB 서비스 활성화
sudo systemctl enable --now mariadb
```

![figure20](./images/figure20.png)

### 참고
- `bind-address = 0.0.0.0` : 외부 노드 접속 허용 (다중 노드 환경 대비)
- `max_connections = 4096` : OpenStack 다수 서비스 동시 접속 대응
- `innodb_file_per_table` : 테이블별로 파일 분리하여 관리 용이성 확보

---

## 6. RabbitMQ 설치 및 구성

`RabbitMQ`는 OpenStack 서비스 간 비동기 메시지 큐 역할을 수행

```bash
# RabbitMQ 서버 설치
sudo apt install rabbitmq-server -y

# openstack 계정 추가 ([RABBITMQ_PASSWORD]는 자신만의 비밀번호로 대체)
sudo rabbitmqctl add_user openstack [RABBITMQ_PASSWORD]

# 모든 권한 부여
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

![figure21](./images/figure21.png)

### 참고
- Nova, Neutron 등 분산 서비스가 모두 RabbitMQ를 통해 통신
- `[RABBITMQ_PASSWORD]`는 이후 Nova/Neutron 설정에서 동일하게 사용됨

---

## 7. Memcached 설치 및 설정

`Memcached`는 Keystone 토큰 검증 결과를 캐싱하여 데이터베이스 I/O 부하를 감소시킴

```bash
# Memcached 설치
sudo apt install memcached python3-memcache -y

# 외부 접근 허용 (127.0.0.1 → 0.0.0.0)
sudo sed -i 's/-l 127.0.0.1/-l 0.0.0.0/' /etc/memcached.conf

# 설정 변경 확인
cat /etc/memcached.conf | grep -E "^-l|-p"

# 서비스 재시작
sudo systemctl restart memcached
```

![figure22](./images/figure22.png)

### 참고
- 다중 노드 환경에서는 모든 노드의 Keystone이 **같은 Memcached 인스턴스를 공유**
- 토큰 검증 캐싱으로 인증 속도 대폭 향상

---

## 8. Keystone 설치 및 구성

`Keystone`은 OpenStack의 **인증/인가 서비스**로, 모든 서비스의 출입을 관리하는 핵심 구성요소

### 8-1. Keystone 데이터베이스 생성

> ⚠️ **주의**: 아래 명령어에서 `[KEYSTONE_PW]` 부분은 **본인이 정한 실제 비밀번호로 치환**해야 함. 대괄호째 그대로 복사하면 비밀번호가 `[KEYSTONE_PW]` 문자열 자체로 등록되어 이후 모든 단계에서 인증 실패. 본 자료에서는 예시로 `[KEYSTONE_PW], boan`을 사용

```bash
# MariaDB 접속 (비밀번호 설정 안 했으므로 ENTER 키 입력)
sudo mysql -u root -p
```

```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY '[KEYSTONE_PW]';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '[KEYSTONE_PW]';
EXIT;
```

![figure23](./images/figure23.png)

### 8-2. Keystone 패키지 설치

```bash
sudo apt install keystone apache2 libapache2-mod-wsgi-py3 python3-openstackclient -y
```

### 8-3. Keystone 설정 파일 수정

```bash
sudo vi /etc/keystone/keystone.conf
```

```ini
# 기존 [database], [token] 섹션에 추가
# [KEYSTONE_PW]는 자신만의 비밀번호로 대체
# 695-696 라인의 [database]에서 수정
[database]
connection = mysql+pymysql://keystone:[KEYSTONE_PW]@localhost/keystone

# 2576-2577 라인의 [token]에 추가
[token]
provider = fernet
```

### 8-4. Keystone DB 동기화 및 키 초기화

```bash
# DB 스키마 생성
sudo su -s /bin/sh -c "keystone-manage db_sync" keystone

# Fernet 키 초기화
sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

![figure26](./images/figure26.png)

### 8-5. Keystone Bootstrap

Bootstrap은 관리자(admin) 계정과 기본 리전, Identity 엔드포인트를 한 번에 등록해 Keystone을 사용 가능한 초기 상태로 만드는 명령입니다.

```bash
# 관리자 계정 및 엔드포인트 자동 생성
# [KEYSTONE_PW]는 자신만의 비밀번호로 대체 (위에서 설정한대로, 이 비밀번호는 이후 Keystone 인증 시 admin 계정의 비밀번호로 사용)
sudo keystone-manage bootstrap --bootstrap-password [KEYSTONE_PW] \
  --bootstrap-admin-url http://localhost:5000/v3/ \
  --bootstrap-internal-url http://localhost:5000/v3/ \
  --bootstrap-public-url http://localhost:5000/v3/ \
  --bootstrap-region-id RegionOne
```

![figure27](./images/figure27.png)

### 8-6. Keystone 실행을 위해 Apache 설정 및 재시작

```bash
echo "ServerName localhost" | sudo tee -a /etc/apache2/apache2.conf > /dev/null
sudo systemctl enable apache2
sudo systemctl restart apache2
```

![figure28](./images/figure28.png)

### 8-7. 인증 파일 생성

```bash
# Keystone 인증 정보 파일 생성 (admin-openrc)
# [KEYSTONE_PW]는 Bootstrap에서 설정한 관리자 비밀번호로 대체
cat <<EOF > ~/admin-openrc
export OS_USERNAME=admin
export OS_PASSWORD=[KEYSTONE_PW]
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://localhost:5000/v3
export OS_IDENTITY_API_VERSION=3
EOF
```

![figure29](./images/figure29.png)

### 8-8. 토큰 발급 테스트

```bash
# 인증 정보 로드 후 토큰 발급
source ~/admin-openrc
openstack token issue
```

![figure30](./images/figure30.png)

### 참고
- 토큰이 정상 발급되면 `Keystone 설치 완료`
- `[KEYSTONE_PW]`는 **동일한 비밀번호**를 모든 설정 파일에서 일관되게 사용
- Bootstrap은 admin 계정과 RegionOne 리전을 자동 등록하는 초기화 명령

---

## 9. Glance 설치 및 구성

`Glance`는 **VM 부팅용 디스크 이미지**를 검색, 등록, 조회하는 이미지 서비스

### 9-1. Glance 데이터베이스 생성

> ⚠️ **주의**: 아래 명령어에서 `[GLANCE_PW]` 부분은 **본인이 정한 실제 비밀번호로 치환**해야 함. 대괄호째 그대로 복사하면 비밀번호가 `[GLANCE_PW]` 문자열 자체로 등록되어 이후 모든 단계에서 인증 실패. 본 자료에서는 예시로 `[GLANCE_PW], boan`을 사용

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY '[GLANCE_PW]';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '[GLANCE_PW]';
EXIT;
```

![figure31](./images/figure31.png)

### 9-2. Glance 서비스 등록

```bash
# service 프로젝트 생성
openstack project create --domain Default --description "Service Project" service

# glance 사용자 및 권한 설정
openstack user create --domain Default --password [GLANCE_PW] glance
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
```

![figure32](./images/figure32.png)
![figure33](./images/figure33.png)

### 9-3. Glance API 엔드포인트 등록

```bash
openstack endpoint create --region RegionOne image public http://localhost:9292
openstack endpoint create --region RegionOne image internal http://localhost:9292
openstack endpoint create --region RegionOne image admin http://localhost:9292
```

![figure34](./images/figure34.png)
![figure35](./images/figure35.png)
![figure36](./images/figure36.png)

### 9-4. Glance 패키지 설치

```bash
sudo apt install glance -y
```

### 9-5. Glance 설정 파일 수정

```bash
sudo vi /etc/glance/glance-api.conf
```

```ini
# /etc/glance/glance-api.conf
# 1767-1768 라인의 [database]에서 수정
[database]
connection = mysql+pymysql://glance:[GLANCE_PW]@localhost/glance

# 3165-3168 라인의 [glance_store] 섹션을 아래 내용으로 교체
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

# 5036-5045 라인의 [keystone_authtoken] 섹션을 아래 내용으로 교체
[keystone_authtoken]
www_authenticate_uri = http://localhost:5000
auth_url = http://localhost:5000
memcached_servers = localhost:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = [GLANCE_PW]

# 5935-5936 라인의 [paste_deploy] 섹션을 아래 내용으로 교체
[paste_deploy]
flavor = keystone
```

### 9-6. Glance DB 동기화 및 서비스 시작

```bash
sudo glance-manage db_sync
sudo systemctl enable glance-api
sudo systemctl restart glance-api
```

![figure41](./images/figure41.png)
![figure42](./images/figure42.png)

### 9-7. 테스트용 VM 이미지 다운로드

```bash
# CirrOS 이미지 다운로드
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img -P ~/

# 인증 정보 로드
source ~/admin-openrc
```

### 9-8. Glance에 이미지 등록

```bash
# Glance에 이미지 등록
openstack image create "cirros" \
  --file ~/cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

# 등록 확인
openstack image list
```

![figure43](./images/figure43.png)
![figure44](./images/figure44.png)

### 참고
- `service` 프로젝트는 OpenStack 내부 서비스 계정용 **공용 프로젝트**
- 각 서비스(Glance, Nova, Neutron 등)는 모두 `service` 프로젝트의 admin 권한으로 동작
- Glance의 기본 백엔드는 **로컬 파일시스템** (`/var/lib/glance/images/`)

---

## 10. Placement 설치 및 구성

`Placement`는 클라우드 내 **자원 인벤토리와 사용량을 추적**하는 API 서비스

### 10-1. Placement 데이터베이스 생성

> ⚠️ **주의**: 아래 명령어에서 `[PLACEMENT_PW]` 부분은 **본인이 정한 실제 비밀번호로 치환**해야 함. 대괄호째 그대로 복사하면 비밀번호가 `[PLACEMENT_PW]` 문자열 자체로 등록되어 이후 모든 단계에서 인증 실패. 본 자료에서는 예시로 `[PLACEMENT_PW], boan`을 사용

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY '[PLACEMENT_PW]';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY '[PLACEMENT_PW]';
EXIT;
```

![figure45](./images/figure45.png)

### 10-2. Placement 서비스 등록

```bash
openstack user create --domain Default --password [PLACEMENT_PW] placement
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
```

![figure46](./images/figure46.png)
![figure47](./images/figure47.png)

### 10-3. Placement API 엔드포인트 생성

```bash
openstack endpoint create --region RegionOne placement public http://localhost:8778
openstack endpoint create --region RegionOne placement internal http://localhost:8778
openstack endpoint create --region RegionOne placement admin http://localhost:8778
```

![figure48](./images/figure48.png)
![figure49](./images/figure49.png)
![figure50](./images/figure50.png)

### 10-4. Placement 패키지 설치

```bash
sudo apt install placement-api -y
```

### 10-5. Placement 설정 파일 수정

```bash
sudo vi /etc/placement/placement.conf
```

```ini
# /etc/placement/placement.conf
# 194-195 라인의 [api]에서 수정
[api]
auth_strategy = keystone

# 519-520 라인의 [database]에서 수정
[placement_database]
connection = mysql+pymysql://placement:[PLACEMENT_PW]@localhost/placement

[keystone_authtoken]
auth_url = http://openstack:5000/v3
memcached_servers = openstack:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = [PLACEMENT_PW]
```

### 10-6. Placement DB 초기화 및 동작 확인

```bash
sudo su -s /bin/sh -c "placement-manage db sync" placement
sudo service apache2 restart
sudo placement-status upgrade check
```

![figure54](./images/figure54.png)

### 참고
- `Nova 스케줄러`가 VM 배치 시 Placement에 자원 인벤토리를 조회
- `placement-status upgrade check` 결과가 모두 `Success`이면 정상 동작

---

## 11. Nova 설치 및 구성

`Nova`는 OpenStack의 핵심 **컴퓨팅 서비스**로, VM 인스턴스의 생성/삭제/관리를 담당

### 11-1. Nova 데이터베이스 생성

`nova_api`, `nova`, `nova_cell0` **3개 데이터베이스 필요**

> ⚠️ **주의**: 아래 명령어에서 `[NOVA_PW]` 부분은 **본인이 정한 실제 비밀번호로 치환**해야 함. 대괄호째 그대로 복사하면 비밀번호가 `[NOVA_PW]` 문자열 자체로 등록되어 이후 모든 단계에서 인증 실패. 본 자료에서는 예시로 `[NOVA_PW], boan`을 사용

```bash
sudo mysql -u root -p
```

```sql
-- nova_api 데이터베이스
CREATE DATABASE nova_api;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY '[NOVA_PW]';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '[NOVA_PW]';

-- nova 데이터베이스
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY '[NOVA_PW]';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '[NOVA_PW]';

-- nova_cell0 데이터베이스
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY '[NOVA_PW]';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '[NOVA_PW]';
EXIT;
```

![figure55](./images/figure55.png)
![figure56](./images/figure56.png)

### 11-2. Nova 서비스 등록

```bash
openstack user create --domain Default --password [NOVA_PW] nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
```

![figure57](./images/figure57.png)


### 11-3. Nova API 엔드포인트 생성

```bash
openstack endpoint create --region RegionOne compute public http://localhost:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://localhost:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://localhost:8774/v2.1
```

![figure58](./images/figure58.png)
![figure59](./images/figure59.png)
![figure60](./images/figure60.png)

### 11-4. Nova 패키지 설치

```bash
sudo apt install nova-api nova-conductor nova-novncproxy nova-scheduler nova-compute -y
```

### 11-5. Nova 설정 파일 수정

```bash
sudo vi /etc/nova/nova.conf
```

```ini
# /etc/nova/nova.conf
# 1-4 라인의 [DEFAULT] 섹션을 아래 내용으로 교체
[DEFAULT]
log_dir = /var/log/nova
lock_path = /var/lock/nova
state_path = /var/lib/nova
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:[RABBITMQ_PASSWORD]@localhost:5672
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
# [Machine IP] 부분은 현재 VM의 실제 IP로 변경
my_ip = [Machine IP]

# 1103-1104 라인의 [api_database]에서 수정
[api_database]
connection = mysql+pymysql://nova:[NOVA_PW]@localhost/nova_api

# 1884-1885 라인의 [database]에서 수정
[database]
connection = mysql+pymysql://nova:[NOVA_PW]@localhost/nova

# 4767 라인의 [placement] 섹션을 아래 내용으로 교체
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://localhost:5000/v3
username = placement
password = [PLACEMENT_PW]

# 2822 라인의 [keystone_authtoken] 섹션을 아래 내용으로 교체
[keystone_authtoken]
www_authenticate_uri = http://localhost:5000
auth_url = http://localhost:5000
memcached_servers = localhost:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = [NOVA_PW]

# 5258 라인의 [service_user] 섹션을 아래 내용으로 교체
[service_user]
send_service_user_token = true
auth_url = http://localhost:5000/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = [NOVA_PW]

# 5821 라인의 [vnc] 섹션을 아래 내용으로 교체
[vnc]
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://localhost:6080/vnc_auto.html

# 2192 라인의 [glance] 섹션을 아래 내용으로 교체
[glance]
api_servers = http://localhost:9292

# 4014 라인의 [oslo_concurrency] 섹션을 아래 내용으로 교체
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

### 11-6. Nova DB 초기화 및 Cell 생성

```bash
# nova_api DB 동기화
sudo su -s /bin/sh -c "nova-manage api_db sync" nova

# cell0 매핑
sudo su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

# cell1 생성
sudo su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

# nova DB 동기화
sudo su -s /bin/sh -c "nova-manage db sync" nova
```

![figure61](./images/figure61.png)

### 11-7. Nova 서비스 시작

```bash
sudo systemctl enable nova-api nova-scheduler nova-conductor nova-novncproxy nova-compute
sudo systemctl restart nova-api nova-scheduler nova-conductor nova-novncproxy nova-compute
```

![figure62](./images/figure62.png)

### 11-8. Nova 서비스 동작 확인

```bash
source ~/admin-openrc
openstack compute service list
```

![figure63](./images/figure63.png)

### 참고
- Nova는 DB 3개(`nova_api`, `nova`, `nova_cell0`)와 **Cell 구조**를 사용하여 대규모 환경에서도 확장 가능
- `nova-api`, `nova-scheduler`, `nova-conductor`, `nova-novncproxy`, `nova-compute` **5개 서비스**가 모두 정상 동작해야 함
- 서비스 목록에서 `State: up`으로 표시되면 정상

---

## 12. Neutron 설치 및 구성

`Neutron`은 OpenStack 환경에서 **SDN 기반 가상 네트워크**를 프로비저닝하고 관리하는 서비스

### 12-1. Neutron 데이터베이스 생성

> ⚠️ **주의**: 아래 명령어에서 `[NEUTRON_PW]` 부분은 **본인이 정한 실제 비밀번호로 치환**해야 함. 대괄호째 그대로 복사하면 비밀번호가 `[NEUTRON_PW]` 문자열 자체로 등록되어 이후 모든 단계에서 인증 실패. 본 자료에서는 예시로 `[NEUTRON_PW], boan`을 사용

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '[NEUTRON_PW]';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '[NEUTRON_PW]';
EXIT;
```

![figure64](./images/figure64.png)

### 12-2. Neutron 서비스 등록

```bash
source ~/admin-openrc
openstack user create --domain Default --password [NEUTRON_PW] neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
```

![figure65](./images/figure65.png)

### 12-3. Neutron API 엔드포인트 생성

```bash
openstack endpoint create --region RegionOne network public http://localhost:9696
openstack endpoint create --region RegionOne network internal http://localhost:9696
openstack endpoint create --region RegionOne network admin http://localhost:9696
```

![figure66](./images/figure66.png)
![figure67](./images/figure67.png)
![figure68](./images/figure68.png)

### 12-4. Neutron 패키지 설치

```bash
sudo apt install neutron-server neutron-plugin-ml2 \
  neutron-openvswitch-agent neutron-dhcp-agent \
  neutron-metadata-agent openvswitch-switch -y
```

### 12-5. Netplan 설정 (OVS 브리지)

```bash
# 기존 netplan 파일 백업
sudo mv /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.backup

# 새 netplan 파일 작성
sudo vi /etc/netplan/50-cloud-init.yaml
```

```yaml
# /etc/netplan/50-cloud-init.yaml
# <IP-ADDRESS>는 현재 VM의 IP, <GATEWAY>는 게이트웨이 주소로 변경
network:
  version: 2
  renderer: networkd
  ethernets:
    br-ex:
      dhcp4: false
      addresses: [<IP-ADDRESS>/24]
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      routes:
        - to: default
          via: <GATEWAY>
```

![figure69](./images/figure69.png)

### 12-6. OVS 브리지 생성 및 연결

> ⚠️ **주의**: `add-port` 실행 순간 ens3의 IP가 br-ex로 이동하면서 **SSH 연결이 끊김**. `netplan apply`를 SOLID CLOUD VNC 콘솔에서 수행

```bash
# netplan 파일 권한 제한
sudo chmod 600 /etc/netplan/50-cloud-init.yaml

# OVS 브리지 생성 및 NIC 연결 
sudo ovs-vsctl add-br br-ex
sudo ovs-vsctl add-port br-ex ens3

# netplan 적용(VNC 콘솔에서 실행)
sudo netplan apply

# 브리지 구성 확인
sudo ovs-vsctl show

# IP 할당 확인
ip addr show br-ex

# 외부 통신 테스트
ping -c 3 8.8.8.8
```

![figure70](./images/figure70.png)

```bash
# IP 할당이 안되어 통신이 안되는 문제를 방지하기 위해 systemd-run으로 별도 유닛에서 실행
sudo systemd-run --unit=net-setup bash -c '
  ovs-vsctl add-br br-ex
  ovs-vsctl add-port br-ex ens3
  chmod 600 /etc/netplan/50-cloud-init.yaml
  netplan apply
'
# 결과 확인
sudo journalctl -u net-setup -f
```

### 참고
- `add-port br-ex ens3`로 SSH가 끊기는 이유: ens3의 IP/라우팅이 br-ex로 이동하면서 기존 TCP 세션이 깨짐
- `systemd-run --unit=net-setup`: 명령을 systemd가 관리하는 일회성 서비스로 실행하여 셸 종료의 영향을 받지 않음 명령어가 전부 실행되면 바로 SSH 재접속 가능
- VNC 콘솔로도 동일하게 진행 가능하며, SSH 끊김에 대한 부담이 적음

### 12-7. Neutron 설정 파일 수정

```bash
sudo vi /etc/neutron/neutron.conf
```

```ini
# /etc/neutron/neutron.conf
# 1-2 라인의 [DEFAULT] 섹션을 아래 내용으로 교체
[DEFAULT]
core_plugin = ml2
service_plugins =
transport_url = rabbit://openstack:[RABBITMQ_PASSWORD]@localhost:5672
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

# 946-947 라인의 [database] 섹션을 아래 내용으로 교체
[database]
connection = mysql+pymysql://neutron:[NEUTRON_PW]@localhost/neutron

# 1318 라인의 [keystone_authtoken] 섹션을 아래 내용으로 교체
[keystone_authtoken]
www_authenticate_uri = http://localhost:5000
auth_url = http://localhost:5000
memcached_servers = localhost:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = [NEUTRON_PW]

# 1489 라인의 [nova] 섹션을 아래 내용으로 교체
[nova]
auth_url = http://localhost:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = [NOVA_PW]

# 1608 라인의 [oslo_concurrency] 섹션을 아래 내용으로 교체
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

### 12-8. ML2 플러그인 설정

```bash
sudo vi /etc/neutron/plugins/ml2/ml2_conf.ini
```

```ini
# /etc/neutron/plugins/ml2/ml2_conf.ini
# 156 라인의 [ml2] 섹션을 아래 내용으로 교체
[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = openvswitch
extension_drivers = port_security

# 218 라인의 [ml2_type_flat] 섹션을 아래 내용으로 교체
[ml2_type_flat]
flat_networks = provider
```

### 12-9. Open vSwitch Agent 설정

```bash
sudo vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
```

```ini
# /etc/neutron/plugins/ml2/openvswitch_agent.ini
# 303 라인의 [ovs] 섹션을 아래 내용으로 교체
[ovs]
bridge_mappings = provider:br-ex

# 455 라인의 [securitygroup] 섹션을 아래 내용으로 교체
[securitygroup]
enable_security_group = true
firewall_driver = openvswitch
```

### 12-10. DHCP Agent 설정

```bash
sudo vi /etc/neutron/dhcp_agent.ini
```

```ini
# /etc/neutron/dhcp_agent.ini
# 1 라인의 [DEFAULT] 섹션을 아래 내용으로 교체
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

### 12-11. Metadata Agent 설정

```bash
sudo vi /etc/neutron/metadata_agent.ini
```

```ini
# /etc/neutron/metadata_agent.ini
# 1 라인의 [DEFAULT] 섹션을 아래 내용으로 교체
# [META_PW]는 자신만의 비밀번호로 대체
[DEFAULT]
nova_metadata_host = localhost
metadata_proxy_shared_secret = [META_PW]
```

### 12-12. Nova 설정에 Neutron 연동 추가

```bash
sudo vi /etc/nova/nova.conf
```

```ini
# /etc/nova/nova.conf 파일 끝에 [neutron] 섹션 추가
# 3677 라인의 [neutron] 섹션을 아래 내용으로 교체
[neutron]
auth_url = http://localhost:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = [NEUTRON_PW]
service_metadata_proxy = true
metadata_proxy_shared_secret = [META_PW]
```

### 12-13. Neutron DB 초기화

```bash
sudo neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head
```

![figure71](./images/figure71.png)

### 12-14. 서비스 재시작

```bash
# Nova API 재시작 (Neutron 연동 적용)
sudo systemctl restart nova-api

# Neutron 서비스 활성화 및 시작
sudo systemctl enable neutron-server neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent
sudo systemctl restart neutron-server neutron-openvswitch-agent neutron-dhcp-agent neutron-metadata-agent
```

![figure72](./images/figure72.png)

### 12-15. 가상 네트워크 생성

```bash
source ~/admin-openrc

# Provider 네트워크 생성
openstack network create --share --external \
  --provider-physical-network provider \
  --provider-network-type flat provider
```

![figure73](./images/figure73.png)

### 12-16. 서브넷 생성

```bash
# 서브넷 생성 (IP 대역은 환경에 맞게 조정)
openstack subnet create --network provider \
  --allocation-pool start=192.168.0.101,end=192.168.0.199 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.0.1 \
  --subnet-range 192.168.0.0/24 provider
```

![figure74](./images/figure74.png)

### 12-17. Neutron 동작 확인

```bash
openstack network agent list
```

![figure75](./images/figure75.png)

### 참고
- 모든 에이전트(`Open vSwitch agent`, `DHCP agent`, `Metadata agent`)가 `:-)` **(alive) 상태**이면 정상
- Provider 네트워크는 외부 통신용 flat 네트워크로 구성
- IP 대역(`192.168.0.0/24`)은 `isolated network` 네트워크 환경에 맞게 변경

---

## 13. Horizon 설치 및 구성

`Horizon`은 OpenStack의 공식 **웹 대시보드**로, Django 기반의 사용자 인터페이스 제공

### 13-1. Horizon 패키지 설치

```bash
sudo apt install openstack-dashboard -y
```

### 13-2. Horizon 설정 파일 수정

```bash
sudo vi /etc/openstack-dashboard/local_settings.py
```

```python
# /etc/openstack-dashboard/local_settings.py
# 105 라인의 SESSION_ENGINE을 아래 내용으로 교체
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

# 119 라인의 OPENSTACK 관련 설정을 아래 내용으로 교체
OPENSTACK_HOST = "localhost"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/identity/v3" % OPENSTACK_HOST
TIME_ZONE = "Asia/Seoul"

# 405 라인 아래로 추가(제일 아래)
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_fip_topology_check': False,
}
```
### 13-3. Apache 재시작

```bash
sudo systemctl reload apache2
```

### 13-4. Horizon 접속

웹 브라우저에서 `http://[Machine IP]/horizon` 접속

| 항목 | 값 |
|------|-----|
| **도메인** | `Default` |
| **사용자명** | `admin` |
| **비밀번호** | `[KEYSTONE_PW]` |

![figure76](./images/figure76.png)
![figure77](./images/figure77.png)

### 참고
- Horizon은 **Django 기반**의 OpenStack 웹 대시보드
- 로그인 후 인스턴스 / 이미지 / 네트워크 / 보안 그룹 등을 GUI에서 관리 가능

---

## 14. Skyline 설치 및 구성

`Skyline`은 OpenStack의 **차세대 웹 대시보드**로, Vue.js + FastAPI 기반의 SPA 구조

### 14-1. Skyline 데이터베이스 생성

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE skyline DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL PRIVILEGES ON skyline.* TO 'skyline'@'localhost' IDENTIFIED BY '[SKYLINE_PW]';
GRANT ALL PRIVILEGES ON skyline.* TO 'skyline'@'%' IDENTIFIED BY '[SKYLINE_PW]';
EXIT;
```

![figure78](./images/figure78.png)

### 14-2. Skyline 서비스 사용자 등록

```bash
source ~/admin-openrc

# skyline 사용자 생성 및 admin 권한 부여
openstack user create --domain Default --password [SKYLINE_PW] skyline
openstack role add --project service --user skyline admin
```

![figure79](./images/figure79.png)

### 14-3. Docker 설치

Skyline은 Vue.js + FastAPI 기반의 컨테이너화 된 대시보드로 제공되는데, Docker를 사용하지 않으면 직접 vue.js를 빌드해야 해 실행이 복잡해짐. 따라서 Docker 설치 권장

```bash
# 필수 의존성 설치
sudo apt update
sudo apt install -y ca-certificates curl

# Docker 공식 GPG 키 추가
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Docker 공식 저장소 등록
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 패키지 인덱스 업데이트
sudo apt update

# Docker Engine 설치
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Docker 서비스 활성화
sudo systemctl enable --now docker

# 현재 사용자에게 Docker 그룹 권한 부여 
sudo usermod -aG docker $USER

# Docker 그룹 권한 적용
newgrp docker

# Docker 버전 확인
docker --version
```

![figure80](./images/figure80.png)

### 14-4. Skyline 설정 디렉토리 생성

`Skyline` 설정 및 로그를 저장할 디렉토리를 미리 생성

```bash
sudo mkdir -p /etc/skyline /var/log/skyline
```

### 참고
- `/etc/skyline` : 설정 파일 (`skyline.yaml`) 저장
- `/var/log/skyline` : 컨테이너 외부에서 Skyline 로그 확인 시 사용 (마운트 옵션은 제외)
- `/var/lib/skyline`은 **호스트에 생성하지 않음**. Unix 소켓이 호스트 파일시스템과 충돌하므로 컨테이너 내부 경로에만 둠

---

### 14-5. Skyline 설정 파일 작성

```bash
sudo vi /etc/skyline/skyline.yaml
```

```yaml
# /etc/skyline/skyline.yaml
default:
  database_url: mysql+pymysql://skyline:[SKYLINE_PW]@localhost:3306/skyline
  debug: false
  log_dir: /var/log/skyline
  log_file: skyline.log
  secret_key: aCtmgbcUqYUy_HNVg5BDXCaeJgJQzHJXwqbXr0Nmb2o
  access_token_expire: 3600
  access_token_renew: 1800
  cors_allow_origins: []
  session_name: session
  ssl_enabled: false

openstack:
  base_domains:
    - heat_user_domain
  default_region: RegionOne
  enforce_new_defaults: true
  interface_type: public
  keystone_url: http://localhost:5000/v3
  nginx_prefix: /api/openstack
  service_mapping:
    compute: nova
    identity: keystone
    image: glance
    network: neutron
    placement: placement
  sso_enabled: false
  system_admin_roles:
    - admin
    - system_admin
  system_project: service
  system_project_domain: Default
  system_reader_roles:
    - system_reader
  system_user_domain: Default
  system_user_name: skyline
  system_user_password: [SKYLINE_PW]
```

### 참고
- **`database_url`은 반드시 `mysql+pymysql://`로 시작**. `mysql://`만 적으면 컨테이너 안에서 `MySQLdb` 모듈을 찾지 못해 워커 부팅 실패
- `keystone_url`은 `localhost` 사용 가능 (`--net=host` 옵션 덕분에 컨테이너가 호스트 네트워크 공유)
- `system_user_password`는 §14-2에서 생성한 `skyline` 사용자 비밀번호와 **반드시 일치**
- `secret_key`는 실습용 임시 값. 운영 환경에서는 `openssl rand -hex 32`로 새로 생성

---

### 14-6. Skyline DB 초기화 (Bootstrap)

`Skyline` 데이터베이스 스키마를 생성하기 위해 부트스트랩 컨테이너 실행

```bash
# bootstrap 컨테이너 실행 (DB 스키마 생성)
sudo docker run -d --name skyline_bootstrap \
  -e KOLLA_BOOTSTRAP="" \
  -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml \
  --net=host 99cloud/skyline:latest
```



```bash
# 부트스트랩 로그 확인 (10초 대기 후)
sleep 10
sudo docker logs skyline_bootstrap
```

![figure81](./images/figure81.png)

```bash
# bootstrap 완료 후 컨테이너 제거
sudo docker rm -f skyline_bootstrap
```

### 참고
- 로그에 `Running upgrade -> 000, init` + `exit 0`이 출력되면 정상
- Bootstrap 컨테이너는 1회 실행 후 종료되는 일회성 작업이며, 완료 후 반드시 제거
- DB 스키마 생성 실패 시 `skyline.yaml`의 `database_url`과 MariaDB 사용자 권한 재확인

---

### 14-7. Skyline 서비스 컨테이너 실행

```bash
sudo docker run -d --name skyline --restart=always \
  -v /etc/skyline/skyline.yaml:/etc/skyline/skyline.yaml \
  --net=host 99cloud/skyline:latest

# 10초 대기 후 실행 상태 확인
sudo docker ps
curl -I http://localhost:9999
```
![figure82](./images/figure82.png)

### 참고
- `STATUS`가 `Up X seconds` (재시작 없이) 으로 표시되면 정상
- `Restarting` 상태가 반복되면 다음 사항 점검
  - `database_url`이 `mysql+pymysql://`로 시작하는지
  - `skyline` 사용자가 Keystone에 등록되어 있고 비밀번호가 yaml과 일치하는지
  - OpenStack 엔드포인트가 각 서비스별로 정확히 3개씩(public/internal/admin)만 존재하는지
- `--net=host` 옵션은 컨테이너가 호스트의 네트워크 스택을 공유하여 `localhost`로 Keystone에 접근 가능하게 함
- **추가 볼륨 마운트(`/var/lib/skyline` 등) 금지** — Unix 소켓 충돌로 컨테이너 재시작 루프 발생

### 14-8. Skyline 접속

웹 브라우저에서 `http://[Machine IP]:9999` 접속

| 항목 | 값 |
|------|-----|
| **도메인** | `Default` |
| **사용자명** | `admin` |
| **비밀번호** | `[KEYSTONE_PW]` |

![figure83](./images/figure83.png)
![figure84](./images/figure84.png)

### 참고
- Skyline은 **Vue.js + FastAPI 기반**의 차세대 OpenStack 대시보드
- 기본 포트는 `9999`이며, Horizon(`/horizon`)과 **동시 운영 가능**
- 두 대시보드 모두 동일한 OpenStack REST API를 호출하므로, 어느 쪽에서 자원을 생성하든 다른 쪽에서 **즉시 조회 가능**

---

## 15. 설치 완료 확인

```bash
source ~/admin-openrc

# 모든 서비스 카탈로그 확인
openstack catalog list
# Keystone, Glance, Nova, Neutron, Placement 서비스가 모두 등록되어 있어야 함
```

![figure85](./images/figure85.png)

```bash
# 등록된 엔드포인트 확인
openstack endpoint list
```

![figure86](./images/figure86.png)

```bash
# 컴퓨트 노드 동작 확인
openstack compute service list

# 네트워크 에이전트 확인
openstack network agent list

# 등록된 이미지 확인
openstack image list
```

![figure87](./images/figure87.png)
![figure88](./images/figure88.png)

### 참고
- 각 명령어가 모두 정상 응답하면 **OpenStack 핵심 서비스 설치 완료**
- 이후 Horizon 또는 Skyline 대시보드를 통해 VM 생성 / 네트워크 구성 / 이미지 관리 등 실습 진행 가능
- 트러블슈팅 시 각 서비스 로그 위치
  - Keystone : `/var/log/keystone/`
  - Glance : `/var/log/glance/`
  - Nova : `/var/log/nova/`
  - Neutron : `/var/log/neutron/`
  - Apache (Keystone/Placement) : `/var/log/apache2/`
  - Skyline : `/var/log/skyline/` 또는 `sudo docker logs skyline`

---

## Q & A

박찬욱  
cupark@dankook.ac.kr

남재현  
namjh@dankook.ac.kr

## Networked Systems and Security Lab (BoanLab) @ DKU
<img src="../images/boanlab_logo.svg" width="25%"/>
