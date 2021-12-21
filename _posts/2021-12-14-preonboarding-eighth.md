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

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 여덟 번째 과제를 수행한 경험을 정리합니다. [Github Repository](https://github.com/chinsanchung/preonboarding-wantedlab)에서 코드를 확인하실 수 있습니다.

## 개요

### 과제 안내

각 언어로 구분하여 회사, 회사명과 태그를 등록하고, 그것을 조회하는 API를 제작합니다.

1. 회사명의 일부만 들어가도 검색이 되는 회사명 자동 완성 기능 API
2. 회사 이름으로 회사를 검색하는 API
3. 새로운 회사를 등록하는 API

## 회고

### flask_restx

플라스크에서 REST API를 간단하고 쉽게 만드는 것을 돕는 [Flask-RESTX](https://flask-restx.readthedocs.io/en/latest/)를 사용했습니다. 더 이상 업데이트를 하지 않는 [Flask-RESTPlus](https://github.com/noirbizarre/flask-restplus)를 기반으로 만든 라이브러리로, 일관된 데코레이터와 도구로 API를 구현하는 것을 돕고 Swagger 문서로 추출할 수 있습니다.

### 컨트롤러, 서비스 패턴으로 구현하기

코드의 가독성을 높이고 역할을 명확히 구분하기 위해, 요청과 응답을 수행하는 컨트롤러와 비즈니스 로직을 수행하는 서비스로 기능을 분리했습니다.

#### 컨트롤러에서 요청 값을 파싱하기

플라스크의 요청 데이터에 접근하기 위해 인자 값을 파싱할 필요가 있습니다. 파싱하는 과정에서 데이터의 유효성을 검증하고, 딕셔너리 형식으로 변환하여 사용할 수 있도록 합니다.

아래는 회사를 등록할 때의 요청 값을 파싱하는 코드입니다.

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

`reqparse.RequestParser`, `parser.add_argument`으로 요청 값을 불러와 검증을 수행합니다. 만약 형식이 다르거나 없을 경우 에러를 응답으로 보냅니다. 마지막으로 `parser.parse_args`으로 딕셔너리 형식으로 변환합니다.

### SQLAlchemy 로 데이터 다루기

데이터를 다룰 ORM 은 [SQLAlchemy](https://www.sqlalchemy.org/)를 사용했습니다. [점프 투 플라스크](https://wikidocs.net/book/4542)에서 가장 많이 사용하는 ORM 이라고 언급한 점을 참고했습니다.

SQLAlchemy 에서 가장 생소했던 개념은 조회로 얻은 데이터가 딕셔너리가 아닌 SQLAlchemy 형식이라는 것입니다. 처음 조회했던 데이터를 그대로 응답으로 보내면 `Object of type 000 is not JSON serializable` 에러가 발생하고, 딕셔너리처럼 키로 조회하려고 하면 `000 object is not subscriptable` 에러가 발생합니다.

이것을 해결하기 위해서 딕셔너리로 변환하는 과정이 필요합니다. stack overflow 의 ["How to convert SQLAlchemy row object to a Python dict?"](https://stackoverflow.com/a/37350445)을 참고해서 작성한 코드입니다.

```python
from sqlalchemy import inspect

def convert_orm_to_dict(row):
    return {
        column.key: getattr(row, column.key) for column in inspect(row).mapper.column_attrs
    }

```

Object 의 속성 값을 가져오는 `getattr`을, SQLAlchemy objects 에 대한 정보를 얻는 `inspect` 모듈을 사용했습니다. `inspect(row)`로 데이터의 정보를 얻은 후, 컬럼 프로퍼티의 네임스페이스를 가져오는 `mapper.column_attrs`를 반복문을 사용해 키와 값을 딕셔너리에 저장합니다.

## 프로젝트를 마치며

1. 한 번도 다뤄본 적이 없는 Python 플라스크를 다루면서, 제게 맞는 공부법이 무엇인지에 대해 다시 생각했습니다. 교재와 공식 문서를 보면서 작성했는데, 이 방법은 어느 정도 지식이 있는 상태에서는 유용하지만 아무것도 모르는 상태에서는 제게 맞는 방법이 아님을 깨달았습니다. 현재까지 가장 효율이 좋았던 것은 인터넷 강의에서 제시한대로 하나씩 따라하며 개념을 익히는 것인데, 상대적으로 오랜 시간이 걸린다는 단점이 있습니다. 그래서 앞으로 새로운 개념을 배울 때 우선 인터넷 강의로 기본적인 개념과 필요한 부분을 듣고, 나머지는 책이나 인터넷으로 보충하는 형식으로 공부해보려 합니다.
2. 특정 회사를 조회하는 API 를 작성할 때 하루 안에 완성하려는 마음에, 구현한 후 POSTMAN 으로 성공함을 확인하자마자 커밋과 PR, 머지를 한 적이 있었습니다. 그런데 test_app.py 로 테스트를 한 결과 회사를 발견하지 않았을 경우를 처리하지 않은 것을 찾았습니다. 다시 수정해서 머지를 했지만, 실제 현업에서 해선 안 되는 행동을 했다는 생각이 들어 크게 반성했습니다. 다음부터는 기능을 작성하자마자 테스트 코드를 작성하거나, 또는 테스트 주도 개발으로 테스트 케이스를 우선 작성하고 거기에 코드를 리펙토링하는 방법을 택해 이런 일이 발생하지 않도록 할 것입니다.
