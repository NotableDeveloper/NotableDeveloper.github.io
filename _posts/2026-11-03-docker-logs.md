---
title: "Docker 컨테이너 로그 조회와 로깅 드라이버"
description: "컨테이너 로그를 확인하는 명령, 로깅 드라이버의 역할, 드라이버 옵션과 저장 경로를 정리한다."
date: 2026-11-03 21:06:00 +0900
categories: [Docker]
tags: [docker, logging, observability]
---

컨테이너 로그는 애플리케이션 문제를 추적하고 성능을 살피며 보안 이벤트를 확인하는 자료다. 법률이나 조직 정책에 따라 기록과 보관이 필요한 정보가 있는지도 확인해야 한다.

## 로그 조회

```sh
docker container logs [옵션] <컨테이너 ID 또는 이름>
```

자주 쓰는 조회 옵션은 다음과 같다.

| 옵션 | 설명 |
| --- | --- |
| `--details` | 로그에 제공된 추가 세부 정보를 표시한다. |
| `--follow`, `-f` | 로그 출력을 계속 따라간다. 새 로그가 생기면 이어서 출력한다. |
| `--since` | 지정한 타임스탬프 또는 상대 시간 이후의 로그만 표시한다. |
| `--tail` | 로그 끝에서 표시할 줄 수를 정한다. |
| `--timestamps`, `-t` | 타임스탬프를 표시한다. |
| `--until` | 지정한 타임스탬프 또는 상대 시간 이전의 로그만 표시한다. |

로그 드라이버는 컨테이너 로그를 수집하거나 외부 서비스에 전달하는 역할을 한다. Docker 데몬의 기본 드라이버는 `json-file`이며 로그를 JSON 형식으로 기록한다. `local` 드라이버는 로그 회전을 지원하고 저장 효율을 고려한 형식을 사용한다. `awslogs` 드라이버는 Amazon CloudWatch Logs로 로그를 보낼 수 있다. 사용 사례에 맞는 내장 드라이버가 없으면 플러그인 드라이버를 검토할 수 있다.

축약 예시 — 실제 환경에서 관측한 로그가 아니다.

```text
$ docker container logs web
2026-11-03T12:00:00Z [info] server started
2026-11-03T12:00:03Z [error] request failed: connection timeout
```

## 드라이버 옵션

로그 드라이버와 드라이버별 옵션은 서로 다르다. 아래는 문서에서 소개하는 대표적인 설정 항목이다.

| 옵션 | 의미 |
| --- | --- |
| `max-size` | 로그 파일의 최대 크기 |
| `max-file` | 로그 파일의 최대 개수 |
| `labels` | 로그에 포함할 라벨 |
| `env` | 로그에 포함할 환경 변수 |
| `mode` | 로그 버퍼링 모드 |
| `syslog-address` | 외부 syslog 서버 주소 |
| `syslog-facility` | 사용할 syslog 시설 |

드라이버 지정과 옵션은 컨테이너를 만들 때 설정한다. 드라이버마다 지원 항목이 다르므로 적용 전 공식 드라이버 문서를 확인한다. 예를 들어 `json-file`과 `local` 드라이버는 각자의 설정과 회전 동작을 가진다.

```text
컨테이너 stdout/stderr
          │
          ▼
Docker 데몬 ── 로깅 드라이버 ── 로그 전달기 ── 중앙 로그 관리 시스템
                ├─ json-file                  ├─ Fluentd/syslog 계열
                ├─ local                     └─ 플러그인 드라이버
                └─ awslogs ───────────────────▶ CloudWatch Logs
```

드라이버를 구성할 때는 `max-size`, `max-file` 같은 파일 회전 관련 옵션과 전달 대상이 요구하는 `syslog-address`, `syslog-facility`를 확인한다. `labels`, `env`, `mode`는 로그에 포함할 메타데이터와 버퍼링 동작을 정하며 지원 여부는 드라이버마다 다르다. `--log-driver`와 `--log-opt`는 로그 조회 옵션이 아니라 컨테이너를 만들 때 지정하는 로깅 설정이다.

```sh
docker run --log-driver=<드라이버> --log-opt=<키>=<값> <이미지>
```

## 기본 로그 파일 위치

| 환경 | 경로 예시 |
| --- | --- |
| Linux | `/var/lib/docker/containers/<container_id>/<container_id>-json.log` |
| Windows Docker Desktop | `C:\ProgramData\Docker\containers\<container_id>\<container_id>-json.log` |
| Windows WSL 2 | `\\wsl$\docker-desktop-data\data\docker\containers\<container_id>\<container_id>-json.log` |

호스트 파일에 직접 접근하기보다 우선 `docker logs`와 설정된 로깅 드라이버를 통해 확인한다. 로그 보관 기간과 민감 데이터 기록 여부도 운영 정책에 맞춰 정한다.
