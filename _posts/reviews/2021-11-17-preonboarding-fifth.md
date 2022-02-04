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
description: 휴먼스케이프에서 제시해주신 임상 정보 수집 및 조회 API 과제의 제작 과정을 정리합니다.
excerpt: 휴먼스케이프에서 제시해주신 임상 정보 수집 및 조회 API 과제의 제작 과정을 정리합니다.
tags:
  - 위코드
  - 원티드
  - 휴먼스케이프
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 다섯 번째 과제를 수행한 경험을 정리하고자 이 글을 작성했습니다. [Github Repository](https://github.com/chinsanchung/preonboarding-humanscape)에서 작성한 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [휴먼스케이프](https://humanscape.io/kr/index.html)에서 제시한 과제로, **휴먼스케이프**는 희귀난치성 질환을 가진 환자들이 직접 업로드한 건강 데이터를 기반으로 자신에게 꼭 맞는 치료제 개발 및 임상시험 정보를 확인하고 참여할 수 있는 [레어노트](https://humanscape.io/kr/service_rarenote.html) 등 헬스케어 서비스를 제공하는 기업입니다.

### 과제 안내

주제는 임상 정보를 수집하고 그것을 조회하는 API 제작입니다.

- 전부다 쌓는 API
- 임상 정보를 수집하는 batch task
  - 참고: [https://www.data.go.kr/data/3074271/fileData.do#/API 목록/GETuddi%3Acfc19dda-6f75-4c57-86a8-bb9c8b103887](https://www.data.go.kr/data/3074271/fileData.do#/API%20%EB%AA%A9%EB%A1%9D/GETuddi%3Acfc19dda-6f75-4c57-86a8-bb9c8b103887)
- 수집한 임상 정보에 대한 API
  - 특정 임상 정보 읽기(키 값은 자유)
- 수집한 임상 정보 리스트 API
  - 최근 일주일내에 업데이트(변경사항이 있는) 된 임상정보 리스트
    - pagination 기능

## 회고

### date-fns 를 이용해 임상 정보를 조회하기

#### date-fns

이번에는 시간을 제어하기 위해 [moment-timezone](https://www.npmjs.com/package/moment-timezone) 대신 [date-fns](https://www.npmjs.com/package/date-fns)를 사용했습니다. 그 이유로는 2가지를 들 수 있습니다. (아래 내용은 [momentjs vs date-fns](https://medium.com/@k2u4yt/momentjs-vs-date-fns-6bddc7bfa21e)을 참고하여 작성했습니다.)

1. momentjs 는 날짜를 계산하려면 인스턴스를 만들어야하는데, 즉 불필요한 함수까지 전부 가져와 애플리케이션을 빌드했을 때 용량이 커집니다. 반면 date-fns 는 필요한 함수만을 따로 가져와 사용할 수 있어 빌드했을 떄의 용량을 줄일 수 있습니다.
2. momentjs 에서 인스턴스를 생성하여 계산하는 과정에서 date-fns 보다 많은 시간을 소요합니다.

이러한 이류로 moment-timezone 대신 date-fns 의 함수를 이용했습니다.

#### 임상 정보의 시간 조건

```typescript
if (query.APPROVAL_TIME) {
  const UTCZeroApprovalTime = subHours(new Date(query.APPROVAL_TIME), 9);
  const datePeriod = [
    UTCZeroApprovalTime.toISOString(),
    add(UTCZeroApprovalTime, {
      hours: 23,
      minutes: 59,
      seconds: 59,
    }).toISOString(),
  ];

  Object.assign(whereOption, {
    APPROVAL_TIME: Between(datePeriod[0], datePeriod[1]),
  });
} else {
  // ...
}
```

승인 시간이 존재하면 그 날짜 하루를 조건으로 정합니다. SQLite3 에서는 한국과 달리 9시간 이전으로 앞당긴 UTC+0 시간대를 사용합니다. 그에 따라 9시간을 빼 UTC+0 시간대로 만들고, 거기에 23시 59분 59초를 더해 조건을 만들었습니다.

```typescript
if (query.APPROVAL_TIME) {
  // ...
} else {
  Object.assign(whereOption, {
    APPROVAL_TIME: MoreThanOrEqual(
      subDays(
        set(new Date(), {
          hours: 0,
          minutes: 0,
          seconds: 0,
          milliseconds: 0,
        }),
        6
      ).toISOString()
    ),
  });
}
```

승인 시간이 존재하지 않을 경우 오늘 날짜를 0시 0분 0초로 지정해 불러온 후 6일을 빼서 6일 전 ~ 오늘까지 7일의 임상 정보를 조회하도록 설정했습니다. 위의 경우와 달리 UTC+0 시간대로 만들지 않은 이유는 date-fns 의 `set` 함수로 변환한 값이 UTC+0 시간대로 변환하기 때문입니다.

## 코드 리펙토링

### 임상 정보를 저장할 때의 승인 시간 설정

[clinical.service.ts](https://github.com/chinsanchung/preonboarding-humanscape/blob/master/src/clinical/clinical.service.ts)에서의 리펙토링입니다.

API 에서 제공하는 승인 시간은 "2012-02-28 00:00:00"으로 시, 분, 초를 모두 0으로 하고 있습니다. 하지만 기존의 코드에서는 저장할 당시의 모든 시간을 저장하고, UTC+0 시간대로 변환하지 않으며, date-fns 가 아닌 moment-timezone 을 사용하고 있었습니다.

```typescript
// 기존의 코드
function convertKstToUtc(time): string {
  const KSTApprovalTime = new Date(time).getTime();
  const modifiedApprovalTime = moment(KSTApprovalTime).format(
    'YYYY-MM-DD HH:mm:ss'
  );

  return modifiedApprovalTime;
}
clinical.APPROVAL_TIME = this.convertKstToUtc(clinical.APPROVAL_TIME);
```

```typescript
// 수정한 코드
clinical.APPROVAL_TIME = set(new Date(clinical.APPROVAL_TIME), {
  hours: 0,
  minutes: 0,
  seconds: 0,
  milliseconds: 0,
});
```

- 빌드할 용량을 줄이고 시간을 절약하기 위해 moment-timezone 을 제거하고 date-fns 의 `set` 함수를 사용했습니다.
  - `set` 함수로 시, 분, 초를 설정할 수 있는데, UTC+0 시간대에 시, 분, 초를 모두 0으로 설정하여 저장하도록 수정했습니다.

### 임상 정보를 조회할 때의 승인 시간 설정

[clinical.repository.ts](https://github.com/chinsanchung/preonboarding-humanscape/blob/master/src/clinical/clinical.repository.ts)에서의 리펙토링입니다.

승인 시간의 시, 분, 초를 0으로 설정하면서, 조회할 때의 시간 설정도 변경할 필요가 있었습니다.

1. 승인 시간을 지정한 경우

```typescript
// 기존의 코드
Object.assign(whereOption, {
  APPROVAL_TIME: Between(
    subDays(subHours(new Date(query.APPROVAL_TIME), 9), 1).toISOString(),
    subDays(addHours(new Date(query.APPROVAL_TIME), 15), 1).toISOString()
  ),
});
```

```typescript
// 수정한 코드
const UTCZeroApprovalTime = subHours(new Date(query.APPROVAL_TIME), 9);
const datePeriod = [
  UTCZeroApprovalTime.toISOString(),
  add(UTCZeroApprovalTime, {
    hours: 23,
    minutes: 59,
    seconds: 59,
  }).toISOString(),
];

Object.assign(whereOption, {
  APPROVAL_TIME: Between(datePeriod[0], datePeriod[1]),
});
```

- `new Date(query.APPROVAL_TIME)`을 두 번 선언해 중복이 발생하는 것을 막기 위해 `UTCZeroApprovalTime` 변수를 만들었습니다.
- 시간 조건을 0시 0분 0초 ~ 23시 59분 59초로 설정했습니다. 위의 코드를 그대로 사용하면 이틀을 조회하기 떄문입니다.

2. 승인 시간을 지정하지 않은 경우

```typescript
subDays(
  set(new Date(), {
    hours: 0,
    minutes: 0,
    seconds: 0,
    milliseconds: 0,
  }),
  6
).toISOString();
```

date-fns 의 `set` 함수로 변환한 값이 UTC+0 시간대로 변환하기에 수정하지 않았습니다.

### 로컬 환경에서의 테스트를 위해 getAllBatchDataForLocalTest 메소드 작성

[clinical.controller.ts](https://github.com/chinsanchung/preonboarding-humanscape/blob/master/src/clinical/clinical.controller.ts)에서의 리펙토링입니다.

원래 의도는 @nestjs/schedule 으로 지정 시간에 데이터를 저장하는 방식이지만, 로컬 환경에서의 테스트를 위해 데이터를 직접 저장하는 API 를 따로 제작했습니다. `POST localhost:3000/clinical`으로 테스트를 위한 임상 시험 정보를 in memory 데이터베이스에 저장합니다.

## 프로젝트를 마치며

momoentjs 와 date-fns 를 비교한 글을 보면서, 앞으로는 date-fns 를 사용하기로 결심했습니다.

데이터베이스에서의 UTC+0 시간대에 맞추는 과정을 겪으며, 글로벌 애플리케이션은 어떤 방식으로 각 지역에 따라 시간대를 바꿔가며 화면으로 출력하는 것인지 그 방식이 궁금해졌습니다.
