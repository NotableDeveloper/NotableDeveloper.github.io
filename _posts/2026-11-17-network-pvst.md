---
title: 'PVST 적용하기'
description: "Cisco Packet Tracer에서 VLAN별 루트 스위치를 지정하는 PVST 실습."
date: 2026-11-17 23:04:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, pvst, stp]
---

아래와 같은 네트워크 구성에서 파란색 스위치와 빨간색 스위치에는 VLAN이 각각 10과 20으로 적용되어 있다. 이 때, 두 스위치를 각각 VLAN 10, VLAN 20에 대한 **루트 스위치**로 설정해보자.

![Untitled](/assets/images/network/practice/pvst/figure-01.webp)

파란색 스위치에 접근하여 다음과 같이 입력해주면 된다.

```
Switch> enable
Switch# configure terminal
Switch(config)# spanning-tree vlan 10 priority 4096
```

빨간색 스위치에 접근하여 다음과 같이 입력해준다.

```
Switch> enable
Switch# configure terminal
Switch(config)# spanning-tree vlan 20 priority 4096
```
