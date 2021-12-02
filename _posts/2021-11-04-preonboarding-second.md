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
description: 프리온보딩 백엔드 코스의 두 번째 과제를 수행하면서 겪은 경험을 작성합니다.
tags:
  - 위코드
  - 원티드
  - freshcode
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 두 번째 과제를 수행한 경험을 정리합니다. [GitHub Repository](https://github.com/chinsanchung/preonboarding-freshcode)에서 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [freshcode](https://www.freshcode.me/)에서 제시한 과제로, **freshcode** 는 "샐러드는 배고픈 다이어트 음식"이라는 편견을 깨고 대한민국 직장인의 건강한 식사 문화를 만드는 것이 목표로, 샐러드 구독·건강간편식 프리미엄 배송 서비스"를 제공하는 기업입니다.

### 과제 안내

1. 로그인 기능

사용자 인증을 통해 상품을 관리할 수 있어야 합니다.

- JWT 인증 방식을 이용합니다.
- 서비스 실행시 데이터베이스 또는 In Memory 상에 유저를 미리 등록해주세요.
- Request시 Header에 Authorization 키를 체크합니다.
- Authorization 키의 값이 없거나 인증 실패시 적절한 Error Handling을 해주세요.
- 상품 추가/수정/삭제는 admin 권한을 가진 사용자만 이용할 수 있습니다.
- 사용자 인증 / 인가

2. 상품 관리 기능

아래 상품 JSON 구조를 이용하여 데이터베이스 및 API를 개발해주세요.

- 서비스 실행시 데이터베이스 또는 In Memory 상에 상품 최소한 5개를 미리 생성해주세요.
- 상품 조회는 하나 또는 전체목록을 조회할 수 있으며, 전체목록은 페이징 기능이 있습니다.
  - 한 페이지 당 아이템 수는 5개 입니다.
- 사용자는 상품 조회만 가능합니다.
- 관리자는 상품 추가/수정/삭제를 할 수 있습니다.
- 상품 관리 API 개발시 적절한 Error Handling을 해주세요.

## 회고
