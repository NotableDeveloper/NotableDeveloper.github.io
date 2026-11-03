---
title: "05. Computer Architecture"
description: "버스, 인터럽트, 트랩, 입출력 컨트롤러와 DMA의 동작을 정리한다."
date: 2026-11-03 23:08:00 +0900
categories: [Operating System]
tags: [os, computer-architecture, interrupt]
---

## Bus

버스는 CPU, RAM, 입출력 장치 사이에서 데이터와 주소를 전달한다. 구성 요소마다 속도가 다르면 병목이 생긴다. 속도가 비슷한 모듈끼리 계층적인 버스로 연결한다.

```text
                 +----- CPU -----+
                 |               |
       고속 버스 ─┴─ RAM          |
       중속 버스 ─── 장치 제어기  |
       저속 버스 ─── 느린 I/O 장치|
```

## 이벤트 처리: Interrupt와 Trap

인터럽트는 키보드 입력이나 네트워크 데이터 도착처럼 비동기적으로 발생한다. CPU는 한 번에 하나씩 인터럽트를 감지하고, 상태를 저장한 뒤 ISR(Interrupt Service Routine)로 이동한다. 처리가 끝나면 상태를 복원하고 실행을 재개한다. 하드웨어마다 우선순위가 다르며 ISR은 짧게 유지해야 한다. 타이머 인터럽트는 시분할을 가능하게 한다.

```text
인터럽트 감지 → CPU 상태 저장 → ISR 실행
       → 상태 복구 → 중단된 실행 재개
```

Trap은 0으로 나누기처럼 동기적으로 발생하는 이벤트다. Trap Handler(Trap Service Routine)가 처리한다. 인터럽트와 달리 실행 상태를 별도로 저장·복원하지 않는다고 설명한다. 시스템 콜은 Trap이다.

## 입출력 레지스터

입출력 장치 레지스터는 Control, Status, Input, Output 네 종류이며 주 메모리에 매핑된다. I/O Controller는 레지스터를 확인하며 입출력 명령을 수행한다.

Polling(Programmed I/O)에서는 CPU가 입출력을 직접 수행하고, 지연 시간 동안 기다린다.

## DMA

DMA 컨트롤러는 CPU의 요청을 받은 뒤 장치와 RAM 사이의 데이터 전송을 맡는다. 전송이 끝나면 CPU에 알린다.

```text
CPU ── 전송 요청/설정 ──▶ DMA Controller
                              │
장치 ◀──── 데이터 전송 ──────▶ RAM
                              │ 완료 알림
CPU ◀─────────────────────────┘
```

```text
Polling I/O: CPU ↔ 장치 레지스터 (CPU가 대기하며 처리)
DMA I/O:     CPU 설정 → DMA가 장치↔RAM 전송 → CPU에 완료 알림
```
