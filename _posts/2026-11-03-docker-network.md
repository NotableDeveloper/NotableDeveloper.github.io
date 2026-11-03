---
title: "Docker 네트워크와 컨테이너 통신"
description: "네트워크 드라이버의 개념과 bridge, host, overlay 유형 및 네트워크 관리 명령을 설명한다."
date: 2026-11-03 21:05:00 +0900
categories: [Docker]
tags: [docker, network, container]
---

컨테이너는 격리된 환경에서 실행되며 네트워크 인터페이스와 IP 주소를 가질 수 있다. Docker 네트워크를 구성하면 컨테이너끼리 통신하거나 호스트와 포트를 연결할 수 있다.

## 네트워크 드라이버

- **bridge**: 기본 드라이버다. 같은 Docker 호스트의 컨테이너를 연결할 때 주로 사용한다.
- **host**: 컨테이너가 호스트의 네트워크를 사용한다. 격리가 약해지고 호스트 네트워크 자원에 직접 접근할 수 있으므로 보안과 포트 충돌을 고려한다.
- **overlay**: 여러 Docker 데몬 사이에 네트워크를 구성해 Swarm 노드의 컨테이너와 서비스가 통신하게 한다.

```text
Bridge:  호스트의 bridge 네트워크 ── 컨테이너 A ↔ 컨테이너 B
Host:    호스트 네트워크 공간을 컨테이너가 공유
Overlay: Docker 노드 A ── 오버레이 네트워크 ── Docker 노드 B
```

Docker는 네트워크 이름과 드라이버를 사용해 네트워크를 만들고, 컨테이너를 네트워크에 연결한다.

## 주요 명령

```sh
docker network ls
docker network inspect <네트워크>
docker network create [옵션] <이름>
docker network rm <네트워크>
docker network connect [옵션] <네트워크> <컨테이너>
docker network disconnect [옵션] <네트워크> <컨테이너>
docker run --network=<네트워크> --name=<이름> <이미지>
docker container create --network=<네트워크> <이미지>
```

`--expose`는 컨테이너가 사용하는 포트를 선언하는 설정이고, 외부에서 호스트 포트로 접근하도록 연결하는 작업은 `-p` 또는 `--publish`로 지정한다. 포트 매핑은 `호스트포트:컨테이너포트` 형태다.

```sh
docker run -p 127.0.0.1:8080:80 nginx
```

호스트의 특정 주소에 포트를 묶을 수도 있다. 포트를 공개할 때는 외부 노출 범위를 확인해 필요 이상으로 넓게 열지 않는다.
