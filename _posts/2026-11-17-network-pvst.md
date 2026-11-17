---
title: "PVST 적용하기"
description: "Cisco Packet Tracer에서 VLAN별 루트 스위치를 지정하는 PVST 실습."
date: 2026-11-17 23:04:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, pvst, stp]
---

이 글의 실습은 Cisco Packet Tracer로 진행한다. VLAN 10과 VLAN 20이 적용된 두 스위치를 각각 해당 VLAN의 루트 스위치로 설정한다.

파란색 스위치:

```text
Switch> enable
Switch# configure terminal
Switch(config)# spanning-tree vlan 10 priority 4096
```

빨간색 스위치:

```text
Switch> enable
Switch# configure terminal
Switch(config)# spanning-tree vlan 20 priority 4096
```

## 실습 화면

![PVST 구성 실습 화면 01](/assets/images/network/practice/pvst/figure-01.webp)
