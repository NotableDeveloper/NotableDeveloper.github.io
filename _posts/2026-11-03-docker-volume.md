---
title: "Docker Volume과 컨테이너 데이터 관리"
description: "볼륨과 바인드 마운트, tmpfs의 차이와 생성·조회·연결 명령을 정리한다."
date: 2026-11-03 21:04:00 +0900
categories: [Docker]
tags: [docker, volume, storage]
---

컨테이너를 제거해도 데이터가 남아야 하거나 여러 컨테이너가 같은 데이터를 읽고 써야 할 때 저장 공간을 컨테이너의 생명주기와 분리해야 한다. Docker는 볼륨, 바인드 마운트, tmpfs 마운트를 제공한다.

## 세 가지 저장 방식

| 방식 | 데이터 위치와 용도 |
| --- | --- |
| Volume | Docker가 만들고 관리하는 호스트 저장 공간이다. 호스트의 임의 경로와 분리해 관리할 수 있고 백업·이동에 적합하다. |
| Bind mount | 호스트의 지정한 파일이나 디렉터리를 컨테이너에 연결한다. 호스트 디렉터리 구조에 의존한다. |
| tmpfs | 호스트 디스크에 보존하지 않는 임시 메모리 기반 저장 공간이다. 컨테이너 생명주기 동안 필요한 임시 데이터에 쓸 수 있다. |

볼륨은 영구 데이터 저장, 여러 컨테이너 간 공유, 태스크 내 임시 저장 공간, 외부 볼륨 드라이버 연동에 활용한다.

## 생성·삭제·조회

```sh
docker volume create [옵션] <이름>
docker volume rm [옵션] <이름>
docker volume prune [옵션]
docker volume ls [옵션]
docker volume inspect [옵션] <볼륨>
```

이름을 지정하지 않으면 임의 이름을 받을 수 있다. `--opt`는 로컬 드라이버의 마운트 옵션을 지정한다. 예를 들어 tmpfs 유형과 크기를 지정하는 구성은 다음과 같이 표현할 수 있다.

```sh
docker volume create --driver local \
  --opt type=tmpfs --opt device=tmpfs \
  --opt o=size=100m,uid=1000 sample
```

사용하지 않는 볼륨을 정리할 때는 먼저 연결된 컨테이너와 보존해야 할 데이터가 있는지 확인한다. 강제 삭제 옵션은 실행 중인 컨테이너에 영향을 줄 수 있다.

## 컨테이너에 연결하기

컨테이너를 시작하면서 Docker 볼륨을 연결하는 예다.

```sh
docker volume create sample
docker run -it --name sample-container -v sample:/vol ubuntu /bin/bash
```

호스트의 특정 경로를 연결하는 바인드 마운트는 다음처럼 쓴다.

```sh
docker run -it --name sample-container \
  -v /home/user/docker:/vol ubuntu /bin/bash
```

컨테이너 안에서 `/vol`에 쓴 파일은 연결된 저장 공간에 기록된다. `docker container create --volume <볼륨> <이미지>`로 생성 단계에서 볼륨을 지정할 수도 있다. 이미 만들어진 컨테이너의 마운트 구성을 바꾸기는 어렵기 때문에 필요한 마운트로 새 컨테이너를 만드는 방식이 적절하다.

```text
Volume:      Docker가 관리하는 영역 ──마운트──▶ 컨테이너 경로
Bind mount:  호스트의 특정 경로    ──마운트──▶ 컨테이너 경로
```
