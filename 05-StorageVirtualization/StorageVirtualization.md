# 클라우드 가상화 기술

## 05. 스토리지 가상화 기술 실습

---

## 1. NFS 설치 및 운영

NFS(Network File System)는 네트워크를 통해 원격 서버의 디렉토리를 로컬처럼 마운트하여 여러 호스트가 파일을 공유할 수 있게 해주는 파일 수준 공유 방식입니다. 본 실습에서는 서버와 클라이언트 역할을 나누어 NFS의 동작을 직접 확인합니다.

- 가상 머신 2개를 준비  
- 하나는 `NFS 서버`, 다른 하나는 `NFS 클라이언트`로 이용  

![figure1](./images/figure1.png)

---

## 2. NFS 서버 설정

먼저 공유를 제공할 NFS 서버를 구성합니다. NFS 서버 패키지를 설치하고, 클라이언트와 공유할 디렉토리를 만든 뒤 적절한 소유자와 권한을 부여합니다.

```bash
# NFS 서버로 이용할 호스트에서 수행

# 패키지 업데이트
sudo apt update

# NFS 서버 설치
sudo apt install -y nfs-kernel-server
```

```bash
# 공유할 디렉토리 생성
sudo mkdir -p /srv/nfs_share

# 디렉토리 소유자를 nobody:nogroup으로 변경
sudo chown nobody:nogroup /srv/nfs_share

# 모든 사용자가 읽기/쓰기/실행 가능하도록 권한 설정
sudo chmod 777 /srv/nfs_share

# 소유자 및 권한 확인
ls -ld /srv/nfs_share
```

![figure2](./images/figure2.png)

### 참고
- `nobody:nogroup`으로 소유자를 설정하는 이유
  - NFS는 클라이언트가 파일에 접근할 때 클라이언트의 UID/GID를 서버에 그대로 전달함
  - 클라이언트마다 UID/GID가 다를 수 있으므로, 특정 사용자에 종속되지 않는 중립적인 계정인 `nobody:nogroup`으로 설정
  - `nobody` : 권한이 거의 없는 시스템 계정으로, 소유권 충돌을 방지하기 위해 사용
  - `nogroup` : 특정 그룹에 속하지 않는 시스템 그룹
- `chmod 777`과 함께 사용하여 어떤 클라이언트든 접근 가능하도록 설정하는 것으로, 실습 환경용 설정임  
  (실무에서는 보안상 권장하지 않음)

- 실무에서의 권한 설정
  - 특정 사용자/그룹만 접근할 수 있도록 소유자와 권한을 제한하여 설정
  ```bash
    # 예: 서버와 클라이언트 모두 동일한 UID(1000)의 사용자가 접근하는 경우
    sudo chown 1000:1000 /srv/nfs_share
    sudo chmod 755 /srv/nfs_share
  ```
  - 서버와 클라이언트의 UID/GID를 일치시키거나, Kerberos 인증(`sec=krb5`)을 사용하여 사용자 인증을 강화함
    - `Kerberos` : MIT에서 개발한 오픈소스 네트워크 인증 프로토콜로, 별도의 인증 서버(KDC)를 통해 사용자를 검증하여 UID 위조를 방지할 수 있음
  - `/etc/exports`에서 `*` 대신 허용할 클라이언트 IP 또는 대역을 명시적으로 지정
  ```bash
    # 예: 특정 IP 대역만 허용
    /srv/nfs_share 192.168.1.0/24(rw,sync,no_subtree_check)
  ```

---

## 3. NFS 공유 설정

어떤 디렉토리를 어떤 클라이언트에게 어떤 권한으로 공개할지는 `/etc/exports` 파일에 정의합니다. 아래 명령으로 공유 디렉토리와 접근 옵션을 등록합니다.

```bash
# /etc/exports 파일에 공유 디렉토리 설정 추가
echo "/srv/nfs_share *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
```

### 참고
- `rw / ro`  
  - `rw` - NFS 서버 디렉토리에 대해 읽기/쓰기 허용
  - `ro` - NFS 서버 디렉토리에 대해 읽기 전용으로 허용  
  
- `sync / async`  
  - `sync` - 클라이언트가 파일을 작성/수정/삭제하면 데이터를 디스크에 기록한 뒤 응답
  - `async` - 클라이언트가 파일을 작성/수정/삭제하면 데이터를 메모리에만 저장한 뒤 응답  
    (성능 향상/데이터 손실 위험)

- `subtree_check / no_subtree_check`  
  - `subtree_check` - NFS 서버가 요청받은 파일이 실제 공유된 하위 디렉토리 내에 있는지 매번 검사  
    (보안 강화, 오버헤드 발생)
  - `no_subtree_check` - 파일의 개별 위치를 매번 엄밀하게 검증하지 않고 파일 시스템 단위로 접근 허용  
    (성능 향상, 연결 안정성 유리)

- `no_subtree_check`를 사용하는 이유  
  - 가상화 환경에서는 디스크 I/O가 물리 서버보다 느릴 수 있으므로 성능 최적화에 유리함  
  - 또한, 가상 머신이 자주 생성/삭제되는 환경에서는 디렉토리 구조가 자주 변경될 수 있으므로 `no_subtree_check`를 사용하여 연결 안정성을 높이는 것이 좋음

---

## 4. NFS 서버 적용

`/etc/exports`에 추가한 설정을 실제로 반영하기 위해 NFS 서비스를 재시작하고, 현재 내보내기(export)된 공유 목록을 확인합니다.

```bash
# NFS 서버 재시작
sudo systemctl restart nfs-kernel-server

# 현재 export 설정 확인
sudo exportfs -v
```

![figure3](./images/figure3.png)

---

## 5. NFS 클라이언트 설정

이제 다른 가상 머신에서 클라이언트를 구성합니다. 클라이언트 패키지를 설치한 뒤, 서버가 공유한 디렉토리를 로컬 디렉토리에 마운트하여 마치 자신의 디렉토리처럼 사용할 수 있게 합니다.

```bash
# NFS 클라이언트로 이용할 호스트에서 수행

# 패키지 업데이트
sudo apt update

# NFS 클라이언트 패키지 설치
sudo apt install -y nfs-common
```

```bash
# 마운트할 디렉토리 생성
sudo mkdir -p /mnt/nfs_share

# NFS 서버의 공유 디렉토리를 클라이언트에 마운트
sudo mount [NFS Server IP]:/srv/nfs_share /mnt/nfs_share

# 마운트 확인
mount | grep nfs

ls -ld /mnt/nfs_share
```

![figure4](./images/figure4.png)

---

## 6. NFS 동작 확인

마운트가 정상적으로 동작하는지 확인하기 위해, 클라이언트에서 공유 디렉토리에 파일을 생성해 봅니다. 생성한 파일은 네트워크를 거쳐 서버 측 디렉토리에 실제로 저장됩니다.

```bash
# 마운트된 디렉토리에 파일 생성 테스트
echo "Storage Virtualization" > /mnt/nfs_share/test.txt
```

![figure5](./images/figure5.png)
![figure6](./images/figure6.png)

---

## 7. NFS 마운트 해제

공유 사용을 마치면 클라이언트에서 마운트를 해제합니다. 마운트를 해제하면 서버에 저장된 파일에는 더 이상 접근할 수 없게 됩니다.

```bash
# NFS 마운트 해제
sudo umount /mnt/nfs_share

# 마운트 해제 확인
mount | grep nfs

ls -l /mnt/nfs_share
```

![figure7](./images/figure7.png)

### 참고
- 생성한 `test.txt` 파일은 `NFS 서버`에 저장되어 있으므로, 마운트를 해제하면 해당 파일에 접근할 수 없음  

- NFS는 네트워크 기반 파일 시스템이므로 서버가 중지되거나 네트워크 연결이 끊기면 마운트된 디렉토리에 접근 시  
 지연 또는 오류 발생 가능   

- NFS는 클라이언트 재부팅 시 자동으로 마운트되지 않음  
  (`/etc/fstab` 파일에 NFS 마운트 설정 추가하여 자동 마운트 가능)  

- `umount` 시 `device is busy` 오류가 발생하면 해당 디렉토리를 사용하는 프로세스가 존재하는 것이므로 종료 후 다시 시도
  (`lsof +D /mnt/nfs_share` 또는 `fuser -m /mnt/nfs_share`로 확인 가능)  

![figure8](./images/figure8.png)

---

## 8. NFS 자동 마운트 설정 (/etc/fstab)

기본적으로 NFS 마운트는 재부팅 시 유지되지 않습니다. 부팅할 때마다 자동으로 마운트되도록 하려면, 마운트 정보를 시스템의 파일시스템 테이블인 `/etc/fstab`에 등록해야 합니다.

재부팅 후에도 NFS가 자동으로 마운트되도록 `/etc/fstab`에 설정을 추가할 수 있음

```bash
# /etc/fstab에 NFS 마운트 설정 추가
echo "[NFS Server IP]:/srv/nfs_share /mnt/nfs_share nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# 설정 확인
cat /etc/fstab
```

```bash
# 재부팅 없이 fstab 기반으로 즉시 마운트 테스트
sudo mount -a

# 마운트 확인
mount | grep nfs
```

### 참고
- `/etc/fstab` 필드 구성: `[장치] [마운트포인트] [파일시스템] [옵션] [dump] [fsck]`
  - `nfs` : 파일시스템 타입
  - `defaults` : rw, suid, exec, auto, nouser, async 옵션을 포함하는 기본값
  - 마지막 두 필드 `0 0` : dump 백업 비활성화, 부팅 시 fsck 검사 비활성화  
    (NFS는 로컬 파일시스템이 아니므로 fsck 대상이 아님)

- `_netdev` 옵션을 추가하면 네트워크가 활성화된 이후에 마운트를 시도하도록 보장함  
  - 이 옵션이 없으면 부팅 시 네트워크 준비 전에 마운트를 시도하여 실패할 수 있음

---

## 9. iSCSI 설치 및 운영

iSCSI(Internet Small Computer Systems Interface)는 SCSI 명령을 TCP/IP 네트워크 위로 전송하여, 원격 저장장치를 마치 로컬에 직접 연결된 디스크처럼 사용할 수 있게 하는 블록 수준 스토리지 프로토콜입니다. 파일 단위로 공유하는 NFS와 달리, 클라이언트는 연결된 블록 장치를 직접 포맷하여 사용합니다. 여기서 저장소를 제공하는 쪽을 `Target`, 이를 연결해 사용하는 쪽을 `Initiator`라고 부릅니다.

- 가상 머신 2개를 준비  
- 하나는 `iSCSI 서버`, 다른 하나는 `iSCSI Initiator`로 이용

![figure9](./images/figure9.png)

---

## 10. iSCSI 서버 설정

저장소를 제공하는 Target 서버를 구성합니다. iSCSI 타겟 서버 패키지(`tgt`)를 설치하고, Initiator에게 제공할 디스크 역할을 할 이미지 파일을 생성합니다.

```bash
# iSCSI 서버로 이용할 호스트에서 수행

# 패키지 업데이트
sudo apt update

# iSCSI 타겟 서버 설치
sudo apt install -y tgt
```

```bash
# iSCSI 디스크 파일을 저장할 디렉토리 생성
sudo mkdir /srv/iscsi_disks

# 1GB 크기의 디스크 이미지 파일 생성
sudo dd if=/dev/zero of=/srv/iscsi_disks/disk01.img bs=1G count=1
```

![figure10](./images/figure10.png)

---

## 11. iSCSI 타겟 설정

앞서 만든 디스크 이미지를 실제 iSCSI 타겟으로 노출하기 위해, 타겟 설정 파일을 작성합니다. 각 타겟은 IQN이라는 전역 고유 식별자로 구분하며, 어떤 Initiator의 접근을 허용할지 IP로 지정합니다.

```bash
# iSCSI 타겟 설정 파일 작성
# [Target IQN]은 iqn.2026-05.kr.ac.dankook:[고유식별자]와 같이 고유하게 설정할 것
# [Initiator IP]는 후에 iSCSI Initiator를 설치할 호스트의 IP 주소로 설정할 것
sudo tee /etc/tgt/conf.d/iscsi-target.conf <<EOF
<target [Target IQN]>
  backing-store /srv/iscsi_disks/disk01.img
  initiator-address [Initiator IP]
</target>
EOF
```

![figure11](./images/figure11.png)

```bash
# iSCSI 서비스 재시작
sudo systemctl restart tgt

# 타겟 설정 확인
sudo tgtadm --mode target --op show
```

![figure12](./images/figure12.png)

### 참고
- `iSCSI의 IQN(iSCSI Qualified Name)`은 고유한 식별자로, 전 세계적으로 중복되지 않아야 함  
  예: `iqn.2026-05.com.boanlab:storage.disk1`
  - `iqn` : iSCSI 규격을 의미하는 고정 문자열 (필수)
  - `2026-05` : 도메인 이름을 역순으로 표현한 날짜 (예: 2026년 5월)
  - `com.boanlab` : 조직의 도메인 이름을 역순으로 표현한 부분 (예: boanlab.com)
  - `storage.disk1` : 해당 타겟을 구분하기 위해 사용자가 정하는 이름

---

## 12. iSCSI CHAP 인증 설정

### CHAP(Challenge Handshake Authentication Protocol)란?

- iSCSI에서 Initiator가 Target에 접속할 때 사용하는 인증 방식
- Target이 Initiator에게 랜덤 Challenge를 보내고, Initiator는 공유된 비밀(password)로 응답값을 생성하여 전송
- 비밀번호 자체가 네트워크에 평문으로 전송되지 않으므로 `initiator-address`만으로 제한하는 것보다 보안성이 높음
- **단방향 CHAP**: Target이 Initiator를 인증 (일반적)
- **상호 CHAP(Mutual CHAP)**: Target과 Initiator가 서로 인증

### CHAP 설정 — iSCSI 서버

```bash
# 기존 타겟 설정 파일에 CHAP 인증 추가
sudo tee /etc/tgt/conf.d/iscsi-target.conf <<EOF
<target [Target IQN]>
  backing-store /srv/iscsi_disks/disk01.img
  initiator-address [Initiator IP]
  incominguser [username] [password]
</target>
EOF

# iSCSI 서비스 재시작
sudo systemctl restart tgt

# 설정 확인
sudo tgtadm --mode target --op show
```

- `incominguser [username] [password]` : Initiator가 Target에 접속할 때 사용하는 자격증명
- password는 12자 이상으로 설정 권장 (tgt 정책)

### CHAP 설정 — iSCSI Initiator

```bash
# CHAP 인증 방식 활성화
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] \
  --op update -n node.session.auth.authmethod -v CHAP

# 사용자 이름 설정
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] \
  --op update -n node.session.auth.username -v [username]

# 비밀번호 설정
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] \
  --op update -n node.session.auth.password -v [password]
```

```bash
# 기존 세션 로그아웃 후 재로그인 (CHAP 적용)
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] --logout
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] --login

# 세션 확인
sudo iscsiadm -m session
```

### 참고
- CHAP 설정 정보는 `/etc/iscsi/nodes/` 디렉토리 하위에 저장됨
- 잘못된 자격증명으로 로그인 시도 시 `Login failed` 오류 발생

---

## 13. iSCSI Initiator 설정

이번에는 저장소를 사용하는 Initiator 호스트를 구성합니다. Initiator 패키지(`open-iscsi`)를 설치하고, 네트워크에서 사용 가능한 타겟을 검색(discovery)한 뒤 해당 타겟에 로그인하여 디스크를 연결합니다.

```bash
# iSCSI Initiator로 이용할 호스트에서 수행

# 패키지 업데이트
sudo apt update

# iSCSI Initiator 설치
sudo apt install -y open-iscsi
```

```bash
# 타겟 서버 검색
sudo iscsiadm -m discovery -t sendtargets -p [Target IP]

# 검색된 타겟에 로그인
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] --login
```

![figure13](./images/figure13.png)

```bash
# 타겟 서버에서 Initiator의 로그인 상태 확인
sudo tgtadm --mode target --op show
```

![figure14](./images/figure14.png)

---

## 14. 연결된 디스크 확인

타겟에 로그인하면, Initiator 입장에서는 새로운 블록 디바이스가 추가됩니다. `lsblk` 명령으로 연결된 디스크가 로컬 장치처럼 인식되는지 확인합니다.

```bash
# 연결된 블록 디바이스 확인
lsblk
```

![figure15](./images/figure15.png)
![figure16](./images/figure16.png)

### 참고
- 타겟 서버 호스트에서는 `iSCSI 타겟`이 `디스크 이미지 파일`로 설정되어 있기 때문에, 실제 블록 디바이스로 나타나지 않음  
(`/srv/iscsi_disks/disk01.img` 파일로 존재)  

- Linux의 저장장치 네이밍 규칙
  - `sd` (SCSI Disk): SATA, SAS, USB 장치 및 iSCSI
  - `vd` (Virtio Disk): KVM/QEMU 같은 가상화 환경에서 Virtio 드라이버를 사용하는 가상 디스크
  - `nvme` (NVMe Disk): NVMe SSD 장치
  - `sr` (SCSI ROM): CD-ROM 또는 DVD-ROM 드라이브
  - 앞의 prefix를 붙인 후 알파벳과 숫자가 조합되어 디바이스 이름이 생성됨  
    (예: `sda`, `sdb`, `vda`, `nvme0n1` 등)

- **iSCSI Multipath (다중 경로)**
  - 실무 환경에서 iSCSI는 단일 네트워크 경로로 연결하는 경우가 드물며,  
    **Multipath I/O (MPIO)** 를 통해 여러 경로를 동시에 구성하는 것이 일반적
  - Multipath를 사용하는 이유
    - **고가용성(HA)**: 하나의 경로(NIC, 스위치, 케이블)가 장애가 나더라도 다른 경로로 자동 전환(Failover)
    - **부하 분산**: 여러 경로에 I/O를 분산시켜 처리량(Throughput) 향상
  - 구성 예시

     ```
    Initiator NIC1 ──── Switch A ──── Target Port1
                                            ↕ (같은 디스크)
    Initiator NIC2 ──── Switch B ──── Target Port2
     ```

    - 위 구성에서 `/dev/sdb`, `/dev/sdc` 두 개의 블록 디바이스가 생성되지만,  
    실제로는 동일한 디스크를 가리킴
    - `multipath-tools`를 사용하면 이를 하나의 논리 장치(`/dev/mapper/mpathX`)로 묶어 사용 가능

---

## 15. 디스크 포맷 및 마운트

iSCSI로 연결된 디스크는 비어 있는 블록 장치이므로, 사용하기 전에 파일시스템을 직접 생성(포맷)해야 합니다. 여기서는 ext4로 포맷한 뒤 디렉토리에 마운트하여 사용할 수 있도록 준비합니다.

```bash
# 연결된 디스크를 ext4 파일시스템으로 포맷
sudo mkfs.ext4 /dev/<디스크이름>

# 마운트할 디렉토리 생성
sudo mkdir /mnt/iscsi_disk

# 디스크를 마운트
sudo mount /dev/<디스크이름> /mnt/iscsi_disk

# /dev/<디스크이름>가 정상적으로 마운트되었는지 확인
mount | grep /dev/<디스크이름>

lsblk | grep <디스크이름>
```

![figure17](./images/figure17.png)

---

## 16. 디스크 사용 테스트

마운트한 디스크가 정상적으로 동작하는지 확인합니다. 권한을 설정한 뒤 파일을 생성하고, 사용량을 조회하여 실제 저장이 이루어지는지 점검합니다.

```bash
# 마운트 포인트의 소유자를 현재 사용자(ubuntu)로 변경
sudo chown ubuntu:ubuntu /mnt/iscsi_disk

# 모든 사용자에게 읽기/쓰기/실행 권한 부여
sudo chmod 777 /mnt/iscsi_disk

# 마운트된 디스크에 파일 생성 테스트
echo "iSCSI Storage Virtualization" > /mnt/iscsi_disk/test.txt

# 생성된 파일 확인
cat /mnt/iscsi_disk/test.txt

# 마운트된 디스크의 사용량 확인
df -h /mnt/iscsi_disk
```

![figure18](./images/figure18.png)

---

## 17. iSCSI 연결 해제

사용을 마치면 디스크를 안전하게 분리합니다. 먼저 마운트를 해제한 뒤 타겟 세션에서 로그아웃하면, Initiator에서 해당 블록 디바이스가 사라집니다.

```bash
# Initiator 호스트에서 수행

# 디스크 마운트 해제
sudo umount /dev/<디스크이름>

# 디스크 마운트 해제 확인
mount | grep /dev/<디스크이름>

lsblk | grep <디스크이름>

# iSCSI 타겟 세션에서 로그아웃
sudo iscsiadm -m node -T [Target IQN] -p [Target IP] --logout

# 디스크가 완전히 해제되었는지 확인
lsblk
```

![figure19](./images/figure19.png)
![figure20](./images/figure20.png)

```bash
# 타겟 서버에서 Initiator의 로그인 상태 확인
sudo tgtadm --mode target --op show
```

![figure21](./images/figure21.png)

### 참고
- `umount` 시 `device is busy` 오류가 발생하면 [NFS에서 설명한 방법](#7-nfs-마운트-해제)과 같이 해결

---

## NFS vs iSCSI 비교

앞서 다룬 두 방식은 모두 네트워크를 통해 스토리지를 제공하지만, 공유 수준과 용도가 다릅니다. NFS는 파일 수준에서 여러 호스트가 손쉽게 공유하기에 적합하고, iSCSI는 블록 수준으로 단일 호스트가 디스크처럼 사용하기에 적합합니다. 아래 표에서 주요 차이를 정리합니다.

| 항목 | NFS | iSCSI |
|------|-----|-------|
| **공유 수준** | 파일 시스템 수준 | 블록 수준 |
| **접근 방식** | 파일/디렉토리 단위 접근 | 디스크 장치 단위 접근 |
| **클라이언트 파일시스템** | 서버의 파일시스템을 그대로 사용 | 클라이언트가 직접 포맷 및 파일시스템 구성 |
| **동시 접근** | 다수의 클라이언트가 동시에 마운트 가능 | 단일 클라이언트 전용 (기본적으로 공유 불가) |
| **프로토콜** | TCP/UDP (포트 2049) | TCP (포트 3260) |
| **인증** | IP 기반 접근 제어 | IP 기반 + CHAP 인증 |
| **성능** | 상대적으로 낮음 (메타데이터 오버헤드) | 상대적으로 높음 (블록 I/O 직접 접근) |
| **주요 용도** | 공유 데이터, 홈 디렉토리, 로그 공유 | 데이터베이스, VM 디스크 이미지, 고성능 스토리지 |
| **운영 복잡도** | 낮음 | 높음 |

---

## Q & A

박찬욱  
cupark@dankook.ac.kr

남재현  
namjh@dankook.ac.kr  

## Networked Systems and Security Lab (BoanLab) @ DKU
<img src="../images/boanlab_logo.svg" width="25%"/>