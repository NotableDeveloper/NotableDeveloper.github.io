---
title: "Dockerfile로 이미지 빌드와 컨테이너 실행 정의하기"
description: "Dockerfile이 배포 절차를 반복 가능하게 만드는 방식과 주요 지시문, 빌드 컨텍스트를 설명한다."
date: 2026-11-03 21:07:00 +0900
categories: [Docker]
tags: [docker, dockerfile, image]
---

컨테이너에 애플리케이션을 배포하려면 베이스 이미지를 고르고, 패키지를 설치하고, 환경 변수를 설정하고, 프로그램을 실행해야 한다. 이 작업을 매번 수동으로 반복하면 환경별 차이가 생기기 쉽다. Dockerfile은 이미지 생성과 실행에 필요한 내용을 코드로 기록한다.

```text
Dockerfile + 빌드 컨텍스트
          │ docker build
          ▼
      Docker 이미지
          │ docker run
          ▼
      실행 컨테이너
```

빌드 컨텍스트는 이미지 빌드에서 사용할 파일의 집합이다. 기본적으로 현재 디렉터리가 컨텍스트가 되며, Dockerfile 경로와 컨텍스트를 명시해 빌드한다.

```sh
docker build [옵션] <PATH | URL | ->
```

## 주요 지시문

| 지시문 | 역할과 사용 시점 |
| --- | --- |
| `FROM` | 베이스 이미지를 지정한다. 보통 Dockerfile의 첫 지시문이다. 태그나 digest로 기반 이미지를 고정할 수 있다. |
| `RUN` | 이미지 빌드 중 명령을 실행한다. 셸 형식이나 exec 배열 형식을 사용할 수 있다. 패키지 설치 등 이미지에 필요한 작업에 쓴다. |
| `CMD` | 컨테이너 시작 시 기본 명령이나 인수를 지정한다. 실행할 때 인자를 주면 기본 CMD가 대체될 수 있으며, Dockerfile에는 일반적으로 마지막 CMD만 적용된다. |
| `ENTRYPOINT` | 컨테이너가 시작될 때 실행할 기본 프로그램을 지정한다. `CMD`는 기본 인자를 제공할 수 있다. 실행 예의 `ENTRYPOINT ["top"]`과 `CMD ["-d", "10"]`은 `top -d 10`이 된다. |
| `ONBUILD` | 이미지를 기반으로 다른 이미지를 빌드할 때 실행할 작업을 등록한다. 파생 이미지 빌드에서 실행되므로 흐름 추적이 어려워지지 않도록 절제해 쓴다. |
| `HEALTHCHECK` | 컨테이너 안의 프로세스 상태를 확인할 명령과 간격, 제한 시간, 재시도 횟수 등을 지정한다. |
| `ENV` | 이미지와 컨테이너에서 사용할 환경 변수를 설정한다. |
| `WORKDIR` | 이후 지시문이 실행될 작업 디렉터리를 설정한다. 필요하면 디렉터리를 만든다. |
| `USER` | 이후 명령과 컨테이너 프로세스를 실행할 사용자를 지정한다. |
| `LABEL` | 이미지에 메타데이터를 붙인다. |
| `EXPOSE` | 컨테이너가 사용할 포트를 문서화한다. 호스트 포트 공개는 실행 시 `-p`로 설정한다. |
| `ARG` | 이미지 빌드 시 전달할 매개변수를 정의한다. 런타임 환경 변수와 구분한다. |
| `SHELL` | 셸 형식의 `RUN` 명령에 사용할 기본 셸을 바꾼다. |
| `ADD` | 빌드 컨텍스트의 파일을 이미지에 추가한다. 단순 복사에는 `COPY`가 더 명확한 경우가 많다. |
| `VOLUME` | 컨테이너 데이터가 외부 저장 공간과 연결될 위치를 선언한다. |
| `.dockerignore` | 빌드 컨텍스트에서 제외할 경로를 지정해 불필요한 파일 전달을 줄인다. |

### 포트 선언과 빌드 매개변수

`EXPOSE`는 컨테이너가 사용할 포트를 문서화한다. 호스트 포트 공개는 `docker run -p` 또는 Compose의 `ports`로 설정한다. `EXPOSE 80`만으로 호스트에서 접속할 수 있는 포트가 열리지는 않는다.

```dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y net-tools
EXPOSE 80
```

`ARG`는 이미지 빌드 중에만 쓰는 매개변수다. 빌드 때 기본값을 줄 수 있고, `docker build --build-arg your_name=...`처럼 값을 전달한다. `ENV`와 달리 `ARG` 값이 자동으로 컨테이너의 런타임 환경 변수로 남지는 않는다.

```dockerfile
FROM ubuntu:latest
ARG your_name="SeongJin"
ENV name=${your_name}
```

위 예제처럼 `ARG` 값을 `ENV`에 대입하면 결과 환경 변수는 이미지에 포함되므로, 비밀 값 전달 용도로 쓰면 안 된다.

### 셸과 파일 복사

`SHELL`은 이후 셸 형식으로 작성한 `RUN` 명령이 사용할 셸을 지정한다. 예를 들어 Ubuntu에서 Bash를 기본 셸로 설정할 수 있다.

```dockerfile
FROM ubuntu:latest
SHELL ["/bin/bash", "-c"]
RUN echo "$SHELL"
```

`ADD`는 빌드 컨텍스트 안의 파일이나 디렉터리를 이미지에 복사한다. 다음 예제는 `example.txt`를 `/root/`에 넣는다.

```dockerfile
FROM ubuntu:latest
ADD ["./example.txt", "/root/"]
```

와일드카드와 URL 입력, 로컬 tar 아카이브 자동 풀기도 지원한다. 단순 파일 복사에는 동작 범위가 더 분명한 `COPY`를 우선 고려한다. 복사 원본은 빌드 컨텍스트 아래에 있어야 하며, 컨텍스트 밖의 경로를 `../`나 절대 경로로 가져올 수 없다. URL로 원격 파일을 가져와야 하는 작업은 재현성과 검증을 고려해 별도 다운로드 단계로 명시하는 편이 낫다.

### 빌드 컨텍스트 제외 규칙

`.dockerignore` 파일은 빌드 컨텍스트에서 제외할 경로 패턴을 적는다. 아래 규칙은 `add` 디렉터리 아래의 `.log` 파일을 빌드 컨텍스트에서 제외한다.

```text
add/*.log
```

예를 들어 `ADD ./add /root/`를 사용하더라도 제외된 로그 파일은 컨텍스트에 포함되지 않는다. 불필요한 파일과 비밀 설정이 이미지 빌드로 전달되지 않도록 프로젝트에 맞는 제외 규칙을 관리한다.

### 볼륨 위치 선언

`VOLUME`은 컨테이너에서 외부 저장 공간으로 연결할 데이터 경로를 이미지에 선언한다. 다음 예제는 `/data`를 볼륨 위치로 지정한다.

```dockerfile
FROM ubuntu:latest
RUN mkdir /data
WORKDIR /data
RUN echo "hello from volume" > test.txt
VOLUME ["/data"]
```

이 선언만으로 호스트의 특정 경로가 정해지는 것은 아니다. 실제 저장 위치를 관리하려면 컨테이너 실행 시 named volume이나 bind mount를 연결하고, 데이터 보존 수명과 백업 방법을 따로 정한다.

## 빌드 예제에서 확인할 점

Dockerfile 경로와 빌드 컨텍스트를 함께 지정하려면 `-f`로 Dockerfile을 지정하고 컨텍스트 디렉터리를 마지막 인자로 둔다. `-t`에는 이미지 태그를 전달한다.

```sh
docker build -t web-sample -f ./1207/dockerfile ./
```

컨테이너 내부 서비스가 80번 포트에서 응답한다면 `HEALTHCHECK`도 컨테이너 내부 주소인 `http://localhost/`를 검사한다. `-p 8080:80`의 호스트 포트 `8080`은 컨테이너 내부 검사 주소에 쓰지 않는다.

`RUN`은 이미지를 만드는 시점에 실행하고, `CMD`와 `ENTRYPOINT`는 컨테이너가 시작될 때 적용한다. Ubuntu 기반 이미지에 Nginx를 설치하고 포트 80을 선언한 뒤 Nginx를 전경 실행하도록 구성할 수 있다.

```dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y nginx curl
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
HEALTHCHECK --interval=5m --timeout=3s \\
  CMD curl -f http://localhost/ || exit 1
```

`HEALTHCHECK`는 지정한 명령을 주기적으로 실행해 컨테이너 상태를 `healthy` 또는 `unhealthy`로 표시한다. 이 설정은 애플리케이션을 자동으로 고쳐준다는 뜻은 아니다.

`ONBUILD`는 기반 이미지가 파생되는 시점에 예약한 작업을 실행한다. 다음 기반 이미지를 빌드할 때는 실행되지 않고, 이를 `FROM`으로 지정한 자식 이미지를 빌드할 때 `touch /hello.txt`가 실행된다.

```dockerfile
# parent/Dockerfile
FROM ubuntu:latest
ONBUILD RUN touch /hello.txt
```

```dockerfile
# child/Dockerfile
FROM parent-image
```

## 환경 변수와 작업 디렉터리 예

```dockerfile
FROM ubuntu:latest
ENV HELLO=1234
WORKDIR /root
RUN touch hello.txt
WORKDIR /tmp
RUN touch hi.txt
CMD ["sh", "-c", "echo ${HELLO}"]
```

이 예제의 `RUN`은 빌드 중 `/root/hello.txt`와 `/tmp/hi.txt`를 만들고, `ENV` 값은 컨테이너 시작 시에도 사용할 수 있다. 빌드 매개변수 `ARG`를 사용하면 Compose의 `build.args`로 값을 전달할 수 있다.

실제 프로젝트에서는 베이스 이미지를 고정하고, 민감한 값은 이미지에 굽지 않으며, 빌드 컨텍스트에 불필요한 파일이 들어가지 않도록 `.dockerignore`를 관리한다.
