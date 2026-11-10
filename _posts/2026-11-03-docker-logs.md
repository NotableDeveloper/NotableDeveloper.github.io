---
title: "Docker 컨테이너 로그 조회와 로깅 드라이버"
description: "컨테이너 로그를 조회하는 방법과 Docker 로깅 드라이버, 로그 전달 흐름 및 저장 경로를 살펴본다."
date: 2026-11-03 21:06:00 +0900
categories: [Docker]
tags: [docker, logging, observability]
---

Docker에서 로깅은 컨테이너가 출력하는 로그를 기록하고 조회하는 작업이다. 로그는 애플리케이션 문제를 추적하고, 성능을 모니터링하고, 보안 이벤트를 확인하는 데 사용한다. 법률이나 조직의 규정에 따라 기록하고 보관해야 하는 정보가 있는지도 확인해야 한다.

- **디버깅**: 애플리케이션이 예상대로 작동하지 않을 때 로그를 확인해 문제가 발생한 위치와 원인을 찾는다.
- **성능 모니터링**: 로그를 분석해 시스템 성능 저하를 감지하고 필요한 최적화를 수행한다.
- **보안**: 비정상적인 접근 시도나 시스템 침입과 같은 보안 이벤트를 추적한다.
- **규정 준수**: 법률이나 규정에서 요구하는 정보를 기록하고 보관한다.

## 컨테이너 로그 조회

`docker container logs` 명령은 컨테이너의 로그를 출력한다. 컨테이너 ID나 이름으로 대상을 지정한다.

```sh
docker container logs [옵션] <컨테이너 ID 또는 이름>
```

| 옵션 | 설명 |
| --- | --- |
| `--details` | 로그에 제공된 추가 세부 정보를 표시한다. |
| `--follow`, `-f` | 로그 출력을 계속 따라간다. 새 로그가 생성되면 이어서 표시한다. |
| `--since` | 지정한 타임스탬프 또는 상대 시간 이후의 로그만 표시한다. 예: `2013-01-02T13:23:37Z`, `42m` |
| `--tail`, `-n` | 로그 끝에서 표시할 줄 수를 지정한다. |
| `--timestamps`, `-t` | 로그에 타임스탬프를 표시한다. |
| `--until` | 지정한 타임스탬프 또는 상대 시간 이전의 로그만 표시한다. 예: `2013-01-02T13:23:37Z`, `42m` |

## Docker 로깅 드라이버

Docker 로깅 드라이버는 컨테이너 로그를 수집하고 파일이나 외부 로그 서비스로 전달한다. Docker는 여러 내장 드라이버를 제공하지만, 모든 사용 사례를 내장 드라이버만으로 지원할 수는 없다. 필요하면 로깅 드라이버 플러그인으로 별도의 드라이버를 구현해 사용할 수 있다.

Docker 데몬의 기본 로깅 드라이버는 `json-file`이다. 이 드라이버는 컨테이너 로그를 JSON 형식으로 저장한다. `local` 드라이버는 로그 회전을 기본으로 수행하고 저장 효율을 고려한 파일 형식을 사용한다. `awslogs` 드라이버는 로그를 Amazon CloudWatch Logs로 전송한다.

컨테이너의 표준 출력과 표준 오류는 Docker 데몬의 로깅 드라이버로 전달된다. 드라이버는 로그를 파일, 시스템 로그, 로그 수집기 또는 중앙 로그 관리 시스템으로 보낸다.

```text
컨테이너 stdout/stderr
        │
        ▼
Docker 데몬
        │
        ├─ json-file ──▶ 파일 ─────────────────────────▶ 로그 수집기 ──▶ 중앙 로그 관리 시스템
        ├─ journald ───▶ system journal ───────────────▶ 로그 수집기 ──▶ 중앙 로그 관리 시스템
        ├─ syslog ─────▶ UDP / TCP / TLS ──▶ syslog 수집기 ──HTTP/HTTPS──▶ 중앙 로그 관리 시스템
        ├─ fluentd ────▶ TCP ──▶ Fluentd 수집기 ───────▶ 중앙 로그 관리 시스템
        ├─ etwlogs ────▶ Windows Event Store ──────────▶ 로그 수집기 ──▶ 중앙 로그 관리 시스템
        ├─ splunk ─────▶ TCP (HTTP / HTTPS) ───────────▶ Splunk
        ├─ gelf ───────▶ UDP ──────────────────────────▶ Graylog 등
        ├─ logentries ─▶ TCP / HTTPS ──────────────────▶ Logentries
        ├─ awslogs ────▶ TCP / HTTPS ──────────────────▶ Amazon CloudWatch Logs
        └─ gcplogs ────▶ TCP / HTTPS ──────────────────▶ Google Cloud Logging
```

Docker에서 제공하는 드라이버와 로그가 전달되는 위치는 다음과 같다.

| 드라이버 | 로그 처리 |
| --- | --- |
| `none` | 컨테이너 로그를 저장하지 않는다. `docker logs` 명령도 출력하지 않는다. |
| `local` | 로그를 저장 효율을 고려한 형식으로 저장하고 기본 로그 회전을 수행한다. |
| `json-file` | 로그를 JSON 형식으로 저장한다. Docker의 기본 로깅 드라이버다. |
| `syslog` | 로그 메시지를 syslog 시설에 기록한다. syslog 데몬은 호스트에서 실행되어야 한다. |
| `journald` | 로그 메시지를 `journald`에 기록한다. `journald` 데몬은 호스트에서 실행되어야 한다. |
| `gelf` | 로그 메시지를 Graylog Extended Log Format(GELF) 엔드포인트로 전송한다. 예를 들어 Graylog 또는 Logstash에 기록할 수 있다. |
| `fluentd` | 로그 메시지를 Fluentd에 전달한다. Fluentd 데몬은 호스트에서 실행되어야 한다. |
| `awslogs` | 로그 메시지를 Amazon CloudWatch Logs에 기록한다. |
| `splunk` | HTTP Event Collector를 사용해 로그 메시지를 Splunk에 기록한다. |
| `etwlogs` | 로그 메시지를 Windows Event Tracing(ETW) 이벤트로 기록한다. Windows에서 사용할 수 있다. |
| `gcplogs` | 로그 메시지를 Google Cloud Platform(GCP) 로깅에 기록한다. |
| `logentries` | 현재 Docker의 내장 드라이버 목록에는 포함되지 않는다. 사용할 로그 서비스에 맞는 드라이버 플러그인 지원 여부를 확인한다. |

드라이버별 설정과 지원 옵션은 서로 다르다. 예를 들어 `local` 드라이버와 `json-file` 드라이버는 각자의 로그 저장 및 회전 옵션을 제공한다. 사용할 드라이버의 공식 문서에서 지원 범위와 설정 방법을 확인한다.

## 로깅 드라이버 설정

`--log-driver`와 `--log-opt`는 로그 조회 명령의 옵션이 아니다. 컨테이너를 만들 때 사용할 로깅 드라이버와 해당 드라이버의 옵션을 지정한다.

```sh
docker run \
  --log-driver=<드라이버> \
  --log-opt <키>=<값> \
  <이미지>
```

대표적인 드라이버 설정 항목은 다음과 같다. 각 옵션의 지원 여부와 의미는 드라이버마다 다르다.

| 옵션 | 의미 |
| --- | --- |
| `max-size` | 로그 파일의 최대 크기 |
| `max-file` | 보관할 로그 파일의 최대 개수 |
| `labels` | 로그에 포함할 컨테이너 라벨 |
| `env` | 로그에 포함할 환경 변수 |
| `mode` | 로그 전달 버퍼링 모드 |
| `syslog-address` | 외부 syslog 서버 주소 |
| `syslog-facility` | 사용할 syslog 시설 |

## Docker 로그 파일 경로

`json-file` 드라이버를 사용할 때 로그 파일은 Docker 데이터 디렉터리 아래에 저장된다. 운영체제와 Docker Desktop 구성에 따라 경로가 다르다.

| 환경 | 경로 |
| --- | --- |
| Linux | `/var/lib/docker/containers/<container_id>/<container_id>-json.log` |
| Windows Docker Desktop | `C:\ProgramData\Docker\containers\<container_id>\<container_id>-json.log` |
| Windows WSL 2 | `\\wsl$\docker-desktop-data\data\docker\containers\<container_id>\<container_id>-json.log` |

먼저 `docker container logs` 명령과 현재 설정된 로깅 드라이버를 통해 로그를 확인한다. 로그 파일에 직접 접근해야 하는 경우에는 Docker 엔진이 사용하는 데이터 경로와 드라이버의 저장 형식을 확인한다.
