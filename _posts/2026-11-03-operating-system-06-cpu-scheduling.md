---
title: "06. CPU Scheduling"
description: "CPU 스케줄링의 목표, 평가 기준과 대표 알고리즘을 정리한다."
date: 2026-11-03 23:07:00 +0900
categories: [Operating System]
tags: [os, cpu-scheduling]
---

## CPU 스케줄링

CPU가 하나이고 프로세스가 여러 개일 때 스케줄링 정책은 Ready 큐에서 실행할 프로세스를 선택한다. 목표는 처리량을 높이고 CPU 유휴 시간을 줄이는 것이다.

프로세스는 CPU burst와 I/O burst를 반복한다. CPU-bound 프로세스는 CPU를 오래 사용하고 데이터 입출력이 적다. I/O-bound 프로세스는 입출력이 길고 CPU burst가 짧다.

```text
프로세스 실행: [CPU burst] → [I/O burst] → [CPU burst] → …
CPU-bound:    [──── 긴 CPU ────] [짧은 I/O]
I/O-bound:    [짧은 CPU] [──── 긴 I/O ────]
```

스케줄링 결정은 Running→Waiting, Running→Ready(인터럽트/I/O 선점), 프로세스 종료, Ready→Running 전환 때 이루어진다. 비선점 스케줄링은 1번과 4번 시점에서만 스케줄링한다. 선점 방식은 그 외에도 CPU 실행을 강제로 중단할 수 있다.

## 평가 기준

- CPU 이용률: CPU가 사용 중인 시간의 비율
- 처리량: 단위 시간에 끝낸 프로세스 수
- 응답 시간: 요청부터 첫 응답까지의 시간
- 대기 시간: Ready 큐에서 기다린 시간
- 반환 시간: 제출부터 종료까지의 시간

모든 기준을 동시에 최대화하거나 최소화하기는 어렵다.

## 알고리즘

### FCFS

먼저 도착한 프로세스부터 실행한다.

### SJF

CPU burst가 짧은 프로세스를 먼저 실행한다. 선점형과 비선점형이 있다.

### Priority

우선순위가 높은 프로세스부터 실행한다. 선점형과 비선점형이 있다. 낮은 우선순위가 계속 기다리는 기아가 생길 수 있으며, Aging으로 기다린 시간만큼 우선순위를 높여 완화한다.

### Round Robin (RR)

각 프로세스에 time quantum을 할당하고 시간이 끝나면 큐의 뒤로 보낸다. 문맥 전환이 늘지만 응답성이 좋다.

```text
Ready Queue: [A] [B] [C]
CPU: A(quantum 만료) → B → C → A → …
```

### Multilevel Queue

서로 다른 큐를 두고 각 큐마다 알고리즘을 적용한다.

### Multilevel Feedback Queue

큐 사이를 이동할 수 있다. 예를 들어 quantum 8, 16을 사용하는 RR 큐를 거쳐 충분히 오래 실행된 작업을 FCFS 큐로 내린다.

```text
새 작업
  ▼
Q0: RR, quantum 8  ── 시간 소진 ──▶ Q1
                                      │
Q1: RR, quantum 16 ── 시간 소진 ─────┤
                                      ▼
                                  Q2: FCFS
```
