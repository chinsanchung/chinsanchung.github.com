---
title: '원티드 프리온보딩 코스 네 번째 과제 후기'
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
description: 8퍼센트에서 제시해주신 계좌 거래 API 과제의 제작 과정을 정리합니다.
excerpt: 8퍼센트에서 제시해주신 계좌 거래 API 과제의 제작 과정을 정리합니다.
tags:
  - 위코드
  - 원티드
  - 8퍼센트
---

원티드와 위코드에서 지원하는 [원티드 프리온보딩 백엔드 코스](https://www.wanted.co.kr/events/pre_onboarding_course_4)의 네 번째 과제를 수행한 경험을 정리합니다. [Github Repository](https://github.com/chinsanchung/preonboarding-eightpercent)에서 코드를 확인하실 수 있습니다.

## 개요

이번 프로젝트는 [8퍼센트](https://8percent.kr/)에서 제시한 과제로, **8퍼센트**는 온라인 플랫폼을 통해 대출자와 투자자를 연결하는 핀테크 서비스를 제공하는 기업입니다. 고금리 대출을 받아야 했던 대출자에게는 중금리 대출을, 저금리 시대에 투자자에게는 중수익 투자처를 제공하며 대한민국 P2P 금융시장을 개척하고 있습니다.

### 과제 안내

주제는 계좌 거래 API 를 만드는 것으로, 세부적인 사항은 아래와 같습니다.

1. 작성해야 하는 API 목록

- 거래내역 조회 API
  - 계좌의 소유주만 요청 할 수 있어야 합니다.
  - 거래일시에 대한 필터링이 가능해야 합니다.
  - 출금, 입금만 선택해서 필터링을 할 수 있어야 합니다.
  - Pagination이 필요 합니다.
  - 다음 사항이 응답에 포함되어야 합니다.
    - 거래일시
    - 거래금액
    - 잔액
    - 거래종류 (출금/입금)
    - 적요
- 입금 API
  - 계좌의 소유주만 요청 할 수 있어야 합니다.
- 출금 API
  - 계좌의 소유주만 요청 할 수 있어야 합니다.
  - 계좌의 잔액내에서만 출금 할 수 있어야 합니다. 잔액을 넘어선 출금 요청에 대해서는 적절한 에러처리가 되어야 합니다.

2. 주요 고려 사항

- 계좌의 잔액을 별도로 관리해야 하며, 계좌의 잔액과 거래내역의 잔액의 무결성의 보장
- DB를 설계 할때 각 칼럼의 타입과 제약
- 테스트의 편의성을 위해 sqlite 를 사용할 것

## 회고

### typeorm 으로 필터링

이번 과제에서 오랜 시간이 걸린 부분입니다. 위의 요구 사항에 따르면 거래일시, 입금 또는 출금 필터링, Pagination을 넣어야 합니다. 또한 거래를 수행한 계좌의 소유주인지 여부도 확인하는 과정이 필요합니다.

#### 1) 엔티티 설계

프로젝트의 엔티티를다음과 같이 작성했습니다.

```typescript
@Entity()
export class Transaction {
  @PrimaryGeneratedColumn()
  id: number;

  @Index()
  @Column()
  trans_type: string;

  @Column()
  amount: number;

  @Column({ default: '' })
  comments: string;

  @Column({ default: 0 })
  balance: number;

  @CreateDateColumn()
  createdAt: Date;

  @ManyToOne((_type) => Account, (account) => account.transactions, {
    eager: true,
    onDelete: 'CASCADE',
  })
  account: Account;
}
```

주목할 점은 `@ManyToOne`으로, 이것을 통해 계좌의 정보와의 관계를 N:1 관계라 정의하게 됩니다. N:1 인 이유는 계좌 하나에 여러 거래 기록이 있기 때문인데, Accout 엔티티에서는 반대로 `@OneToMany` 데코레이터로 거래 내역을 연결합니다.

```typescript
@Entity()
export class Account extends CoreEntity {
  @Column({ unique: true })
  acc_num: string;

  @Column({ default: 0 })
  money: number;

  @ManyToOne((_type) => User, (user) => user.accounts, {
    eager: false,
    onDelete: 'CASCADE',
  })
  user: User;

  @OneToMany((_type) => Transaction, (transaction) => transaction.account, {
    eager: false,
    cascade: true,
  })
  transactions: Transaction[];
}
```

#### 2) 계좌의 소유주인지 확인하기

오직 계좌의 소유주만 해당 계좌의 거래 내역을 조회할 수 있습니다. 요구 사항을 충족시키기 위해선 로그인한 유저와 계좌의 작성자를 비교해 소유주인지를 확인하는 과정이 필요합니다.

```typescript
// transaction.service.ts
const account = await this.accountRepository.findOne({
  where: { acc_num: query.acc_num },
  join: {
    alias: 'account',
    leftJoinAndSelect: { user: 'account.user' },
  },
});
if (!account) {
  throw new BadRequestException('거래 내역에 등록한 계좌가 존재하지 않습니다.');
}
if (account.user.user_id !== query.user.user_id) {
  throw new NotAcceptableException(
    '오직 계좌의 소유주만 해당 계좌의 거래 내역을 조회하실 수 있습니다.'
  );
}
```

계좌의 정보를 얻은 후, 계좌가 존재하는지 그리고 소유주가 로그인한 유저인지를 검증합니다.

#### 3) createQueryBuilder 를 이용해 필터링 및 데이터 출력

검증이 끝난 후, 마지막으로 transactionRepository 에서 데이터를 추출합니다. 우선 전체 코드입니다.

```typescript
  async getAllTransactions({
    limit,
    offset,
    trans_type,
    startDate,
    endDate,
    acc_num,
  }: ListWithPageAndUserOptions): Promise<Transaction[]> {
    // * 거래일시에 대한 필터링을 수행합니다. 처음과 끝 날짜를 계산하여 문자열 형식으로 반환합니다.
    const [startDateString, endDateString] = this.getDatePeriod(
      startDate,
      endDate,
    );
    // * 입금, 출금 필터링. 1. 입금, 2. 출금, 3. 입출금 으로 구분합니다.
    let transTypeQuery: any = [
      'transaction.trans_type = :trans_type',
      { trans_type },
    ];
    if (!trans_type) {
      transTypeQuery = [
        'transaction.trans_type IN (:...trans_type)',
        { trans_type: ['in', 'out'] },
      ];
    }
    const transaction = await this.createQueryBuilder('transaction')
      .leftJoinAndSelect('transaction.account', 'account')
      .where('account.acc_num = :acc_num', { acc_num })
      .andWhere('transaction.createdAt >= :startDate', {
        startDate: startDateString,
      })
      .andWhere('transaction.createdAt <= :endDate', {
        endDate: endDateString,
      })
      .andWhere(transTypeQuery[0], transTypeQuery[1])
      .limit(limit) // * Pagination 기능
      .offset(offset) // * Pagination 기능
      .select([
        'transaction.id',
        'transaction.createdAt',
        'transaction.amount',
        'transaction.balance',
        'transaction.trans_type',
        'transaction.comments',
      ])
      .getMany();
    // * select 로 특정 컬럼만 응답에 포함합니다. [거래일시, 거래금액, 잔액, 거래종류, 적요]
    return transaction;
  }
```

각 과정을 차례대로 설명하겠습니다.

##### 특정 계좌 번호의 거래만을 가져오기

where 조건절으로 계좌 번호가 일치하는 거래 내역만을 불러옵니다.

```typescript
where('account.acc_num = :acc_num', { acc_num });
```

##### 거래일시 계산

typeorm, SQLite 에서 시간을 조건절에 넣으려면 `YYYY-MM-DD HH:mm:ss`형식으로 작성해야 합니다.
그래서 요청의 날짜 쿼리 `startDate`, `endDate` 를 `YYYY-MM-DD` 형식으로 입력하도록 하는 한편, 반대로 날짜를 입력하지 않았을 경우도 고려했습니다. 입력하지 않을 때의 기본값을 3개월 전부터 오늘까지로 설정했습니다. 시간을 계산하고 문자열 형식으로 변환하기 위해 [date-fns](https://date-fns.org/)을 사용했습니다.

```typescript
private getDatePeriod(
  startDate: string | undefined,
  endDate: string | undefined,
): [string, string] {
  // * 처음과 마지막을 쿼리로 전달하지 않을 경우, 3개월 전부터 오늘까지를 기준으로 정합니다.
  let startDateString = '';
  let endDateString = '';
  const UTCZeroToday = subHours(
    set(new Date(), {
      hours: 0,
      minutes: 0,
      seconds: 0,
      milliseconds: 0,
    }),
    9,
  );
  if (startDate) {
    startDateString = `${startDate} 00:00:00`;
  } else {
    startDateString = format(subMonths(UTCZeroToday, 3), 'yyyy-MM-dd HH:mm:ss');
  }
  if (endDate) {
    endDateString = `${endDate} 23:59:59`;
  } else {
    const endDate = add(UTCZeroToday, {
      hours: 23,
      minutes: 59,
      seconds: 59,
    });
    endDateString = format(endDate, 'yyyy-MM-dd HH:mm:ss');
  }
  return [startDateString, endDateString];
}
```

1. SQLite3 은 UTC+0 시간대를 기준으로 잡고 있습니다. 그에 맞춰 `UTCZeroToday` 변수의 값을 UTC+0 시간대로 변환한 오늘 날짜로 하고, 0시 0분 0초로 설정합니다.
2. `UTCZeroToday`을 이용해 3개월 전의 날짜를 구하고 `format` 함수로 'yyyy-MM-dd HH:mm:ss' 형식의 문자열로 변환합니다.
3. 마찬가지로, `UTCZeroToday`에 23시 59분 59초를 더해 오늘의 마지막 시간으로 설정한 후, 문자열로 변환합니다.

##### 입급과 출금을 필터링하는 쿼리

입금과 출금, 또는 둘 다 출력하기 위해선, 쿼리문을 두 개로 나눌 필요가 있었습니다. 왜냐면 모두 출력하는 where 조건절이 기존의 방식과 달리 `IN` 연산자를 사용하기 때문입니다.

```typescript
let transTypeQuery: any = [
  'transaction.trans_type = :trans_type',
  { trans_type },
];
if (!trans_type) {
  transTypeQuery = [
    'transaction.trans_type IN (:...trans_type)',
    { trans_type: ['in', 'out'] },
  ];
}
```

이제 위의 `transTypeQuery` 를 아래처럼 적용시킬 수 있습니다. mongoDB 의 aggregate 문을 작성하며 얻은 아이디어로, `transTypeQuery`같이 변수로 선언하면 다른 쿼리문에서도 사용할 수 있어 코드의 중복을 줄일 수 있습니다.

```typescript
andWhere(transTypeQuery[0], transTypeQuery[1]);
```

##### 특정 컬럼의 내용만을 출력하기

거래일시, 거래금액, 잔액, 거래종류, 적요를 출력해야 하는데 `select`를 이용해 특정 컬럼의 내용만을 출력할 수 있습니다.

```typescript
.select([
  'transaction.id',
  'transaction.createdAt',
  'transaction.amount',
  'transaction.balance',
  'transaction.trans_type',
  'transaction.comments',
])
```

## 프로젝트를 마치며

이번 프로젝트의 주 목적은 typeorm 을 이용해 데이터를 추출하는 것으로, `createQueryBuilder` 그리고 where 조건절을 어떻게 다뤄야 하는지에 대해 배울 수 있었습니다.

하지만 지금의 코드로는 우대 사항에 적힌대로 "1억 건"이 있을 경우 제대로 성능을 내는 쿼리문인지는 확신하기 어렵다고 생각합니다. 그리고 기존의 방식과 어떤 차별점을 두어야 데이터의 양이 많아질 때도 성능을 유지할 수 있는지, 대용량 서비스에 대한 이해도를 높일 필요가 있음을 느꼈습니다.
