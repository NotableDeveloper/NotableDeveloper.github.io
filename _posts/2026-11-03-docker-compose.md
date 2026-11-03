---
title: "Docker Compose로 여러 컨테이너 구성하기"
description: "Compose YAML의 구조와 서비스·네트워크·볼륨 정의, 개발·테스트 환경에서의 활용 방법을 정리한다."
date: 2026-11-03 21:08:00 +0900
categories: [Docker]
tags: [docker, compose, yaml]
---

Docker Compose는 여러 컨테이너로 이뤄진 애플리케이션의 서비스, 네트워크, 볼륨과 설정을 YAML 파일로 정의하고 함께 실행하는 도구다. 프로젝트별 로컬 개발 환경을 구성하거나 CI/CD 과정에서 격리된 테스트 환경을 만드는 데 쓸 수 있다.

Compose 파일을 재사용하면 같은 구성을 다른 환경에서 재현할 수 있다. 한 번의 명령으로 관련 서비스를 함께 시작하고 중지할 수 있어 수동 설정을 줄인다.

```text
compose.yaml
├─ services: 애플리케이션과 의존 서비스
├─ networks: 서비스 사이 통신 경로
├─ volumes: 영구 데이터 저장
├─ configs: 런타임 설정 파일
└─ secrets: 민감한 설정 값
```

Compose는 주로 개발과 테스트, 단일 호스트에서의 선언적 운영에 적합한 경량 도구다. 대규모 프로덕션 운영에서는 서비스 검색, 자동 복구, 확장 같은 기능이 필요한지 따져 더 강한 오케스트레이션 도구를 검토한다.

## 기본 서비스 실행

Compose 정의 파일에서 서비스별 이미지나 빌드 경로를 지정한 뒤 다음 명령으로 묶어서 실행한다.

```yaml
services:
  app:
    image: ubuntu:latest
```

```sh
docker compose up
```

서비스가 여러 개면 Compose가 파일에 선언된 대로 각각 컨테이너를 만들고 하나의 프로젝트 단위로 관리한다. 프로젝트 이름과 서비스 이름은 컨테이너 식별에 반영된다. Compose 스키마는 설치된 Docker Compose에서 지원되는 형식을 사용한다. 오래된 `version` 표기와 명령 사용 예는 현재 설치 환경에서 호환성을 확인해야 한다.

## Dockerfile 빌드

서비스에 Dockerfile을 지정해 Compose가 빌드한 이미지로 컨테이너를 실행할 수 있다.

```yaml
services:
  app:
    build:
      context: ./dfile
      dockerfile: Dockerfile
      args:
        buildno: "1"
```

`context`는 빌드에 사용할 파일 경로이고 `dockerfile`은 파일 이름을 지정한다. Dockerfile의 `ARG` 값은 빌드 시점에 `args`로 전달한다.

## 서비스 설정의 대표 항목

- `tty`: 컨테이너에 터미널을 연결할 수 있도록 TTY를 할당한다.
- `ports`: 호스트 포트와 컨테이너 포트를 연결한다. 예: `"8080:80"`.
- `working_dir`: 컨테이너의 작업 디렉터리를 지정한다.
- `environment`: 서비스 컨테이너의 런타임 환경 변수를 지정한다. `.env` 파일이나 `--env-file`을 활용할 수 있다.
- `command`: 이미지 기본 명령을 덮어쓴다. Compose의 `command`가 Dockerfile의 기본 `CMD`보다 우선한다.
- `depends_on`: 서비스 시작 순서의 의존성을 표현한다. 의존 서비스가 애플리케이션 수준에서 준비되었음을 항상 보장하는 것은 아니다.
- `volumes`: 저장 공간을 서비스에 연결한다. 오래된 예에는 `volumes_from`이 나오지만, 명시적인 `volumes` 정의가 데이터 연결 의도를 더 잘 드러낸다.
- `healthcheck`: 서비스 컨테이너의 상태를 확인하는 명령과 주기를 정의한다.

```yaml
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    depends_on:
      - db
  db:
    image: mariadb:latest
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
```

민감한 암호를 YAML에 직접 기록하거나 버전 관리에 포함하지 않는다. 환경 변수 파일과 배포 환경의 비밀 관리 방식을 구분하고, 저장소에 올릴 파일을 점검한다.

## YAML 파일의 상위 항목

Compose 구성은 `services`를 중심으로 네트워크, 볼륨, 설정 파일과 민감 데이터 항목을 함께 정의한다. `services`는 필수이며 각 서비스는 이미지와 런타임 옵션으로 기술한다. 네트워크는 기본 또는 사용자 정의 드라이버를 연결하고, 볼륨은 데이터를 저장한다. `configs`는 런타임에 서비스에 제공할 구성 자료를 다루며, `secrets`는 암호·토큰처럼 보호해야 할 데이터를 전달하기 위한 항목이다.

YAML은 들여쓰기로 구조를 표현하는 데이터 직렬화 형식이다. `.yml` 또는 `.yaml` 확장자를 사용하며 공백 들여쓰기를 일관되게 유지해야 한다.

## Compose 설치와 버전

Docker Desktop은 보통 Compose를 함께 제공한다. 별도로 설치하는 환경은 Docker 엔진과 Compose 플러그인의 호환되는 설치 절차를 공식 문서에서 확인한다. 예전 바이너리를 직접 내려받는 설치 방법은 버전 및 운영체제에 따라 달라질 수 있으므로 고정된 명령을 그대로 실행하지 않는다.

## 네트워크 구성

Compose는 기본 네트워크에 서비스를 연결한다. 서비스별 네트워크를 지정하면 한 서비스에 하나 이상의 네트워크를 연결할 수 있다. `bridge`, `host`, `overlay`, `none` 등 네트워크 드라이버를 선택할 수 있으며 지원 여부는 플랫폼에 따라 다르다. 미리 만든 네트워크는 `external: true`로 참조한다.

IPAM 설정으로 서브넷, 할당 범위, 게이트웨이, 예약 주소를 선언할 수도 있다. 고정 IP를 쓸 때는 네트워크의 IPAM 범위와 충돌하지 않도록 주소 계획을 함께 관리한다. 포트 매핑은 짧은 형식인 `"8080:80"` 또는 긴 형식으로 작성한다. 긴 형식에는 컨테이너 포트(`target`), 공개 포트(`published`), 호스트 주소(`host_ip`), 프로토콜(`protocol`)을 지정할 수 있다.

다음은 `frontend`와 `backend` 네트워크를 만들고 서비스를 필요한 네트워크에 연결하는 형태다. 같은 네트워크에 연결된 서비스는 서비스 이름을 이용해 서로 찾을 수 있다.

```yaml
services:
  web:
    image: nginx:latest
    networks:
      - frontend
  app:
    image: example/app
    networks:
      - frontend
      - backend
  db:
    image: mariadb:latest
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
```

사용자 정의 네트워크에 `ipam`을 설정하면 `subnet`, `ip_range`, `gateway`, `aux_addresses` 등을 지정할 수 있다. `ipv4_address` 같은 고정 주소는 해당 네트워크에서 유효한 범위를 사용해야 한다. 이미 존재하는 네트워크를 쓸 때는 `external: true`를 선언한다.

## 볼륨

최상위 `volumes`에서 여러 서비스가 재사용할 볼륨을 정의하고, 서비스의 `volumes`에서 이를 컨테이너 경로에 연결한다. 이미 만든 볼륨은 `external: true`로 지정할 수 있다.

```yaml
services:
  web:
    image: nginx:latest
    volumes:
      - web-logs:/var/log/nginx
volumes:
  web-logs: {}
```

## 상태 확인

서비스의 `healthcheck`에는 상태 확인 명령(`test`), 확인 간격(`interval`), 실행 제한 시간(`timeout`), 초기 대기 기간(`start_period`), 연속 실패 허용 횟수(`retries`)를 설정한다. 아래 예시는 Nginx의 로컬 응답을 확인하는 구성이다.

```yaml
services:
  web:
    image: nginx:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 10s
      timeout: 30s
      retries: 3
      start_period: 30s
```

`healthcheck`는 컨테이너 안에서 지정한 명령이 실행될 수 있어야 한다. 이미지에 `curl`이 없으면 다른 검사 방법을 써야 한다.

## 배포 메타데이터

`deploy` 항목에는 배포 플랫폼이 사용할 라벨, 실행 모드와 복제 수, CPU·메모리 한계와 예약량, 재시작 정책 등을 적을 수 있다. `global`은 노드마다 하나씩, `replicated`는 지정한 개수만큼 실행하는 모델이다. 재시작 정책에는 종료 조건과 재시도 사이 지연을 설정할 수 있다. 이 항목의 동작은 실행 플랫폼과 Compose 모드에 따라 달라지므로 적용 환경에서 지원 여부를 확인한다.

## Compose 명령

```text
docker compose <하위 명령>
```

주요 하위 명령은 `up`(서비스 생성과 시작), `ps`(컨테이너 목록), `logs`(로그), `run`(일회성 명령), `start`, `stop`, `restart`, `pause`, `unpause`, `port`, `config`, `kill`, `rm`, `down`이다. `up`에는 백그라운드 실행(`-d`), 의존 서비스 생략(`--no-deps`), 빌드 여부(`--build`, `--no-build`), 대기 제한(`-t/--timeout`), 서비스 복제 수(`--scale`)를 지정할 수 있다.

`down`은 프로젝트 리소스를 내리는 명령이다. `--rmi all`은 이미지를, `--rmi local`은 사용자 지정 태그가 없는 이미지를, `-v/--volumes`는 Compose가 정의한 데이터 볼륨을 대상으로 한다. 볼륨 옵션은 데이터를 삭제할 수 있으므로 실행 전에 보존 대상인지 확인한다.
