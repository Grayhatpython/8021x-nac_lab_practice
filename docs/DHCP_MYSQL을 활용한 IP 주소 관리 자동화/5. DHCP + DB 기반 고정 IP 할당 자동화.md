# DHCP + DB 기반 고정 IP 할당 자동화 (db_to_dhcp_sync)

> **목표:** DB에 저장된 “서브넷(스코프) 정보 + IP 풀 + MAC 바인딩”을 기준으로 `isc-dhcp-server` 설정을 **자동 생성/갱신**하고, 단말에 **고정 IP(Reservation)** 를 배포한다.  
> **방식:** Shell Script + MySQL + cron(주기 실행)

---

## 0. 작업 요약

- DB에 **VLAN/서브넷 프로파일(스코프)** 과 **IP 풀/바인딩(고정 IP)** 정보를 저장
- Shell Script로 DB를 조회해 `dhcpd.conf`를 **자동 생성**
- VLAN별로 `range` 생성 방식을 분기(연속 range vs IP 풀 기반 range 나열)
- MAC이 등록된 항목은 `host { hardware ethernet ...; fixed-address ...; }` 형태로 **예약(Reservation)** 생성
- 생성된 설정을 `dhcpd`에 반영 후 서비스 재시작
- 실제로 `/var/lib/dhcp/dhcpd.leases`에 **예약 IP가 lease로 기록**되는 것까지 확인

---

## 1. 구성(환경) / 역할

- **OS:** Ubuntu (DHCP 서버)
- **DHCP:** `isc-dhcp-server`
- **DB:** MySQL
- **자동화 스크립트:** `/root/radius/db_to_dhcp_sync.sh` (예시 경로)
- **출력 파일**
  - DHCP 헤더 템플릿: `dhcpd_head.conf`
  - DHCP 바디 생성물: `dhcpd_body.conf` (subnet/host 블록)
  - 최종 반영 파일: `/etc/dhcp/dhcpd.conf`
  - Lease 확인 파일: `/var/lib/dhcp/dhcpd.leases`

---

## 2. DB에 어떤 데이터를 넣어두는가?

핵심은 “DHCP 설정에 필요한 정보를 DB로 정규화” 해두고, 스크립트가 그걸 읽어서 **설정 파일을 조립**하는 구조다.

### 2.1 VLAN/서브넷 프로파일(스코프) 테이블

스샷 기준으로 VLAN 300~340, 999(test) 등의 프로파일이 있고,
각 프로파일은 다음 같은 정보를 가진다.

- `id`(또는 `vlan_id`) / 프로파일명
- `network_addr`(subnet) / `subnet_mask`(netmask)
- `gateway_addr`
- `dns` / `domain_name` 등 옵션
- `ip_start` / `ip_end` (연속 range가 필요한 VLAN 용)
- `default_lease_time` / `max_lease_time`

📌 DB에 저장된 프로파일 목록 예시  
<img width="980" height="573" alt="01_db_vlan_profiles" src="https://github.com/user-attachments/assets/4299d0b5-0ab6-442a-8f6a-de0a50fd26a1" />


---

### 2.2 IP 풀 + MAC 바인딩 테이블

- `ipaddr`(또는 `ip_addr`)
- `macaddr`(또는 `mac_addr`)
- `vlan_id`
- `status`(또는 `ip_alloc_state`)  
  - 예: `U`(사용 가능/대기), `A`(할당 상태) 등

📌 VLAN 999 구간에서 특정 IP에 MAC이 매핑된 예시(고정 IP 후보)  
<img width="958" height="517" alt="02_db_ip_pool" src="https://github.com/user-attachments/assets/3dc00167-4c3e-457d-99db-cb7a48d6a7e6" />


---

## 3. 설정 파일 생성 구조

### 3.1 헤더 템플릿(`dhcpd_head.conf`)

공통 설정(deny bootp, authoritative, ddns-update-style, 기본 lease 등)을 헤더로 두고,  
서브넷/host 블록은 스크립트가 생성해서 뒤에 붙인다.

<img width="753" height="413" alt="03_template_dhcpd_head" src="https://github.com/user-attachments/assets/f7bd9cce-6d57-44bd-b349-e5bfa478f30e" />


---

### 3.2 바디 생성물(`dhcpd_body.conf`)

각 VLAN/서브넷마다 `subnet { ... }` 블록이 생성된다.

- `option routers`
- `option subnet-mask`
- `option domain-name-servers`
- `default-lease-time`, `max-lease-time`
- `range` 생성(연속 range 또는 풀 기반 나열)

<img width="806" height="639" alt="04_generated_dhcpd_body_subnets" src="https://github.com/user-attachments/assets/e2f4751b-3c62-49fe-92e9-ee2c72162ae6" />


---

## 4. 고정 IP(Reservation) 생성 방식

### 4.1 `host` 블록 생성 규칙

DB에서 **MAC이 등록된 레코드**를 뽑아 아래 형태로 만든다.

```conf
host <mac_underscore_vlan> {
    hardware ethernet <mac>;
    fixed-address <ip>;
}
```

VLAN 999에서 `aa:bb:cc:11:22:33` → `192.168.99.15`가 예약으로 생성된 예시:

<img width="688" height="429" alt="05_generated_vlan999_host_reservation" src="https://github.com/user-attachments/assets/69850544-193c-4d42-9db7-e43d41c30af6" />


---

## 5. 실제 적용 결과 확인

### 5.1 Lease 파일 반영 확인

DHCP 데몬이 정상 동작하면 `/var/lib/dhcp/dhcpd.leases`에 해당 IP/MAC lease가 기록된다.

<img width="788" height="421" alt="06_dhcpd_leases_result" src="https://github.com/user-attachments/assets/02cebf16-60b9-4411-bce5-af69b48b3cda" />

---

## 6. 스크립트 구현 포인트(핵심 로직)

아래는 오늘 작업에서 중요했던 “구현 포인트”를 스크린샷 기반으로 정리한 것이다.

### 6.1 here-doc 기반 SQL 실행 + 결과를 변수로 받기

`RAW_SCOPE_OUTPUT=$( mysql ... <<'SQL' ... SQL )`

- 종료 토큰(`SQL`)은 반드시 **라인 맨 앞**에 단독으로 있어야 함
- `$( ... )` 괄호 닫힘 누락 시 `unexpected EOF` 발생

<img width="976" height="174" alt="07_script_raw_scope_query" src="https://github.com/user-attachments/assets/56988cb7-ea0d-4afd-945d-c53d49780fd8" />


---

### 6.2 `printf` 기반으로 subnet 블록 조립

- `printf`로 줄/탭/공백을 통제하면 출력 포맷이 안정적
- 옵션/lease/range를 서브넷 블록 안에 순서대로 출력

<img width="951" height="383" alt="08_script_scope_printf_generator" src="https://github.com/user-attachments/assets/be63685a-83ad-4838-aa7e-cba169a729d8" />


---

### 6.3 host reservation 생성 + 2T/1T 토큰 치환

레거시 구현에서 SQL 결과를 한 줄 문자열로 만든 뒤 토큰을 치환하는 방식이 있었고,
이 때 `2T/1T`의 **대소문자 불일치(2t)** 로 치환이 안 되던 문제가 있었다.

- 해결: 토큰을 대소문자 통일 또는 sed에서 `[Tt]`로 처리

<img width="822" height="515" alt="09_script_raw_host_query_sed_tokens" src="https://github.com/user-attachments/assets/63de65c5-d459-4775-bf2e-7c76c180b7f5" />


---

### 6.4 Lease 텍스트 생성(분석/검증용)

DB에 있는 MAC 바인딩을 기반으로 lease 스타일 텍스트를 만드는 로직.
(실운영에서는 `dhcpd.leases`를 직접 생성하기보다는 dhcpd가 관리하는 것이 일반적이라, 이 부분은 참고/분석 목적)

<img width="959" height="315" alt="10_script_raw_lease_query" src="https://github.com/user-attachments/assets/86fa5048-3c07-4317-89a3-b742073894e9" />


<img width="637" height="295" alt="11_script_lease_printf_generator" src="https://github.com/user-attachments/assets/2917b514-4e2b-4dbc-a285-c85bb91868a9" />


---

### 6.5 최종 반영 + 서비스 재시작

- 헤더 + 바디를 합쳐 `/etc/dhcp/dhcpd.conf`를 갱신
- 필요 시 `isc-dhcp-server stop/start`로 반영

<img width="566" height="182" alt="12_script_apply_restart" src="https://github.com/user-attachments/assets/105b6b12-37dd-4191-b9fc-c1f502b61780" />


---

## 7. 운영 적용 팁

### 7.1 적용 전 검증 (권장)
```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl status isc-dhcp-server --no-pager
```

### 7.2 cron 등록 예시(`/etc/crontab`)
매시 1분 실행 + 로그 저장:

```cron
1 * * * * root /bin/sh /root/radius/db_to_dhcp_sync.sh >> /var/log/db_to_dhcp_sync.log 2>&1
```
