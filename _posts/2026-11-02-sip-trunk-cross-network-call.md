---
title: "SIP-Trunk로 서로 다른 네트워크 간 통화 연결하기"
description: "두 Asterisk 서버를 SIP-Trunk로 연결하고 5000번대와 7000번대 번호를 서로 다른 네트워크로 라우팅하는 구성을 정리한다."
date: 2026-11-02 23:50:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, sip, pjsip, sip-trunk, voip, networking]
---

서로 다른 네트워크에 있는 VoIP 전화기끼리 통화하려면 전화기만 연결해서는 부족하다. 각 전화기는 자신이 속한 Asterisk 서버에 등록되어 있고, 두 Asterisk 서버도 서로 SIP 요청을 전달할 수 있어야 한다.

이번 구성에서는 왼쪽 네트워크에 5001번 전화기와 Asterisk 서버를 두고, 오른쪽 네트워크에 7001번 전화기와 다른 Asterisk 서버를 둔다.

```text
5001 전화기 ──▶ Asterisk A ── SIP-Trunk ──▶ Asterisk B ──▶ 7001 전화기
     ◀────────── SIP/RTP ◀─────── SIP/RTP ──────── SIP/RTP ────◀
```

5001번 전화기는 7001번 전화기의 IP 주소를 직접 알 필요가 없다. 5001번의 요청은 먼저 자신이 등록된 Asterisk A로 전달되고, Asterisk A의 Dialplan이 7000번대 번호를 SIP-Trunk로 보낸다. Asterisk B는 요청을 받아 자신의 Dialplan에 따라 7001번으로 연결한다.

## 로컬 전화기 계정 등록하기

먼저 각 Asterisk 서버에 자신이 관리할 전화기 계정을 등록한다. 왼쪽 서버에서는 5001번을, 오른쪽 서버에서는 같은 방식으로 7001번을 등록한다.

왼쪽 서버의 `/etc/asterisk/pjsip.conf` 예시는 다음과 같다.

```ini
[simple-transport]
type=transport
protocol=udp
bind=0.0.0.0:5060

[5001]
type=endpoint
context=5XXX_default
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
qualify_frequency=30
max_contacts=1
```

`5001` endpoint의 `context`는 전화기가 발신할 때 사용할 Dialplan 컨텍스트다. `auth`는 전화기 인증 정보를 연결하고, `aor`는 해당 계정에 등록될 실제 단말의 위치를 관리한다.

오른쪽 서버에도 같은 구조로 `7001` endpoint와 인증 객체, AOR을 만든다. 양쪽 계정의 이름과 비밀번호는 각 서버의 Dialplan과 전화기 설정에 맞게 일치해야 한다.

## 로컬 번호 Dialplan 작성하기

왼쪽 서버의 `/etc/asterisk/extensions.conf`에는 우선 로컬 번호를 처리할 규칙을 작성한다.

```ini
[5XXX_default]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup()
```

이 규칙은 5000번대 번호를 입력받으면 같은 번호의 PJSIP endpoint를 호출한다. 예를 들어 5001번에서 5002번으로 전화하면 왼쪽 Asterisk가 `PJSIP/5002`를 호출한다.

## 상대 Asterisk를 Trunk로 등록하기

이제 두 Asterisk 서버가 서로를 SIP 계정처럼 사용할 수 있도록 설정한다. 서버와 서버 사이에서 통화를 전달하기 위해 만든 endpoint를 일반적으로 SIP-Trunk라고 부른다.

왼쪽 서버의 `/etc/asterisk/pjsip.conf`에 오른쪽 Asterisk를 등록한다.

```ini
[Another_Trunk]
type=endpoint
transport=simple-transport
context=Another_context
disallow=all
allow=ulaw
aors=Another_Trunk

[Another_Trunk]
type=aor
max_contacts=1
qualify_frequency=30
contact=sip:<Another Asterisk Server IP>:<Port>

[Another_Trunk]
type=identify
endpoint=Another_Trunk
match=<Another Asterisk Server IP>
```

여기서는 세 가지 PJSIP 객체를 사용한다.

- `endpoint`: 상대 서버와 통신할 프로토콜, 코덱, Dialplan 컨텍스트를 정의한다.
- `aor`: 상대 서버로 요청을 보낼 주소를 정의한다.
- `identify`: 특정 IP 주소에서 들어온 SIP 요청을 `Another_Trunk` endpoint와 연결한다.

일반 전화기 등록에서는 사용자 이름과 비밀번호를 이용해 인증하지만, 이 예제의 서버 간 연결은 `identify`의 IP 주소를 기준으로 상대를 식별한다. 따라서 `match`에 지정한 주소가 아닌 곳에서 들어온 요청을 허용하지 않도록 하고, 방화벽에서도 SIP 포트를 양쪽 서버 주소로 제한해야 한다.

## Trunk 수신용 컨텍스트 만들기

상대 서버에서 들어온 요청을 처리할 Dialplan도 필요하다. `context=Another_context`로 지정했으므로 같은 이름의 컨텍스트를 `extensions.conf`에 추가한다.

```ini
[Another_context]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup()
```

오른쪽 Asterisk 입장에서는 왼쪽 서버가 보낸 7001번 통화 요청을 받아야 하므로, 오른쪽 서버에서는 `7001`에 맞는 endpoint와 수신 Dialplan을 준비한다. 번호 대역은 실제 사용하는 계정 체계에 맞게 조정해야 한다.

## 번호 대역을 Trunk로 라우팅하기

왼쪽 서버의 기존 컨텍스트에 7000번대 번호에 대한 외부 라우팅 규칙을 추가한다.

```ini
[5XXX_default]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup()

exten => _7XXX,1,Dial(PJSIP/${EXTEN}@Another_Trunk)
same => n,Hangup()

[Another_context]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup()
```

이제 5000번대는 기존처럼 왼쪽 네트워크 내부로 전달되고, 7000번대는 `Another_Trunk`를 통해 오른쪽 Asterisk로 전달된다.

통화 요청이 이동하는 과정은 다음과 같다.

1. 5001번 전화기가 7001번으로 발신한다.
2. 5001번 전화기는 자신의 Asterisk A로 SIP 요청을 보낸다.
3. Asterisk A는 `5XXX_default`에서 7000번대 규칙을 찾는다.
4. Asterisk A는 요청을 `Another_Trunk` 주소로 전달한다.
5. Asterisk B는 `Another_context`에서 7001번을 찾는다.
6. Asterisk B는 PJSIP를 통해 7001번 전화기를 호출한다.
7. 응답이 완료되면 두 Asterisk 서버를 거쳐 음성 RTP가 전달된다.

```text
5001
  │ INVITE 7001
  ▼
Asterisk A
  │ Dial(PJSIP/7001@Another_Trunk)
  ▼
Asterisk B
  │ Dial(PJSIP/7001)
  ▼
7001
```

## 설정 적용하기

두 서버에서 PJSIP 설정과 Dialplan을 다시 읽는다.

```text
pjsip reload
dialplan reload
```

설정이 반영되지 않거나 transport와 endpoint를 처음 추가한 경우에는 Asterisk 서비스를 재시작해야 할 수 있다.

```bash
sudo systemctl restart asterisk
```

## 통화 상태 확인하기

통화가 연결되면 각 Asterisk의 CLI에서 발신 번호, Trunk endpoint, 착신 번호를 확인할 수 있다. 다음은 발신 서버와 수신 서버의 로그 이미지에 나온 핵심 순서를 옮긴 것이다. 주소와 동적 채널 식별자는 생략했다.

```text
발신 서버: 5001 → PJSIP/7001@Another_Trunk 호출
수신 서버: Another_Trunk → PJSIP/7001 호출
PJSIP/7001 is ringing
PJSIP/7001 answered
발신·착신 채널이 브리지에 연결됨
```

첫 번째 Asterisk에서 `PJSIP/7001@Another_Trunk`가 호출되고, 두 번째 Asterisk에서는 수신 컨텍스트의 `PJSIP/7001`이 호출되는 것이 핵심이다. Trunk 호출이 보이지 않으면 발신 서버의 번호 패턴과 endpoint 이름을 먼저 확인한다. 요청이 상대 서버까지 도달하지만 전화기가 울리지 않으면 수신 서버의 `identify`, context, 착신 번호 규칙을 확인한다.

## 서로 다른 네트워크에서 확인할 항목

서버가 서로 다른 네트워크에 있으면 SIP 설정 외에도 네트워크 조건을 함께 확인해야 한다.

- 두 서버가 서로의 SIP 포트에 접근할 수 있어야 한다.
- `contact`에 입력한 주소와 포트가 실제 상대 서버 주소와 일치해야 한다.
- `identify.match`가 상대 서버의 실제 송신 IP를 포함해야 한다.
- 방화벽과 NAT가 SIP 신호 및 RTP 포트를 차단하지 않아야 한다.
- 외부에 노출되는 Trunk에는 허용 IP 제한과 적절한 인증 방식을 적용해야 한다.
- SIP 신호가 연결되더라도 RTP 경로가 막히면 통화는 연결되고 음성만 들리지 않을 수 있다.

## 마무리

SIP-Trunk 연동은 두 Asterisk 서버가 서로를 하나의 통화 대상처럼 사용하도록 만드는 구성이다.

1. 각 Asterisk에 로컬 전화기 계정을 등록한다.
2. 각 서버의 로컬 번호를 Dialplan에 등록한다.
3. 상대 Asterisk를 endpoint, AOR, identify 객체로 등록한다.
4. 상대 서버에서 들어온 요청을 처리할 context를 만든다.
5. 번호 대역에 따라 로컬 endpoint 또는 Trunk로 라우팅한다.
6. SIP 신호와 RTP가 양쪽 네트워크를 통과하는지 확인한다.

전화기는 상대 전화기의 IP를 직접 알지 못해도 된다. 자신의 Asterisk에만 요청을 보내고, 두 서버 사이의 경로 선택과 전달은 SIP-Trunk와 Dialplan이 담당한다.
