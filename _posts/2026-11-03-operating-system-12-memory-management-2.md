---
title: "12. Memory Management 2"
description: "Page Replacement, Swapping과 페이지 교체 알고리즘을 정리한다."
date: 2026-11-03 23:01:00 +0900
categories: [Operating System]
tags: [os, memory-management, page-replacement]
---

## 1. Page Replacement

Multi Programming System에서 메모리 내에 적재된 Application의 수가 증가함에 따라 메모리 공간이 부족해지는 상황이 발생한다. Page 수보다 물리 메모리의 Frame 수가 훨씬 적은 상황이다.

이럴 경우 Page Fault Handling(Trap)에 Page Replacement를 추가하면 해결할 수 있다. Page Replacement란 물리 메모리에 위치한 Page를 Disk에 임시로 저장하고, 요구된 페이지가 해당 프레임을 할당받도록 하는 방법이다.

Page Fault Handler는 Disk를 탐색하여 요구된 페이지의 위치를 찾는다. 이후 물리 메모리에서 할당되지 않은 프레임을 찾는다. 할당되지 않은 프레임이 있으면 그대로 사용하고, 찾지 못하면 Page Replacement Algorithm으로 교체될 프레임(Victim Frame)을 선택한다. 교체할 프레임의 내용을 Disk에 저장하고 Page Table에서 해당 프레임의 내용을 갱신한 뒤 Application을 재시작한다.

```text
Page Fault
   │
요구된 페이지의 Disk 위치 탐색
   │
할당되지 않은 Frame 탐색
   ├─ Frame 있음 ───────────────▶ 요구 페이지 적재
   └─ Frame 없음 ─▶ Victim Frame 선택
                       │
                       ▼
               기존 내용 Disk 저장
                       │
                       ▼
                Page Table 갱신
                       │
                       ▼
                요구 페이지 적재
                       │
                       ▼
                Application 재시작
```

Page Replacement를 고려할 때는 다음과 같은 문제가 있다.

- 각각의 User Process에게 Frame을 어떻게 분배할 것인가? 공평하게 분배할 것인가, 메모리를 많이 사용하는 프로세스 위주로 분배할 것인가?
- 페이지 교체가 필요할 때 교체할 페이지, 즉 희생될 Frame을 어떻게 선정할 것인가?

이러한 고려 사항은 페이지 교체에 따른 I/O 작업 횟수를 최대한 줄이려는 목적을 갖는다. Page Replacement Algorithm을 살펴보기 전에 다음과 같은 상황을 가정한다.

- 세 개의 Frame이 할당된다.
- Page 참조는 총 20회이며 순서는 다음과 같다.

```text
7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1
```

- 한 번 참조된 Page는 교체가 일어나기 전까지 물리 메모리에 그대로 위치한다.

### 1. First In First Out Algorithm

먼저 Frame이 할당된 Page를 먼저 교체하는 알고리즘이다. 가장 단순한 알고리즘이며 FIFO Queue로 구현할 수 있다. 멀티미디어 데이터를 처리하는 데 특화되어 있다.

위의 참조열에서 FIFO 알고리즘은 Page Fault가 15번 발생한다.

### 2. Second Change Replacement Algorithm

FIFO 알고리즘을 보완하기 위해 Reference Bit를 추가한 알고리즘이다. 오랜 시간 메모리에 상주하는 페이지 중 자주 사용되는 페이지가 교체되는 것을 방지한다.

참조 비트는 프레임에 최초로 적재될 때와 페이지가 참조될 때마다 1로 설정한다. 참조 비트는 주기적으로 0으로 초기화한다. 페이지 교체 시 참조 비트가 1이면 최근에 사용된 페이지이므로 교체하지 않고 참조 비트만 0으로 바꾼 뒤 다른 페이지를 교체한다.

```text
교체 대상의 Reference Bit 확인
   ├─ 0 → 페이지 교체
   └─ 1 → Bit를 0으로 바꾸고 다음 페이지 검사
```

### 3. Clock Algorithm

SCR Algorithm을 발전시킨 형태로, 환형 Queue를 사용하여 프레임을 관리한다. 다음에 교체될 페이지를 가리키는 포인터 `Hand`를 두고, `Hand`는 Queue를 따라 한 칸씩 이동하며 페이지를 교체한다.

`Hand`가 가리키는 페이지의 참조 비트가 1이면 최근에 사용된 페이지로 보고 교체하지 않는다. 참조 Bit만 0으로 초기화하고 다음 페이지로 이동한다. 이 알고리즘도 참조 Bit를 주기적으로 0으로 초기화한다.

```text
                 ┌─────────┐
             ┌──▶│ Page A  │──┐
             │   └─────────┘  ▼
         ┌─────────┐      ┌─────────┐
         │ Page D  │      │ Page B  │ ← Hand
         └────▲────┘      └────┬────┘
              └─────┌─────────┐◀┘
                    │ Page C  │
                    └─────────┘
```

### 4. Least Frequently Used Algorithm

사용 빈도가 낮은 페이지를 교체하는 기법이다. 지금까지 가장 적게 참조된 페이지를 교체 대상으로 선택한다.

프로그램 실행 초기에 많이 사용된 페이지는 이후 사용되지 않더라도 사용 빈도가 높은 것으로 기록되어 교체되지 않고 계속 프레임을 차지할 수 있다.

### 5. Not Recently Used Algorithm

최근에 사용하지 않은 페이지를 교체하는 기법이다. 페이지마다 참조 비트와 변형 비트(Modified Bit)를 두어 관리한다.

페이지 내 데이터가 수정되면 해당 페이지를 다시 사용될 것으로 기대하여 변형 비트를 관리한다. 프레임에 최초로 적재될 때는 0으로 설정하고, 페이지 내용이 바뀌면 1로 바꾼다.

페이지 교체가 필요한 시점에는 참조 비트와 변형 비트 값을 비교해 교체 대상을 결정한다. 두 비트 값이 모두 0인 페이지가 가장 높은 교체 순위를 갖는다.

| 참조 비트 | 변형 비트 | 교체 순위 |
|---:|---:|---|
| 0 | 0 | 가장 높음 |
| 0 | 1 | 다음 |
| 1 | 0 | 다음 |
| 1 | 1 | 가장 낮음 |

### 6. Least Recently Used Algorithm

가장 오랜 시간 동안 참조되지 않은 페이지부터 교체하는 기법이다. 가장 이상적인 페이지 교체 알고리즘이며, 구현 방법은 다음과 같다.

- **Counter**: 카운터를 사용하여 참조된 시간을 기록한다.
- **Queue**: 한 번 사용한 페이지를 Queue의 가장 위로 이동시킨다. Queue의 가장 위는 가장 최근에 사용한 페이지이고, 가장 아래는 가장 오래 전에 사용된 페이지다.

```text
가장 최근에 사용한 페이지
          ▲
       [ Page ]
       [ Page ]
       [ Page ]
          ▼
가장 오래 전에 사용한 페이지 → 교체 대상
```

위의 참조열에서 LRU Algorithm의 Page Fault는 총 12번 발생한다.

## 2. Swapping

Page Replacement만으로 메모리 부족을 해결하지 못할 때 사용하는 기법이다. 대상 프로세스 전체를 Disk로 이동시켜 두었다가 다시 필요할 때 불러온다. 이때 사용하는 Disk 영역을 Swap 영역이라고 한다.

```text
물리 메모리 부족
      │
      ▼
프로세스 전체 ──▶ Disk의 Swap 영역
      │                 │
      └──── 필요할 때 다시 적재 ◀────┘
```

## 3. Contiguous Memory Allocation

물리 메모리 영역을 User Process가 어떻게 나누어 사용하는지 다룬다.
