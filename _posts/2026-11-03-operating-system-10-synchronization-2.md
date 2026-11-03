---
title: "10. 동기화 2"
description: "유한 버퍼, 독자-작성자, 식사하는 철학자 문제와 세마포어 풀이를 정리한다."
date: 2026-11-03 23:03:00 +0900
categories: [Operating System]
tags: [os, synchronization, semaphore]
---

## 고전적인 동기화 문제

### Bounded Buffer (Producer-Consumer)

N개의 칸으로 된 버퍼에서 생산자는 빈 칸을 채우고 소비자는 채워진 칸을 비운다. `Full=0`, `Empty=N`, `Mutex=1`로 시작한다.

```text
Producer: 항목 생성 → P(empty) → P(mutex) → 항목 추가
          → V(mutex) → V(full)

Consumer: P(full) → P(mutex) → 항목 제거
          → V(mutex) → V(empty) → 항목 소비
```

### Readers-Writers

여러 독자는 동시에 읽을 수 있다. 작성자는 독점적으로 쓰며, 쓰는 동안 독자는 접근할 수 없다. `Mutex=1`, `Wrt=1`, `ReadCount=0`으로 시작한다.

```text
Writer:
  P(Wrt)
  쓰기
  V(Wrt)

Reader:
  P(Mutex)
  ReadCount++
  if ReadCount == 1: P(Wrt)
  V(Mutex)
  읽기
  P(Mutex)
  ReadCount--
  if ReadCount == 0: V(Wrt)
  V(Mutex)
```

독자가 계속 들어와 `ReadCount`가 0이 되지 않으면 작성자가 기아 상태에 빠질 수 있다.

### Dining Philosophers

철학자 다섯 명이 원형 탁자에 앉고 젓가락 다섯 개를 공유한다. 식사하려면 양옆의 젓가락 두 개를 모두 얻어야 한다.

```text
         P0
      C4    C0
   P4          P1
    C3        C1
      P3—C2—P2
```

각 철학자가 왼쪽 젓가락을 잡은 뒤 오른쪽 젓가락을 기다리는 방식은 모두가 하나씩 잡은 상태에서 교착 상태가 될 수 있다.

```text
P(i): P(chopstick[i]);
      P(chopstick[(i+1) % 5]);
      식사;
      V(chopstick[i]);
      V(chopstick[(i+1) % 5]);
```

해결 방법은 최대 네 명만 동시에 앉게 하거나, 양쪽 젓가락을 모두 사용할 수 있는지 확인한 뒤 얻거나, 홀수와 짝수 철학자의 젓가락 획득 순서를 반대로 하는 것이다.
