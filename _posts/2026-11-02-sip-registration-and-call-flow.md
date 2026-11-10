---
title: "SIP 패킷으로 살펴보는 전화기 등록과 통화 흐름"
description: "Asterisk에 SIP 전화기를 등록할 때와 통화를 시작할 때 오가는 패킷을 REGISTER, Digest 인증, INVITE, SDP 중심으로 정리한다."
date: 2026-11-02 23:45:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, sip, pjsip, voip, packet]
---

Asterisk에 VoIP 전화기를 연결하면 내부적으로 여러 SIP 패킷이 오간다. 화면에서 계정 등록 버튼을 누르고 통화 버튼을 누르는 동안 실제로는 등록 요청, 인증 요청, 세션 협상 요청과 응답이 순서대로 교환된다.

이번 글에서는 5001번 전화기가 Asterisk에 등록되는 과정과 5001번에서 5002번으로 통화를 시작하는 과정을 패킷 흐름으로 나누어 살펴본다.

```text
5001 전화기  ── REGISTER ──▶  Asterisk
5001 전화기  ◀─ 401 인증 ──  Asterisk
5001 전화기  ── 인증 REGISTER ▶ Asterisk
5001 전화기  ◀─ 200 OK ─────  Asterisk

5001 전화기  ── INVITE ─────▶  Asterisk  ─────▶  5002 전화기
             ◀─ 100/180/200 ─           ◀───── 응답
```

## SIP 패킷 로그 활성화하기

Asterisk CLI에서 특정 전화기와 주고받는 SIP 메시지를 확인하려면 해당 호스트에 대한 PJSIP 로거를 활성화한다.

```text
pjsip set logger host <VoIP IP Address>
```

이후 지정한 IP 주소에서 Asterisk로 요청이 들어오거나 Asterisk가 응답을 보내면 SIP 메시지의 시작과 본문을 CLI에서 확인할 수 있다. 테스트가 끝나면 다음 명령으로 로거를 끌 수 있다.

```text
pjsip set logger off
```

## 전화기 최초 등록

SIP 전화기에는 보통 계정 이름, 비밀번호, SIP 서버 주소를 입력한다. 전화기가 서버에 자신의 위치를 알리고 통화를 받을 수 있는 상태가 되려면 먼저 `REGISTER` 요청을 보내야 한다.

### 첫 번째 REGISTER

전화기가 보내는 첫 요청은 `REGISTER`다. 이 요청에는 등록할 SIP 주소, 전화기의 현재 접속 주소, 요청을 식별하기 위한 정보가 포함된다.

```text
<--- Received SIP request (539 bytes) from UDP:<IP:PORT> --->
REGISTER sip:<Asterisk IP> SIP/2.0
Via: SIP/2.0/UDP <IP:PORT>;rport;branch=<branch-id>
Max-Forwards: 70
From: "5001" <sip:5001@<Asterisk IP>>;tag=<tag>
To: "5001" <sip:5001@<Asterisk IP>>
Call-ID: <registration-call-id>
CSeq: 41892 REGISTER
User-Agent: MicroSIP/3.21.4
Contact: "5001" <sip:5001@<IP:PORT>;ob>
Expires: 300
Allow: PRACK, INVITE, ACK, BYE, CANCEL, UPDATE, INFO, SUBSCRIBE,
       NOTIFY, REFER, MESSAGE, OPTIONS
Content-Length: 0
```

`REGISTER`는 통화를 요청하는 메시지가 아니다. 전화기가 어느 주소에서 연결되어 있는지 Asterisk에 등록하는 메시지다. `Contact`에 적힌 주소를 통해 Asterisk는 이후 해당 계정으로 통화를 전달할 위치를 알 수 있다.

## Digest 인증 과정

Asterisk는 첫 REGISTER를 바로 허용하지 않고 인증 정보를 요구한다. 이때 `401 Unauthorized` 응답을 보낸다. 여기서 401은 등록이 영구적으로 거부되었다는 뜻이 아니라, 인증 없이 보낸 첫 요청에 대해 인증 절차를 시작하라는 의미다.

```text
<--- Transmitting SIP response (570 bytes) to UDP:<IP:PORT> --->
SIP/2.0 401 Unauthorized
Via: SIP/2.0/UDP <IP:PORT>;rport=<port>;received=<Asterisk IP>;branch=<branch-id>
Call-ID: <registration-call-id>
From: "5001" <sip:5001@<Asterisk IP>>;tag=<tag>
To: "5001" <sip:5001@<Asterisk IP>>;tag=<server-tag>
CSeq: 41892 REGISTER
WWW-Authenticate: Digest realm="asterisk", nonce="<nonce>",
                  opaque="<opaque>", algorithm=MD5, qop="auth"
Server: Asterisk PBX 18.22.0
Content-Length: 0
```

응답의 `WWW-Authenticate`에는 클라이언트가 인증 응답을 계산할 때 사용할 값들이 들어 있다. 중요한 값은 다음과 같다.

- `realm`: 인증 영역을 나타낸다.
- `nonce`: 서버가 생성한 일회성 값이다.
- `opaque`: 서버가 전달한 상태 정보다.
- `algorithm`: 응답값 계산에 사용할 알고리즘이다.
- `qop`: 인증 품질 옵션이다.

전화기는 계정 이름과 비밀번호, 서버가 보낸 challenge 값을 사용해 `response` 값을 계산한다. 비밀번호를 그대로 패킷에 넣어 보내는 방식이 아니라, 서버가 제시한 값과 결합해 Digest 응답을 만드는 방식이다.

### 인증 정보를 포함한 두 번째 REGISTER

전화기는 인증 정보를 포함해 같은 등록 요청을 다시 보낸다. 요청의 `CSeq`는 증가하고 `Authorization` 헤더가 추가된다.

```text
<--- Received SIP request (827 bytes) from UDP:<IP:PORT> --->
REGISTER sip:<Asterisk IP> SIP/2.0
Via: SIP/2.0/UDP <IP:PORT>;rport;branch=<new-branch-id>
Max-Forwards: 70
From: "5001" <sip:5001@<Asterisk IP>>;tag=<tag>
To: "5001" <sip:5001@<Asterisk IP>>
Call-ID: <registration-call-id>
CSeq: 41893 REGISTER
User-Agent: MicroSIP/3.21.4
Contact: "5001" <sip:5001@<IP:PORT>;ob>
Expires: 300
Authorization: Digest username="5001", realm="asterisk",
               nonce="<nonce>", uri="sip:<Asterisk IP>",
               response="<digest-response>", algorithm=MD5,
               cnonce="<client-nonce>", opaque="<opaque>",
               qop=auth, nc=00000001
Content-Length: 0
```

Asterisk도 같은 규칙으로 기대하는 `response` 값을 계산한 뒤 전화기가 보낸 값과 비교한다. 두 값이 일치하면 인증이 성공한다.

```text
<--- Transmitting SIP response (521 bytes) to UDP:<IP:PORT> --->
SIP/2.0 200 OK
Via: SIP/2.0/UDP <IP:PORT>;rport=<port>;received=<Asterisk IP>;branch=<branch-id>
Call-ID: <registration-call-id>
From: "5001" <sip:5001@<Asterisk IP>>;tag=<tag>
To: "5001" <sip:5001@<Asterisk IP>>;tag=<server-tag>
CSeq: 41893 REGISTER
Date: Wed, 04 Sep 2024 06:22:09 GMT
Contact: <sip:5001@<IP:PORT>;ob>;expires=299
Expires: 300
Server: Asterisk PBX 18.22.0
Content-Length: 0
```

`200 OK`가 반환되면 전화기 등록이 완료된 것이다. 이후 전화기는 만료 시간이 끝나기 전에 다시 REGISTER를 보내 등록 상태를 갱신한다.

## Call-ID로 요청과 응답 묶기

SIP 메시지에는 `Call-ID`가 들어간다. 이름 때문에 음성 통화에만 사용되는 값처럼 보이지만, 등록 과정에서도 요청과 응답을 한 쌍으로 묶는 식별자로 사용된다.

같은 흐름에 속한 메시지는 같은 `Call-ID`를 공유한다. 반면 새로운 등록이나 새로운 통화가 시작되면 별도의 `Call-ID`가 만들어진다. 패킷을 분석할 때 `Call-ID`를 먼저 기준으로 삼으면 여러 전화기가 동시에 통신하는 환경에서도 하나의 흐름을 분리하기 쉽다.

## 5001번에서 5002번으로 통화 시작하기

두 전화기가 등록된 뒤 5001번에서 5002번으로 전화를 걸면 `INVITE` 요청이 시작된다. `INVITE`는 상대방을 멀티미디어 세션에 초대하는 메시지다.

```text
Received SIP request from UDP:<5001-ip>:<port>
INVITE sip:5002@<asterisk-ip> SIP/2.0
From: "5001" <sip:5001@<asterisk-ip>>
To: <sip:5002@<asterisk-ip>>
Call-ID: <call-id>
CSeq: 32111 INVITE
Contact: <sip:5001@<5001-ip>:<port>>
Content-Type: application/sdp
Content-Length: 343

v=0
o=- <session-id> <session-version> IN IP4 <5001-ip>
s=pjmedia
m=audio 4002 RTP/AVP 8 0 101
a=rtpmap:8 PCMA/8000
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=sendrecv
```

이 요청에는 SIP 제어 정보뿐 아니라 SDP도 포함된다. SDP는 음성 데이터를 어떤 주소와 포트로 주고받을지, 어떤 코덱을 사용할지 제안한다. 예시에서 `m=audio`는 오디오 포트와 전송 프로파일을, `a=rtpmap`은 코덱과 샘플링 레이트를 나타낸다.

## 통화 요청 인증과 응답

Asterisk는 INVITE에도 인증을 요구할 수 있다. 이 경우 첫 INVITE에 대해 401 응답을 보낸다.

```text
Transmitting SIP response to UDP:<5001-ip>:<port>
SIP/2.0 401 Unauthorized
Call-ID: <call-id>
CSeq: 32111 INVITE
WWW-Authenticate: Digest realm="asterisk",
                 nonce="<invite-nonce>",
                 algorithm=MD5,
                 qop="auth"
```

5001번 전화기는 ACK로 401 응답을 확인한 뒤, 인증 정보가 추가된 INVITE를 다시 보낸다. 이때 같은 `Call-ID`를 유지하면서 `CSeq`의 번호와 메서드 순서가 바뀐다.

```text
Received SIP request from UDP:<5001-ip>:<port>
INVITE sip:5002@<asterisk-ip> SIP/2.0
Call-ID: <call-id>
CSeq: 32112 INVITE
Authorization: Digest username="5001",
                      realm="asterisk",
                      nonce="<invite-nonce>",
                      uri="sip:5002@<asterisk-ip>",
                      response="<calculated-response>",
                      algorithm=MD5,
                      qop=auth,
                      nc=00000001
Content-Type: application/sdp
```

인증이 끝나면 Asterisk는 통화 처리 상태를 단계별 응답으로 알린다.

```text
SIP/2.0 100 Trying
SIP/2.0 180 Ringing
SIP/2.0 200 OK
Content-Type: application/sdp

v=0
o=- <asterisk-session-id> <version> IN IP4 <asterisk-ip>
s=Asterisk
m=audio <asterisk-rtp-port> RTP/AVP 0 101
a=rtpmap:0 PCMU/8000
a=rtpmap:101 telephone-event/8000
a=sendrecv
```

`100 Trying`은 요청을 처리 중이라는 의미다. `180 Ringing`은 착신 단말에 벨을 울리는 단계다. 5002번이 전화를 받으면 `200 OK`와 함께 Asterisk가 사용할 SDP를 보낸다. 양쪽이 SDP 정보를 교환하고 ACK까지 완료하면 SIP 세션이 성립한다.

그 이후의 음성 데이터는 SIP 메시지 자체가 아니라 SDP로 협상한 주소와 포트를 사용하는 RTP 패킷으로 전달된다. SIP는 통화의 시작과 종료, 세션 변경 같은 제어를 담당하고 RTP는 실제 음성 데이터를 운반한다.

## 패킷을 볼 때 확인할 순서

SIP 로그를 처음 볼 때는 메시지 전체를 한 번에 해석하기보다 다음 순서로 확인하면 된다.

1. `REGISTER`가 서버에 도착했는지 확인한다.
2. `401 Unauthorized` 뒤에 `Authorization`이 추가된 요청이 다시 왔는지 확인한다.
3. 인증 후 `200 OK`가 반환되었는지 확인한다.
4. 통화에서는 `INVITE`와 `CSeq` 변화를 확인한다.
5. `Content-Type: application/sdp`가 있는지 확인하고 미디어 주소, 포트, 코덱을 읽는다.
6. `100 Trying`, `180 Ringing`, `200 OK` 순서로 통화 상태가 진행되는지 확인한다.
7. SIP 협상 이후 RTP가 실제로 협상된 포트로 흐르는지 확인한다.

## 마무리

SIP 전화기 등록은 단순히 계정 정보를 한 번 보내는 동작이 아니다. 전화기는 REGISTER를 보내고, Asterisk는 인증 challenge를 반환하며, 전화기는 계산한 Digest 응답을 포함해 다시 요청한다. 검증이 끝나야 `200 OK`와 함께 등록이 완료된다.

통화도 같은 구조로 진행된다. `INVITE`가 세션을 제안하고, 인증이 필요하면 다시 INVITE를 보내며, 응답 코드와 SDP를 통해 상대방과 통화 조건을 맞춘다. 이 흐름을 알고 있으면 SIP 로그에서 문제가 인증 단계에 있는지, 통화 라우팅에 있는지, SDP나 RTP 협상에 있는지 구분할 수 있다.
