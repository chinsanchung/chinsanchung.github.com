---
title: '원티드 프리온보딩 코스 두 번째 과제 후기'
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
description: 프리온보딩 백엔드 코스의 e두 번째 과제를 수행하면서 겪은 경험을 작성합니다.
tags:
  - 위코드
  - 원티드
  - freshcode
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 두 번째 과제를 수행한 경험을 정리하고자 이 글을 작성했습니다.

## 1. 개요

이번 프로젝트는 [freshcode](https://www.freshcode.me/)에서 제시한 과제입니다.

freshcode 는 "샐러드는 배고픈 다이어트 음식"이라는 편견을 깨고 대한민국 직장인의 건강한 식사 문화를 만드는 것이 목표로, 샐러드 구독·건강간편식 프리미엄 배송 서비스"를 제공하는 기업입니다.

주제는 게시글 API 를 만드는 것으로, 세부적인 사항은 아래와 같습니다.

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

## 2. 회고

## 3. 프로젝트를 마치며
