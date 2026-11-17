---
title: "BGP 설정하기"
description: "Cisco Packet Tracer에서 세 영역 사이에 BGP를 설정하고 내부에는 OSPF를 적용하는 실습."
date: 2026-11-17 23:10:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, bgp, ospf]
---

이 글의 실습은 Cisco Packet Tracer로 진행한다.

```text
router bgp 111
neighbor 88.1.2.2 remote-as 222
neighbor 100.1.1.2 remote-as 333
redistribute connected
```

```text
router bgp 333
neighbor 99.1.2.2 remote-as 222
neighbor 100.1.1.1 remote-as 333
redistribute connected
```

```text
router bgp 222
neighbor 88.1.2.1 remote-as 111
neighbor 99.1.2.1 remote-as 333
network 88.1.3.0 mask 255.255.255.0
```

사설 영역 정보는 라우팅 테이블에 올리지 않고 DMZ 영역만 올린다. 내부에는 다음과 같이 OSPF를 가동한다.

```text
router ospf 100
network 10.0.0.0 0.255.255.255 area 0
```

## 실습 화면

![BGP 구성 실습 화면 01](/assets/images/network/practice/bgp/figure-01.webp)
