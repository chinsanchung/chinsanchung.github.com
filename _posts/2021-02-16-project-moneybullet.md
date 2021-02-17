---
title: "MoneyBullet 프로젝트 제작기"
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
description: MoneyBullet 웹 애플리케이션을 개발했던 경험을 작성합니다.
tags:
  - javascript
  - database
  - mongo
  - NoSQL
---

> 2021년 1월부터 2월 초까지 가맹점주의 결제 데이터를 가공해서 시각화하는 MoneyBullet 웹 애플리케이션을 개발했습니다. 우선 프로젝트에 대해 간략하게 소개한 후, 작업하면서 기억에 남던 부분을 설명하겠습니다.

## 프로젝트 구성

이번 프로젝트는 크게 매장, 상품 데이터로 구분하고, 그에 맞춰 데이터를 분류, 가공해서 출력하는 과정을 거칩니다.
우선 각 매장 또는 상품으로 묶어 테이블로 출력하는 목록 페이지, 그리고 매장(상품)에 대한 상세 보기 페이지로 구성되었습니다.

### 목록 페이지

테이블 상단에는 결제일이 오늘인 데이터를 묶어 다양한 형식(예시: 상위 매출 매장)으로 분류한 후 그래프에 출력했습니다.

테이블에 결과를 출력하기에 앞서 정렬 조건을 지정할 수 있습니다. 기본값은 기간(7일 전 ~ 오늘) / 매출액의 내림차순 / 한 번에 10개의 행을 출력합니다.

- 기간 (1일, 7일, 30일, 90일, 직접 선택) / 기간 + 상품명(최대 2개) / 기간 + 업체명(최대 2개)
- 매출액 / 가나다 / 상위 결제 시간의 오름차순, 내림차순
- 한 번에 출력할 테이블 행의 개수: 10, 30, 50, 100

목록의 컬럼들은 이름, 코드, 현재 기간의 매출액뿐만 아니라, 이전 기간(예시: 14일 전 ~ 7일 전 / 7일 전 ~ 오늘)의 매출액을 표기해서 현재 매출액과 비교할 수 있도록 했습니다.
그 외 매장(상품)의 결제 데이터를 한 시간 단위로 묶고 가장 매출이 많은 시간 순으로 표기한 "상위 결제 시간", 매장의 경우 해당 매장에서 가장 많이 팔린 상품을 보여주는 "상위 매출 상품", 상품의 경우 가장 많은 매출을 낸 매장을 보여주는 "상위 결제 매장" 컬럼을 보여줍니다.

<!-- 사진1 -->

### 보기 페이지

해당 매장(상품)을 지정한 기간으로 묶은 후, 그것을 가공해서 그래프로 출력합니다.

<!-- 사진2 -->

## 작업 1. 데이터 가공하기

이번 프로젝트는 데이터 시각화가 최우선 사항이었고, 따라서 데이터를 정확하게 분류, 추출, 가공하는 작업이 필요했습니다. 데이터베이스를 MongoDB, ODM(Object Document Mapping)은 mongoose를 사용했습니다.

### 목록 페이지에서의 데이터 가공

#### 1) 서로 다른 결과를 하나의 aggregate 쿼리문에 합치기

목록 페이지의 데이터는 네 갈래로 나뉩니다. 현재 기간의 데이터(줄여서 현재값이라 부르겠습니다.), 이전 기간의 매출액, 상위 결제 시간, 그리고 상위 매출 상품(매장)입니다. 처음에는 하나의 aggregate 쿼리문으로 작성하려 했지만, 각 영역을 구하는 절차가 서로 달라 결국 결과를 따로 구한 후 합치는 방법을 택했습니다.
예시로 이전 기간의 매출액과 현재값을 합치는 과정을 코드로 작성하겠습니다.

```javascript
// $let, $filter 예시 - 매장의 경우
const previousProfit = "aggregate문으로 구한 이전 기간 매출액";
const response = await Data.aggregate([
  // 그룹화는 생략하고 $let, $filter 함수만 보여드리겠습니다.
  {
    $addFields: {
      previousProfit: {
        $let: {
          vars: { value: previousProfit },
          in: "$$value",
        },
      },
    },
  },
  {
    $project: {
      _id: 0,
      previousProfit: {
        $filter: {
          input: "$previousProfit",
          as: "prevVal",
          cond: {
            $eq: ["$$prevVal.shopName", "$_id.shopName"],
          },
        },
      },
    },
  },
]);
```

- `$let`: 외부의 값을 쿼리문에서의 변수로 지정할 수 있습니다. `$addFields`로 외부값 previousProfit를 `previousProfit`변수로 지정했습니다.
- `$filter`: `$let`으로 저장한 previousProfit 변수를 `input`으로 대입해 `prevVal`으로 임시로 지정하고, cond로 previousProfit의 매장(상품)명과 현재값의 매장(상품)명을 비교해서 서로 일치하는 데이터를 연결할 수 있습니다.

- 각 영역을 구한 방법
- `$let`, `$filter`
- aggregate 작성 순서

### 보기 페이지에서의 데이터 가공

## 작업 2. 데이터 시각화

<!-- 데이터 시각화는 [ReCharts](https://recharts.org/en-US/)를 사용했습니다. -->
<!-- recharts 의 용어들. 반응형 작업 -->
<!-- 매장(상품)명 -->
