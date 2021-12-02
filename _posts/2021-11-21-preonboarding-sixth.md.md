---
title: '원티드 프리온보딩 코스 여섯 번째 과제 후기'
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
description: 프리온보딩 백엔드 코스의 여섯 번째 과제를 수행하면서 겪은 경험을 작성합니다.
tags:
  - 위코드
  - 원티드
  - 디어코퍼레이션
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 여섯 번째 과제를 수행한 경험을 정리합니다. [Github Repository](https://github.com/chinsanchung/preonboarding-deer)에서 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [디어코퍼레이션](https://web.deering.co/)에서 제시한 과제로, **디어코퍼레이션**은 전동킥보드 공유서비스를 제공합니다. 사소하지만 거대한 일상의 문제들을 똑똑하고 독특한 방식으로 해결해보자는 비전으로 2019년 4월 서비스를 출시했습니다.

### 과제 안내

주제는 킥보드의 요금을 계산하는 API 입니다.

- 디어는 사용자의 요금을 계산하기 위해 다양한 상황을 고려합니다.
  - 우선 지역별로 다양한 요금제를 적용하고 있습니다. 예를 들어 건대에서 이용하는 유저는 기본요금 790원에 분당요금 150원, 여수에서 이용하는 유저는 기본요금 300원에 분당요금 70원으로 적용됩니다.
  - 할인 조건도 있습니다. 사용자가 **파킹존에서 반납**하는 경우 요금의 30%를 할인해주며, 사용자가 마지막 이용으로부터 30분 이내에 다시 이용하면 기본요금을 면제해줍니다.
  - 벌금 조건도 있습니다. 사용자가 **지역 바깥에 반납**한 경우 얼마나 멀리 떨어져있는지 거리에 비례하는 벌금을 부과하며, **반납 금지로 지정된 구역에 반납**하면 6,000원의 벌금을 요금에 추과로 부과합니다.
  - 예외도 있는데, 킥보드가 고장나서 정상적인 이용을 못하는 경우의 유저들을 배려하여 1분 이내의 이용에는 요금을 청구하지 않고 있습니다.
- 최근에 다양한 할인과 벌금을 사용하여 지자체와 협력하는 경우가 점점 많아지고 있어 요금제에 새로운 할인/벌금 조건을 추가하는 일을 쉽게 만드려고 합니다.
  - 어떻게 하면 앞으로 발생할 수 있는 다양한 할인과 벌금 조건을 기존의 요금제에 쉽게 추가할 수 있는 소프트웨어를 만들 수 있을까요?
- 우선은 사용자의 이용에 관한 정보를 알려주면 현재의 요금 정책에 따라 요금을 계산해주는 API를 만들어주세요. 그 다음은, 기능을 유지한 채로 새로운 할인이나 벌금 조건이 쉽게 추가될 수 있게 코드를 개선하여 최종 코드를 만들어주세요.

### 데이터 구조

![데이터 ERD](https://user-images.githubusercontent.com/57168321/142719190-3f0dda31-26b1-4aef-8cd1-49750ed7ae34.PNG)

## 회고

### 거리 데이터를 데이터베이스에 저장하기

거리 데이터를 저장할 때 사용한 공간 타입은 x 축, y 축으로 구성되어 특정 점의 위치를 보여주는 [Point](https://dev.mysql.com/doc/refman/5.7/en/gis-class-point.html),
여러 개의 `Point` 의 집합이지만 연결이 되진 않은 [MultiPoint](https://dev.mysql.com/doc/refman/5.7/en/gis-class-multipoint.html), 그리고 다각형의 표현을 나타내는 [Polygon](https://dev.mysql.com/doc/refman/5.7/en/gis-class-polygon.html) 3가지입니다.

임의로 여러 개의 지역, 주차 구역 그리고 금지 구역을 초기 데이터로 저장하는 작업을 [임유라](https://github.com/BangleCoding)님께서 하셨습니다. 이 데이터를 바탕으로 킥보드를 주차한 위치의 x, y 축과 위의 타입으로 선언한 값을 비교하여 어디에 주차했는지를 확인합니다.

### typeorm 으로 거리 계산, 주정차 여부 확인하기

이번 요금 계산에서 까다로운 부분은 주차한 곳이 지역 안인지, 주차 구역인지 아니면 금지 구역인지를 비교하는 것입니다.

1. `leftJoin`으로 데이터 연결하기

```
.leftJoin('history.deer', 'deer')
.leftJoin('deer.area', 'area')
.leftJoin('area.parkingzones', 'parkingzone')
.leftJoin('area.forbidden_areas', 'forbidden_area')
```

`leftJoin`을 이용해 이용 내역으로부터 킥보드, 지역, 주차 구역, 금지 구역의 정보를 한꺼번에 불러옵니다. 그 결과 지역 내의 주차 구역, 금지 구역의 개수만큼 데이터가 늘어납니다.

2. MySQL 에서 지원하는 Spatial Relation Functions 으로 요청한 위도, 경도(킥보드의 주차 위치)와 특정 위치(지역, 주차 구역, 금지 구역)을 비교합니다. 특정 위치 안에 킥보드가 있으면 1(true), 아니면 0(false)을 리턴합니다. 그리고 `SUM`, `groupBy`를 이용해 그 결과를 합칩니다.

```
.addSelect(
  'SUM(ST_Contains(area.area_boundary, ST_GeomFromText(:p)))',
  'isInArea',
)
.addSelect(
  'SUM(ST_DISTANCE(ST_GEOMFROMTEXT(ST_ASTEXT(parkingzone.parkingzone_center_coord), 4326), ST_GEOMFROMTEXT(:p, 4326)) < parkingzone.parkingzone_radius)',
  'isInParkingzone',
)
.addSelect(
  'SUM(ST_Contains(forbidden_area.forbidden_area_boundary, ST_GeomFromText(:p)))',
  'isInForbiddenArea',
)
.groupBy('history_id')
```

3번으로 넘어가기 전에 사용한 Spatial Relation Functions 에 대해 간단하게 언급하겠습니다.

- [ST_Contains](https://dev.mysql.com/doc/refman/5.6/en/spatial-relation-functions-object-shapes.html#function_st-contains): `ST_Contains(g1, g2)`은 g1 이 g2 에 포함하면 1을 아니면 0을 리턴합니다.
- [ST_GeomFromText](https://dev.mysql.com/doc/refman/8.0/en/gis-wkt-functions.html#function_st-geomfromtext): WKT representation 과 SRID(spatial reference system identifier) 를 이용해 기하학 값을 구성합니다.
  - [Well-Known Text 포맷](https://dev.mysql.com/doc/refman/8.0/en/gis-data-formats.html#gis-wkt-format): WKT representation 기하학 값은 기하학 데이터를 아스키 폼으로 교환하기 위해 고안된 것입니다.
  - [Spatial Reference system IDentifier](https://docs.microsoft.com/ko-kr/sql/relational-databases/spatial/spatial-reference-identifiers-srids?view=sql-server-ver15): 비교할 대상 g1, g2 의 SRID 를 같은 값으로 해야 함수의 기능을 수행할 수 있습니다. `4326`은 일반적으로 사용하는 SRID 로, 지구 표면의 경도 및 위도 좌표를 사용하여 공간 데이터를 나타내며, 이는 GPS(Global Positioning System)에도 사용됩니다. [출처: cockroachlabs](https://www.cockroachlabs.com/docs/stable/srid-4326.html)
- [ST_DISTANCE](https://dev.mysql.com/doc/refman/5.6/en/spatial-relation-functions-object-shapes.html#function_st-distance): `ST_Distance(g1, g2)`은 g, g2 사이의 거리를 리턴합니다. 이 거리와 금지 구역의 반지름을 비교하여 거리가 더 짧으면 금지 구역 내에 주차한 것으로 간주합니다.

3. 결과를 해석해보면, isInArea 값이 1 이상이면 지역에 주차한 것이고, isInParkingzone 또는 isInForbiddenArea 값이 1이면 킥보드가 주차/금지 구역에 주차한 것입니다.

### 정규표현식으로 이벤트 조건을 검증하기

이벤트 조건은 `isInParkingzone > 0`(주차 구역에 정차한 이벤트)같이 문자열으로 저장하기로 했는데, 조건을 양식에 맞게 정확히 작성을 해야 이벤트를 활용할 때 오류가 발생하지 않습니다.

조건의 검증을 정규표현식으로 수행하기로 했습니다.

```typescript
function checkConditionValidate(condition: string): boolean {
  /**
   * 1. 처음은 무조건 영어(변수명)으로 시작합니다. 이벤트 대상은 "가격", "지역 아이디", "이용 시간", "지역에 주차", "주차장에 주차", "금지구역 주차"으로 고정합니다.
   * 그 다음 반드시 띄어쓰기를 합니다.(\s)
   * 2. 연산자는 >=, >, <=, <, ==, != 만 가능합니다.
   * 그 다음 반드시 띄어쓰기를 합니다.(\s)
   * 3. 마지막은 반드시 숫자로 끝맺습니다.
   */
  const regex =
    /^(price|deer.area.area_id|useMin|isInArea|isInParkingzone|isInForbiddenArea)\s(>=|>|<=|<|==|!=)\s[0-9]+$/;
  if (condition.match(regex)) {
    return true;
  }
  return false;
}
```

- `^`: 문장의 처음을 뜻합니다.
- `(문자|문자)`: 특정 문자 값들을 나열하는데, 이 값들이 존재해야 true 를 리턴합니다.
- `\s`: 띄어쓰기 공백입니다.
- `[0-9]`: 숫자 값을 의미합니다.
- `+`: 앞의 표현식이 1회 이상 반복하는 것입니다. 위의 경우 1개든 2개든 숫자 값이면 true 를 리턴합니다.
- `$`: 문장의 마지막을 뜻합니다.

## 프로젝트를 마치며

아직 익숙하지 않은 SQL, typeorm 에 더해 처음으로 공간 데이터까지 다룬 이번 과제는 그동안 수행해온 것보다 확연히 어려웠습니다. 공간 데이터가 그동안 다뤄왔던 숫자, 문자, 날짜 타입과 확연히 다른 개념인 부분도 어려움을 가중시켰습니다.
다행히 MySQL 의 공간 관계 함수를 이용한 거리 계산, 그리고 그룹으로 묶어 어디에 주차했는지를 확인하는 방법이 의도한 대로 작동되어 생각보다 오랜 시간이 걸리지 않았습니다. 다만, 이번 과제에서는 단순한 거리 계산에 그쳐 기초적인 겉핥기 수준으로만 공간 데이터를 접했다고 생각합니다.
