---
title: "DMZ Network"
description: "DMZ Network에 관한 네트워크 이론."
date: 2026-11-13 08:05:59 +0900
categories: [Network, Theory]
tags: [network]
---

DMZ(Demilitarized Zone) 네트워크란 외부에 서비스 제공 시에 내부 자원을 보호하기 위해 내부 네트워크 구간과 외부 네트워크 구간 사이에 접근 제한을 수행하는 영역을 말한다.

![DMZ Network 그림 1](/assets/images/network-theory/dmz-network/figure-01.png)

보통 DMZ에는 외부 공격자의 침입으로부터 안전하게 보호되어야 하는 메일 서버, 웹 서버, DNS 서버 등이 위치한다. 웹 서버들은 인터넷을 통해 외부에서 접속이 가능하지만 DB 서버의 경우 상단에 별도의 Firewall을 구축시켜 분리된 네트워크의 DMZ 존을 구성하여 외부로부터의 접근을 제한하게 된다. 

외부에서 내부로 트래픽이 들어오지 못하게 하려면 이처럼 외부 네트워크와 내부 네트워크를 분리하고 외부에서 DMZ는 자유롭게 들어올 수 있게 설정하되 외부에서 내부로는 들어오지 못하도록 필터링을 하는 것이 좋다. 이 때, 내부 보안정책에 따라 내부에서 외부나 DMZ로 가는 것을 제한하는 것이 **방화벽**이다.
