# 클라우드 가상화 기술

## 네트워크 가상화 기술 실습 (현대)

## 1. Mininet

`Mininet`은 가상화된 환경에서 가상의 스위치, 호스트, 링크를 생성하여 실제 네트워크 동작을 시뮬레이션하는 도구

- Linux 기반 Network Namespace 활용
- 빠르고 가벼운 네트워크 토폴로지 구성 가능

---

## 2. Mininet 설치

```bash
# 패키지 업데이트
sudo apt-get update

# 필수 패키지 설치
sudo apt-get install -y git python3-pip openvswitch-switch

# Mininet 소스코드 클론
git clone https://github.com/mininet/mininet

# 설치 스크립트 실행
cd mininet

# Mininet 설치
sudo ./util/install.sh -a

# 설치 확인
mn --version
```

![figure1](./images/figure1.png)

---

## 3. POX 컨트롤러

`POX`는 Python 기반의 `OpenFlow 컨트롤러`로 학습 및 프로토타이핑에 적합

- `OpenFlow 기반 SDN 컨트롤러`
- 가볍고 사용이 간단

---

## 4. POX 설치

```bash
# 홈 디렉토리에서 POX 클론
cd ~
git clone https://github.com/noxrepo/pox
# Mininet 설치 시 이미 포함되어 있어 별도로 하지 않아도 됨

# 실행 테스트
cd ~/pox
./pox.py -h
```

![figure2](./images/figure2.png)

---

## 5. Mininet 기본 실행

```bash
# 최소 토폴로지 실행
sudo mn --topo minimal
```

![figure3](./images/figure3.png)  

---

## 6. Tree 토폴로지 생성

```bash
# Tree 구조 토폴로지 생성
# depth: 트리의 깊이
# fanout: 각 스위치의 하위 호스트 수
sudo mn --topo tree,depth=2,fanout=2
```

![figure4](./images/figure4.png)

---

## 7. Mininet CLI 명령어

```bash
# Mininet CLI에서 실행
mininet> h1 ping -c 1 h2
mininet> h2 ping -c 3 h4
mininet> h3 ip a
mininet> h4 ip route
```

![figure5](./images/figure5.png)
![figure6](./images/figure6.png)

---

## 8. 호스트에서 Mininet 내부 네트워크 정보 확인

```bash
# 호스트에서 실행
ps -ef | grep mininet

# mininet의 PID를 통해 h3 네트워크 네임스페이스 네트워크 정보 확인
sudo nsenter -t <h3 PID> -n ip a
```

![figure7](./images/figure7.png)

### s1, s2, s3 간 연결 관계 확인

```bash
# OVS 스위치 정보 확인
sudo ovs-vsctl show

# 인터페이스 정보 확인
ip a | grep s
```
`ovs-vsctl show와 ip a 결과를 비교하면 스위치 간 인터페이스 연결 관계를 확인할 수 있음`

---

## 9. POX 실행

```bash
cd ~/pox

# L2 Learning 스위치 실행
./pox.py log.level --DEBUG forwarding.l2_learning
```

![figure8](./images/figure8.png)

---

## 10. Mininet에서 컨트롤러 연결

```bash
sudo mn --topo single,3 \
--controller=remote,ip=127.0.0.1,port=6633
```

![figure9](./images/figure9.png)

---

## 11. L2 Learning 모듈 동작

경로: `~/pox/pox/forwarding/l2_learning.py`

### 주요 동작

- PacketIn 이벤트 발생 시 처리
- MAC 주소 학습
- 목적지 MAC 확인 후 forwarding 수행

```python
# MAC 학습
self.macToPort[packet.src] = event.port
```

### 동작 흐름

- 목적지 MAC 존재 → 유니캐스트 전달 (FlowMod)
- 목적지 MAC 없음 → Flood 처리 (PacketOut)
- Flow 설치 후 스위치에서 직접 처리 가능

### h1에서 h3로 ping 테스트

```bash
h1 ping -c 3 h3

# Flow Table 확인
sudo ovs-ofctl dump-flows s1
```

![figure10](./images/figure10.png)  
![figure11](./images/figure11.png)
![figure12](./images/figure12.png)
`어떻게 학습되는지 확인해보기`

---

## 12. OVS 스위치 정보 확인

```bash
# OVS 스위치 정보 확인

# s1 스위치의 포트와 연결 상태 확인
sudo ovs-ofctl show s1

# Flow Table 확인
# 트래픽이 없으면 빈 테이블이지만, ping 테스트 후에는 학습된 Flow가 보임
sudo ovs-ofctl dump-flows s1

# Port 정보 확인
sudo ovs-ofctl dump-ports s1
```

---

## 13. 방화벽 구현

`l2_learning` 코드를 기반으로 특정 트래픽 차단 구현
- 특정 트래픽(출발지 MAC/IP, 목적지 MAC/IP, 포트 등)에 대한 차단 규칙을 적용
- 편의상, 코드에 차단 규칙을 하드 코딩해도 됨
- ~/pox/pox/forwarding/l2_learning.py 파일에서 PacketIn 이벤트 처리 부분 수정


### 방법 1

- 특정 트래픽이 들어오면 `PacketOut`, `FlowMod` 메시지 없이 패킷 드랍 
- [예시코드](./codes/l2_learning_1.py) s1 - h1, h2, h3, h4 구조일 때

![figure13](./images/figure13.png)
![figure14](./images/figure14.png)

### 방법 2

- 특정 트래픽이 들어오면 패킷 드랍과 동시에 차단을 위한 `FlowMod` 메시지 생성하여 설치하기
- [예시코드](./codes/l2_learning_2.py) s1 - h1, h2, h3, h4 구조일 때

![figure15](./images/figure15.png)
![figure16](./images/figure16.png)
![figure17](./images/figure17.png)
![figure18](./images/figure18.png)

---

## 14. 심화 과제 - 로드 밸런서 구현

`라운드 로빈 방식으로 서버 선택하는 로드밸런서 구현해보기`

---

## 15. 기타 SDN 컨트롤러

### Ryu

- https://github.com/faucetsdn/ryu

---

### ONOS

```bash
# ONOS 실행
docker run -d --net=host --name onos onosproject/onos:2.1.0
```
- 접속:  
  - http://<IP주소>:8181/onos/ui/login.html  
- 계정:  
  - onos / rocks  
- 설정:
  - Applications -> OpenFlow Provider Suite, Reactive Forwarding (Activation)

```bash
# Mininet에서 ONOS 연결
sudo mn --topo tree,depth=2,fanout=2 \
--controller=remote,ip=<IP주소>,port=6653 \
--switch ovs,protocols=OpenFlow13
```

---

### OpenDayLight

- https://docs.opendaylight.org/en/latest/getting-started-guide/installing_opendaylight.html


---

## Q & A

박찬욱  
cupark@dankook.ac.kr

남재현  
namjh@dankook.ac.kr  

## Networked Systems and Security Lab (BoanLab) @ DKU
<img src="../images/boanlab_logo.svg" width="25%"/>