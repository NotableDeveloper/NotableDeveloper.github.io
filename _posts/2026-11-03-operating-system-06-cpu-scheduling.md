---
title: "06. CPU Scheduling"
description: "CPU 스케줄링의 목표, 평가 기준과 대표 알고리즘을 정리한다."
date: 2026-11-03 23:07:00 +0900
categories: [Operating System]
tags: [os, cpu-scheduling]
---

## CPU 스케줄링

CPU가 하나이고 프로세스가 여러 개일 때 CPU는 한 번에 하나의 프로세스만 실행할 수 있다. CPU 자원은 한정되어 있으므로 스케줄링 정책은 메모리에 있는 Ready 상태 프로세스 중 CPU를 할당할 프로세스를 결정한다. Multiprogramming 환경에서 CPU가 놀지 않게 하고 단위 시간당 처리량을 높이는 것이 목표다.

프로세스는 CPU burst와 I/O burst를 반복한다. CPU-bound 프로세스는 CPU를 오래 사용하고 데이터 입출력이 적다. I/O-bound 프로세스는 입출력이 길고 CPU burst가 짧다.

```text
프로세스 실행 → CPU burst → I/O burst → CPU burst → …
CPU-bound → 긴 CPU burst → 짧은 I/O burst
I/O-bound → 짧은 CPU burst → 긴 I/O burst
```

스케줄링 결정은 다음 네 시점에서 일어난다.

1. Running에서 Waiting으로 바뀔 때: 프로세스가 잠시 멈추므로 다른 프로세스를 실행한다.
2. Running에서 Ready로 바뀔 때: 인터럽트나 입출력 등의 이유로 실행 중인 프로세스를 멈추고 다른 프로세스를 실행한다.
3. 프로세스가 종료될 때.
4. Ready에서 Running으로 바뀔 때: CPU를 할당할 프로세스를 결정한다. 인터럽트 처리가 끝나면 중단되었던 프로세스를 다시 실행할 수 있다.

비선점(Nonpreemptive) 스케줄링은 1번과 4번 시점에서만 CPU 할당을 결정한다. 운영체제가 프로세스에게 CPU를 맡긴 뒤 강제로 뺏지 않는다. 선점(Preemptive) 스케줄링은 다른 시점에도 실행 중인 프로세스를 강제로 중단할 수 있다.

## 평가 기준

- CPU 이용률: CPU가 사용 중인 시간의 비율
- 처리량: 단위 시간에 끝낸 프로세스 수
- 응답 시간: 요청부터 첫 응답까지의 시간
- 대기 시간: Ready 큐에서 기다린 시간
- 반환 시간(Turnaround Time): 프로세스 실행을 시작한 시점부터 끝날 때까지의 시간

이상적으로는 CPU 이용률과 처리량을 높이고 응답 시간, Ready 큐 대기 시간, 반환 시간을 줄이고 싶다. 그러나 이 기준들을 모두 동시에 최적화하기는 어렵다.

## 알고리즘

### FCFS

먼저 도착한 프로세스가 먼저 CPU를 할당받는다(FIFO). 프로세스 도착 순서에 따라 평균 대기 시간이 달라질 수 있다.

### SJF

CPU Burst Time을 비교해 가장 짧은 프로세스를 먼저 실행한다. 비선점형 SJF는 CPU를 할당받은 프로세스가 해당 Burst를 마칠 때까지 실행한다. 선점형 방식은 더 짧은 남은 Burst를 가진 프로세스가 도착하면 실행 중인 프로세스가 CPU를 양보한다.

### Priority

우선순위가 높은 프로세스부터 실행한다. 비선점형에서는 할당받은 프로세스가 실행을 계속하고, 선점형에서는 더 높은 우선순위 프로세스가 도착하면 CPU를 양보한다. 낮은 우선순위 프로세스가 계속 기다리는 Starvation이 생길 수 있다. Aging은 기다린 시간에 따라 우선순위를 높여 이 문제를 완화한다.

### Round Robin (RR)

각 프로세스에 time quantum을 할당하고 시간이 끝나면 큐의 뒤로 보낸다. 문맥 전환이 늘지만 응답성이 좋다.

```text
Ready Queue → A → B → C
CPU → A (quantum 만료) → B → C → A → …
```

### Multilevel Queue

Ready Queue를 여러 개로 나누고 각 큐에 서로 다른 스케줄링 알고리즘을 적용한다.

### Multilevel Feedback Queue

프로세스가 큐 사이를 이동할 수 있다. 예를 들어 세 개의 큐를 두고 첫 번째는 RR, time quantum 8ms, 두 번째는 RR, time quantum 16ms, 세 번째는 FCFS로 운영한다. 새 프로세스는 Q0에서 8ms 실행한 뒤 끝나지 않으면 Q1로 내려간다. Q1에서 16ms를 더 실행한 뒤에도 끝나지 않으면 Q2로 내려가 FCFS로 처리된다. 따라서 CPU를 오래 쓰는 작업은 하위 큐에서 실행된다.

```text
새 작업 → Q0 (RR, quantum 8)
Q0 → 시간 소진 → Q1 (RR, quantum 16)
Q1 → 시간 소진 → Q2 (FCFS)
```
