---
title: "AMI와 AGI로 Asterisk 통화 제어하기"
description: "Java 애플리케이션에서 AMI로 Asterisk에 접속해 통화를 시작하고, Dialplan에서 AGI 스크립트를 실행하는 구성을 정리한다."
date: 2026-11-02 23:59:00 +0900
categories: [Asterisk, Practice]
tags: [asterisk, ami, agi, java, pjsip, voip]
---

Asterisk의 통화 동작을 외부 애플리케이션에서 제어하려면 Asterisk Manager Interface(AMI)를 사용할 수 있다. Java 애플리케이션이 AMI로 Asterisk에 접속하고, Asterisk는 등록된 Dialplan과 AGI 스크립트를 이용해 통화를 처리한다.

```text
Java 애플리케이션 ── TCP 5038 ──▶ Asterisk AMI
                                      │
                                      ├─▶ Dialplan ──▶ PJSIP 전화기
                                      └─▶ AGI ──▶ 외부 스크립트
```

AMI는 서버 관리와 통화 제어를 위한 인터페이스다. AGI는 Dialplan에서 Python이나 컴파일된 실행 파일을 호출해 통화 흐름 안에서 애플리케이션 로직을 실행하는 방식이다.

## Asterisk에 전화기 등록하기

Asterisk에 5001번과 5002번 단말을 등록한다. 두 계정 모두 simple_context를 사용하고, 인증 객체와 AOR을 각각 연결한다.

```ini
[5001]
type=endpoint
context=simple_context
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

5002번도 같은 구조로 auth5002와 5002 AOR을 만든다. 실제 운영에서는 계정별 비밀번호를 별도로 관리해야 한다.

## Dialplan에서 통화와 AGI 연결하기

simple_context에는 일반 통화와 AGI 호출을 구분하는 규칙을 둔다.

```ini
[simple_context]
exten => _5XXX,1,Dial(PJSIP/${EXTEN})
same => n,Hangup

exten => _6XXX,1,AGI(hello_world.py)
same => n,Hangup

exten => _8XXX,1,AGI(good_world)
same => n,Hangup
```

5000번대 번호는 PJSIP 단말로 직접 연결한다. 6000번대와 8000번대 번호는 각각 Python 스크립트와 C로 컴파일한 실행 파일을 AGI로 호출한다. 두 파일은 /var/lib/asterisk/agi-bin에 배치해야 한다.

hello_world.py는 Python 3 인터프리터를 가리키는 shebang으로 시작한다.

```python
#!/usr/bin/python3

if __name__ == "__main__":
    print("Hello, world!\n")
```

good_world.c는 소스 파일과 같은 위치에 실행 파일을 만들어 두어야 한다.

```c
#include <stdio.h>

void main() {
    printf("Good World!\n");
}
```

## AMI 계정과 접속 포트

Java 애플리케이션이 AMI를 사용하려면 Manager 계정에 로그인과 통화 제어 권한을 부여해야 한다. 예시에서는 AMI가 TCP 5038번 포트에서 동작하고, Manager 계정에 originate와 agi를 포함한 읽기·쓰기 권한을 설정한다.

```ini
[manager]
secret = 1234
read = system,call,log,verbose,agent,user,config,dtmf,reporting,cdr,dialplan
write = system,call,agent,user,config,command,reporting,originate,agi,message
```

외부에서 접속하는 경우에는 AMI 포트를 무조건 공개하기보다 방화벽과 허용 IP를 함께 제한해야 한다. 예시 비밀번호는 실습용 값이므로 운영 환경에서 그대로 사용하면 안 된다.

## Java 프로젝트 의존성

예제는 Asterisk-Java를 사용하고, AMI 통신 로그를 남기기 위해 Logback을 함께 사용한다.

```groovy
dependencies {
    implementation group: 'org.asteriskjava',
        name: 'asterisk-java',
        version: '3.40.1'
    implementation group: 'ch.qos.logback',
        name: 'logback-classic',
        version: '1.4.12'
    implementation group: 'ch.qos.logback',
        name: 'logback-access',
        version: '1.2.9'
}
```

## AMI 로그인 코드

서버 주소와 Manager 계정 정보는 환경 변수에서 읽는다. 먼저 연결 객체를 만들고 AMI에 로그인한다.

```java
private ManagerConnection managerConnection;

ManagerConnectionFactory factory =
    new ManagerConnectionFactory(
        ASTERISK_ADDRESS,
        MANAGER_USERNAME,
        MANAGER_PASSWORD
    );

managerConnection = factory.createManagerConnection();
managerConnection.login();
```

환경 변수에서 설정을 읽는 Manager 클래스는 다음처럼 구성할 수 있다.

```java
public class AsteriskManager {
    private static final String ASTERISK_ADDRESS =
        System.getenv("ASTERISK_ADDRESS");
    private static final String MANAGER_USERNAME =
        System.getenv("MANAGER_USERNAME");
    private static final String MANAGER_PASSWORD =
        System.getenv("MANAGER_PASSWORD");

    private final ManagerConnection managerConnection;

    public AsteriskManager() throws Exception {
        ManagerConnectionFactory factory =
            new ManagerConnectionFactory(
                ASTERISK_ADDRESS,
                MANAGER_USERNAME,
                MANAGER_PASSWORD
            );

        managerConnection = factory.createManagerConnection();
        managerConnection.login();
    }

    public ManagerConnection connection() {
        return managerConnection;
    }
}
```

## 통화 요청 코드

로그인한 Connection으로 OriginateAction을 전송하면 5001번 채널에서 5002번 Extension으로 통화를 시작할 수 있다.

```java
OriginateAction originateAction = new OriginateAction();
originateAction.setChannel("PJSIP/5001");
originateAction.setExten(5002);
originateAction.setPriority(1);
originateAction.setCallerId(5001);
originateAction.setContext("simple_context");
originateAction.setAccount(5001);
originateAction.setAsync(true);

ManagerResponse response =
    managerConnection.sendAction(originateAction);
```

통화에 필요한 값을 별도 객체로 묶으면 요청 생성과 실행을 분리할 수 있다.

```java
public class CallAction {
    private final OriginateAction action;

    public CallAction(String channel, int extension, String context) {
        action = new OriginateAction();
        action.setChannel(channel);
        action.setExten(extension);
        action.setPriority(1);
        action.setContext(context);
        action.setAsync(true);
    }

    public OriginateAction action() {
        return action;
    }
}
```

애플리케이션 진입점에서는 명령행 인자로 발신자, 목적지, Context를 받아 요청을 보낸다.

```java
public class Client {
    public static void main(String[] args) throws Exception {
        String source = args[0];
        int destination = Integer.parseInt(args[1]);
        String context = args[2];

        AsteriskManager manager = new AsteriskManager();
        CallAction call =
            new CallAction("PJSIP/" + source, destination, context);

        manager.connection().sendAction(call.action());
    }
}
```

## Java 애플리케이션의 역할 나누기

Java 쪽 구현은 한 클래스에 모든 통신을 넣기보다 역할을 나누는 편이 이해하기 쉽다.

```text
Client
  └─ AsteriskManager: AMI 연결과 로그인
       └─ CallAction: 통화 시작에 필요한 Action 생성
```

흐름은 다음과 같다.

1. AsteriskManager가 Asterisk의 주소와 5038번 포트로 연결한다.
2. Manager 사용자 이름과 비밀번호로 로그인한다.
3. CallAction이 발신 채널, 대상 번호, 사용할 Context와 Extension을 지정한다.
4. AMI의 통화 시작 요청을 전송한다.
5. Asterisk가 Dialplan을 평가하고 PJSIP 단말 또는 AGI 스크립트로 흐름을 넘긴다.

통화 요청을 만들 때에는 호출할 채널과 대상 번호, Context, Extension을 실제 Asterisk 설정과 일치시켜야 한다. 예를 들어 5002번으로 연결하려면 PJSIP/5001 같은 발신 채널과 simple_context의 5002번 Extension을 조합하는 방식으로 요청을 구성한다.

## AMI와 AGI의 경계

AMI는 Asterisk 밖에 있는 프로그램이 통화 제어 요청을 보내는 통로다. AGI는 Dialplan이 실행되는 과정에서 Asterisk가 외부 프로그램을 호출하는 통로다.

```text
외부 프로그램 ──요청──▶ AMI ──▶ Asterisk
Asterisk ──실행──▶ AGI 스크립트
```

따라서 Java 애플리케이션에서 통화를 시작하는 문제는 AMI 인증과 Action 전송으로 나누고, 통화가 연결된 뒤 별도 로직을 실행하는 문제는 Dialplan과 AGI 스크립트로 나누어 생각하면 된다.

## 실행 예시

5001번에서 5002번으로 일반 PJSIP 통화를 요청한다.

```bash
java -jar app.jar 5001 5002 simple_context
```

6001번을 호출하면 Dialplan이 hello_world.py를 실행한다.

```bash
java -jar app.jar 5001 6001 simple_context
```

8001번을 호출하면 컴파일된 good_world 실행 파일을 AGI로 실행한다.

```bash
java -jar app.jar 5001 8001 simple_context
```

AGI 로그를 확인하려면 Asterisk CLI의 AGI 로그 레벨을 Debug로 올린다. 그러면 Python 또는 C 프로그램이 출력한 문자열이 AGI 송수신 로그와 함께 표시된다.

## 정리

두 단말과 Dialplan을 Asterisk에 준비한 뒤 Manager 계정으로 AMI에 로그인하면 Java 애플리케이션에서 통화 시작을 요청할 수 있다. Dialplan은 일반 번호를 PJSIP으로 연결하고, 특정 번호 대역은 AGI로 넘긴다. AMI는 외부 제어, AGI는 통화 흐름 내부의 애플리케이션 실행을 담당한다.

