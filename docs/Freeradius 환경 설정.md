# FreeRADIUS 802.1X(PEAP) 환경 구성 & 트러블슈팅 로그

> **목표:** 신입 단말이 스위치 포트에 연결된 뒤 **802.1X 인증 성공 시 VLAN 810을 동적 할당**받고, **VLAN 810의 DHCP 풀**에서 IP를 받는 흐름을 실습 환경(EVE-NG)에서 구성한다.

---

## 1) 오늘 한 일 요약
- FreeRADIUS 기본 설치 후 **EAP(PEAP / inner MSCHAPv2)** 설정
- OpenSSL로 **CA / Server / Client 인증서 생성** 및 FreeRADIUS 경로에 배치
- 스위치를 RADIUS Client로 등록(`clients.conf`)
- 테스트 사용자 등록 + **RADIUS VLAN 속성(Tunnel-*)**으로 VLAN 810 할당 유도
- **클라이언트(Windows)에서 CA 인증서(ca.der) 내려받아 신뢰할 수 있도록 준비**
- L3 라우팅은 정상인데 **Windows Ping이 안 되던 문제(방화벽)** 해결

---

## 2) 실습 환경(간단)
- EVE-NG 내 Ubuntu(FreeRADIUS 서버)
- Access Switch(802.1X 수행, RADIUS에 인증 요청)
- L3 라우팅 스위치(Distribution 역할) ↔ Edge Router 연결
- 테스트 단말: Windows / Linux

---

## 3) 인증서(OpenSSL) 생성 & 배치
### 3-1. 인증서 템플릿 위치 확인
FreeRADIUS 예제 인증서 템플릿을 먼저 찾았다.

<img width="1048" height="883" alt="01-cert-templates-location" src="https://github.com/user-attachments/assets/72c0b158-2df5-42bf-8584-95e532d32589" />


### 3-2. cnf 수정 포인트
`ca.cnf / server.cnf / client.cnf`에서 주로 아래를 수정했다.

- `default_days`: 인증서 유효기간(일 단위)
- `input_password / output_password`: 개인키 암호(Passphrase)
- 인증서 주체 정보(C, ST, L, O, CN, email 등)

<img width="443" height="387" alt="02-ca-default-days" src="https://github.com/user-attachments/assets/93f150c5-548c-4d4e-bd63-29c55d82ca37" />

<img width="522" height="171" alt="03-ca-req-password" src="https://github.com/user-attachments/assets/298d9ab9-b2af-4d73-b39b-a7584c2536ee" />

<img width="611" height="174" alt="04-ca-subject" src="https://github.com/user-attachments/assets/dacd4169-b3ec-408a-a5eb-582f0ab301b7" />

<img width="444" height="388" alt="05-server-default-days" src="https://github.com/user-attachments/assets/11c4a392-0c50-4f68-af41-69293a02c3fe" />

<img width="553" height="170" alt="06-server-subject" src="https://github.com/user-attachments/assets/12065848-9653-48a6-af02-8d2f8be7d7cb" />


### 3-3. 인증서 생성
예제 스크립트/명령으로 CA 인증서를 생성했다.

<img width="883" height="308" alt="07-generate-ca-openssl" src="https://github.com/user-attachments/assets/e436f76e-d18c-40e4-9870-0d896c7b97d0" />


생성된 산출물(예: `ca.der`, `ca.pem`, `server.key`, `server.pem` 등)을 확인했다.

<img width="858" height="181" alt="08-generated-files-ls" src="https://github.com/user-attachments/assets/273591a9-6201-4298-a067-daf7f4d3cfe9" />


### 3-4. FreeRADIUS 인증서 경로에 배치
FreeRADIUS가 읽는 인증서 디렉터리에 생성된 인증서를 배치했다.

> 일반적으로 `/etc/freeradius/3.0/certs` 경로를 사용했다.

<img width="737" height="397" alt="09-certs-in-freeradius-dir" src="https://github.com/user-attachments/assets/704373af-0bd6-4f11-ae02-64dc2ab7c583" />


권한 문제로 읽지 못하는 경우가 많아서 `.pem`과 `.key` 권한을 정리했다.

```bash
cd /etc/freeradius/3.0/certs
chmod 644 *.pem
chmod 600 *.key
```

<img width="883" height="90" alt="10-fix-cert-permissions" src="https://github.com/user-attachments/assets/3d5884a9-b147-47ed-ba92-87dd7fbd7bac" />


---

## 4) FreeRADIUS EAP(PEAP) 설정
### 4-1. default EAP 타입을 PEAP로
EAP 모듈 설정에서 `default_eap_type`을 `peap`로 바꿨다.

<img width="339" height="38" alt="11-default-eap-type-peap" src="https://github.com/user-attachments/assets/7d3aee04-29f6-4625-9629-2699f5876432" />


### 4-2. PEAP 내부 인증(inner) = MSCHAPv2
PEAP는 “바깥은 TLS 터널”, “안쪽은 사용자 비번 인증” 구조라서, 내부 인증 타입을 `mschapv2`로 둔다.

<img width="366" height="46" alt="17-peap-inner-mschapv2" src="https://github.com/user-attachments/assets/99b30b8a-8244-4288-8ed6-69d58dad50d4" />


### 4-3. TLS 개인키 암호(passphrase)
인증서 생성 시 개인키 암호를 설정했기 때문에, FreeRADIUS 설정에도 동일한 암호가 필요하다.

<img width="755" height="81" alt="16-eap-tls-common-key-pass" src="https://github.com/user-attachments/assets/c77e5275-3ca7-4176-adb6-da67072d0b40" />


> ✅ 체크 포인트
> - `private_key_file`, `certificate_file`, `ca_file`가 **내가 생성한 server.pem/ca.pem 경로**를 가리키는지 확인 필요
> - (처음엔 기본(snakeoil) 경로가 남아있을 수 있음)

---

## 5) Access Switch를 RADIUS Client로 등록 (clients.conf)
스위치가 FreeRADIUS에 인증 요청을 보낼 수 있도록, 스위치 IP와 공유 시크릿을 등록했다.

```conf
client access_switch_1 {
  ipaddr     = 10.0.11.4
  secret     = test123
  shortname  = ASW1
  nastype    = cisco
}
```

<img width="910" height="548" alt="12-clients-conf-add-switch" src="https://github.com/user-attachments/assets/185ba2d0-fd31-41ea-ada0-9bff12b8fbea" />


---

## 6) 테스트 사용자 등록 + VLAN 810 동적 할당 속성(users)
테스트 사용자 계정(예: `haru / haru123`)을 만들고, **인증 성공 시 VLAN 810을 내려주도록** Tunnel 속성을 추가했다.

```conf
haru   Cleartext-Password := "haru123"
       Tunnel-Type = VLAN,
       Tunnel-Medium-Type = IEEE-802,
       Tunnel-Private-Group-ID = 810
```

<img width="903" height="584" alt="13-users-add-vlan-810" src="https://github.com/user-attachments/assets/3e6e0f09-c1d6-4f95-be57-bc52b30429a1" />


### ✅ Tunnel-* 속성 의미 (쉽게)
- **Tunnel-Type = VLAN**
  - “이 사용자는 VLAN 기반으로 네트워크에 붙여라”라는 의미
- **Tunnel-Medium-Type = IEEE-802**
  - “이 VLAN은 이더넷(802 계열) 환경에서 적용되는 VLAN이다”
- **Tunnel-Private-Group-ID = 810**
  - 실제로 할당할 VLAN 번호(= 810)

> 즉, **인증이 성공하면 스위치가 이 값을 보고 ‘이 단말은 VLAN 810에 넣자’**고 처리할 수 있다.

---

## 7) 변경사항 적용 (서비스 재시작)
```bash
systemctl restart freeradius
```

<img width="894" height="48" alt="18-restart-freeradius" src="https://github.com/user-attachments/assets/b12d798e-57c3-4063-8c00-0c9bf1a01bac" />


---

## 8) 클라이언트에 CA 인증서 배포 (ca.der)
클라이언트가 서버 인증서를 신뢰하려면 **CA 인증서(루트)** 를 설치해야 한다.

나는 Ubuntu에서 `ca.der`를 단말로 옮기기 위해 간단히 Python HTTP 서버를 사용했다.

```bash
# 예) 홈 디렉터리로 복사 후
cp /etc/freeradius/3.0/certs/ca.der ~/
cd ~
python3 -m http.server 8000
```

<img width="1072" height="875" alt="14-export-ca-der-python-http" src="https://github.com/user-attachments/assets/5ab31a73-c752-48ff-b334-2056ea95f76b" />


### 트러블: 브라우저에서 접속 거부(포트 실수)
처음에 포트를 `8000`이 아니라 `800`으로 접속해서 연결 거부가 났다.

<img width="1041" height="844" alt="15-windows-download-ca-der-wrong-port" src="https://github.com/user-attachments/assets/1cc7a23e-6009-4e93-8acc-9218e94266f9" />


---

## 9) 트러블슈팅: VLAN 간 라우팅은 되고 윈도우에서 Ping이 되지만 Linux에서만 Ping이 안 됨
### 증상
- 라우팅 스위치/리눅스에서 Windows(예: `10.0.31.5`)로 ping이 실패

<img width="707" height="126" alt="20-ping-fail-linux-to-windows" src="https://github.com/user-attachments/assets/761e3bc5-445c-4bd4-9aa2-d0ad9ba75f43" />


### 원인
- **Windows Defender 방화벽이 ICMP Echo 요청을 기본 차단**

<img width="827" height="391" alt="21-windows-firewall-currentprofile" src="https://github.com/user-attachments/assets/0319d5b2-dd8e-4b3f-8cb6-12ba931c2d01" />


### 해결 1) 빠른 원인 확인: 방화벽 OFF로 즉시 검증
```bat
netsh advfirewall set allprofiles state off
```

<img width="593" height="217" alt="22-firewall-off-netsh" src="https://github.com/user-attachments/assets/d69e28d1-b8ed-49a8-a3d6-2339b0309051" />


> 이 방법은 “원인 확인”용으로만 쓰고, 실사용은 규칙을 열어주는 방식으로 정리하는 게 안전함.

### 해결 2) 권장: ICMPv4 Echo(핑) 인바운드 규칙만 허용
Windows Defender 방화벽 고급 설정에서 아래 규칙을 활성화했다.
- **파일 및 프린터 공유(에코 요청 - ICMPv4-In)**

<img width="1033" height="834" alt="23-enable-icmpv4-inbound-rule-list" src="https://github.com/user-attachments/assets/6f07a7d4-8e5b-4aa4-b498-0e14d6564fd9" />

<img width="589" height="556" alt="24-icmpv4-rule-properties-allow" src="https://github.com/user-attachments/assets/b628959a-2bcb-4c19-b497-c1b68077cfca" />


### 결과
- Linux → Windows ping 성공
- 라우팅 스위치(DSW) → Windows ping도 성공

<img width="646" height="226" alt="25-ping-success-linux-to-windows" src="https://github.com/user-attachments/assets/b278dc8b-ba9e-472c-995b-99df7b45f2df" />

<img width="562" height="69" alt="26-dsw1-ping-success" src="https://github.com/user-attachments/assets/af30f467-5de9-42a4-8744-3cbb769b1a16" />


양방향 통신도 같이 확인했다.

<img width="2129" height="838" alt="27-bidirectional-ping-verification" src="https://github.com/user-attachments/assets/8e023085-0232-4fc1-a000-1ce5cbfd5ecf" />


---

## 10) 오늘 얻은 교훈(짧게)
- “라우팅이 안 된다”가 아니라, **단말 OS 방화벽이 막고 있을 수 있다**
- 인증서 작업은 “생성”보다 **배치 경로 + 권한 + FreeRADIUS 설정 경로**가 더 자주 문제된다
- 802.1X에서 VLAN 할당은 스위치 설정도 중요하지만, RADIUS에서 내려주는 **Tunnel-* 속성**이 핵심이다

---

## 11) 다음 할 일 (To-Do)
- Access Switch 포트에 802.1X 적용
  - `dot1x system-auth-control`
  - `authentication port-control auto`
  - RADIUS 서버 주소/secret 적용
- VLAN 810의 DHCP 풀 구성 + 실제 단말이 VLAN 810에서 IP 받는지 확인
- FreeRADIUS 디버그 모드로 인증 흐름 로그 수집
  - `freeradius -X`

