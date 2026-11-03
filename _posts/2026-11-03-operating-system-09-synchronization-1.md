---
title: "09. 동기화 1"
description: "Race Condition과 Critical Section 문제, 상호 배제 해결 방법을 정리한다."
date: 2026-11-03 23:04:00 +0900
categories: [Operating System]
tags: [os, synchronization]
---

## Race Condition

공유 데이터에 여러 실행 흐름이 동시에 접근해 실행 순서에 따라 결과가 달라지는 현상이다. 예를 들어 잔액 1000원에 500원을 입금하고 동시에 500원을 출금하면 올바른 최종 잔액은 1000원이어야 한다. 하지만 기계어 명령의 실행 순서가 섞이면 1500원 또는 500원이 될 수 있다.

```text
입금: 잔액 읽기(1000) → 더하기(500) → 저장(1500)
출금: 잔액 읽기(1000) → 빼기(500) → 저장(500)

두 흐름이 섞이면 마지막 저장값에 따라 1500 또는 500이 남을 수 있다.
```

## Critical Section

공유 데이터를 접근하는 코드 영역을 Critical Section이라고 한다. 해결책은 다음 조건을 만족해야 한다.

1. **Mutual Exclusion**: 한 번에 하나의 프로세스만 임계 영역에 들어간다.
2. **Progress**: 임계 영역이 비어 있고 진입을 원하는 프로세스가 있으면 진입을 결정할 수 있어야 한다.
3. **Bounded Waiting**: 진입 요청 후 무한정 기다리지 않아야 한다.

## 시도와 한계

### turn 변수

공유 정수 `turn`으로 어느 쪽 차례인지 표시한다. 상호 배제는 만족하지만 진행과 bounded waiting을 보장하지 못한다.

### flag 배열

각 프로세스가 임계 영역에 들어가고 싶은지 Boolean flag로 표시한다. 두 프로세스가 동시에 flag를 올리면 서로 기다려 진행 조건이 깨질 수 있다.

### Peterson's Solution

flag와 turn을 함께 사용해 두 프로세스가 임계 영역을 공유하도록 한다. 세 조건을 만족한다.

```text
프로세스 i 진입:
  flag[i] = true
  turn = j
  while (flag[j] && turn == j) 기다림
  critical section
  flag[i] = false
```

이 방법은 두 프로세스에 대한 해결책이다.

## 하드웨어 동기화

하드웨어 동기화 명령은 원자적으로 수행된다. 원자적이라는 것은 명령이 실행되는 도중 인터럽트되지 않는다는 뜻이다. `lock` Boolean을 두고 Test-and-Set 또는 Swap 명령을 사용하는 방식이 있다.

```text
lock = false
진입 시 원자적으로 lock 확인 및 설정
  이미 잠김 → 대기
  열림      → 잠그고 임계 영역 진입
임계 영역 종료 → lock 해제
```
