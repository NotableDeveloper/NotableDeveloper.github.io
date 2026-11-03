---
title: "Real-time으로 Asterisk PJSIP 설정 관리하기"
description: "파일을 직접 수정하고 reload하던 Asterisk 계정 관리를 ODBC와 Sorcery, 데이터베이스 테이블 기반으로 전환하는 흐름을 정리한다."
date: 2026-11-02 23:59:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, realtime, pjsip, odbc, mariadb]
---

Asterisk의 PJSIP 계정을 파일로 관리하면 계정을 추가하거나 변경할 때마다 /etc/asterisk/pjsip.conf를 수정하고 CLI에서 reload를 실행해야 한다. 계정 수가 많아지면 이 작업은 반복적이고 운영하기 어렵다.

Asterisk Realtime을 사용하면 계정 정보를 데이터베이스에 저장하고, 필요한 테이블을 조회하도록 설정할 수 있다. 그러면 계정 등록·수정·삭제를 파일 편집이 아닌 SQL 작업으로 관리할 수 있다.

```text
SQL 변경
   │
   ▼
데이터베이스 테이블 ──▶ ODBC ──▶ Asterisk Sorcery ──▶ PJSIP 객체
```

## ODBC란

ODBC(Open Database Connectivity)는 애플리케이션이 특정 데이터베이스 제품에 종속되지 않고 데이터베이스에 접근할 수 있도록 해주는 표준 인터페이스다. Asterisk는 ODBC를 통해 MariaDB나 PostgreSQL 같은 데이터베이스에 연결하고, DSN에 지정된 드라이버와 접속 정보를 사용해 테이블을 조회한다.

구성 요소의 역할은 다음과 같다.

- DSN: 데이터베이스 주소, 이름, 포트, 사용할 드라이버 같은 접속 정보를 묶은 이름이다.
- ODBC 드라이버: ODBC 요청을 해당 DBMS가 이해하는 방식으로 변환한다.
- Asterisk ODBC 설정: Asterisk가 어떤 DSN과 인증 정보로 연결할지 지정한다.

따라서 Asterisk Realtime에서는 Asterisk가 직접 DBMS별 접속 방식을 구현하는 대신 ODBC를 중간 계층으로 사용한다. 이후 Sorcery가 PJSIP 객체와 데이터베이스 테이블을 연결한다.

## ODBC 연결 정의

/etc/asterisk/res_odbc.conf에 연결 객체를 만든다. 이 객체는 /etc/odbc.ini에 정의한 DSN과 데이터베이스 인증 정보를 참조한다.

```ini
[ENV]
[asterisk]
enabled => yes
dsn => [DSN object name]
username => [Database user name]
password => [Database password]
```

/etc/odbc.ini에는 데이터베이스 주소, 이름, 드라이버, 사용자 정보, 포트를 포함한 DSN을 작성한다.

```ini
[sample]
Driver = MariaDB
Description = MariaDB connection to 'sample' database
Server = 10.10.1.1
Port = 3306
Database = sample
UserName = root
Password = 1234
```

Driver는 /etc/odbcinst.ini에 등록된 드라이버 이름과 연결된다. 이 파일에는 DBMS별 ODBC 드라이버와 라이브러리 경로를 정의한다.

## Sorcery에서 데이터 소스 바꾸기

Sorcery는 Asterisk가 설정 객체를 어디에서 읽고 저장할지 결정하는 추상화 계층이다. 파일 기반 매핑은 다음과 같다.

```ini
[res_pjsip]
endpoint=config,pjsip.conf,criteria=type=endpoint
auth=config,pjsip.conf,criteria=type=auth
aor=config,pjsip.conf,criteria=type=aor
```

Realtime 매핑에서는 각 객체를 데이터베이스 테이블로 연결한다.

```ini
[res_pjsip]
endpoint=realtime,ps_endpoints
auth=realtime,ps_auths
aor=realtime,ps_aors
domain_alias=realtime,ps_domain_aliases
```

즉 endpoint, auth, aor, domain_alias를 각각 지정된 테이블에서 조회하겠다는 의미다.

## 인증 테이블 만들기

예시에서는 인증 객체를 저장할 ps_auths 테이블을 사용한다.

```sql
CREATE TABLE ps_auths (
  id varchar(40) NOT NULL,
  auth_type enum('md5', 'userpass') DEFAULT NULL,
  nonce_lifetime int DEFAULT NULL,
  md5_cred varchar(40) DEFAULT NULL,
  password varchar(80) DEFAULT NULL,
  realm varchar(40) DEFAULT NULL,
  username varchar(40) DEFAULT NULL,
  PRIMARY KEY (id)
);
```

MD5 인증을 사용할 때에는 사용자 이름, Realm, 비밀번호를 조합한 값을 md5_cred에 저장한다.

```sql
INSERT INTO ps_auths
  (id, auth_type, md5_cred, realm, username)
VALUES
  ('7777', 'md5', MD5('7777:sbpdx.sdbx.co.kr:ippbx7777'),
   'sbpdx.sdbx.co.kr', '7777');
```

인증 요청이 들어오면 Asterisk는 같은 알고리즘으로 자격 증명을 계산하고 데이터베이스에 저장된 값과 비교한다. 계정 데이터를 DB에 넣는 것만으로 끝나는 것이 아니라 PJSIP 객체 매핑과 ODBC 연결도 모두 정상이어야 한다.

## 정리

Realtime 구성은 파일을 없애는 기능이 아니라 설정 데이터의 저장소를 데이터베이스로 옮기는 구조다. res_odbc.conf가 연결 정보를 제공하고, sorcery.conf가 PJSIP 객체와 테이블을 연결하며, 실제 계정 정보는 ps_auths 같은 테이블에 저장된다.

