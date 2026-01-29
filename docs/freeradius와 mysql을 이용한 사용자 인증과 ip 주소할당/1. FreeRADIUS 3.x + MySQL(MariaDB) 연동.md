# FreeRADIUS 3.x + MySQL(MariaDB) 연동 기록 (SQL 모듈 / NAS(DB) 등록 / default·inner-tunnel 적용)

> **목표:** FreeRADIUS 3.x에서 인증/정책/로그를 **MySQL(radius DB)** 기반으로 관리하고,  
> (선택) NAS(액세스 스위치)도 **`nas` 테이블**에서 로드하도록 구성한다.

---

## 구성 개요

- FreeRADIUS 3.x (Ubuntu/Debian 패키지 기준)
- MySQL/MariaDB: `radius` 데이터베이스
- SQL 스키마: `schema.sql` (radcheck/radreply/radacct/nas 등)
- 사이트 적용:
  - `sites-enabled/default` : outer(EAP 처리, 로깅/세션 등)
  - `sites-enabled/inner-tunnel` : inner(PEAP/TTLS에서 **실제 계정 검증**)

---

## 1) SQL 스키마 파일 위치 확인

FreeRADIUS 3.x에서 MySQL 스키마/쿼리 파일은 보통 아래 경로에 위치한다.

```bash
cd /etc/freeradius/3.0/mods-config/sql/main/mysql/
ls -l
```

<img width="937" height="227" alt="01_schema_files_list" src="https://github.com/user-attachments/assets/4cdeaec9-810a-4aa5-915c-cb20f4eeeb50" />


---

## 2) `schema.sql`로 radius DB 테이블 생성

이미 `radius` DB와 `radius` 계정(권한 포함)을 준비했다는 전제에서, `schema.sql`을 임포트한다.

```bash
cd /etc/freeradius/3.0/mods-config/sql/main/mysql/
mysql -u radius -p radius < schema.sql
```

<img width="951" height="104" alt="02_import_schema_sql" src="https://github.com/user-attachments/assets/09560f09-dca9-4ec1-9230-06351b9c7334" />


---

## 3) 테이블 생성 확인

```sql
USE radius;
SHOW TABLES;
SELECT USER();
```

<img width="416" height="649" alt="03_show_tables" src="https://github.com/user-attachments/assets/04f4cade-c6f3-442a-9f6e-d97c3af39279" />


> 이 시점부터 FreeRADIUS가 SQL 모드로 동작하면 아래 테이블들을 활용한다:
> - `radcheck` / `radreply` / `radusergroup` : 인증/인가 정책
> - `radacct` : Accounting(접속 기록)
> - `radpostauth` : 인증 성공/실패 로그
> - `nas` : NAS(스위치) 목록 (옵션: DB에서 NAS 읽을 때)

---

## 4) SQL 모듈 설정: `/etc/freeradius/3.0/mods-available/sql`

### 4-1. dialect(=DB 종류) 지정

`sql { }` 블록 **안에서** `dialect`를 `mysql`로 지정한다.

<img width="683" height="515" alt="04_sql_module_dialect" src="https://github.com/user-attachments/assets/918aa883-69fa-4e91-b030-925cbc7db643" />


### 4-2. DB 접속정보 설정 (중요)

아래 항목은 **`sql { }` 블록에 작성**한다.

- `driver` : `rlm_sql_mysql`
- `server` / `port`
- `login` / `password`
- `radius_db`
- (옵션) `read_clients = yes` : NAS를 DB의 `nas` 테이블에서 읽기

> `mysql { }` 블록은 주로 TLS/문자셋 같은 **드라이버 세부 옵션** 영역이고,  
> 접속정보(server/login/password/radius_db)는 보통 `sql { }`에 둔다.

```conf
sql {
    dialect = "mysql"
    driver  = "rlm_sql_mysql"

    server   = "localhost"
    port     = 3306
    login    = "radius"
    password = "<DB_PASSWORD>"
    radius_db = "radius"

    # NAS를 DB(nas 테이블)에서 로드하려면
    read_clients = yes
}
```

<img width="428" height="244" alt="05_sql_module_conninfo" src="https://github.com/user-attachments/assets/5ff4b876-bd7f-486f-80ee-11ef565d28f2" />


---

## 5) SQL 모듈 활성화: `mods-enabled` 심볼릭 링크

FreeRADIUS는 `mods-enabled/`에 존재하는 모듈만 로드한다.  
따라서 `mods-available/sql`을 `mods-enabled/sql`로 **심볼릭 링크**하여 활성화한다.

```bash
ls -l /etc/freeradius/3.0/mods-enabled/ | grep sql

# 없다면 활성화
sudo ln -s ../mods-available/sql /etc/freeradius/3.0/mods-enabled/sql

# 다시 확인
ls -l /etc/freeradius/3.0/mods-enabled/ | grep sql
```

<img width="954" height="209" alt="06_enable_sql_module_symlink" src="https://github.com/user-attachments/assets/f8312220-f3fc-4e1f-acff-ef1c2a2b7676" />


> 출력이 `sql -> ../mods-available/sql` 형태면 **활성화 완료**.

---

## 6) 사이트 적용: default / inner-tunnel

> **핵심 포인트**  
> - `default` = outer 단계(EAP 처리 + 로깅/세션 등)  
> - `inner-tunnel` = inner 단계(PEAP/TTLS에서 **실제 계정 검증**)  
>
> 따라서 **PEAP/TTLS를 쓰면** `inner-tunnel`의 `authorize { sql }`이 사실상 필수다.

### 6-1. `sites-enabled/default`

`authorize`에서 `sql`을 호출하면 SQL 기반 정책/속성을 조회할 수 있다.

<img width="687" height="114" alt="07_default_site_edit" src="https://github.com/user-attachments/assets/a4c8417f-b5a7-4c1f-9982-1422fc446eb0" />

<img width="626" height="176" alt="08_default_authorize_sql" src="https://github.com/user-attachments/assets/18aa7317-b6a5-4050-aec0-1f7e694c4f9a" />


또한 (필요 시) `session`/`post-auth`에서도 SQL을 호출해 다음을 수행할 수 있다.

- `session { sql }` : 동시접속/Simultaneous-Use 체크 (radacct 기반)
- `post-auth { sql }` : 인증 성공/실패를 `radpostauth`에 기록

<img width="801" height="189" alt="09_default_session_sql" src="https://github.com/user-attachments/assets/7fb60d2e-b7bb-4391-8d80-d50108d09c95" />

<img width="274" height="189" alt="10_default_postauth_edit" src="https://github.com/user-attachments/assets/5936be32-1d1e-4e01-ad7d-6930afd2d0f6" />

<img width="165" height="45" alt="11_default_postauth_sql" src="https://github.com/user-attachments/assets/6f89afea-3881-4f74-9a4d-7d743a9a08e1" />


> 운영 관점에서 **반드시 필요한 건 `authorize`(정책 조회)**이고,  
> `session/post-auth`는 요구사항(감사/동시접속 제어)에 따라 선택한다.

### 6-2. `sites-enabled/inner-tunnel` (PEAP/TTLS라면 중요)

PEAP/MSCHAPv2 환경에서 **실제 사용자 계정 조회는 inner-tunnel에서 발생**한다.  
그래서 DB 계정 기반 인증을 하려면 `inner-tunnel`의 `authorize`에도 `sql`을 넣는다.

<img width="743" height="103" alt="12_inner_tunnel_edit" src="https://github.com/user-attachments/assets/d84d0d07-d559-47f3-9194-0290573a251f" />

<img width="317" height="33" alt="13_inner_tunnel_authorize_sql" src="https://github.com/user-attachments/assets/93728a75-c511-4c1b-b7d7-c87b45eb6bfe" />


---

## 7) NAS(스위치) 등록: 파일 기반 vs DB 기반

NAS 등록은 **두 가지 방식** 중 하나를 선택한다.

### A) 파일 기반: `clients.conf`

```conf
# /etc/freeradius/3.0/clients.conf
client access_switch_2 {
    ipaddr = 10.0.11.5
    secret = <RADIUS_SHARED_SECRET>
    shortname = ASW2
    nastype = cisco
}
```

<img width="712" height="381" alt="14_clients_conf_example" src="https://github.com/user-attachments/assets/68c49ceb-af51-42e9-8efb-be902b2deb18" />


### B) DB 기반: `nas` 테이블 (read_clients=yes일 때)

`mods-available/sql`에서 `read_clients = yes`로 설정했다면,  
NAS를 DB의 `nas` 테이블에 등록해도 된다.

```sql
INSERT INTO nas (nasname, shortname, type, ports, secret, description)
VALUES ('10.0.11.5', 'ASW2', 'cisco', 1812, '<RADIUS_SHARED_SECRET>', 'Company');

SELECT * FROM nas;
```

<img width="969" height="370" alt="15_nas_table_insert" src="https://github.com/user-attachments/assets/87d75271-c943-4173-9d37-2a4387be6599" />


> `secret`는 **스위치의 RADIUS shared secret**과 반드시 동일해야 한다.  
> (사용자 비밀번호/DB 비밀번호가 아님)

---

### 참고
- FreeRADIUS 3.x는 `mods-available/`(설정 원본) + `mods-enabled/`(활성화 링크) 구조를 사용한다.
- `sites-available/`와 `sites-enabled/`도 동일하게 **링크 기반 활성화** 구조다.
