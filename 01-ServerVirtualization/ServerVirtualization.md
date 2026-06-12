# 클라우드 가상화 기술

## 01. 서버 가상화 기술 실습

---

## 1. KVM (Kernel-based Virtual Machine)

서버 가상화의 핵심 구성 요소부터 살펴봅니다. KVM은 리눅스 `커널 모듈 기반의 가상화 기능`으로, Intel VT-x 또는 AMD-V와 같은 하드웨어 가상화 기능을 이용해 가상 머신(VM) 실행을 가속합니다.

- 리눅스를 호스트 OS로 이용하면서 하이퍼바이저 역할 수행
- VM 실행 시 QEMU와 함께 사용됨
- QEMU는 디바이스 에뮬레이션 및 VM 실행 기능 담당

![figure1](./images/figure1.png)

---

## 2. QEMU (Quick Emulator)

QEMU는 `PC 환경을 에뮬레이션`하는 VM 실행기이자 프로세스 에뮬레이터로, KVM과 함께 사용하여 VM을 실행합니다. KVM이 CPU 가상화를 담당한다면, QEMU는 가상 머신에 필요한 하드웨어를 흉내 내는 역할을 맡습니다.

- CPU와 주변 장치(Disk, NIC 등)를 에뮬레이션
- CPU 명령을 변환하여 실행
- CPU 가상화 지원이 없어도 동작 가능(단, 성능 저하 발생)

<img src="./images/figure2.png" width="50%"/>

---

## 3. Libvirt

Libvirt는 `가상화 환경을 관리`하기 위한 관리 프레임워크입니다. 하이퍼바이저마다 제어 방식이 다른 점을 추상화해, 동일한 방식으로 VM을 다룰 수 있도록 해 줍니다.

QEMU-KVM, Xen, VMware 등 다양한 하이퍼바이저를 관리하고 제어하기 위한 **통합 API**를 제공하며, 이후 실습에서 사용하는 `virsh` 명령어도 Libvirt를 통해 동작합니다.

![figure3](./images/figure3.png)

---

## 4. QEMU-KVM 설치

앞서 살펴본 구성 요소를 실제로 설치합니다. 먼저 호스트의 CPU가 하드웨어 가상화를 지원하는지 확인한 뒤, QEMU와 KVM을 설치합니다.

### CPU 가상화 지원 확인

KVM 가속을 사용하려면 CPU가 하드웨어 가상화를 지원해야 하므로, 설치 전에 지원 여부를 먼저 확인합니다.

```bash
# CPU 가상화 지원 여부 확인 명령어
egrep -c '(vmx|svm)' /proc/cpuinfo
```

- 결과가 **0이 아니라면 CPU가 하드웨어 가상화(Intel VT-x 또는 AMD-V)를 지원**
- 일부 시스템에서는 **BIOS 또는 UEFI에서 해당 기능이 비활성화**되어 있을 수 있음

![figure4](./images/figure4.png)

### 참고

- 하드웨어 가상화가 지원되고 활성화되어 있다면 **QEMU + KVM 사용 가능**  

- 지원하지 않는 경우 **QEMU만 사용하여 소프트웨어 에뮬레이션 방식으로 실행 가능 (성능 저하)**

---

### QEMU (+ KVM) 설치

QEMU, KVM, 그리고 Libvirt 관련 패키지를 한 번에 설치해 가상화 환경을 구성합니다.

```bash
sudo apt-get update

sudo apt-get install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager
```

- QEMU와 QEMU + KVM 모두 기본적인 설치 과정은 동일
- CPU가 가상화를 지원하지 않으면 **KVM 가속 사용 불가**

![figure5](./images/figure5.png)

---

### 권한 설정

기본 설정에서는 가상화 관련 명령을 실행할 때 루트 권한이 필요합니다. 일반 사용자 계정을 `libvirt`와 `kvm` 그룹에 추가하면 sudo 없이도 VM을 관리할 수 있습니다.

```bash
# 루트 권한 없이 KVM 명령어를 실행할 수 있도록 설정
sudo usermod -aG libvirt $USER

sudo usermod -aG kvm $USER
```

로그아웃 후 재로그인하여 권한 적용

---

## 5. VM 네트워크 확인

Libvirt를 설치하면 VM이 외부와 통신할 수 있도록 가상 네트워크가 자동으로 구성됩니다. VM을 만들기 전에 어떤 네트워크 인터페이스가 준비되어 있는지 확인합니다.

```bash
# VM을 위해 생성된 네트워크 인터페이스 확인
ip a
```

출력에서 `virbr0` 인터페이스를 확인합니다. 이는 Libvirt의 기본 가상 네트워크(default)를 위한 브리지 인터페이스입니다.

![figure6](./images/figure6.png)

---

### 가상 네트워크 정보 확인

`virsh` 명령어로 가상 네트워크의 목록과 상세 설정을 확인할 수 있습니다.

```bash
# 모든 가상 네트워크 확인
virsh net-list --all
```
![figure7](./images/figure7.png)


```bash
# 특정 네트워크 정보 확인   
virsh net-info default
```
![figure8](./images/figure8.png)


```bash
# 네트워크 설정 XML 출력
virsh net-dumpxml default
```
![figure9](./images/figure9.png)
---

## 6. ISO 기반 VM 생성

가장 일반적인 방식으로, 설치용 ISO 이미지를 이용해 OS를 직접 설치하면서 VM을 생성합니다.

### Ubuntu ISO 다운로드

설치에 사용할 Ubuntu 서버 ISO 이미지를 내려받아, Libvirt가 디스크 이미지를 관리하는 디렉터리로 옮깁니다.

```bash
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso

sudo mv ubuntu-24.04.3-live-server-amd64.iso /var/lib/libvirt/images/
```

---

### VM 생성

`virt-install` 명령어로 VM의 사양(vCPU, 메모리, 디스크)과 부팅에 사용할 ISO, 네트워크, 그래픽 방식을 지정해 새 VM을 생성합니다.

```bash
virt-install --name ubuntu-vm \
--vcpus 2 --ram 2048 \
--disk path=/var/lib/libvirt/images/ubuntu-vm.img,size=20,format=qcow2 \
--cdrom /var/lib/libvirt/images/ubuntu-24.04.3-live-server-amd64.iso \
--network network=default \
--graphics vnc,listen=0.0.0.0 \
--os-variant ubuntu24.04
```

---

## 7. VM 접속

생성한 VM의 화면에 접속해 OS 설치를 진행합니다. `--graphics vnc`로 생성한 VM은 VNC 뷰어로 화면에 접근할 수 있습니다.

### RealVNC Viewer 다운로드

VNC 접속에 사용할 뷰어를 아래 주소에서 내려받아 설치합니다.

https://www.realvnc.com/en/connect/download/viewer/

---

### VM 화면 연결

RealVNC Viewer를 이용하여 VM 화면 접속

![figure10](./images/figure10.png)

---

### 여러 VM 생성 시 VNC 포트

VM을 여러 개 생성하면 각 VM에 서로 다른 VNC 포트가 할당되므로, 접속하려는 VM의 포트를 구분해 두어야 합니다. 포트는 다음 방식으로 자동 할당됩니다.

```
5900 + display 번호
```

예시

| VM | Display | Port |
|----|--------|------|
| VM1 | 0 | 5900 |
| VM2 | 1 | 5901 |
| VM3 | 2 | 5902 |

VM 종료 시 포트는 재사용됨

![figure11](./images/figure11.png)
![figure12](./images/figure12.png)

---
### SSH를 이용한 VM 접속

화면 접속 외에 SSH로도 VM에 접속할 수 있습니다. 다만 ISO 기반으로 설치한 VM은 SSH 서버가 기본적으로 설치되어 있지 않으므로, VM 내부에서 직접 설치하고 서비스를 활성화해야 합니다.

```bash
sudo apt-get update

sudo apt-get install -y openssh-server

# ssh 서버 시작
sudo systemctl start ssh

# ssh 자동 시작 설정
sudo systemctl enable ssh
```

---

## 8. VM 관리 명령어 (virsh)

`virsh`는 Libvirt가 제공하는 명령행 관리 도구로, VM의 조회, 시작과 종료, 일시정지, 삭제 등 생애주기 전반을 제어할 수 있습니다.

### VM 목록 확인

```bash
# -all 입력 시 비활성화(종료) 된 VM도 출력
virsh list --all
```

---

### VM 제어

```bash
# VM 시작
virsh start <vm_name> 

# VM 종료
virsh shutdown <vm_name> 

# VM 재시작
virsh reboot <vm_name> 
```

---

### 강제 종료

정상 종료(`shutdown`)가 응답하지 않을 때, 전원을 강제로 차단하듯 VM을 즉시 중단합니다.

```bash
# 전원 차단과 같음
virsh destroy <vm_name>
```

---

### 일시정지 / 재개

```bash
# VM 일시정지
virsh suspend <vm_name>
# VM 재개
virsh resume <vm_name>
```

---

### VM 삭제

VM 정의를 제거합니다. 기본적으로 디스크 이미지는 남겨 두며, 디스크까지 함께 지우려면 별도 옵션을 사용합니다.

```bash
# VM 삭제 (디스크는 유지)
virsh undefine <vm_name>
```

```bash
# VM 삭제 (디스크도 함께 삭제)
virsh undefine <vm_name> --remove-all-storage
```

---

## 9. OS 설치 없이 VM 생성 (Cloud Image)

ISO로 OS를 직접 설치하는 방식과 달리, 클라우드 환경에서는 OS가 미리 설치된 Cloud Image를 사용해 설치 과정 없이 빠르게 VM을 만듭니다. 클라우드에서 인스턴스가 즉시 부팅되는 것과 같은 방식입니다.

### 패키지 설치

Cloud Image와 cloud-init 설정을 다루는 데 필요한 유틸리티를 설치합니다.

```bash
sudo apt-get install -y cloud-image-utils
```

---

### Ubuntu Cloud Image 다운로드

OS가 미리 설치된 Ubuntu Cloud Image를 내려받아 디스크 이미지 디렉터리로 옮깁니다.

```bash
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

sudo mv noble-server-cloudimg-amd64.img /var/lib/libvirt/images
```

### Ubuntu 코드명

| Version | Codename |
|--------|----------|
| 20.04 | focal |
| 22.04 | jammy |
| 24.04 | noble |

---

## 10. cloud-init 설정

Cloud Image는 처음 부팅될 때 `cloud-init`을 통해 초기 설정을 적용합니다. 호스트 이름, 사용자 계정, 비밀번호, 네트워크 등을 미리 정의해 두면 부팅과 동시에 자동으로 구성됩니다.

### user-data

생성할 계정과 비밀번호, sudo 권한, 로그인 방식 등 VM의 초기 사용자 환경을 정의합니다.

```yaml
#cloud-config
hostname: myvm

users:
  - name: ubuntu # 사용자 계정 이름
    sudo: ["ALL=(ALL) NOPASSWD:ALL"] # sudo 입력 시 PW 미입력 설정 (실제 환경에서는 사용하지 않는 걸 추천)
    groups: users, sudo
    home: /home/ubuntu
    shell: /bin/bash
    lock_passwd: false

ssh_pwauth: true # 비밀번호 로그인 활성화

chpasswd:
  list: | # 계정:비밀번호
    ubuntu:ubuntu
  expire: false
```

---

### meta-data

인스턴스 식별 정보를 담는 파일로, 여기서는 VM의 로컬 호스트 이름을 지정합니다.

```yaml
local-hostname: myvm
```

---

### network-config (ubuntu 기준 netplan) 

VM의 네트워크 구성을 정의합니다. 아래 예시는 DHCP 대신 고정 IP, 게이트웨이, DNS를 직접 지정하는 설정입니다.

```yaml
version: 2
ethernets:
  enp1s0: # NIC 이름은 환경마다 다를 수 있음 (ens3, eth0 등)
    dhcp4: false
    addresses:
      - 192.168.122.20/24 # VM IP Address
    routes:
      - to: default
        via: 192.168.122.1 # Gateway
    nameservers: # DNS Server
      addresses:
        - 8.8.8.8
```

---

## 11. cloud-init 이미지 생성

앞서 작성한 설정 파일들을 cloud-init이 읽을 수 있는 하나의 ISO 이미지(seed)로 묶습니다. 이 이미지를 VM에 연결하면 첫 부팅 시 설정이 적용됩니다.

```bash
# cloud-localds 명령어로 cloud-init ISO 이미지 생성
sudo cloud-localds -v --network-config=network-config myvm-seed.iso user-data meta-data

# 생성된 ISO 이미지를 VM 디스크 이미지 저장 디렉터리로 이동
sudo mv myvm-seed.iso /var/lib/libvirt/images
```

---

## 12. QCOW2 디스크 생성

원본 Cloud Image를 직접 사용하지 않고, 그 위에 변경분만 기록하는 overlay 디스크를 만듭니다. 이렇게 하면 원본 이미지는 그대로 보존되어 여러 VM의 기반으로 재사용할 수 있습니다.

```bash
# 디스크 이미지 저장 디렉터리 이동
cd /var/lib/libvirt/images

# noble 이미지 기반 overlay 디스크 생성
# 최대 디스크 크기 20GB
sudo qemu-img create -F qcow2 -b ./noble-server-cloudimg-amd64.img -f qcow2 ./myvm-base.qcow2 20G
```

---

## 13. Cloud Image 기반 VM 생성

준비한 overlay 디스크와 cloud-init seed 이미지를 연결해 VM을 생성합니다. ISO 설치 방식과 달리 `--import` 옵션으로 기존 디스크를 그대로 가져와 부팅하므로 별도의 OS 설치 과정이 없습니다.

```bash
# cloud-init ISO 이미지 연결하여 VM 생성
virt-install --name myvm \
--vcpus 2 --ram 2048 \
--import \
--disk path=/var/lib/libvirt/images/myvm-base.qcow2,format=qcow2 \
--cdrom /var/lib/libvirt/images/myvm-seed.iso \
--network network=default \
--graphics none \
--os-variant ubuntu24.04
```
### SSH를 이용한 VM 접속

Cloud Image 기반 VM은 SSH 서버가 기본적으로 설치되어 있어, 별도 설치 없이 곧바로 SSH로 접속할 수 있습니다.
```bash
# VM IP 주소는 network-config에서 설정한 IP로 접속
ssh ubuntu@<VM_IP_ADDRESS>

ssh ubuntu@192.168.122.20
```

---

## 14. Snapshot 기능

Snapshot은 특정 시점의 VM 상태를 저장해 두었다가 필요할 때 그 시점으로 되돌릴 수 있는 기능입니다. 설정 변경이나 테스트 전에 안전한 복원 지점을 만들어 둘 때 유용합니다.

### Snapshot 생성

디스크 상태만 저장하는 방식과 메모리(RAM) 상태까지 함께 저장하는 방식이 있습니다. 메모리를 포함하면 복원 시 실행 중이던 상태까지 그대로 되살릴 수 있습니다.

```bash
# 1. VM 디스크 상태를 external snapshot 방식으로 저장
# 원본 디스크는 backing file로 유지되고, 새로운 overlay 파일에 변경분이 기록됨
# 메모리 상태는 저장되지 않음
virsh snapshot-create-as \
--domain myvm \
--name external_disk_snapshot \
--description "External disk-only snapshot" \
--disk-only \
--atomic \
--diskspec vda,snapshot=external,file=/var/lib/libvirt/images/myvm_overlay.qcow2

# 2. 실행 중인 VM의 디스크 + 메모리(RAM) 상태를 함께 저장하는 full snapshot 생성
# 단, 종료된 VM에서는 메모리 상태가 없으므로 디스크 상태만 저장됨
# 복원 시 Snapshot 시점의 실행 상태까지 함께 복원됨
virsh snapshot-create-as \
--domain myvm \
--name full_snapshot \
--description "Snapshot with memory" \
--atomic

# Snapshot 리스트 확인
virsh snapshot-list myvm

# Snapshot 정보 확인
virsh snapshot-info myvm --snapshotname full_snapshot
```

---

### Snapshot 복원

```bash
# 생성된 snapshot(full_snapshot) 상태로 VM 복원
virsh snapshot-revert --domain myvm --snapshotname full_snapshot
```
`snapshot-revert` 명령어는 생성된 snapshot 상태로 VM을 복원하는 기능

VM이 실행 중일 경우 복원 과정에서 VM이 일시 중지되거나 재시작될 수 있음

메모리 포함 snapshot의 경우 실행 상태까지 함께 복원되며  

disk-only snapshot의 경우 디스크 상태만 복원되고 VM은 재시작 상태로 동작

---

### Snapshot 삭제

더 이상 필요하지 않은 Snapshot을 제거합니다. 데이터까지 함께 지울 수도 있고, 메타데이터만 정리할 수도 있습니다.

```bash
# 스냅샷 메타데이터와 데이터를 함께 삭제
virsh snapshot-delete --domain myvm --snapshotname full_snapshot

# 스냅샷 메타데이터만 삭제
virsh snapshot-delete --domain myvm --snapshotname full_snapshot --metadata
```
--- 

## Q & A

박찬욱   
cupark@dankook.ac.kr

남재현  
namjh@dankook.ac.kr

## Networked Systems and Security Lab (BoanLab) @ DKU
<img src="../images/boanlab_logo.svg" width="25%"/>