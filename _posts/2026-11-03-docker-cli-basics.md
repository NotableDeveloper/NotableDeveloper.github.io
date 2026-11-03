---
title: "Docker CLI 기본 명령어"
description: "Docker 사용자 권한을 설정하고 엔진 상태와 리소스 현황을 조회하는 기본 명령을 살펴본다."
date: 2026-11-03 21:01:00 +0900
categories: [Docker]
tags: [docker, cli, linux]
---

Docker는 엔진 상태부터 이미지, 컨테이너, 볼륨, 네트워크까지 CLI로 관리한다. 모든 명령과 옵션을 외우기보다 필요한 작업에 맞춰 명령 레퍼런스를 확인하는 편이 좋다.

## Docker 그룹 권한

Linux에서 일반 사용자에게 Docker 데몬을 사용할 권한을 주려면 사용자를 `docker` 그룹에 추가한다.

```sh
sudo usermod -aG docker <username>
```

그룹 변경은 새 로그인 세션에서 적용될 수 있다. `sudo`를 붙이면 현재 명령만 관리자 권한으로 실행할 수 있지만, 그룹 설정과는 권한 관리 방식이 다르다. Docker 그룹에 속한 사용자는 호스트에서 높은 권한을 행사할 수 있으므로 신뢰할 수 있는 사용자만 추가해야 한다.

## 엔진 상태와 리소스 확인

```sh
docker version       # 클라이언트와 엔진 버전
docker system info   # 엔진 및 실행 환경 정보
docker system df     # Docker 리소스의 디스크 사용량
docker system events # 엔진 이벤트 스트림
```

`docker system info`는 실행 중인 컨테이너와 이미지 수, 운영체제와 커널 등 엔진 정보를 보여준다. `docker system events`에는 이미지 생성이나 컨테이너 시작 같은 이벤트가 나타난다.

## 사용하지 않는 리소스 정리

`docker system prune`은 사용되지 않는 Docker 리소스를 정리할 때 쓰인다. 삭제 범위는 옵션에 따라 달라질 수 있으므로 실행 전에 `docker system prune --help`로 대상과 옵션을 확인한다. 필요하다고 생각한 이미지나 네트워크가 함께 삭제되지 않도록 주의한다.

Docker CLI의 상세 옵션은 공식 명령 레퍼런스를 기준으로 확인한다. Docker 엔진 버전과 명령의 하위 호환성도 함께 살펴보면 환경 차이로 인한 혼란을 줄일 수 있다.
