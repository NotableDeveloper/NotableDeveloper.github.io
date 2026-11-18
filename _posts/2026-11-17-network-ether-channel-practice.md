---
title: '이더 채널 적용하기'
description: "Cisco Packet Tracer에서 두 스위치 사이의 링크를 EtherChannel로 묶는 실습."
date: 2026-11-17 23:05:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, etherchannel]
---

> **실습 목표** · 두 스위치 사이의 물리 포트들을 같은 채널 그룹에 넣어 여러 회선을 하나의 논리 링크로 구성한다.

아래의 두 스위치의 링크를 이중으로 연결하고, 이더 채널을 설정해보자.

![Untitled](/assets/images/network/practice/ether-channel-practice/figure-01.webp)

우선 두 스위치 간에 링크를 하나 더 추가해준다.

![Untitled](/assets/images/network/practice/ether-channel-practice/figure-02.webp)

이제 각각의 스위치에 아래와 같이 입력해준다.

```
Switch> enable
Switch# configure terminal
Switch(config)# interface fastethernet 0/4
Switch(config-if)# channel-group 1 mode on
Switch(config)# interface fastethernet 0/6
Switch(config-if)# channel-group 1 mode on
```
