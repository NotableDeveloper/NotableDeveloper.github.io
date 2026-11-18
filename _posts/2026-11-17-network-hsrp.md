---
title: 'HSRP 설정 하기'
description: "Cisco Packet Tracer에서 VLAN별 액티브 라우터를 설정하는 HSRP 실습."
date: 2026-11-17 23:06:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, hsrp, fhrp]
---

> **실습 목표** · 단말이 가상 IP 주소를 기본 게이트웨이로 사용하도록 HSRP를 구성한다. VLAN별로 우선순위를 다르게 설정해 두 라우터가 액티브 역할을 나누도록 한다.

아래와 같은 네트워크 구성에서 파란색 라우터과 빨간색 라우터를 각각 VLAN 10과 VLAN 20에 대한 **액티브 라우터**로 설정해보자.

![Untitled](/assets/images/network/practice/hsrp/figure-01.webp)

파란색 라우터에는 아래와 같이 입력해준다.

```
R1(config)# interface fastethernet 2/0
R1(config-if)# standby 10 ip 10.1.6.1
R1(config-if)# standby 10 priority 150
// 액티브 라우터가 되기 위한 첫 번째 조건을 무시한다.
R1(config-if)# standby 10 preempt

R1(config)# interface fastethernet 3/0
R1(config-if)# standby 20 ip 10.1.7.1
R1(config-if)# standby 20 priority 50
```

빨간색 라우터에는 아래와 같이 입력해준다.

```
R1(config)# interface fastethernet 2/0
R1(config-if)# standby 10 ip 10.1.6.1
R1(config-if)# standby 10 priority 50

R1(config)# interface fastethernet 3/0
R1(config-if)# standby 20 ip 10.1.7.1
R1(config-if)# standby 20 priority 150
// 액티브 라우터가 되기 위한 첫 번째 조건을 무시한다.
R1(config-if)# standby 20 preempt
```

이제 두 라우터가 **각각 논리적**으로 10.1.6.1과 10.1.7.1이 하나씩 생겨나게 된 것이다.
