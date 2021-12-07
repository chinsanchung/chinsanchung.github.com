---
title: "Flask 초기 환경 설정"
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
description: 파이썬 Flask 를 설치하면서 겪은 환경 설정을 정리합니다.
excerpt: 파이썬 Flask 를 설치하면서 겪은 환경 설정을 정리합니다.
tags:
  - python
  - Flask
---

지금까지 TypeScript, Node.js, NestJs 로 백엔드를 개발했는데, 프리온보딩 과정을 진행하면서 파이썬을 백엔드 개발에 활용하는 기업이 꽤 많다는 것을 알았습니다. 그래서 이번에 [Flask(이하 플라스크)](https://flask-docs-kr.readthedocs.io/ko/latest/index.html) 를 설치하여 어떤 방식으로 개발하는지 알아보려 합니다.

## 초기 설정

우선 가장 처음 설치해야 하는 도구는 virtualenv 입니다.

### virtualenv

파이썬 프로젝트마다 다른 파이썬 버젼을 사용하는 경우가 많은데(예시로 플라스크는 파이썬 3.x 버전을 지원하지 않습니다.) 데스크톱의 파이썬을 매번 바꾸는 것은 어렵고 번거롭습니다. 그래서 [virtualenv](https://virtualenv.pypa.io/en/latest/)으로 가상 환경을 만들어 한 데스크톱 안에 여러 버전의 파이썬과 라이브러리를 설치해 운영하는 것이 필요합니다.

공식 문서에서는 `sudo pip install virtualenv`으로 설치하지만, 제 맥북에서는 "pip: command not found" 에러가 발생했습니다. `pip` 대신 python3 과 함께 자동으로 설치되는 `pip3`으로 virtualenv 를 설치했습니다.

설치가 완료되면, 프로젝트를 수행할 폴더에 venv 로 가상 환경을 설정합니다.

```bash
$ mkdir myproject
$ cd myproject
$ virtualenv venv
```

이제 아래의 명령어로 가상 환경을 실행할 수 있습니다.

```bash
# Mac OS
$ . venv/bin/activate
# 윈도우
$ . venv/bin/activate
```

가상 환경을 실행할 때마다 activate 명령을 입력하는 것은 번거롭습니다. 필요한 명령을 한 파일에 모아 실행시켜주는 [virtualenvwrapper](https://virtualenvwrapper.readthedocs.io/en/latest/)가 필요한 시점입니다.

### virtualenvwrapper

virtualenvwrapper 으로 배치 파일을 만들어 프로젝트에 필요한 명령 및 환경 변수를 설정할 수 있습니다.

`sudo pip3 install virtualenvwrapper`으로 도구를 설치합니다.

프로젝트 폴더에 가상 환경을 설정했기에 "/Users/username/Documents/Code/flask_project/venv/bin" 경로에 virtualenvwrapper.sh 파일을 작성합니다.(프로젝트의 경로는 임의로 설정하시면 됩니다.)

> 편의성을 위해 앞으로 `/Users/username/Documents/Code/flask_project` 을 `프로젝트 경로`라고 줄여서 작성하겠습니다.

```bash
#!/bin/zsh

cd /Users/username/Documents/Code/flask_project
source /Users/username/Documents/Code/flask_project/venv/bin/activate
```

- `chmod a+x 프로젝트 경로/venv/bin/virtualenvwrapper.sh`을 터미널에 입력해 실행 권한을 부여합니다. ("zsh: permission denied" 에러를 방지합니다.)
- 그 다음 `source 프로젝트 경로/venv/bin/virtualenvwrapper.sh`을 터미널에 입력해 엔터를 누르면 virtualenvwrapper.sh 를 실행합니다.

## 플라스크 설치하기

가상 환경을 실행한 상태에서 `pip install Flask`를 명령어로 입력해 가상 환경에서 플라스크를 설치합니다.

플라스크를 사용하면서 여러 환경 변수를 적용할 텐데, 프로젝트를 실행할 때마다 환경 변수를 입력하는 것은 번거롭습니다. 이번에도 virtualenvwrapper.ts 에 환경 변수를 설정할 수 있습니다.

```bash
#!/bin/zsh

cd /Users/username/Documents/Code/flask_project
export FLASK_APP=test.py
export FLASK_ENV=development
source /Users/username/Documents/Code/flask_project/venv/bin/activate
```

- `export FLASK_APP=test.py`: FLASK_APP 환경 변수가 지정되지 않은 경우 자동으로 app.py를 기본 애플리케이션으로 인식합니다. 예시로 app.py 대신 test.py 를 기본 애플리케이션으로 하고 싶을 때 환경 변수로 입력합니다.
- `export FLASK_ENV=development`: 개발 환경으로 설정해 디버그 모드를 활성화합니다.

virtualenvwrapper.sh 파일을 터미널에서 실행한 후 `flask run`을 입력하면 위의 명령을 적용하여 플라스크 앱을 실행합니다.

## 참고 문서

- [플라스크 문서 한글판](https://flask-docs-kr.readthedocs.io/ko/latest/)
- [점프 투 플라스크](https://wikidocs.net/book/4542)
