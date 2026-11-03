---
title: "07. Inter Process Communication"
description: "프로세스 간 통신의 공유 메모리와 메시지 전달 방식, 대표 IPC 수단을 살펴본다."
date: 2026-11-03 23:06:00 +0900
categories: [Operating System]
tags: [os, ipc]
---

## IPC

IPC(Inter Process Communication)는 프로세스가 데이터와 정보를 주고받는 방법이다. 커널은 시스템 콜을 통해 IPC를 제공하며, 프로세스들이 협력할 수 있게 한다.

## 공유 메모리

프로세스들이 메모리 영역을 공유한다. 커널이 영역을 만든 뒤에는 읽기와 쓰기가 커널을 거치지 않아 빠르고 유연하다. 동시에 값을 변경하지 않도록 동기화해야 한다.

```text
프로세스 A ──┐     ┌── 프로세스 B
             └─────┘
          공유 메모리
       (읽기/쓰기, 동기화 필요)
```

다음 예제는 공유 메모리를 만들고 자식 프로세스가 값을 변경하는 흐름을 보여준다.

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

프로세스가 커널을 통해 고정 또는 가변 크기 메시지를 보내고 받는다. 공유 메모리가 필요 없고 커널이 중재한다. 원문에서는 동기화가 필요하지 않은 방식으로 설명한다.

### Pipe

단방향 통신이며 양방향으로 통신하려면 파이프 두 개를 사용한다. 데이터는 FIFO 순서로 전달되고 용량은 제한되어 있다.

```text
부모 프로세스 ── pipe(hello world) ──▶ 자식 프로세스
```

### Signal

미리 정해진 이벤트를 비동기적으로 전달한다. 받는 쪽은 처리하거나 무시하거나 핸들러를 등록할 수 있다. 예제에서는 자식이 `SIGUSR1` 핸들러를 등록하고, 부모가 3초 기다린 뒤 `kill`로 시그널을 보내고 자식 프로세스를 기다린다.

```text
자식: SIGUSR1 핸들러 등록 → 대기
부모: sleep(3) → kill(child, SIGUSR1) → wait
```

### Message Queue

고정 크기 메시지를 연결 리스트 형태로 보관한다. 여러 프로세스가 동시에 접근할 수 있어 동기화가 필요하다. 예제 메시지는 type이 1이고 내용은 `Hello, world.`다.

### Socket

통신 끝점이다. 로컬 프로세스는 `port`, 원격 프로세스는 `IP + port`로 구분한다. 예제는 9999 포트의 TCP 서버가 `listen`과 `accept`를 수행하고, 자식 프로세스가 연결해 데이터를 받는 흐름이다.

```text
서버: socket → bind(9999) → listen → accept → 데이터 전송
클라이언트: socket → connect(IP, 9999) → 데이터 수신
```
