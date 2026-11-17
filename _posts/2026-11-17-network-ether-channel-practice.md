---
title: "이더 채널 적용하기"
description: "Cisco Packet Tracer에서 두 스위치 사이의 링크를 EtherChannel로 묶는 실습."
date: 2026-11-17 23:05:00 +0900
categories: [Network, Practice]
tags: [network, packet-tracer, etherchannel]
---

이 글의 실습은 Cisco Packet Tracer로 진행한다. 두 스위치 사이의 링크를 하나 더 추가하고 각각의 스위치에 다음 명령을 입력한다.

```text
Switch> enable
Switch# configure terminal
Switch(config)# interface fastethernet 0/4
Switch(config-if)# channel-group 1 mode on
Switch(config)# interface fastethernet 0/6
Switch(config-if)# channel-group 1 mode on
```

## 실습 화면

![EtherChannel 구성 실습 화면 01](/assets/images/network/practice/ether-channel-practice/figure-01.webp)

![EtherChannel 구성 실습 화면 02](/assets/images/network/practice/ether-channel-practice/figure-02.webp)
