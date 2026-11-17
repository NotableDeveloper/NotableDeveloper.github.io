---
title: "방화벽 적용하기"
description: "Cisco Packet Tracer에서 Access list와 CBAC 필터로 웹 서버 접근을 제한하는 실습."
date: 2026-11-17 23:12:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, firewall, acl, cbac]
---

이 글의 실습은 Cisco Packet Tracer로 진행한다. 보라색 컴퓨터만 웹 서버 `www.aaa.com`에 접근하도록 주황색 라우터에 Access list와 CBAC 필터를 적용한다.

```text
access-list 100 permit tcp host 10.1.2.2 host 10.1.3.2 eq 80
access-list 100 deny tcp any host 10.1.3.2 eq 80
access-list 100 permit ip any any

interface fa0/0
ip access-group 100 in
```

```text
ip inspect name 123 http
interface fa1/0
ip inspect 123 in
```

## 실습 화면

![방화벽 적용 실습 화면 01](/assets/images/network/practice/firewall/figure-01.webp)
