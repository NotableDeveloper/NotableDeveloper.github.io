---
title: "Routing Protocol - IGP"
description: "Routing Protocol - IGP에 관한 네트워크 이론."
date: 2026-11-13 08:05:59 +0900
categories: [Network, Theory]
tags: [network]
---

라우터는 간접적인 네트워크 영역에 대한 정보를 수집하기 위해 라우팅 프로토콜을 사용한다. 이 때, 라우팅 프로토콜은 적용 범위에 따라 IGP 혹은 BGP로 나눈다. 

**IGP**(Inter domain Routing Protocol)는 <u>**AS(Autonomous System) 안에서 동작하는 라우팅 프로토콜**</u>을 말하며 다음과 같은 세 가지 종류로 나눌 수 있다.

-  (RIP)
    
    라우팅 테이블 전체를 주기적으로 반복 전달하는 방식이다. Bandwidth 소모량이 많고, 긴 시간을 주기로 테이블 내용을 전파하기 때문에 Convergence Time(= 네트워크의 라우팅 테이블 갱신 시간)이 느린 편에 속한다. 단순한 알고리즘을 사용하기 때문에 CPU 소모량이 적다
    
-  (OSPF)
    
    라우터의 변화를 감지하여 그 정보만을 전달하는 방식이다. Bandwidth 소모가 적으며, 변화를 감지하는 즉시 그 정보를 전달하기 때문에 Convergence Time이 빠른 편이다. 알고리즘이 복잡하여 CPU 소모량이 많다. 이 문제는 네트워크 영역을 다음과 같이 분할하는 것으로 극복할 수 있다.
    
    ![Routing Protocol - IGP 그림 1](/assets/images/network-theory/routing-protocol-igp/figure-01.png)
    

- Hybrid (EIGRP)
    
    Distance Vector와 Link State의 장점을 수용하는 방식으로 변화된 네트워크 정보만을 전달하여 Bandwidth 소모가 적고, Convergence Time이 빠르다. 또, 알고리즘은 Distance Vector 방식이기 때문에 CPU 소모량도 적다. 단, 표준이 아닌 시스코 프로토콜이기 때문에 장비 의존성이 강하다.
