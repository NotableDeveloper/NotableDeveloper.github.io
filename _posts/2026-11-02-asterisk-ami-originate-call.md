---
title: "AMI로 Asterisk 통화 시작하기"
description: "Asterisk Manager Interface를 활성화하고 외부 클라이언트에서 Originate 명령을 보내 5001번 전화기로 5002번에 전화를 거는 과정을 정리한다."
date: 2026-11-02 23:55:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, ami, pjsip, voip, originate]
---

Asterisk에는 외부 프로그램이나 터미널에서 통화 제어 명령을 보낼 수 있는 AMI(Asterisk Manager Interface)가 있다. 이번에는 이미 Asterisk에 연결된 5001번과 5002번 전화기를 대상으로, AMI를 통해 5001번에서 5002번으로 통화를 시작해본다.

전체 흐름은 다음과 같다.

```text
외부 클라이언트 ── AMI/TCP ──▶ Asterisk ── PJSIP ──▶ 5001 전화기
                                      │
                                      └── Dialplan ──▶ 5002 전화기
```

PJSIP와 Dialplan은 앞서 구성되어 있다고 가정한다. 여기서는 AMI 접속 설정과 통화 시작 명령에 집중한다.

## AMI 활성화하기

AMI는 기본적으로 TCP 연결을 통해 Asterisk의 Manager 기능을 외부에 제공한다. `/etc/asterisk/manager.conf`에서 AMI를 활성화하고 접속 주소와 포트를 지정한다.

```ini
[general]
enabled = yes
port = 5038
bindaddr = 0.0.0.0
```

`enabled`가 `yes`이면 AMI가 활성화된다. `port`는 Manager 클라이언트가 접속할 TCP 포트이며, 일반적인 기본 포트는 5038이다. `bindaddr`를 `0.0.0.0`으로 지정하면 서버의 모든 인터페이스에서 접속을 받는다.

외부 네트워크에서 접근해야 한다면 방화벽에서 포트를 열어야 한다. 다만 모든 주소에 AMI를 공개하면 계정 탈취나 통화 제어로 이어질 수 있으므로, 실제 운영에서는 허용할 IP 대역을 제한하고 필요한 권한만 부여해야 한다.

## Manager 계정 만들기

같은 파일에 AMI에 로그인할 Manager 계정을 추가한다.

```ini
[sample]
secret = 1234
read = system, call, log, originate, agi, ...
write = system, call, agent, user, config, ...
```

이 설정에서는 `sample`이라는 사용자를 만들고 `1234`를 비밀번호로 사용한다. `read`는 Asterisk 상태나 이벤트를 읽을 수 있는 권한이고, `write`는 명령을 실행하거나 시스템 상태를 바꿀 수 있는 권한이다.

통화를 시작하려면 `originate` 권한이 필요하다. 예제처럼 여러 권한을 한 번에 허용할 수도 있지만, 운영 환경에서는 실제로 필요한 권한만 남기는 편이 안전하다. 특히 `system`, `config`처럼 범위가 넓은 권한은 AMI 사용 목적에 맞는지 확인해야 한다.

설정 파일을 수정한 뒤 Asterisk CLI에서 Manager 설정을 다시 읽는다.

```text
manager reload
```

## Telnet으로 AMI 접속하기

AMI는 텍스트 기반 프로토콜이므로 Telnet으로도 동작을 확인할 수 있다. Asterisk 서버 외부에서 다음 명령을 실행한다.

```bash
telnet <Asterisk Server IP> 5038
```

연결에 성공하면 Asterisk Manager 인터페이스의 응답을 받을 수 있다. 접속이 되지 않으면 다음 항목을 순서대로 확인한다.

1. Asterisk에서 AMI가 활성화되어 있는지 확인한다.
2. `manager.conf`의 포트와 접속 주소가 올바른지 확인한다.
3. 서버 방화벽이 TCP 5038을 허용하는지 확인한다.
4. 외부 클라이언트와 Asterisk 사이의 네트워크 경로를 확인한다.

## AMI 로그인

Telnet 연결이 만들어지면 `Action: Login`을 보내 Manager 계정으로 인증한다. AMI 명령은 헤더를 줄 단위로 작성하고, 마지막에 빈 줄을 보내 하나의 명령이 끝났음을 알린다.

```text
Action: Login
Username: sample
Secret: 1234

```

로그인이 성공하면 다음과 같은 응답을 받을 수 있다.

```text
Response: Success
Message: Authentication accepted
```

이제 같은 연결에서 Asterisk Manager 명령을 보낼 수 있다. 로그인에 실패하면 계정 이름과 비밀번호, `manager.conf`의 권한 설정, 설정 reload 여부를 확인한다.

## Originate로 통화 시작하기

AMI에서 전화를 시작할 때 사용하는 대표적인 명령이 `Originate`다. 다음 명령은 5001번 전화기를 먼저 호출한 뒤, 5001번이 응답하면 `simple` Dialplan의 5002번으로 연결한다.

```text
Action: Originate
Channel: PJSIP/5001
Exten: 5002
Context: simple
Priority: 1
CallerID: 5001
Account: 5001
Async: true

```

각 항목의 역할은 다음과 같다.

- `Channel`: 먼저 호출할 Asterisk 채널이다. 여기서는 5001번 endpoint를 지정한다.
- `Exten`: 채널이 응답한 뒤 실행할 Dialplan의 내선 번호다.
- `Context`: 내선 번호를 찾을 Dialplan 컨텍스트다.
- `Priority`: 해당 컨텍스트에서 실행할 우선순위다.
- `CallerID`: 발신자 표시와 통화 식별에 사용할 값이다.
- `Account`: 회계나 과금 정보로 사용할 계정 값이다.
- `Async`: 통화가 끝날 때까지 명령을 기다리지 않고 요청을 비동기로 처리할지 결정한다.

이 명령에서 `Channel`과 `Exten`은 서로 다른 역할을 가진다. `Channel: PJSIP/5001`은 먼저 울릴 전화기를 의미하고, `Exten: 5002`와 `Context: simple`은 5001번이 응답한 뒤 Dialplan에서 연결할 대상을 의미한다.

## 통화 흐름 확인하기

AMI는 Originate 요청을 받은 뒤 Asterisk 내부에서 채널을 생성하고, PJSIP와 Dialplan을 통해 통화를 진행한다. 다음은 실제 원문 로그를 그대로 옮기지 않고 흐름에 맞춰 재구성한 예시다.

```text
Manager 'sample' logged on
Called 5001
PJSIP/5001 is ringing
PJSIP/5001 answered
Executing [5002@simple:1] Dial("PJSIP/5001", "PJSIP/5002")
Called PJSIP/5002
PJSIP/5002 is ringing
PJSIP/5002 answered PJSIP/5001
Channel PJSIP/5001 joined 'simple_bridge'
Channel PJSIP/5002 joined 'simple_bridge'
```

흐름을 단계로 나누면 다음과 같다.

1. AMI Manager 계정이 로그인한다.
2. Asterisk가 `PJSIP/5001` 채널을 호출한다.
3. 5001번 전화기가 응답한다.
4. Asterisk가 `simple` 컨텍스트의 5002번 extension을 실행한다.
5. Dialplan의 `Dial(PJSIP/5002)`가 5002번 전화기를 호출한다.
6. 5002번이 응답하면 두 채널이 브리지에 연결된다.

`Async: true`를 사용하면 Originate 요청에 대한 AMI 응답을 받은 뒤에도 통화 과정은 계속 진행될 수 있다. 통화가 실제로 연결되었는지 확인하려면 Asterisk CLI 로그나 AMI 이벤트를 함께 확인해야 한다.

## AMI 명령을 사용할 때의 주의점

AMI는 단순한 조회 인터페이스가 아니라 통화 생성과 시스템 제어까지 수행할 수 있는 관리 인터페이스다. 따라서 테스트가 끝난 뒤에는 다음 항목을 정리해야 한다.

- 예제 비밀번호를 실제 환경에서 그대로 사용하지 않는다.
- `bindaddr = 0.0.0.0`을 유지해야 하는지 검토한다.
- 방화벽에서 필요한 클라이언트만 5038 포트에 접근하도록 제한한다.
- Manager 계정에 불필요한 `read`와 `write` 권한을 부여하지 않는다.
- 사용하지 않는 Manager 계정은 제거하거나 비활성화한다.
- AMI 연결을 평문 Telnet으로 운영하지 않고 전송 구간 보호 방법을 검토한다.

## 마무리

AMI를 이용하면 외부 클라이언트에서 Asterisk의 통화 기능을 명령 단위로 제어할 수 있다.

1. `manager.conf`에서 AMI와 Manager 계정을 설정한다.
2. Asterisk CLI에서 `manager reload`를 실행한다.
3. TCP 5038 포트로 AMI에 접속한다.
4. `Action: Login`으로 인증한다.
5. `Action: Originate`에 채널과 Dialplan 정보를 전달한다.
6. Asterisk CLI나 AMI 이벤트로 통화 상태를 확인한다.

핵심은 AMI가 직접 SIP 통화 과정을 모두 구현하는 것이 아니라, Asterisk에 통화 시작을 지시하는 제어 계층이라는 점이다. 실제 단말 연결은 PJSIP가 담당하고, 어느 번호로 연결할지는 Dialplan이 결정한다. AMI는 그 과정을 외부에서 시작하게 만드는 인터페이스다.
