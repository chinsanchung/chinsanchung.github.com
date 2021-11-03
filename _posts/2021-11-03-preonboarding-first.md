---
title: '원티드 프리온보딩 코스 첫 번째 과제 후기'
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - web
toc: true
toc_sticky: true
toc_labe: 목차
description: 프리온보딩 백엔드 코스의 첫 과제를 수행하면서 겪은 경험을 작성합니다.
tags:
  - 위코드
  - 원티드
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 첫 과제를 수행한 경험을 정리하고자 이 글을 작성했습니다.

## 1. 개요

이번 첫 번째 프로젝트는 [에이모](https://aimmo.co.kr/)에서 제시한 과제입니다. 주제는 게시글 API 를 만드는 것으로, 세부적인 사항은 아래와 같습니다.

- 게시글 카테고리
- 게시글 검색
- 대댓글(1 depth)
  - 대댓글 pagination
- 게시글 읽힘 수
  - 같은 User가 게시글을 읽는 경우 count 수 증가하면 안 됨
- Rest API 설계
- Unit Test
- 1000만건 이상의 데이터를 넣고 성능테스트 진행 결과 필요

에이모에서 선호하는 기술 스택을 python flask 로 적었지만, 그 부분은 선택사항이기에 Express 를 사용해서 작성했습니다. 다만, 데이터베이스는 mongodb 를 필수로 지정해야 했습니다.

프로젝트의 [리포지토리](https://github.com/lhj0621/nodeswork_boards_server)에서 사용했던 기술, 맡은 업무, API 명세 등 더 자세한 정보를 확인하실 수 있습니다.

## 2. 주요 사항 회고

### 데이터 스키마 설계 및 기본 설정

#### 데이터 스키마 설계

데이터 구조를 설계하는 것을 최우선으로 시작했습니다. 모여서 회의를 했는데, 처음의 데이터 구조는 아래 그림과 같았습니다.

![첫 번째 erd](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-11-03-preonboarding-first-erd_1.PNG?raw=true)

하지만 개발을 진행하면서, 댓글의 배열 또는 대댓글의 배열로 하는 구조에서는 pagination, 삭제하지 않은 댓글만을 목록으로 불러오는 처리, 그리고 특성 댓글 삭제를 수행하는 것이 까다롭고 어렵다는 것을 알게 되었습니다.

그래서 다시 고민을 했는데, 다른 사이트의 게시판을 살펴보면서, 1. 게시글을 조회할 때는 본문과 첫 번째 댓글 페이지를 보여준다는 것, 2. 페이지 버튼으로 댓글 pagination 을 수행하지만 url query 가 바뀌지 않는 점, 마지막 3. 댓글의 댓글은 따로 버튼을 클릭해서 불러온다는 것을 확인했습니다.

그래서 생각한 것이 댓글과 대댓글의 API 를 새로 만들어서 댓글들만 새로 받아오도록 수행하는 것이라 가정을 했습니다. 그리고 댓글이나 대댓글 모두 도큐먼트 안의 배열로 하지 말고, 아래 그림처럼 다 따로 저장하는 형식으로 수정하면 댓글이나 대댓글 삭제도 저번보다 더 간단한 작업이 가능하다고 판단했습니다.

조원들과 의논한 결과 아래 그림과 같이 데이터베이스를 다시 설계했습니다.

![두 번째 erd](https://github.com/lhj0621/imagetemp/blob/master/2021-11-03%2004;10;50.PNG?raw=true)

#### 기본 설정

다행히 1년 동안 mongodb 를 이용해서 개발을 했기 때문에 [mongo atlas](https://www.mongodb.com/atlas/database)를 이용한 서버 생성과, 애플리케이션에 코드를 작성해 데이터베이스에 접근할 수 있도록 설정하는 것은 수월하게 진행했습니다.
그리고 Express APP 에 cookie-parser 같은 미들웨어를 연결하고 실행하는 작업을 조원들과 함께 작업했습니다.

### 개발 진행

혼자 하는 것이 아니다보니, 제 마음대로 커밋 메시지나 브랜치 이름을 정하지 않고, 조원들과 서로 합의한 기준으로 형상 관리를 했습니다. 그 기준은 아래와 같습니다.

- 브랜치 원칙
  - `main` - 운영 서버 배포 브런치
  - `develop` - 테스트 서버 배포 브런치
  - `feature` - 각 기능별 브런치 (ex. feature/kafka)
  - 네이밍은 케밥케이스를 사용한다(ex : branch-name-1)
- 커밋의 원칙
  - `feat` - 신규 기능 추가
  - `fix` - 버그 수정
  - `docs` - 문서 수정
  - `style` - 코딩 스타일 관련(로직 변경x)
  - `refactor` - 코드 리팩터링
  - `test` - 테스트 코드 관련
  - `ci` - CI/CD 관련
  - `chore` - 기타
- PR 의 원칙

## 3. 프로젝트를 마치며

이틀이라는 짧은 기간 내에 프로젝트를 제작하는 과정은 어려웠습니다. 조원들과 협업하면서 형상 관리에 신경을 써야하는 부분은 비록 제대로 작성해야 한다는 압박감은 있었지만 그 덕분에 깔끔하게 기록을 남길 수 있었고, 프로젝트의 진행 과정을 보다 상세하게 알 수 있었습니다.
