---
title: "간단한 교환기 구현하기"
description: "Asterisk와 PJSIP를 이용해 5001번과 5002번 전화기가 통화할 수 있는 간단한 교환기를 구현합니다."
date: 2026-11-02 22:30:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, pjsip, sip, voip]
---

아래와 같이 5001번과 5002번의 두 대의 전화기가 존재하는 상황이라 가정하고, Asterisk를 통해 두 대의 전화기에 대한 **교환기**를 구현해보자. 즉, Asterisk를 이용하여 5001번과 5002번이 서로 통화가 가능하도록 전화 신호를 교환해보는 것이다.

다음과 같은 구조로 Asterisk가 두 전화기 사이의 교환기 역할을 한다.

```text
                    Asterisk 교환기
                    /            \
                   /              \
          5001번 전화기      5002번 전화기
             SIP/RTP             SIP/RTP
```

## PJSIP 설정하기

두 대의 전화가 서로 통신하기 위해 두 전화 사이에서 데이터를 중재하기 위해서는 SIP와 RTP이 필요하다. 그 중에서 Asterisk는 SIP 프로토콜을 사용하기 위한 모듈로 PJSIP를 사용하고 있다.

이 모듈을 사용하여 두 전화 사이의 Session을 설정해야 두 전화 사이의 음성 데이터를 RTP를 통해 주고 받을 수 있게 된다.

Asterisk에서 PJSIP 모듈을 사용하기 위해서는 아래와 같은 양식의 설정 파일을 작성해야한다. 이 파일은 ***/etc/asterisk/pjsip.conf*** 이다.

```
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

이렇게 작성한 파일에 대해 하나하나 살펴보면 다음과 같다.

- simpletrans

  전화기로부터 오는 SIP을 수신받기 위한 객체(= transport, listen을 담당하는 객체)를 정의하는 내용을 담고 있다.

  ```
  [simpletrans]
  type=transport
  protocol=udp
  bind=0.0.0.0:5060
  ```

  **simpletrans**이라는 이름으로 정의했으며, UDP 프로토콜을 사용하고 외부의 어떤 IP 주소인지는 상관하지 않고 5060 포트로 오는 모든 요청을 허용하기로 정의했다.

- endpoint (5001번, 5002번 전화기에 대한 설정)

  교환기의 종점에 대한 설정이다. <u>교환기에 연결되기 위한 프로토콜 설정</u> 및 <u>전화기를 등록</u>하는 방법에 대해 설정하는 부분이라고 할 수 있다.

  ```
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
  ```

  **5001**이라는 이름으로 정의했으며, **simple**이라는 context(= Call에 대한 라우팅 정보)를 가지며 인증과 aors에 대한 설정은 **auth5001** 객체와 **5001** 객체(= t<u>ype이 aor이고, 이름이 5001인 객체</u>)를 참조하도록 정의했다.

## Socery 설정하기

앞서 PJSIP 모듈에 대한 설정을 끝마치고 나면 데이터 접근에 관한 설정도 해주어야 한다.

이는 **Socery**라는 모듈을 통해 할 수 있다. Sorcery는 <u>데이터 추상화와 객체 지속성 CRUD API</u>이며, Asterisk의 여러 저장 메커니즘(Asterisk 데이터베이스, 정적 구성 파일, Asterisk Realtime Architecture, 인메모리 저장소 등) 위에 추상화 계층을 제공하여, 모듈들이 이를 통해 쉽게 데이터를 관리할 수 있도록 한다.

또한 캐싱 서비스와 Asterisk REST Interface(ARI)를 통한 푸시 구성 기능도 지원한다.

***/etc/asterisk/socery.conf*** 에서 다음과 같은 내용을 추가해준다. ******

```
[res_pjsip]
endpoint=realtime,ps_endpoints
endpoint=config,pjsip.conf,criteria=type=endpoint

auth=realtime,ps_auths
auth=config,pjsip.conf,criteria=type=auth

aor=realtime,ps_aors
aor=config,pjsip.conf,criteria=type=aor
```

## Dialplan 설정하기

앞선 PJSIP 설정이 전화기 및 교환기에 대한 설정이었다면 이번 설정은 전화 신호를 받았을 때에 어떤 행동을 할 지에 대해 설정하는 것이다.

**/etc/asterisk/extension.conf** 에서 다음과 같은 내용을 추가해준다.

```
[simple]
exten => _5001,1,Dial(PJSIP/5001);
exten => _5002,1,Dial(PJSIP/5002);
```

위 설정은 **simple**이라는 이름으로 교환기에 5001번과 5002번에 <u>통화가 올 때에 대한 행동을 정의한 것</u>이다. 앞서 PJSIP에 관련한 설정을 작성할 때 **endpoint의 context로 simple을 작성했었는데 해당 simple은 dialplan을 작성**한 것이다.

여기서 5001번과 5002번으로 전화가 올 때에 하는 행동은 PJSIP를 이용하여 5001번과 5002번으로 통화를 걸겠다는 것이다.

또, 앞서 작성한 dialplan은 다음과 같이 축약할 수도 있다.

```
[simple]
exten => _5XX.,1,Dial(PJSIP/${EXTEN})
```

축약한 내용은 <u>*첫 번째 자리가 무조건 5이고, 두 번째와 세 번째 자리가 각각 0부터 9사이의 한 자리 숫자이며 마지막 네 번째 자리는 1부터 9 사이의 한 자리*</u>로 오는 번호에 대하여 그 번호 그대로 전화를 걸겠다는 것이다.

## 설정 적용하기

앞서 적용한 내용들은 모두 파일에 쓰기 작업만을 한 것이다. Asterisk가 이미 백그라운드 프로세스(= 데몬)으로 동작하고 있는 와중에 파일의 내용만을 바꾼 것이지 해당 설정 내용들이 적용된 것은 아니다. 따라서 이를 하나하나 적용할 필요가 있다.

가장 먼저 적용해야 할 것은 socery이다. 이 모듈은 서비스를 재기동해야 정상 작동된다. 따라서 다음과 같이 asterisk를 재시작해준다. (asterisk 18.22 버전 기준으로 CLI에서 reload를 해도 설정이 적용되지 않아 서비스를 재기동한다.)

```
$ systemctl restart asterisk
```

이후, Asterisk의 CLI에 접근한다.

```
$ /usr/sbin/asterisk -crvvvvv
```

아래와 같이 PJSIP와 Dialplan의 설정을 적용해준다.

```
localhost*CLI> pjsip reload
```

```
localhost*CLI> dialplan reload
```

## 통화 걸어보기

SIP 통화를 걸 수 있는 장치들을 준비한다. (PC의 경우에는 MicroSIP를 사용하면 된다.)

다음과 같이 SIP 서버(= 교환기 서버, asterisk가 가동중인 서버) 및 PJSIP에서 등록한 계정 정보들을 작성하고 등록해준다.

MicroSIP에는 다음과 같이 5001 계정을 등록한다. `SIP 서버`와 `도메인`은 Asterisk 서버 주소로 설정한다.

| 항목 | 값 |
| --- | --- |
| 계정 이름 | `5001` |
| 사용자 | `5001` |
| 로그인 | `5001` |
| 비밀번호 | `5001` |
| 표시 이름 | `5001` |
| SIP 서버 | Asterisk 서버 주소 |
| 도메인 | Asterisk 서버 주소 |

등록하고 난 이후에 asterisk CLI에서 조회해보면 다음과 같이 설정했던 기기 이름(5001번과 5002번)으로 연결된 것을 볼 수 있다.

등록이 완료되면 `pjsip show contacts` 명령으로 두 전화기의 연락처가 등록된 것을 확인할 수 있다.

```text
localhost*CLI> pjsip show contacts

Contact:  5001/sip:5001@<ASTERISK_SERVER>  ...  <STATUS>  <RTT(ms)>
Contact:  5002/sip:5002@<ASTERISK_SERVER>  ...  <STATUS>  <RTT(ms)>
```

여기서 `5001`과 `5002`가 각각 표시되면 두 단말이 Asterisk에 등록된 상태다.

이제 5001번에서 5002번으로 실제 통화를 시도하면 asterisk 내에서 다음과 같은 로그를 볼 수 있다. 반대의 경우도 마찬가지이다.

통화가 연결되면 Asterisk CLI에는 다음과 같은 흐름이 출력된다.

```text
-- Executing [5002@simple:1] Dial("PJSIP/5001-...", "PJSIP/5002") in new stack
-- Called PJSIP/5002
-- PJSIP/5002-... is ringing
-- PJSIP/5002-... answered PJSIP/5001-...
-- Channel PJSIP/5001-... joined 'simple_bridge'
-- Channel PJSIP/5002-... joined 'simple_bridge'
-- Remotely bridged PJSIP/5001-... and PJSIP/5002-...
```

이 로그는 5001번에서 5002번으로 전화를 걸고, 두 채널이 하나의 브리지로 연결되어 음성 통화가 시작된 과정을 보여준다.
