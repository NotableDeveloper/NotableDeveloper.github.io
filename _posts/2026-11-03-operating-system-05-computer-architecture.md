---
title: "05. Computer Architecture"
description: "버스, 인터럽트, 트랩, 입출력 컨트롤러와 DMA의 동작을 정리한다."
date: 2026-11-03 23:08:00 +0900
categories: [Operating System]
tags: [os, computer-architecture, interrupt]
---

## Bus

버스(Bus)는 CPU, RAM, 입출력 장치 사이에서 데이터를 옮기는 경로다. 전송하는 데이터는 실제 Data일 수도 있고 Address일 수도 있다. 장치가 하나의 경로를 나누어 쓰고 서로 속도가 다르면 병목이 생긴다. 예를 들어 CPU가 초당 5 단위를 처리하고 메모리가 초당 3 단위를 처리한다면 CPU에서 처리할 수 있는 2 단위가 메모리 때문에 손실되어 전체 속도가 메모리에 제한된다. RAM, Disk, Network, I/O 사이의 속도 차이를 고려해 속도가 비슷한 장치끼리 계층적인 버스로 연결한다.

```text
CPU ↔ 고속 버스 ↔ RAM
CPU ↔ 중속 버스 ↔ 장치 제어기
CPU ↔ 저속 버스 ↔ 느린 I/O 장치
```

## 이벤트 처리: Interrupt와 Trap

운영체제는 이벤트를 Interrupt와 Trap으로 나누어 처리한다. Interrupt는 키보드 입력이나 네트워크 패킷 도착처럼 현재 명령과 무관하게 비동기적으로 발생한다. CPU는 한 번에 하나씩 인터럽트를 처리한다. 현재 실행 상태를 저장해 커널 작업을 마친 뒤 돌아올 수 있게 하고, 해당 이벤트를 처리할 ISR(Interrupt Service Routine)로 이동한다. 처리가 끝나면 저장한 상태를 복구하고 중단된 지점부터 실행을 재개한다.

인터럽트 처리 중 다른 인터럽트가 생기면 커널에 전달된다. 하드웨어 장치별 우선순위가 있으며 ISR은 가능한 짧게 유지해야 한다. 타이머 인터럽트를 사용하면 실행 중인 프로세스를 일정 시간마다 중단할 수 있어 Time Sharing이 가능하다.

```text
인터럽트 감지 → CPU 상태 저장 → ISR 실행
       → 상태 복구 → 중단된 실행 재개
```

Trap은 애플리케이션 실행 중 0으로 나누기처럼 동기적으로 발생하는 이벤트다. Trap Handler 또는 Trap Service Routine이 처리한다. 처리 흐름은 인터럽트와 비슷하지만 Trap은 동기 이벤트이므로 인터럽트처럼 실행 상태를 별도로 저장·복원하지 않는다. System Call도 Trap을 통해 커널로 요청한다.

## 입출력 레지스터

입출력 장치에는 Control, Status, Input, Output 레지스터가 있고 이 레지스터들을 주 메모리에 매핑한다. CPU는 메모리 주소를 통해 장치에 접근한다. I/O Controller는 주기적으로 레지스터를 살펴보고 값이 바뀌면 해당 장치 동작을 수행한다.

Polling, 즉 Programmed I/O에서는 CPU가 레지스터를 반복해서 확인하고 입출력 작업을 직접 수행한다. 요청 결과가 레지스터에 바로 나타나는 빠른 장치에서는 괜찮지만, 결과가 늦게 나오는 경우 CPU는 기다려야 한다. Polling 입출력은 데이터 크기가 커질수록 CPU가 처리해야 할 작업과 시간이 증가한다.

## DMA

DMA(Direct Memory Access)는 별도의 컨트롤러 또는 프로세서가 입출력을 처리하는 방식이다. CPU는 장치 레지스터를 하나씩 직접 다루는 대신 전송할 메모리 영역과 요청을 지정한다. DMA가 장치와 RAM 사이에서 데이터를 옮기고 전송이 끝나면 CPU에 알린다. 디스크에서 RAM으로 읽어 들일 때와 RAM의 데이터를 디스크로 기록할 때 모두 이 흐름을 사용할 수 있다.

```text
CPU → 전송 요청 / 설정 → DMA Controller
DMA Controller → 데이터 전송 → 장치 ↔ RAM
DMA Controller → 전송 완료 알림 → CPU
```

```text
Polling I/O → CPU ↔ 장치 레지스터 (CPU가 대기하며 처리)
DMA I/O → CPU 설정 → DMA가 장치 ↔ RAM 사이 전송 → CPU에 완료 알림
```
