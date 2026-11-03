---
title: "10. 동기화 2"
description: "유한 버퍼, 독자-작성자, 식사하는 철학자 문제와 세마포어 풀이를 정리한다."
date: 2026-11-03 23:03:00 +0900
categories: [Operating System]
tags: [os, synchronization, semaphore]
---

## 고전적인 동기화 문제

### Bounded Buffer (Producer-Consumer)

N개의 칸으로 된 버퍼를 여러 생산자와 소비자가 함께 사용한다. 생산자는 항목을 하나 만들고 빈 칸에 넣는다. 소비자는 버퍼가 비어 있지 않을 때 항목을 꺼낸다. 공유 버퍼에 동시에 접근하면 Race Condition이 생기므로 세마포어를 사용한다. `Empty`는 빈 칸 수로 `N`, `Full`은 채워진 칸 수로 `0`, `Mutex`는 상호 배제를 위해 `1`로 초기화한다. 생산자는 빈 칸을 확보하고 Mutex로 임계 영역에 들어가 항목을 넣은 뒤, `Full`을 증가시킨다. 소비자는 반대로 채워진 항목을 기다리고 버퍼에서 제거한 다음 `Empty`를 증가시킨다.

Producer와 Consumer는 다음 절차를 반복한다. 생산자는 다음 항목을 만든 뒤 빈 공간을 기다리고 임계 영역에서 버퍼에 추가한다. 소비자는 항목이 생길 때까지 기다렸다가 임계 영역에서 꺼내고, 빈 공간을 돌려준 뒤 항목을 소비한다.

```text
Producer: 항목 생성 → P(empty) → P(mutex) → 항목 추가
          → V(mutex) → V(full)

Consumer: P(full) → P(mutex) → 항목 제거
          → V(mutex) → V(empty) → 항목 소비
```

```c
function Producers() {
  do {
    produce item in nextp
    P(empty);   // wait buffer space
    P(mutex);   // enter critical section
    add nextp to buffer
    V(mutex);
    V(full);    // item now available
  } while (1);
}

function Consumers() {
  do {
    P(full);    // wait item
    P(mutex);
    remove item from buffer to nextc
    V(mutex);
    V(empty);   // free slot
    consume item in nextc
  } while (1);
}
```

### Readers-Writers

여러 독자는 동시에 읽을 수 있다. 작성자는 독점적으로 쓰며, 쓰는 동안 독자는 접근할 수 없다. `Mutex=1`, `Wrt=1`, `ReadCount=0`으로 시작한다.

`ReadCount`는 현재 읽는 독자의 수다. `Wrt`는 작성자와 독자의 접근을 조정하고, `Mutex`는 `ReadCount`와 관련 상태의 원자적인 변경을 보호한다. 최초 독자는 `Wrt`를 얻어 작성자를 막고, 마지막 독자가 나갈 때 `Wrt`를 돌려준다.

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

첫 독자가 `Wrt`를 얻은 뒤 독자가 계속 들어오면 `ReadCount`가 0이 되지 않는다. 그러면 마지막 독자가 `Wrt`를 반환하지 않아 작성자는 계속 기다릴 수 있고 기아 상태에 빠진다.

### Dining Philosophers

철학자 다섯 명이 원형 탁자에 앉고 젓가락 다섯 개를 함께 사용한다. 각 철학자는 생각하다가 배가 고프면 식사하려 한다. 식사하려면 양옆의 젓가락 두 개를 모두 얻어야 하고, 식사를 마치면 둘 다 내려놓는다. 젓가락은 한 번에 한 명만 사용할 수 있다.

```text
P0 → C0 → P1 → C1 → P2 → C2 → P3 → C3 → P4 → C4 → P0
```

각 철학자가 왼쪽 젓가락을 잡은 뒤 오른쪽 젓가락을 기다리는 방식은 모두가 하나씩 잡은 상태에서 교착 상태가 될 수 있다.

원문 예제에서는 `waiting[n]`과 `chopstick[5]`를 두고, 한 철학자가 왼쪽 젓가락을 `P`로 요청하고 이어 오른쪽 젓가락을 요청한 다음 식사 후 두 젓가락을 `V`로 반환한다. 다섯 명 모두 왼쪽 젓가락을 먼저 얻으면 오른쪽 젓가락을 기다리는 순환 대기가 만들어진다.

```text
P(i): P(chopstick[i]);
      P(chopstick[(i+1) % 5]);
      식사;
      V(chopstick[i]);
      V(chopstick[(i+1) % 5]);
```

원문 풀이에서는 `waiting[n]`과 `chopstick[5]` 배열을 두고 젓가락을 사용 가능 상태로 초기화한 뒤, 철학자가 왼쪽과 오른쪽 젓가락을 차례로 요청하고 식사 후 두 젓가락을 반환하는 절차를 보인다. 이 단순한 순서에서는 모두가 왼쪽 젓가락을 먼저 잡으면 오른쪽을 기다리며 교착 상태에 빠질 수 있다.

해결 방법으로 최대 네 명만 동시에 자리에 앉게 하는 방법, 양쪽 젓가락이 모두 사용 가능한지 확인한 뒤 함께 가져가는 방법, 홀수 번호 철학자는 왼쪽부터 짝수 번호 철학자는 오른쪽부터 가져가도록 순서를 바꾸는 방법을 제시한다. 이런 방식에서도 기아가 발생할 수 있으므로 우선순위를 두어 해결할 수 있다.
