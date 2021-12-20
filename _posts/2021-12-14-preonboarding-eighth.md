---
title: '원티드 프리온보딩 코스 여덟 번째 과제 후기'
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
description: 원티드랩에서 제시해주신 회사 등록 및 조회 API 과제의 제작 과정을 정리합니다.
excerpt: 원티드랩에서 제시해주신 회사 등록 및 조회 API 과제의 제작 과정을 정리합니다.
tags:
  - 위코드
  - 원티드
  - 원티드랩
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 일곱 번째 과제를 수행한 경험을 정리합니다. [Github Repository](https://github.com/chinsanchung/preonboarding_wantedlab)에서 코드를 확인하실 수 있습니다.

파이썬의 플라스크 프레임워크에 대해 이해하기 위해 개인적으로 수행한 과제입니다.

## 회고

### 컨트롤러, 서비스 패턴으로 구현하기

컨트롤러에서 요청과 응답을 처리하고, 비즈니스 로직은 서비스에서 수행하도록 했습니다. 플라스크에서는 요청 값을

### flask_restx 로 REST API 를 제작하기

작성 중입니다.

#### 컨트롤러에서 요청 값을 검증하기

요청으로 들어온 값을 인식해 활용하기 위해 `flask_restx`의 `reqparse`를 사용했습니다. `reqparse`로 요청 값을 올바르게 입력했는지 검증하고, 딕셔너리로 변환하여 파이썬에서 사용할 수 있게 합니다.

```python
parser = reqparse.RequestParser()
parser.add_argument(
    "company_name",
    type=dict,
    required=True,
)
parser.add_argument(
    "tags",
    type=list,
    required=True,
    location="json",
)
parser.add_argument("x-wanted-language", type=str, location="headers")
args = parser.parse_args()
```
