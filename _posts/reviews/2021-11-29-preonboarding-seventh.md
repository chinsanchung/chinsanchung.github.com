---
title: '원티드 프리온보딩 코스 일곱 번째 과제 후기'
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
description: 카닥에서 제시해주신 타이어 정보 저장 및 조회 API 과제의 제작 과정을 정리합니다.
excerpt: 카닥에서 제시해주신 타이어 정보 저장 및 조회 API 과제의 제작 과정을 정리합니다.
tags:
  - 위코드
  - 원티드
  - 카닥
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 일곱 번째 과제를 수행한 경험을 정리합니다. [Github Repository](https://github.com/chinsanchung/preonboarding_cardoc)에서 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [카닥](https://www.cardoc.co.kr/)에서 제시한 과제로, **카닥**은 복잡하고 어려운 자동차 관리 정보를 편리하게 제공하고 신뢰할 수 있는 파트너와 상품, 서비스를 모아 당신의 자동차를 위한 더 나은 선택을 도와주는, "결과적으로 자동차 오너의 소비를 편하게"하는 것이 목표인 기업입니다.

### 과제 안내

주제는 카닥에서 실제로 사용하는 프레임워크를 토대로 타이어 API를 설계 및 구현하는 것입니다.

1. 사용자 생성 API

- ID/Password로 사용자를 생성하는 API 를 제작합니다.
- 인증 토큰을 발급하고 이후의 API는 인증된 사용자만 호출할 수 있습니다.

2. 사용자가 소유한 타이어 정보를 저장하는 API

- 자동차 차종 ID(trimID)를 이용하여 사용자가 소유한 자동차 정보를 저장합니다. 자동차 정보 조회 API 예시:[https://dev.mycar.cardoc.co.kr/v1/trim/5000](https://dev.mycar.cardoc.co.kr/v1/trim/5000)
  - 조회된 정보에서 타이어 정보는 spec → driving → frontTire/rearTire 에서 찾을 수 있습니다.
  - - 타이어 정보는 205/75R18의 포맷이 정상으로, 205는 타이어 폭을 의미하고 75R은 편평비, 그리고 마지막 18은 휠사이즈로써 {폭}/{편평비}R{18}과 같은 구조입니다. 위와 같은 형식의 데이터일 경우만 DB에 항목별로 나누어 서로다른 Column에 저장합니다.
- 한 번에 최대 5명까지의 사용자에 대한 요청을 받습니다. 즉 사용자 정보와 trimId 5쌍을 요청데이터로 하여금 API를 호출할 수 있습니다.

3. 사용자가 소유한 타이어 정보 조회 API

- 사용자 ID를 통해서 2번 API에서 저장한 타이어 정보를 조회할 수 있어야 합니다.

## 회고

### 서비스의 리턴 타입을 통일하기

모든 서비스는 아래의 인터페이스 중에서 하나의 타입대로 값을 리턴합니다. 성공과 실패를 같은 타입으로 리턴한 이유는 우선 서비스의 예측 결과가 무엇일지를 쉽게 파악할 수 있는 점, 그리고 모든 실패에 대한 결과를 통제하는 데 있어서 throw Error보다 수월한 점이 있습니다.

```typescript
export interface IOutput {
  ok: boolean;
  httpStatus?: number;
  error?: string;
}

export interface IOutputWithData<DataType> extends IOutput {
  data?: DataType;
}
```

성공했을 때는 ok: true 를 리턴합니다. 만약 데이터가 존재한다면 data: DataType으로 데이터를 리턴합니다. 제네릭을 이용해 타입을 설정할 수 있습니다.
원하는 결과가 없을 때, 로그인 등 특정 로직을 실패했을 때, 에러가 발생했을 때는 ok: false, httpStatus: http 상태 코드, error: "에러 메시지"를 리턴합니다. 참고로 NestJs 에서는 `throw new InternalServerErrorException` 와 같이 곧바로 HTTP 에러 상태 코드를 응답으로 보내는 기능이 있습니다. 이것을 서비스에서 사용하지 않은 이유는 요청과 응답은 컨트롤러에서 관리하는 것이 옳다고 생각하기 때문입니다.

### 타이어를 저장 할 때 캐시를 활용해 반복을 줄이기

타이어를 저장할 떄, 유저와 타이어를 객체로 따로 저장했습니다. 그 이유는 동일한 유저 또는 trimId 로 인해 로직을 반복하는 과정을 줄이고 싶어서입니다. 이러한 아이디어를 얻은 계기는 프로그래머스의 [실무와 가까워지는 Node.js 백엔드 개발](https://programmers.co.kr/learn/courses/12887) 스터디에서 들은 강의였는데, 데이터베이스에서 읽는 과정도 비용이 발생하기에 그것을 최소화하기 위해서 한 번 불러온 값을 캐시로 저장하고 필요할 때 그것을 호출해 데이터베이스에 다시 접근할 필요를 차단하는 것이었습니다. 그래서 저도 올바른 유저인지 확인하기, 카닥 API 에서 타이어 정보를 추출하는 과정의 결과를 객체에 저장해 재사용하는 방법을 선택했습니다.

```typescript
const userEntities = {};
const tireEntites = {};
const entireTireInfo = {};

// * 유효한 유저 아이디인지 확인합니다. 한 번 확인한 아이디는 다시 확인하지 않습니다.
if (!userEntities[id]) {
  const checkUserEntity = await this.checkAndReturnUserEntity(id);
  if (!checkUserEntity.ok) {
    httpStatus = checkUserEntity.httpStatus;
    error = checkUserEntity.error;
    throw new Error();
  }
  Object.assign(userEntities, { [`${id}`]: checkUserEntity.data });
}

// * 카닥 API 에서 차에 대한 정보를 불러옵니다. trimId 결과를 entireTireInfo에 저장하여 같은 trimId 로 API 를 호출하는 것을 방지합니다.
if (!entireTireInfo[trimId]) {
  const carInfo = await this.getTireInfoFromCarApi(trimId);
  // * 유효한 trimid 로 불러온 것인지 확인합니다.
  if (!carInfo.ok) {
    httpStatus = carInfo.httpStatus;
    error = carInfo.error;
    throw new Error();
  }
  Object.assign(entireTireInfo, { [`${trimId}`]: carInfo.data });
}

// * 타이어에 저장한 것인지 확인하고, 저장하지 않으면 타이어 생성을, 저장했으면 타이어 데이터를 불러옵니다.
const checkTireEntity = await this.checkOrCreateAndReturnTireEntity(
  entireTireInfo[`${trimId}`]
);
if (!checkTireEntity.ok) {
  httpStatus = checkTireEntity.httpStatus;
  error = checkTireEntity.error;
  throw new Error();
}
Object.assign(tireEntites, { [`${id}`]: checkTireEntity.data });
```

## 프로젝트를 마치며

이번 프로젝트는 그동안의 프로젝트와 달리 혼자서 작성해야 했습니다. 여러 명이서 할 때보다 오래 걸릴 것이라 생각했지만, 그동안 NestJs, SQL 과 TypeORM 을 다뤄왔던 경험으로 수월하게 코드를 작성할 수 있었습니다. 또한, 개인으로서는 처음으로 [업무 대시보드](https://github.com/chinsanchung/preonboarding-cardoc/projects/1)와 [이슈](https://github.com/chinsanchung/preonboarding-cardoc/issues?q=is%3Aissue+is%3Aclosed)를 작성하고, 기능별로 브랜치를 나눠 개발한 후 PR 을 올리는 방식으로 개발했습니다. 진도를 직접 확인할 수 있어 의욕이 오르는 한편, PR 을 이슈와 연결시킴으로써 각 기능별로 남긴 커밋 내역과 PR 메시지를 통해 어떤 의도로 작성한 코드인지를 확인할 수 있었습니다. 마지막 프로젝트를 끝마치면서 타입스크립트와 NestJs 에 익숙해지고, 협업을 위한 형상 관리를 하는 법을 배워 이전보다 더 성장했음을 느꼈습니다.
