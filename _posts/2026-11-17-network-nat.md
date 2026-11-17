---
title: "NAT 설정하기"
description: "Cisco Packet Tracer에서 사설 IP와 공인 IP 사이의 NAT 변환을 설정하는 실습."
date: 2026-11-17 23:09:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, nat, ospf]
---

이 글의 실습은 Cisco Packet Tracer로 진행한다. Internet router에서 NAT 서비스를 동작시킨다.

```text
interface fastethernet 0/0
ip nat outside

interface fastethernet 1/0
ip nat inside
exit

access-list 88 permit 10.0.0.0 255.255.255.0
ip nat inside source list 88 int fa 0/0 overload
```

`0/0`은 외부 인터넷으로 향하므로 `ip nat outside`, `1/0`은 내부 네트워크로 향하므로 `ip nat inside`를 설정한다. Access list 88에는 공인 IP로 변환할 사설 대역을 등록하고 마지막 명령으로 NAT 테이블을 만든다.

## 실습 화면

![NAT 구성 실습 화면 01](/assets/images/network/practice/nat/figure-01.webp)
