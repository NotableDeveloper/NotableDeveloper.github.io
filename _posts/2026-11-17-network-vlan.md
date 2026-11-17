---
title: "VLAN 설정하기"
description: "Cisco Packet Tracer에서 VLAN과 Trunk를 사용해 논리적으로 분리된 네트워크를 구성하는 실습."
date: 2026-11-17 23:03:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, vlan, trunk]
---

이 글의 실습은 Cisco Packet Tracer로 진행한다. 하나의 스위치가 논리적인 두 개의 스위치처럼 동작하도록 VLAN을 설정한다.

```text
Switch> enable
Switch# configure terminal
Switch(config)# vlan 10
Switch(config-vlan) vlan 20
Switch(config-vlan) exit
Switch(config-if)# interface fastEthernet 0/4
Switch(config-if)# switchport access vlan 10
Switch(config-if)# exit
Switch(config-if)# interface fastEthernet 0/1
Switch(config-if)# switchport access vlan 20
Switch(config-if)# exit
```

나머지 포트도 같은 방식으로 VLAN 10 또는 VLAN 20을 할당한다. Trunk를 적용하면 케이블을 적게 사용하면서 VLAN을 구성할 수 있다.

```text
Switch(config-if)# interface fastEthernet 0/3
Switch(config-if)# switchport mode trunk
Switch(config-if)# exit
Switch(config-if)# interface fastEthernet 0/4
Switch(config-if)# switchport mode trunk
Switch(config-if)# exit
```

## 실습 화면

![VLAN 구성 실습 화면 01](/assets/images/network/practice/vlan/figure-01.webp)

![VLAN 구성 실습 화면 02](/assets/images/network/practice/vlan/figure-02.webp)

![VLAN 구성 실습 화면 03](/assets/images/network/practice/vlan/figure-03.webp)
