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

2021년 1월부터 2월 초까지 가맹점주의 결제 데이터를 가공해서 시각화하는 MoneyBullet 웹 애플리케이션을 개발했습니다. 우선 프로젝트에 대해 간략하게 소개한 후, 작업하면서 기억에 남던 부분을 설명하겠습니다.

- 2021년 5월 21일, AWS Beanstalk 의 환경을 종로해서 더 이상 사이트에 접속하실 수 없게 됐습니다. 대신 스크린샷을 아래에 추가했습니다.

## 프로젝트 구성

이번 프로젝트는 크게 매장, 상품 데이터로 구분하고, 그에 맞춰 데이터를 분류, 가공해서 출력하는 과정을 거칩니다.
우선 각 매장 또는 상품으로 묶어 테이블로 출력하는 목록 페이지, 그리고 매장(상품)에 대한 상세 보기 페이지로 구성되었습니다.

### 목록 페이지

테이블 상단에는 결제일이 오늘인 데이터를 묶어 다양한 형식(예시: 상위 매출 매장)으로 분류한 후 그래프에 출력했습니다.

테이블에 결과를 출력하기에 앞서 검색 조건을 지정할 수 있습니다. 기본값은 기간(7일 전 ~ 오늘) / 매출액의 내림차순 / 한 번에 10개의 행을 출력합니다.

- 기간 (1일, 7일, 30일, 90일, 직접 선택) / 기간 + 상품명(최대 2개) / 기간 + 업체명(최대 2개)
- 매출액 / 매장명 또는 상품명 / 상위 결제 시간의 오름차순, 내림차순
- 한 번에 출력할 테이블 행의 개수: 10, 30, 50, 100

목록의 컬럼들은 이름, 코드, 현재 기간의 매출액뿐만 아니라, 이전 기간(예시: 14일 전 ~ 7일 전 / 7일 전 ~ 오늘)의 매출액을 표기해서 현재 매출액과 비교할 수 있도록 했습니다.
그 외 매장(상품)의 결제 데이터를 한 시간 단위로 묶고 가장 매출이 많은 시간 순으로 표기한 "상위 결제 시간", 매장의 경우 해당 매장에서 가장 많이 팔린 상품을 보여주는 "상위 매출 상품", 상품의 경우 가장 많은 매출을 낸 매장을 보여주는 "상위 결제 매장" 컬럼을 보여줍니다.

목록 페이지의 스크린샷입니다.

[매장 목록 보기](https://drive.google.com/file/d/141IrCgMdjv_Xk-25nLoDJiWE93xgG62N/view?usp=sharing)

[상품 목록 보기](https://drive.google.com/file/d/1p7EOkTvcUlSi4O9WgNHj2KkiQjTpPK2E/view?usp=sharing)

다음은 매장 목록의 두 항목을 체크해 비교하는 모달입니다.

[두 매장을 비교하는 모달](https://drive.google.com/file/d/1eN2r60M6Eis8u5aJCVlB0z8mRBs5iKqp/view?usp=sharing)

[두 상품을 비교하는 모달](https://drive.google.com/file/d/1izelIqmK2TVZkTF-FJEwEXcWKtaLiNKz/view?usp=sharing)

### 보기 페이지

해당 매장(상품)을 지정한 기간으로 묶은 후, 그것을 가공해서 그래프로 출력합니다. 우선 오늘 결제한 데이터를 기반으로 어제와 오늘의 매출 비교, 오늘의 시간별 매출액, 오늘의 상위 결제 카드사를 추출했습니다.
그리고 기간 선택지에 따라 이전 기간과 현재 기간의 매출액 비교, 기간별(예 - 4주전, 3주전, 2주전, 1주전, 현재) 매출액, 상위 매출 상품(매장), 결제 카드나 시간으로 그룹화해서 매출액 5순위 데이터를 구했습니다.

보기 페이지의 스크린샷입니다.

[매장 상세 보기](https://drive.google.com/file/d/1mlyrlurF9q_SOmirALwmujB6C8gWfovD/view?usp=sharing)

[상품 상세 보기](https://drive.google.com/file/d/1sZS4cQvgzTZ54q1xfndQB1HPHf2kbUqS/view?usp=sharing)

## 작업 1. 데이터 가공하기

이번 프로젝트는 데이터 시각화가 최우선 사항이었고, 따라서 데이터를 정확하게 분류, 추출, 가공하는 작업이 필요했습니다. 데이터베이스를 MongoDB, ODM(Object Document Mapping)은 mongoose를 사용했습니다.

### 1) $let, $filter: 서로 다른 쿼리의 결과들을 하나의 aggregate 쿼리문에 합치기

목록 페이지의 데이터는 네 갈래로 나뉩니다. 현재 기간의 데이터(줄여서 현재값이라 부르겠습니다.), 이전 기간의 매출액, 상위 결제 시간, 그리고 상위 매출 상품(매장)입니다. 처음에는 하나의 aggregate 쿼리문으로 작성하려 했지만, 각 영역을 구하는 절차가 서로 다르기에, 결과를 따로 구한 후 합치는 방법을 택했습니다.
아래의 코드는 이전 기간의 매출액과 현재값을 합치는 과정입니다. 상위 결제 시간과 상위 매출 상품(매장)도 아래와 같은 절차로 작성했습니다.

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

### 2) 각 document에 인덱스 번호 부여하기

목록 테이블의 각 행마다 번호를 부여해야 했는데, 번호를 오름차순 또는 내림차순으로 정렬하기 위해선 데이터를 가공하는 단계에서부터 번호를 미리 부여해야만 했습니다. 그것을 위해 `$group`으로 전체 데이터를 묶은 후에 그것을 다시 풀면서 인덱스를 부여하는 방법을 사용했습니다.

```javascript
const response = await Data.aggregate([
  // 앞의 과정은 생략합니다.
  {
    $group: {
      _id: null,
      data: { $push: "$$ROOT" },
    },
  },
  { $unwind: { path: "$data", includeArrayIndex: "serial_number" } },
  {
    $project: {
      _id: 0,
      name: "$data.name",
      previousProfit: "$data.previousProfit",
      // 나머지 선언은 생략합니다.
    },
  },
]);
```

- `$group`으로 묶을 때 data 라는 임시 변수에 전체 결과값 `$$ROOT`를 담았습니다.
- `$unwind`로 data 변수(배열)을 해체해 객체로 만들면서 `includeArrayIndex`으로 인덱스 숫자를 serial_number 변수에 담도록 했습니다.
- 마지막으로 data 변수(객체) 안에 있던 값들을 새로운 변수로 다시 선언했습니다.

## 작업 2. 데이터 시각화

데이터 시각화는 [ReCharts](https://recharts.org/en-US/)를 사용했습니다. 막대그래프, 꺾은선그래프 등 다양한 그래프 형식을 지원합니다.

### 1) 기본적인 표기

```javascript
# 막대그래프
const data = [{name:'a',value:1000}]
function Bar() {
  return (
    <BarChart data={data}>
      <XAxis dataKey='name' />
      <YAxis />
      <Bar
        dataKey='value'
        label={{position:'top', fill:'#000'}}
        barSize={15}
      >
        {data.map((entry, index) => (<Cell key={index} />))}
      </Bar>
    </BarChart>
  )
}
# 꺾은선그래프
function Line() {
  return (
    {/* 그래프의 마진을 직접 설정할 수 있습니다. */}
    <LineChart
      data={[{name:'a',value1:1000, value2}]}
      margin={{top:10, left: 10, bottom: 10, right: 10}}
    >
      <XAxis dataKey='name' interval={0} />
      <YAxis hide />
      {/* 두 개의 Line 으로 여러 개의 꺾은선그래프를 그릴 수 있습니다. */}
      <Line dataKey='value1' name='value1' stroke='#000' />
      <Line dataKey='value2' name='value2' stroke='#e2e2e2' />
    </LineChart>
  )
}
# 파이차트
const colors = ['green','red','blue','purple','yellow']
function Pie() {
  return (
    <PieChart width={300} height={300}>
      <Pie
        data={data}
        cx='50%'
        cy='50%'
        innerRadius='50%'
        dataKey='value'
      >
        {data.map((entry, index) => (
          <Cell key={index} fill={colors[index]} />
        ))}
      </Pie>
    </PieChart>
  )
}

```

### 2) 그래프를 가공하기

1. 반응형 작업

`<ResponsiveContainer>`컴포넌트로 차트를 감싸주면 반응형으로 그래프를 만들 수 있습니다. props 로 width, height 등 필요한 설정을 추가하시면 됩니다.

2. x축, y축 가공하기

```javascript
# Y축 간격을 임의로 조절하기.
## 최대값을 1.4배로 늘려 라벨이 가려지지 않도록 할 때 사용했습니다.
<YAxis domail={[0, (dataMax) => dataMax * 1.4]} />
# 임의로 축 라벨을 수정하기 예시
const CustomXTic = ({x, y, stroke, payload}) => {
  <g transform={`translate(${x},${y}`)}>
    <text
      x={0}
      y={0}
      dy={0}
      textAnchor='middle'
      fontSize={10}
    >
      {payload.value}
    </text>
  </g>
}
<XAxis dataKey='name' tick={<CustomXTic />} />
```

- 그래프 변경

## 작업 3. 클라이언트 영역에서의 작업

### 1) Date.getDate(): 이전 기간과 현재 기간을 계산해서 yyyy.mm.dd 형식으로 가공하기

기간 검색 조건을 문자열로 보여주려면 계산을 해야합니다. 예를 들어 기간을 7일으로 한다면, `2021.02.04 ~ 2021.02.10 || 2021.02.11 ~ 2021.02.17`이라는 결과가 나오게 해야합니다. 계산하는 방법은 [How to subtract days from a plain Date?](https://stackoverflow.com/questions/1296358/how-to-subtract-days-from-a-plain-date)를 참고했습니다.

```javascript
// 원래는 날짜 조건을 컴포넌트화해서 date, period 를 props로 내려받았지만,
// 여기서는 이해를 돕기 위해 값을 직접 구했습니다.
// URL 쿼리 문자열을 객체로 변환합니다. 참고로 객체를 문자열로 바꾸는 stringify 함수도 가지고 있습니다.
import { parse } from "query-string";
import { useLocation } from "react-router-dom";

const location = useLocation();
// date: 2021.02.17, period: 7
const { date, period } = parse(location.search);

const dateString = date.split(".").join("-");
const dateObj = new Date(dateString);
const fourthDate = new Date(dateString);
const thirdDate = new Date(dateObj.setDate(dateObj.getDate() - period + 1));
const secondDate = new Date(dateObj.setDate(dateObj.getDate() - 1));
const firstDate = new Date(dateObj.setDate(dateObj.getDate() - period + 1));

// 날짜 객체 -> yyyy.mm.dd 문자열로 바꾸는 과정은 생략합니다.
```

1. 클라이언트 URL에 마지막 날짜와 기간을 저장해두었는데("moneybullet.com/shop?date=2021.02.17&period=7"), 여기서 마지막 날짜를 이용해 날짜 객체를 만듭니다. 주의할 점은 "yyyy.mm.dd" 문자열을 "yyyy-mm-dd" 문자열로 변환해야합니다. 왜냐면 익스플로러와 파이어폭스 브라우저는 `new Date(yyyy.mm.dd)`를 작성할 때 오류를 띄우기 때문입니다.
2. 계산에 필요한 날짜 객체 "dateObj"를 구합니다. 따로 구하는 이유는 계산을 진행하면서 날짜 객체가 변화하기 때문입니다.
3. 세번째 날짜를 구합니다. 네번째 날짜를 포함해서 계산하기에 마지막에 1을 더합니다.
4. 두번째 날짜는 프로젝트에서 기획한대로 세번째 날짜의 하루 전입니다.
5. 네번째 날짜 역시 세번째 날짜와 같은 방법으로 구합니다.

#### 수정사항: moment-timezone으로 날짜 계산하기

AWS Beanstalk 애플리케이션에서는 날짜 계산 시 한국 시간을 고려하지 않는 문제를 확인했습니다. 그에 따라 [moment-timezone](https://momentjs.com/timezone/) 패키지를 이용, 타임존을 서울로 지정한 후 계산하는 방식으로 수정했습니다. 또한, moment 에는 문자열 추출과 계산 함수를 지원하기에 Date 보다 더욱 쉽게 작업할 수 있었습니다. 참고로, [자바스크립트에서 타임존 다루기 (2) : NHN Cloud Meetup](https://meetup.toast.com/posts/130)에서 moment-timezone 을 소개받았습니다.

```javascript
import moment from 'moment-timezone';
# 오늘의 날짜 선언
const today = moment().tz('Asia/Seoul')
# 오늘의 날짜 문자열로 추출. format 으로 형식을 자유롭게 지정할 수 있습니다.
const todayStr = today.format('YYYYMMDD')
# 날짜 계산하기. subtract(), add() 함수 두 번째 인수로 시, 분, 초, 일 등 다양하게 계산할 수 있습니다.
const yesterday = moment().tz('Asia/Seoul').subtract(1, 'day')
# 밀리초 계산
const millisecond = today.valueOf()
```
