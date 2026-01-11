# 802.1X 기반 네트워크 접근통제(NAC) 구축 (FreeRADIUS)

본 프로젝트는 **802.1X + FreeRADIUS**를 활용하여 사용자 단말의 네트워크 접속을 인증하고,  
정책에 따라 **스위치 포트 제어 / VLAN 할당 / 서비스 접근 통제**를 수행하는 NAC(Network Access Control) 구축 예시입니다.

---

## 1) 토폴로지

### 전체 구성(연구소 ↔ 기숙사, ISP A/B 분리)

<img width="1773" height="1079" alt="스크린샷 2026-01-11 165702" src="https://github.com/user-attachments/assets/b987a7d2-53ca-4635-96f2-0add3236a1e3" />

### VLAN/데이터 흐름 상세(802.1X 인증 + DHCP 흐름)

<img width="1389" height="593" alt="스크린샷 2026-01-11 165709" src="https://github.com/user-attachments/assets/90f75c41-7581-4742-8c90-c8b616e1be2f" />

---


## 2) VLAN 설계

| VLAN | 목적 | 주요 시스템/대상 | 기본 게이트웨이 | 라우팅/격리 정책 |
|------|------|------------------|------------------|------------------|
| **300** | 보안/인증 인프라 운영 VLAN | 보안장비, **인증 서버(FreeRADIUS)**, **DHCP 서버** | (백본 스위치 NAC_BSW) | 접근통제 정책 적용: **관리자 외 접근 차단**, 서비스 포트 단위 허용 |
| **310** | 네트워크 장비 관리 VLAN | 스위치/게이트웨이 관리 IP, 원격 유지보수 트래픽 | (백본 스위치 NAC_BSW) | 인증 서버와의 통신 허용(인증/운영 관리 목적) |
| **800** | 기숙사 사용자 VLAN | 기숙사 사용자 단말(데스크탑 등) | **NAC_GW_2** | 연구소 내부로 라우팅되지 않음. **ISP B 회선(별도 경로)**로 인터넷 통신 |

---

## 3) 구성 요소와 역할

- **Supplicant (요청자)**: 인증 대상 단말(기숙사/내부 데스크탑)
  - 네트워크 접속 시 802.1X 인증 정보(EAPOL)를 전송

- **Authenticator (인증자)**: 사용자 단말이 직접 연결되는 **Access Switch**
  - 802.1X 인증을 중계하고, 인증 결과에 따라 **스위치 포트 제어(허용/차단/권한 VLAN 할당)** 수행
  - 인증 서버로 RADIUS 요청 전달 및 Accounting 정보 수집(선택)

- **Authentication Server (인증 서버)**: **FreeRADIUS**
  - 인증자(스위치)로부터 인증 정보를 받아 사용자 인증 수행
  - 사용자 권한/정책에 따라 **VLAN ID 할당**, 포트 제어 정책 적용을 지원

- **DHCP Server (DHCP 서버)**  
  - VLAN별 IP 할당 요청 처리(각 VLAN Scope 운영)
  - VLAN800(기숙사)의 경우 **기본 게이트웨이는 NAC_GW_2**로 할당

- **Backbone Switch (NAC_BSW)**  
  - 연구소 내부 네트워크의 중심(집선/정책 적용)
  - VLAN300/310의 정책 기반 접근통제 및 기본 게이트웨이 역할

- **Gateway Switch (NAC_GW_1,NAC_GW_2)**  
  - 기본 게이트웨이
  - 트래픽을 각각 **ISP A or ISP B**로 아웃바운드

---

## 4) 접근통제 정책 개요

### VLAN300 (보안/인증 인프라)
- **관리자 이외 접근 차단**
- 시스템별 서비스 특성에 따라 **허용 포트/허용 대상 호스트를 세분화하여 통제**
  - 예) SSH(22), RADIUS(1812/1813), DHCP(67/68), DNS(53), HTTP/HTTPS(80/443) 등

### VLAN310 (관리 VLAN)
- 네트워크 장비 관리 IP 및 원격 유지보수 트래픽을 위한 VLAN
- 인증 서버(FreeRADIUS)와 인증/운영 관리를 위한 통신이 가능하도록 구성

### VLAN800 (기숙사 VLAN)
- 연구소 내부로 라우팅되지 않음
- NAC_GW_2를 통해 **ISP B로만 통신**하도록 분리

---

## 5) 트래픽 흐름(데이터/인증/DHCP)

### 5.1 물리 연결 경로(Physical)
- Backbone ↔ Bridge ↔ Access 스위치로 L2/L3 경로 구성
- 인증 서버/ DHCP 서버는 Backbone 영역(VLAN300)에 위치

### 5.2 데이터 통신 경로(Data Plane)
- 인증 완료 후:
  - 연구소 내부 VLAN(예: 300/310 등)은 Backbone을 기본 게이트웨이로 통신
  - 기숙사 VLAN800은 NAC_GW_2를 기본 게이트웨이로 ISP B로 통신

### 5.3 802.1X 인증 패킷 전달 경로(Control Plane)
구성 요소: **요청자(Supplicant) → 인증자(Authenticator) → 인증서버(FreeRADIUS)**

1) 단말이 Access Switch 포트에 연결되면 **802.1X(EAPOL) 인증 요청**
2) Access Switch(Authenticator)는 인증 정보를 **RADIUS로 캡슐화**하여 FreeRADIUS로 전달  
3) FreeRADIUS는 사용자 정상 여부를 인증하고, 정책에 따라:
   - 포트 허용/차단
   - VLAN ID 할당(권한 VLAN)
   - (선택) Accounting 정보 저장
4) 인증 성공 시 해당 단말 포트가 데이터 트래픽을 송수신할 수 있도록 전환됨

> 기숙사 ↔ 인증 서버 구간 스위치들에는 **VLAN310이 선언**되어 있으며,  
> 이를 통해 기숙사 측 인증자(Access Switch)와 인증 서버 간 **인증 정보 전달 경로**가 구성된다.

### 5.4 DHCP IP 할당 경로
- DHCP 서버는 VLAN300에서 운영하며, 각 VLAN별 IP 할당 요청을 처리
- VLAN800(기숙사) 단말은 DHCP를 통해 IP를 할당받으며:
  - **기본 게이트웨이(Default Gateway)는 NAC_GW_2의 VLAN800 인터페이스 IP**
- 그 외 VLAN(예: 300/310 등)은 기본 게이트웨이가 Backbone(NAC_BSW)로 설정됨

---

## 진행 현황 (Status)

- 본 프로젝트는 **EVE-NG** 환경에서 구현을 진행한다.
- 현재 단계에서는 전체 아키텍처/토폴로지 및 VLAN 분리(300/310/800), 인증 흐름(802.1X ↔ RADIUS), DHCP 흐름 등 **큰 구조(High-level Design)** 를 우선 설계하였다.
- 세부 구성(장비별 설정, 정책 룰, 인증 방식(EAP) 세부, 예외 처리, 검증 로그/캡처 등)은 **단계적으로 추가/고도화**할 예정이다.

---

## 단계별 구현 계획 (Roadmap)

1. 네트워크 인프라 구성 (EVE-NG)
- [ ] 장치 연결 / 백본 / 연구소 스위치 / 기숙사 스위치 / 연계 스위치
- [ ] 연구소·기숙사 게이트웨이 포함(= L2/L3/VLAN/라우팅 기본 뼈대)
2. 인증 인프라 구성 (FreeRadius)
- [ ] OS 설치 및 환경 설정
- [ ] FreeRADIUS 및 관련 소프트웨어 설치/기본 설정
.....


