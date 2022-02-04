---
title: 'Docker 정리'
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - experience
toc: true
toc_sticky: true
toc_labe: 목차
description: 처음으로 Docker를 접하면서 기본적인 개념과 용어를 정리합니다.
excerpt: 처음으로 Docker를 접하면서 기본적인 개념과 용어를 정리합니다.
tags:
  - docker
---

## 개요

### Docker 소개

Docker(이하 도커)는 컨테이너를 이용해 앱을 특정 환경으로 격리함으로써 "오직 내 컴퓨터에서만 작동이 되던 문제"를 해결해 준 플랫폼입니다. CLI 기반 워크플로우로 누구나 쉽게 빌드, 공유, 실행할 수 있습니다.

### 특징

- 도커가 설치되어 있다면 어디서든 환경 설정을 하지 않고도 컨테이너의 애플리케이션을 실행할 수 있습니다.
- 최소한의 애플리케이션 런타임의 요구 사항을 가진 컨테이너로, 배포 시간을 줄입니다.
- 컨테이너는 서로 독립적입니다. 그래서 하나의 서비스(컨테이너)를 유지보수하거나 갱신하더라도 다른 서비스에 영향을 주지 않습니다.

## 관련 용어

### 1. 컨테이너

컨테이너는 어떠한 컴퓨팅 환경에서도 빠르고 안정적으로 실행할 수 있도록 코드와 종속성을 패키징하는 소프트웨어의 표준 단위입니다.

여러 개의 컨테이너를 동일한 가상 머신에서 격리된 프로세스로 실행합니다. 가상 머신보다 작은 용량을 가지고, 더 많은 애플리케이션을 처리할 수 있으며, 더 적은 수의 가상 머신 또는 OS 를 요구합니다.

### 2. 이미지

아미지는 파일이나 설정 값 등 애플리케이션 실행에 필요한 모든 것을 포함하는 독립적이고 가벼운 파일 시스템입니다. 모든 컨테이너는 이미지를 바탕으로 실행합니다.

### 3. Dockerfile

`Dockerfile`은 이미지를 생성하는 과정을 명령문 형식으로 작성한 파일으로, `docker build` 명령을 통해 자동으로 빌드하여 이미지를 생성합니다.

- `FROM`: Dockerfile 은 반드시 `FROM` 명령어로 시작해야 합니다. 빌드를 할 기본 이미지를 지정합니다. 이후의 모든 명령문은 여기서 지정한 이미지를 기반으로 작동합니다. `AS <name>`으로 이름을 지정하여 이후에 참조할 수 있습니다.
  - `FROM [--platform=<platform>] <image> [AS <name>]`
  - `FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]`: 태그를 찾을 수 없으면 에러를 리턴합니다.
  - `FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]`
- `RUN`: 애플리케이션을 빌드할 때 명령을 수행합니다. 아래의 두 가지 방식으로 작성할 수 있습니다.
  - `RUN <command>`
  - `RUN ["executable", "param1", "param2"]`
- WORKDIR: `RUN`, `CMD`, `COPY`, `ENTRYPOINT`, `ADD` 명령어에 대한 작업 디렉토리를 설정합니다.
- COPY: `<src>`에서 파일이나 디렉토리를 복사하고, `<dest>`의 컨테이너 파일 시스템에 붙여넣기합니다.
  - `COPY <src>... <dest>`
  - 선택 사항으로 `--from=<name>`을 통해 이전 빌드 단계(`FROM .. AS <name>`)를 `<src>` 위치로 참조할 수 있습니다.
- CMD: 컨테이너를 시작할 때 실행합니다. 파일 당 한 번만 작성할 수 있습니다. (여러 번 작성하더라도 마지막 `CMD` 만 실행합니다.)
  - `CMD ["executable","param1","param2"]`: 이 방식을 권장합니다.

### 4. dockerignore

gitignore 처럼 특정 파일이나 디렉토리를 제외할 수 있습니다.

## CLI 명령어

주요 명령어만 정리합니다.

### docker build

`docker build [OPTIONS] PATH | URL | -`

`Dockerfile`으로부터 이미지를 빌드합니다. 빌드할 대상은 지정한 `PATH` 또는 `URL`의 파일들입니다. `URL`에는 깃허브 레포지토리, tar 압축 파일, 텍스트 파일을 참조할 수 있습니다.

- `--label`
- `--output`, `-o`: 결과물의 경로
- `--quiet`, `-q`: 빌드 결과를 압축하여, 성공 시 이미지 ID 만을 프린트합니다.
- `--tag`, `-t`: 이름을 짓거나, 선택적으로 "name:tag" 형식으로 작성하면 이름과 태그를 작성할 수 있습니다.

### docker ps

현재 실행하고 있는 컨테이너의 목록을 보여줍니다.

- `--all`, `-a`: 실행뿐만 아니라 정지한 컨테이너까지 전부 보여줍니다.
- `--filter`, `-f`: 필터링한 목록을 출력합니다. id, name, label 등 여러 가지 필터를 지원합니다. [공식 문서 참고](https://docs.docker.com/engine/reference/commandline/ps/#filtering)
- `--size`, `-s`: 파일 크기도 보여줍니다.
- `--quiet` , `-q`: 컨테이너 ID 만 보여줍니다.

### docker images

이미지 목록을 출력합니다.

- `--all`, `-a`: 재사용성을 높이고 디스크 사용량을 줄이며 각 단계를 캐시할 수 있도록 도와 빌드의 속도를 높이는 중간 레이어까지 전부 표시합니다.
- `--filter`, `-f`: 필터링한 목록을 출력합니다. id, name, label 등 여러 가지 필터를 지원합니다. [공식 문서 참고](https://docs.docker.com/engine/reference/commandline/images/#options)

### docker pull

`docker pull [OPTIONS] NAME[:TAG|@DIGEST]`

레포지토리나 [Docker Hub](https://hub.docker.com/) 레지스트리로부터 이미지를 불러옵니다.

### docker rm, docker rmi

`docker rm [OPTIONS] CONTAINER [CONTAINER...]` 은 하나 또는 여러 개의 컨테이너를 삭제합니다.

`docker rmi [OPTIONS] IMAGE [IMAGE...]` 은 하나 또는 여러 개의 이미지를 삭제합니다.

- `--force`, `-f`: 실행 중인 컨테이너(이미지)를 강제로 종료하고 삭제합니다.

### docker run

새로운 컨테이너를 만들고 실행합니다.

`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

- `--name`: 컨테이너의 이름을 할당합니다.
- `--publish`, `-p`: 컨테이너의 포트를 호스트에게 게시합니다.
  - 예 `docker run -p 127.0.0.1:80:8080/tcp httpd`: 컨테이너 포트 8080을 호스트 127.0.0.1의 TCP 포트 80으로 바인딩합니다.
  - 예 `docker run -p 80:5000 httpd`: 호스트 포트 80을 컨테이너 포트 5000에 연결합니다.

### docker start

`docker start [OPTIONS] CONTAINER [CONTAINER...]`

하나 또는 여러 개의 정지한 컨테이너를 실행합니다.

## 참고 자료

- [Docker 공식 사이트](https://www.docker.com)
- [Docker 공식 문서](https://docs.docker.com)
- [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- ["컨테이너 혁명을 주도하는" 도커의 의미와 장단점](https://www.itworld.co.kr/news/203644)
- [Docker(도커)란? 도커 컨테이너 실행 및 사용법](https://www.redhat.com/ko/topics/containers/what-is-docker)
- [7.2. Advantages of Using Docker](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/7.0_release_notes/sect-red_hat_enterprise_linux-7.0_release_notes-linux_containers_with_docker_format-advantages_of_using_docker)
