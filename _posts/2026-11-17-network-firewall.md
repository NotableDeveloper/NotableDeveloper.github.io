---
title: '방화벽 적용하기'
description: "Cisco Packet Tracer에서 Access list와 CBAC 필터로 웹 서버 접근을 제한하는 실습."
date: 2026-11-17 23:12:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, firewall, acl, cbac]
---

아래와 같은 네트워크 구조에서 보라색 컴퓨터만 웹 서버(www.aaa.com)에 접근할 수 있도록 주황색 라우터에 Access list 및 CBAC 필터를 적용해보자.

![Untitled](/assets/images/network/practice/firewall/figure-01.webp)

주황색 라우터에 다음과 같은 액세스 리스트를 설정하면 된다.

```
access-list  100  permit tcp  host  10.1.2.2  host  10.1.3.2   eq 80
access-list  100  deny tcp  any  host  10.1.3.2   eq 80
access-list  100  permit ip  any   any

in fa 0/0
ip access-group  100 in
```

이후, CBAC 필터를 다음과 같이 적용한다.

```
ip inspcet name 123 http
in fa 1/0
ip inspect 123 in
```
