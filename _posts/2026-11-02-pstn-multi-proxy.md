---
title: "PSTN망과 통신하기 위한 다중 Proxy 서버 구성"
description: "5001번과 7001번 내부 서버, PSTN 연동 서버를 연결하고 사설망·공인망의 경계에 맞춰 통화 라우팅을 구성하는 방법을 정리한다."
date: 2026-11-03 00:20:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, pjsip, pstn, proxy, dialplan, voip]
---

이번 구성의 목표는 내부 전화망과 외부 PSTN망 사이에 통화 경로를 만드는 것이다. 5001번 전화기가 등록된 서버와 7001번 전화기가 등록된 서버는 앞서 만든 SIP-Trunk 환경과 같은 방식으로 준비하고, 여기에 PSTN과 직접 통신하는 Proxy 서버를 추가한다.

```text
5001 전화기 ──▶ 5001 서버 ──▶ PSTN 연동 Proxy ──▶ PSTN망
7001 전화기 ──▶ 7001 서버 ──▶ PSTN 연동 Proxy ──▶ PSTN망
PSTN망 ──▶ PSTN 연동 Proxy ──▶ 내부 서버의 Dialplan ──▶ 5001 또는 7001
```

핵심은 각 서버에 상대방을 PJSIP 계정으로 등록하고, 등록된 상대 endpoint를 Dialplan에서 목적지별로 호출하는 것이다.

## 서버별 사용자 계정 등록

사용자 계정 정보는 이전에 Proxy 서버끼리 서로의 계정을 /etc/asterisk/pjsip.conf에 등록했던 방식과 유사하게 작성한다. 다만 주소의 종류를 구분해야 한다.

내부 사설 네트워크에 있는 서버끼리는 사설 IP를 사용한다. 반면 PSTN망과 연결되는 서버의 계정은 외부 네트워크에서 도달할 수 있는 PSTN 연결용 공인 IP를 사용해야 한다.

```text
5001번 서버
  5001번 전화기
  7001번 서버
  PSTN 연동 서버 (같은 내부망이므로 사설 IP)

7001번 서버
  7001번 전화기
  5001번 서버
  PSTN 연동 서버 (같은 내부망이므로 사설 IP)

PSTN 연동 서버
  5001번 서버
  7001번 서버
  PSTN망 (다른 네트워크이므로 공인 IP)
```

각 서버가 등록해야 하는 계정은 로컬 전화기, 상대 Asterisk 서버, PSTN Peer로 나뉜다.

## 서버별 Dialplan 역할

계정 등록만으로는 통화가 전달되지 않는다. 각 서버는 번호 대역에 따라 어느 endpoint로 전화를 넘길지 Dialplan에 선언해야 한다.

```text
5001번 서버
  로컬 5001번대 내선 ──▶ 내부 단말
  7001번대 내선 ───────▶ 7001번 서버
  010으로 시작하는 번호 ─▶ PSTN 연동 서버

7001번 서버
  로컬 7001번대 내선 ──▶ 내부 단말
  5001번대 내선 ───────▶ 5001번 서버
  PSTN 발신 번호 ───────▶ PSTN 연동 서버

PSTN 연동 서버
  5001번 서버 ──▶ 내부 서버
  7001번 서버 ──▶ 내부 서버
  PSTN에서 들어온 번호 ──▶ 지정된 내부 전화기
```

5001번 서버에서는 로컬 내선, 상대 서버, PSTN 발신, 외부에서 들어오는 통화를 각각 분리한다.

```ini
[5XXX_default]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup

exten => _7XXX,1,Dial(PJSIP/${EXTEN}@Another_Trunk)
same => n,Hangup

exten => _010.,1,Noop(CALLER=${CALLERID(all)})
same => n,Dial(PJSIP/${EXTEN}@Pstn_Trunk)
same => n,Hangup

[7001_Context]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup

[Pstn_Context]
exten => _070.,1,Noop(CALLERID=${CALLERID(all)})
same => n,Dial(PJSIP/5001)
same => n,Hangup
```

## Outbound 라우팅

5XXX_default 컨텍스트는 세 가지 발신 상황을 처리한다.

- _5XXX: 5000번대 로컬 내선으로 연결한다.
- _7XXX: Another_Trunk endpoint를 통해 7001번 서버로 넘긴다.
- _010.: Pstn_Trunk endpoint를 통해 010으로 시작하는 외부 번호로 보낸다.

외부 네트워크로 나가는 통화는 Dial 명령의 endpoint 이름으로 경로를 선택한다.

```ini
exten => _7XXX,1,Dial(PJSIP/${EXTEN}@Another_Trunk)
same => n,Hangup

exten => _010.,1,Noop(CALLER=${CALLERID(all)})
same => n,Dial(PJSIP/${EXTEN}@Pstn_Trunk)
same => n,Hangup
```

5001번 사용자가 7001번으로 전화하면 Another_Trunk가 선택된다. 010으로 시작하는 번호를 입력하면 Pstn_Trunk가 선택되어 PSTN 연동 서버로 통화가 전달된다.

## Inbound 라우팅

반대 방향의 통화는 외부에서 들어온 번호를 내부 전화기로 연결해야 한다. 예시에서는 PSTN 연동 서버가 070으로 시작하는 수신 번호를 5001번으로 전달한다.

```ini
[7001_Context]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup

[Pstn_Context]
exten => _070.,1,Noop(CALLERID=${CALLERID(all)})
same => n,Dial(PJSIP/5001)
same => n,Hangup
```

이 설정은 PSTN에서 070으로 시작하는 번호로 전화가 들어오면 발신자 정보를 남긴 뒤 내부 5001 endpoint를 호출한다. 실제로 연결할 내선이 여러 개라면 수신 번호 대역이나 DID에 따라 다른 내부 endpoint로 분기하면 된다.

7001번이 속한 서버도 같은 원칙으로 Dialplan을 작성한다. 로컬 번호는 자기 서버의 단말로, 5001번대 번호는 상대 서버로, PSTN 발신은 PSTN 연동 서버로 전달한다.

## 주소와 라우팅을 함께 확인하기

문제가 발생했을 때는 다음 순서로 확인하면 된다.

1. 각 서버의 /etc/asterisk/pjsip.conf에 상대 endpoint가 등록되어 있는지 확인한다.
2. 내부 Peer에는 사설 IP가, PSTN Peer에는 공인 IP가 사용됐는지 확인한다.
3. 발신 전화기의 context가 올바른 Dialplan 컨텍스트를 가리키는지 확인한다.
4. 번호 패턴이 실제 입력 번호와 일치하는지 확인한다.
5. Dial(PJSIP/번호@상대_endpoint)에서 endpoint 이름이 계정 설정과 같은지 확인한다.
6. PSTN에서 들어오는 번호가 PSTN 연동 서버의 inbound 컨텍스트로 들어오는지 확인한다.

## 정리

다중 Proxy 구성은 단순히 서버를 하나 더 추가하는 작업이 아니다. 사용자 계정을 등록할 때 네트워크 경계를 고려하고, 번호 대역별로 Outbound와 Inbound 경로를 분리해야 한다.

내부 서버끼리 통신할 때에는 사설 IP와 SIP-Trunk endpoint를 사용한다. PSTN으로 나가는 통화는 Pstn_Trunk로 전달하고, PSTN에서 들어온 통화는 PSTN 연동 서버의 컨텍스트에서 내부 전화기로 연결한다. 이 구조를 기준으로 하면 5001번·7001번 내부 통화와 PSTN 통화를 하나의 Asterisk 환경 안에서 함께 라우팅할 수 있다.

