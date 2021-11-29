---
title: '원티드 프리온보딩 코스 다섯 번째 과제 후기'
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - review
toc: true
toc_sticky: true
toc_labe: 목차
description: 프리온보딩 백엔드 코스의 다섯 번째 과제를 수행하면서 겪은 경험을 작성합니다.
tags:
  - 위코드
  - 원티드
  - 휴먼스케이프
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 다섯 번째 과제를 수행한 경험을 정리하고자 이 글을 작성했습니다. [Github repository 링크](https://github.com/wanted-wecode-subjects/humanscape-subject)에서 작성한 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [휴먼스케이프](https://humanscape.io/kr/index.html)에서 제시한 과제입니다.

**휴먼스케이프**는 희귀난치성 질환을 가진 환자들이 직접 업로드한 건강 데이터를 기반으로 자신에게 꼭 맞는 치료제 개발 및 임상시험 정보를 확인하고 참여할 수 있는 [레어노트](https://humanscape.io/kr/service_rarenote.html) 등 헬스케어 서비스를 제공하는 기업입니다.

휴먼스케이프에서 제시한 주제는 임상 정보를 수집하고 그것을 출력하는 API 제작입니다.

### 과제 안내

다음 사항들을 충족하는 서비스를 구현해주세요.

- 전부다 쌓는 API
- 임상정보를 수집하는 batch task
  - 참고: [https://www.data.go.kr/data/3074271/fileData.do#/API 목록/GETuddi%3Acfc19dda-6f75-4c57-86a8-bb9c8b103887](https://www.data.go.kr/data/3074271/fileData.do#/API%20%EB%AA%A9%EB%A1%9D/GETuddi%3Acfc19dda-6f75-4c57-86a8-bb9c8b103887)
- 수집한 임상정보에 대한 API
  - 특정 임상정보 읽기(키 값은 자유)
- 수집한 임상정보 리스트 API
  - 최근 일주일내에 업데이트(변경사항이 있는) 된 임상정보 리스트
    - pagination 기능

## 회고

작성 중입니다.

### 시간문제...

- 시간 계산. date-fns 를 이용함.
- 하루 단위는 오늘 0시 0분 0초 ~ 다음날 0시 0분 0초.
- 7일 단위는 6일전 0,0,0 ~ 오늘 0,0,0
- toISOString 으로 변환하여 쿼리문에 넣음.
- 저장할 때의 방식도 utc 시간으로 통일하려고 협의 예정.
- TypeOrmModule.forRoot 에서 timezone 을 설정하려 했지만 mysql, mariadb 에서만 지원함.

- 하루 밀려서 검색이 되고 있음. 7/15 입력 -> 범위는 7/14 15:00 ~ 7/15 15:00 인데 2021-07-15T19:39:44.000Z 가 나오고 있음.
  - 검색어를 7/14으로 1일을 빼고 계산하니 정상적으로 나옴. 원인이 무엇일지.

### 데이터베이스

- approval_time 으로 인덱스를 생성?

### 테스트

- 서비스 테스트 중에, ClinicalServie 를 can't resolve dependencies of the clinicalService
- 레포지토리의 선언 방식을 수정함.

## 프로젝트를 마치며

시간 계산.
