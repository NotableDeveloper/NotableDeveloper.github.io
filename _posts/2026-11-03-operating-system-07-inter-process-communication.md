---
title: "07. Inter Process Communication"
description: "프로세스 간 통신의 공유 메모리와 메시지 전달 방식, 대표 IPC 수단을 살펴본다."
date: 2026-11-03 23:06:00 +0900
categories: [Operating System]
tags: [os, ipc]
---

## IPC

IPC(Inter Process Communication)는 프로세스가 데이터와 정보를 주고받는 방법이다. 커널은 시스템 콜을 통해 IPC 기능을 제공하여 프로세스들이 서로 협력할 수 있게 한다. 대표 방식은 공유 메모리와 메시지 전달이다.

## 공유 메모리

프로세스들이 특정 메모리 영역을 함께 사용한다. 공유 영역을 만들 때는 커널의 도움이 필요하지만, 영역이 준비된 뒤에는 프로세스가 직접 읽고 쓸 수 있어 데이터 전달이 빠르고 유연하다. 여러 프로세스가 동시에 같은 메모리에 쓰지 않도록 동기화해야 한다.

```text
프로세스 A ↔ 공유 메모리 ↔ 프로세스 B
공유 메모리 → 읽기 / 쓰기
공유 메모리 → 동기화 필요
```

다음 코드는 공유 메모리를 만들고 자식 프로세스가 값을 변경하는 과정이다.

```c
#define SHM_SIZE 1024

/* 부모 프로세스 */
shmId = shmget(IPC_PRIVATE, SHM_SIZE, SHM_R | SHM_W);
ptrShm = shmat(shmId, NULL, 0);
ptrShm[0] = 11;
ptrShm[1] = 22;
pid = fork();
if (pid == 0) {
    ChildRun(shmId);
} else {
    waitpid(pid, NULL, 0);
    printf("Parent: %d, %d\n", ptrShm[0], ptrShm[1]);
    shmdt(ptrShm);
    shmctl(shmId, IPC_RMID, NULL);
}

/* 자식 프로세스 */
void ChildRun(int shmid) {
    char *ptr = shmat(shmid, NULL, 0);
    printf("Child: %d, %d\n", ptr[0], ptr[1]);
    ptr[0] = 33;
    ptr[1] = 44;
    shmdt(ptr);
}
```

## 메시지 전달

공유 메모리를 두지 않고 고정 크기 또는 가변 크기 메시지를 `send`와 `receive`로 주고받는다. 메시지는 커널을 통해 전달된다. 송수신 과정에서 공유 메모리를 직접 갱신할 때와 같은 별도 동기화는 필요하지 않다.

### Pipe

Pipe는 한쪽 방향으로만 데이터를 전달한다. 양방향 통신에는 서로 반대 방향인 Pipe 두 개를 사용한다. 수신은 FIFO 순서로 이루어지고 Pipe 용량은 제한되어 있으므로 공간이 가득 차면 새 데이터를 쓸 수 없다.

```text
부모 프로세스 → pipe("hello world") → 자식 프로세스
```

### Signal

미리 정해진 이벤트를 비동기적으로 전달한다. 받는 쪽은 처리하거나 무시하거나 핸들러를 등록할 수 있다. 자식 프로세스는 `SIGUSR1` 핸들러를 등록하고, 부모 프로세스는 3초 기다린 뒤 `kill`로 시그널을 보내고 자식 프로세스를 기다린다.

```text
자식: SIGUSR1 핸들러 등록 → 대기
부모: sleep(3) → kill(child, SIGUSR1) → wait
```

### Message Queue

Message Queue는 사용자가 정의한 크기의 메시지를 연결 리스트 형태로 보관한다. 여러 프로세스가 동시에 접근할 수 있으므로 동기화가 필요하다. 실습에서는 message type을 `1`로 두고 메시지 본문에 `Hello, world.`를 넣어 큐로 전달한다.

### Socket

Socket은 프로세스 사이 통신의 끝점이며 시스템 위치와 무관하게 통신할 수 있다. 로컬 프로세스는 Port로, 원격 프로세스는 IP 주소와 Port로 구분한다. 실습의 TCP 서버는 9999번 포트에 소켓을 만들고 `bind`, `listen`, `accept` 순서로 연결을 기다린다. 클라이언트는 서버에 연결해 데이터를 받는다. 서버는 `fork`로 만든 자식 프로세스에서 연결을 처리할 수도 있다.

```text
서버: socket → bind(9999) → listen → accept → 데이터 전송
클라이언트: socket → connect(IP, 9999) → 데이터 수신
```
