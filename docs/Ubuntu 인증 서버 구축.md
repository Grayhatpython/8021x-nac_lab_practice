# [LAB] Ubuntu 서버 구축 실습 기록 (MySQL / FreeRADIUS / VLAN / DHCP)
- Date: 2026-01-15
- Env: EVE-NG QEMU Ubuntu (Server VM)
- Goal: Ubuntu 1대에 MySQL, FreeRADIUS, VLAN(802.1Q), ISC DHCP Server 설치 후 “동작 확인 + 트러블슈팅 포인트”를 실습 문서처럼 기록

---

## 0) 오늘 실습에서 확인된 결과 요약
- ✅ 인터넷 연결 OK (ping 8.8.8.8 성공)
- ✅ MySQL 설치 및 구동 OK (`systemctl status mysql` → active/running, MySQL 접속 + DB 목록 확인)
- ✅ FreeRADIUS 설치 및 로컬 인증 OK
  - 테스트 유저 `test01/test01` 등록
  - `clients.conf` secret = `testing123`
  - `freeradius -X` 디버그 실행 상태에서 `radtest` → Access-Accept 확인
- ✅ VLAN 패키지 설치 완료 (`vlan`)
- ✅ ISC DHCP Server 패키지 설치 완료 (`isc-dhcp-server`)
  - (DHCP는 패키지 설치까지만 완료, 실제 풀/인터페이스 바인딩 구성은 다음 단계로 진행)

---

## 1) 사전 점검 (네트워크/인터페이스)

<img width="1053" height="854" alt="스크린샷 2026-01-15 184213" src="https://github.com/user-attachments/assets/6daf9881-e004-45ae-b727-a5a8b1e8a9f1" />

### 1-1. 인터넷 확인
```bash
ping -c 2 8.8.8.8
```

### 1-2. IP/인터페이스 확인
```bash
ip addr
```

ens3 UP, IPv4: 10.0.10.11/28
ens4 UP (추가 NIC, 추후 VLAN/트렁크 실습용)

---

##  2) MySQL 설치 및 동작 검증

### 2-1. 설치
```bash
sudo apt update
sudo apt install mysql-server msql-client -y
```

### 2-2. 서비스 상태 확인
```bash
systemctl status mysql
```

정상 기준

<img width="1056" height="870" alt="스크린샷 2026-01-15 224411" src="https://github.com/user-attachments/assets/06174cb9-6326-4777-9886-3d4a9df17e6e" />

- Active: active (running)
- Status: "Server is operational"

### 2-3. MySQL 접속 및 기본 DB 확인

<img width="946" height="734" alt="스크린샷 2026-01-15 224852" src="https://github.com/user-attachments/assets/9d08d1ef-6b7d-46bb-b4f8-f34883f1b621" />

```bash
sudo mysql
SHOW DATABASES;
```

---

## 3) FreeRADIUS 설치 및 로컬 인증(radtest) 검증

<img width="950" height="748" alt="스크린샷 2026-01-15 230102" src="https://github.com/user-attachments/assets/3033eee0-7646-49fa-9b1c-0d653c01ea33" />

### 3-1. 설치 (MySQL 모듈 포함)
```bash
sudo apt install freeradius-mysql -y
```

### 3-2. (선택) 프로세스 확인
```bash
ps -ef | grep freeradius
```

### 3-3. 테스트 유저 추가

<img width="969" height="742" alt="스크린샷 2026-01-15 231017" src="https://github.com/user-attachments/assets/6fdb1372-2c33-4830-9040-65a54b089ae3" />

```bash
sudo nano /etc/freeradius/3.0/users
```
아래 라인을 추가:
```bash
test01  Cleartext-Password := "test01"
        Reply-Message := "Hello, %{User-Name}"
```

nano 저장/종료
- 저장: Ctrl + O → Enter
- 종료: Ctrl + X

### 3-4. shared secret 설정 (clients.conf)
radtest 실패(Access-Reject)의 1순위 원인이 secret 불일치라서, 테스트 단계에서는 여기 먼저 맞춰놓는 게 편함.

<img width="970" height="375" alt="스크린샷 2026-01-15 231818" src="https://github.com/user-attachments/assets/4ebe3f57-ad29-4371-acf8-ad3784f56d0c" />

```bash
sudo nano /etc/freeradius/3.0/clients.conf
```

테스트용 secret 예시:
```bash
secret = testing123
```

### 3-5. FreeRADIUS 디버그 모드 실행 (-X)
서비스로 돌리기 전에 -X로 띄우면 요청/응답이 다 보여서 트러블슈팅이 쉬움.

<img width="1038" height="414" alt="스크린샷 2026-01-15 231339" src="https://github.com/user-attachments/assets/21d59f37-c743-41bb-b85b-3bc5237892d9" />

터미널 A:
```bash
sudo freeradius -X
```

정상 기준
- 마지막에 Ready to process requests 비슷한 문구로 대기 상태

### 3-6. 로컬 인증 테스트 (radtest)

<img width="1027" height="797" alt="스크린샷 2026-01-15 231348" src="https://github.com/user-attachments/assets/39b13b9e-5a06-4d9e-9835-5f5592d131c9" />

터미널 B (새 창):
```bash
radtest test01 test01 localhost 0 testing123
```
정상 기준(오늘 확인한 형태)
- Received Access-Accept
- Reply-Message = "Hello, test01"

---

## 4) VLAN(802.1Q) 패키지 설치

<img width="970" height="375" alt="스크린샷 2026-01-15 231818" src="https://github.com/user-attachments/assets/1826ee2d-19fb-462a-bd78-9b00f6fae3fd" />

### 4-1. 설치
```bash
sudo apt install vlan -y
```

---

## 5) ISC DHCP Server 설치

<img width="1048" height="845" alt="스크린샷 2026-01-15 232319" src="https://github.com/user-attachments/assets/2d85d454-4753-4164-ab4c-8c1a9b4122ce" />

### 5-1. 설치
```bash
sudo apt install isc-dhcp-server -y
```

---

## 6) 빠른 명령어 모음 (복붙용)
```bash
# Network
ping -c 2 8.8.8.8
ip addr
ip route

# MySQL
sudo apt install mysql-server -y
systemctl status mysql
sudo mysql
"SHOW DATABASES;"

# FreeRADIUS
sudo apt install freeradius-mysql -y
ls -l /etc/freeradius/3.0/users
sudo nano /etc/freeradius/3.0/users
sudo nano /etc/freeradius/3.0/clients.conf
sudo freeradius -X
radtest test01 test01 localhost 0 testing123

# VLAN
sudo apt install vlan -y

# DHCP
sudo apt install isc-dhcp-server -y
systemctl status isc-dhcp-server
```
