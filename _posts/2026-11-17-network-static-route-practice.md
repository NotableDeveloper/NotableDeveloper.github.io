---
title: 'Static route 설정하기'
description: "Cisco Packet Tracer에서 라우팅 프로토콜 없이 정적 라우팅 테이블을 구성하는 실습."
date: 2026-11-17 23:07:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, static-route]
---

아래와 같은 네트워크에서 라우팅 프로토콜을 사용하지 않고 Static route만을 설정하여 라우팅 테이블을 채워보자.

![Untitled](/assets/images/network/practice/static-route-practice/figure-01.webp)

먼저, 각각의 라우터마다 아래와 같이 라우터의 세 가지 기본 명령을 입력해준다.

```
configure terminal
/*
	라우터의 기본 세 가지 명령
*/
interface fastethernet 0/0
no shutdown
ip address 10.1.0.1 255.255.255.0
```

이후, 간접적으로 연결된 라우터(= not-connected router)와 연결 정보를 정적으로 채우기 위해 다음과 같은 형식으로 명령어를 입력해주면 된다.

```
ip route [간접 라우터의 대역] [Subnet mask] [간접 라우터로 Forwarding 하기 위한 IP 주소]
```

예를 들어, 맨 왼쪽에 위치한 라우터(10.1.0.0 / 24)를 기준으로는 아래와 같이 입력해준다.

```
ip route 10.1.1.0 255.255.255.0 10.1.0.2
ip route 10.1.2.0 255.255.255.0 10.1.0.2
```

위와 같이 입력하였으면 10.1.0.0 / 24 대역과 간접적으로 연결된 두 개의 대역(10.1.1.0 / 24,10.1.2.0 / 24)을 **10.1.0.2를 통하여 Forwarding 할 수 있음**을 명시적으로 나타낸 것이 된다.
