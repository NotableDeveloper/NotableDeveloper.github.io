---
title: "Asterisk로 두 SIP 단말을 연결하는 작은 교환기 만들기"
description: "Asterisk와 PJSIP를 사용해 5001번과 5002번 SIP 단말이 서로 통화하도록 구성하는 과정을 정리한다."
date: 2026-11-02 23:30:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, pjsip, sip, voip]
---

Asterisk를 전화 교환기처럼 사용해 두 개의 SIP 단말을 연결해본다. 이번 구성에서는 5001번과 5002번이라는 두 계정을 만들고, 한 단말에서 다른 단말로 전화를 걸 수 있도록 설정한다.

전체 흐름은 단순하다.

```text
SIP 단말 5001  ◀──── SIP / RTP ────▶  Asterisk  ◀──── SIP / RTP ────▶  SIP 단말 5002
                                      SIP 교환기
```

PJSIP는 단말이 어떤 방식으로 Asterisk에 등록할지 정의한다. Dialplan은 등록된 단말로 전화가 들어왔을 때 Asterisk가 어떤 동작을 수행할지 결정한다.

## PJSIP에서 단말 등록 정의하기

먼저 `/etc/asterisk/pjsip.conf`에 SIP 요청을 받을 transport와 두 단말의 계정을 정의한다.

```ini
[simpletrans]
type=transport
protocol=udp
bind=0.0.0.0:5060

[5001]
type=endpoint
context=simple
disallow=all
allow=ulaw
auth=auth5001
aors=5001

[auth5001]
type=auth
auth_type=userpass
password=5001
username=5001

[5001]
type=aor
max_contacts=1

[5002]
type=endpoint
context=simple
disallow=all
allow=ulaw
auth=auth5002
aors=5002

[auth5002]
type=auth
auth_type=userpass
password=5002
username=5002

[5002]
type=aor
max_contacts=1
```

여기서 transport는 UDP 5060 포트로 들어오는 SIP 요청을 받는다. `endpoint`는 단말의 통화 관련 설정을, `auth`는 인증 정보를, `aor`는 해당 계정에 연결된 실제 단말 정보를 나타낸다.

두 endpoint의 `context`를 모두 `simple`로 지정한 점도 중요하다. 이 값은 나중에 작성할 Dialplan의 컨텍스트 이름과 연결된다.

## Sorcery로 설정 소스 연결하기

Asterisk는 Sorcery라는 추상화 계층을 통해 endpoint, auth, AOR 같은 객체를 여러 저장소에서 읽을 수 있다. 이 구성에서는 파일 설정과 realtime 저장소를 연결하는 규칙을 `/etc/asterisk/sorcery.conf`에 둔다.

```ini
[res_pjsip]
endpoint=realtime,ps_endpoints
endpoint=config,pjsip.conf,criteria=type=endpoint

auth=realtime,ps_auths
auth=config,pjsip.conf,criteria=type=auth

aor=realtime,ps_aors
aor=config,pjsip.conf,criteria=type=aor
```

이 설정은 PJSIP 객체를 조회할 때 realtime 저장소와 정적 설정 파일을 사용할 수 있도록 한다. 이번 예제의 계정 정보는 `pjsip.conf`에 작성했으므로 `config` 경로를 통해 읽힌다.

## 전화번호 동작을 Dialplan으로 정의하기

PJSIP 설정이 단말과 교환기 사이의 연결 방법을 정의한다면, Dialplan은 전화가 걸려왔을 때의 동작을 정의한다. `/etc/asterisk/extensions.conf`에 다음 내용을 추가한다.

```ini
[simple]
exten => _5001,1,Dial(PJSIP/5001)
exten => _5002,1,Dial(PJSIP/5002)
```

`simple`은 앞서 endpoint에 지정한 context 이름이다. 5001로 전화가 들어오면 `PJSIP/5001`로 연결하고, 5002로 전화가 들어오면 `PJSIP/5002`로 연결한다.

번호 규칙이 단순하다면 패턴으로 줄여 쓸 수도 있다.

```ini
[simple]
exten => _5XX.,1,Dial(PJSIP/${EXTEN})
```

이 패턴은 5로 시작하는 번호를 받아 입력된 번호 그대로 PJSIP 대상에게 전달한다. 다만 실제 운영에서는 의도하지 않은 번호까지 연결되지 않도록 허용할 번호 범위를 명확히 제한하는 편이 안전하다.

## 변경한 설정 적용하기

파일을 수정했다고 해서 이미 실행 중인 Asterisk 프로세스가 모든 내용을 즉시 다시 읽는 것은 아니다. 먼저 서비스를 재시작한다.

```bash
sudo systemctl restart asterisk
```

그다음 Asterisk CLI에 접속한다.

```bash
/usr/sbin/asterisk -crvvvvv
```

CLI에서 PJSIP와 Dialplan을 각각 reload한다.

```text
localhost*CLI> pjsip reload
localhost*CLI> dialplan reload
```

Sorcery 설정은 서비스 재시작이 필요한 경우가 있으므로, reload만으로 반영되지 않을 때는 Asterisk를 재시작하는 것이 확실하다.

## 두 단말을 등록하고 통화하기

SIP 클라이언트 두 개를 준비해 하나는 5001, 다른 하나는 5002로 등록한다. PC에서 테스트한다면 MicroSIP 같은 SIP 클라이언트를 사용할 수 있다. 두 클라이언트에는 같은 Asterisk 서버 주소와 각각의 계정 이름, 비밀번호를 입력한다.

등록이 끝난 뒤 CLI에서 연결된 단말을 확인한다.

```text
localhost*CLI> pjsip show contacts

Contact:  5001/sip:5001@<단말 주소>   NonQual
Contact:  5002/sip:5002@<단말 주소>   NonQual
```

두 계정이 보이면 Asterisk가 단말의 등록 정보를 알고 있다는 뜻이다. 이제 5001에서 5002로 전화를 걸면 Dialplan이 `PJSIP/5002`를 호출한다. 반대 방향도 같은 방식으로 처리된다.

```text
-- Executing [5002@simple:1] Dial("PJSIP/5001", "PJSIP/5002")
-- Called PJSIP/5002
-- PJSIP/5002 is ringing
-- PJSIP/5002 answered PJSIP/5001
-- Channel PJSIP/5001 joined the bridge
-- Channel PJSIP/5002 joined the bridge
```

위 로그는 실제 원문 로그를 그대로 옮긴 것이 아니라, 발신·착신·응답·브리지 연결이라는 통화 흐름을 보여주기 위해 재구성한 예시다. 통화가 연결되면 Asterisk는 두 SIP 채널을 하나의 브리지로 묶고 음성 RTP가 오갈 수 있도록 중계한다.

## 마무리

두 SIP 단말로 작은 교환기를 만들려면 역할을 나누어 생각하면 된다.

1. `pjsip.conf`에서 transport, endpoint, 인증, AOR을 정의한다.
2. `sorcery.conf`에서 PJSIP 객체를 읽을 설정 소스를 연결한다.
3. `extensions.conf`에서 번호별 통화 동작을 Dialplan으로 작성한다.
4. Asterisk를 재시작하고 PJSIP와 Dialplan을 reload한다.
5. 두 단말의 등록 상태와 통화 로그를 확인한다.

구성이 작아도 PJSIP와 Dialplan은 서로 다른 책임을 가진다. 단말 등록이 보이지 않으면 PJSIP 인증·AOR·네트워크를 먼저 확인하고, 등록은 되었지만 통화가 연결되지 않으면 context와 Dialplan의 번호 패턴을 확인하면 된다.
