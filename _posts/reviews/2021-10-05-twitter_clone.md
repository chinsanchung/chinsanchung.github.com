---
title: "트위터의 일부 기능을 구현한 witter 제작기"
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
description: 트위터의 일부 기능을 구현한 witter 애플리케이션을 제작하고 heroku 에 배포했던 경험을 정리합니다.
excerpt: 트위터의 일부 기능을 구현한 witter 애플리케이션을 제작하고 heroku 에 배포했던 경험을 정리합니다.
tags:
  - toy_project
---

트위터의 일부 기능을 직접 구현해보는 [witter](https://witterclone.herokuapp.com) 프로젝트를 제작하면서 어떤 방식으로 개발을 했는지, 문제를 해결했던 경험을 정리합니다. [클라이언트 레포지토리](https://github.com/chinsanchung/witter-client), [백엔드 레포지토리](https://github.com/chinsanchung/witter-server)에서 코드를 확인하실 수 있습니다.

## 1. 프로젝트를 시작한 이유

프로젝트를 시작한 계기는 Nomad Coder 라는 온라인 교육 사이트에서 주최한 트위터 클론코딩 콘테스트였습니다. React 와 Firebase 를 이용해 개발한 후 그것을 배포해서 인증하는 콘테스트였는데, Firebase 로 모든 백엔드를 처리하는 대신 직접 모든 기능을 만들어보자는 생각에 프로젝트 제작을 시작했습니다.

## 2. 백엔드 프로젝트의 구조와 설계

실제 트위터에는 다양한 기능들이 있지만, 저는 주로 4가지 기능에 중점을 두었습니다. 로그인, 회원가입, 로그아웃같은 사용자 인증 기능, 트윗의 CRD(Create, Read, Delete)와 트윗과 관련된 기능들(리트윗, 마음에 들어요), 트윗의 목록(타임라인)을 보여주기, 마지막으로 사용자의 정보를 확인하고 갱신하는 기능을 각각 `auth`, `tweet`, `reading`, `user` 폴더로 구분했습니다.

### 기능을 컨트롤러, 서비스, 그리고 인터페이스로 구분하여 작성하기

기능을 담당하는 각 폴더마다 요청과 응답을 처리하는 `컨트롤러`, 컨트롤러의 요청을 받아 실제 데이터베이스와 연동하여 기능을 수행하고 컨트롤러로 요청에 맞는 데이터를 전달하는 `서비스`, 그리고 컨트롤러 또는 서비스 클래스의 인터페이스를 모은 `인터페이스`로 역할에 맞게 구분하여 작성했습니다.

예전에는 하나의 컨트롤러에 모든 내용을 담았는데, 복잡한 기능일 경우 API 함수 하나에 200자를 넘은 적도 있었습니다. 그로인해 코드의 흐름을 파악하기 어려워졌고, 나중에 오류를 발견해 코드를 수정할 때 원인을 찾는데 오랜 시간이 걸렸습니다.

컨트롤러와 서비스로 구분하면서, 코드를 분산하게 되어 코드를 읽는 부담이 줄어들었고 오류의 원인이 어디인지를 가늠하기가 수월해졌습니다. 또한 API 기능의 흐름을 보다 명확하게 이해하게 되었습니다. 현재 API 함수가 작동하는 방식은 다음과 같습니다.

1. 클라이언트에서 API 를 호출하여 요청을 보냅니다.
2. 컨트롤러에서 그 요청을 읽습니다. 요청을 수행하기 위해 서비스를 호출합니다.
3. 서비스에서는 데이터베이스에 접근해 요청을 수행합니다. 만약 GET 메소드일 경우 데이터를 리턴합니다.
4. 다시 컨트롤러로 돌아와 서비스의 리턴값을 읽고, 그것을 응답으로 클라이언트에 전달합니다.

### 클래스 기반으로 작성하기

예전에는 서버의 기능을 각각 함수들로 구현했다면, 이번에는 ES6에서 새로 도입된 클래스 기반으로 작성했습니다. 작성 방식을 바꾼 이유는 첫째로 가독성을 높이기 위해서입니다. 우선 기존의 방식대로 작성한 예시를 보겠습니다.

```typescript
const defaultSettingQuery = [
  {
    $set: {
      create_date: {
        $dateToString: {
          format: "%H:%M · %Y년 %m월 %d일",
          timezone: "+09:00",
          date: "$create_date",
        },
      },
      retweet_count: { $size: "$retweet" },
      like_count: { $size: "$like" },
      comments_count: { $size: "$comments" },
    },
  },
];
const getUserInfoQuery = async (writer_id: string) => {
  return {
    $lookup: {
      from: "users",
      let: { writer_id: `$${writer_id}` },
      pipeline: [
        { $match: { $expr: { $eq: ["$user_id", "$$writer_id"] } } },
        {
          $project: {
            _id: 0,
            name: "$name",
            user_id: "$user_id",
            profile_color: "$profile_color",
            description: "$description",
            follower: "$follower",
            following: "$following",
            follower_count: { $size: "$follower" },
            following_count: { $size: "$following" },
          },
        },
      ],
      as: "user",
    },
  };
};

const getUserTimeLine = async (user_id) => {
  try {
    const response = await TimeLineModel.aggregate([
      { $match: { user_id } },
      getUserInfoQuery("writer_id"),
      defaultSettingQuery,
      // ...
    ]);
    if (response.length > 0) {
      return response;
    } else {
      throw createError(404, "타임라인에 트윗이 없습니다.");
    }
  } catch (error) {
    throw error;
  }
};
const getHomeTimeLine = async (user_id) => {
  // ...
};

export default { getUserTimeLine, getUserTimeLine };
```

쿼리 변수(`getTweetsFromTimeLineQuery`)나 API 함수(`getUserTimeLine`, `getUserTimeLine`), 기능 구현을 위한 부수적인 함수(`getUserInfoQuery`)가 일렬로 나열되어 있어서, 이것이 어떤 역할을 맡은 것인지 한 눈에 들어오질 않습니다. 이번에는 위의 내용을 클래스로 작성하면 아래와 같은 코드가 됩니다.

```typescript
export default class ReadingService implements IReadingService {
  constructor() {
    this.getUserTimeLine = this.getUserTimeLine.bind(this);
    this.getHomeTimeLine = this.getHomeTimeLine.bind(this);
  }
  private defaultSettingQuery = [
    // ...
  ];
  private getUserInfoQuery(writer_id: string) {
    return {
      // ...
    };
  }
  async getUserTimeLine(user_id) {
    try {
      const response = await TimeLineModel.aggregate([
        { $match: { user_id } },
        this.getUserInfoQuery("writer_id"),
        this.defaultSettingQuery,
        // ...
      ]);
      if (response.length > 0) {
        return response;
      } else {
        throw createError(404, "타임라인에 트윗이 없습니다.");
      }
    } catch (error) {
      throw error;
    }
  }
  async getHomeTimeLine(user_id) {
    // ...
  }
}
```

이후에 라우팅에 선언해서 활용할 API 함수는 public 으로, 클래스 내부에서 부수적인 기능을 담당하는 함수와 쿼리 변수를 private 으로 선언하여 역할을 구분할 수 있게 되었습니다. 또한 독립적으로 따로 놀았던 코드들을 `ReadingService`라는 클래스 안에 포함함으로써 코드를 읽기가 편해졌습니다.

작성 방식을 바꾼 두 번째 이유는 클래스의 인터페이스 선언을 위해서입니다. 인터페이스로 클래스의 타입 규칙을 지정함으로써 어떠한 의도를 가지고 작성한 함수인지, 매개변수는 무엇이고 무엇을 리턴하는지 확인하는 한편, 의도하지 에러를 사전에 방지하는데 도움이 되었습니다.

#### 추가: 클래스의 메소드는 화살표 함수가 아니라 일반 함수로 작성해야 한다.

처음에 메소드를 작성할 때 일반 함수로 작성하면서 `bind`로 일일이 묶는 것이 번거로워 화살표 함수로 작성을 했었습니다. 그런데 [Should I write methods as arrow functions in Angular's class](https://stackoverflow.com/a/45882417), [Class에서 arrow function을 사용하지 말아야하는 이유](https://simsimjae.tistory.com/452)를 읽으며 화살표 함수로 클래스 메소드를 작성하면 안된다는 것을 꺠달았습니다.

위의 두 글에 따르면,

1. 부모 클래스의 메소드를 화살표 함수로 작성하면 그 메소드는 자식 클래스에서 오버라이드해서 사용할 수 없게 됩니다.
2. 또한 자식 클래스에서 `super.method()`으로 상속해서 사용할 수 없습니다. 화살표 함수로 선언하면 인스턴스 함수가 되기 떄문에, 상속받은 자식 클래스가 아닌 부모 클래스의 인스턴스에만 화살표 함수가 존재하게 되기 때문입니다.

비록 현재 컨트롤러와 서비스의 메소드들은 상속이나 오버라이드를 하지는 않지만, 위의 조언에 따라 전부 화살표 함수에서 일반 함수로 수정했습니다.

### 추가: REST API 디자인 가이드에 맞춰 URI 수정하기

현재 작성한 URI 는 컨트롤러의 메소드명을 그대로 옮겨적는 경우가 많았습니다. 하지만 [REST API 제대로 알고 사용하기](https://meetup.toast.com/posts/92)를 읽으면서 REST API 디자인 가이드대로 작성하지 않았다는 것을 알았습니다.

가이드에 맞게 기존의 URI 를 다음과 같이 수정했습니다.

- URI 의 리소스 명은 명사를 사용하도록 수정하고 자원을 표현하는 데 집중하도록 작성했습니다.
- URI 에 행위에 대한 표현을 지우고 그 행위는 HTTP Method 로 표현하도록 수정했습니다.

## 3. 트위터의 기능 구현하기

트위터의 기능들을 구현하면서 기억에 남는 점 몇 가지를 설명하겠습니다.

### 타임라인

타임라인이란 간단하게 정의하자면 트윗들의 모음으로, 로그인한 사용자의 홈 타임라인과 특정 사용자의 타임라인 두 가지로 구분합니다.

- 홈 타임라인: 사용자가 작성한 트윗과 리트윗, 그리고 팔로우하고 있는 사용자가 작성한 트윗과 리트윗을 나열한 목록입니다. 현재 트위터에서는 인기순과 최신순으로 정렬하고 있습니다.
- 사용자 타임라인: 특정 사용자가 작성한 트윗과 리트윗을 나열한 목록입니다. 여기서는 최신순으로 정렬합니다.

처음에는 트윗 컬렉션 하나만으로 타임라인을 구현할 수 있을 거라 생각했습니다. 그렇게 구현하려면 아래의 과정으로 타임라인을 가져옵니다.

1. 로그인한 사용자가 작성한 모든 트윗들을 가져옵니다.
2. 로그인한 사용자의 팔로워 목록을 가져오고, 해당 user_id 를 작성자로 하는 트윗들 전부를 가져옵니다.
3. 그 트윗들을 전부 합친 후, 시간순으로 내림차순하여 타임라인을 만듭니다.

이 방식의 문제점은 여러 개의 aggregate 쿼리문을 작성하게 되어, 트윗들을 합치고 정렬하는 것을 직접 수행해야 합니다. 또한, 리트윗은 트윗의 작성일이 아니라 리트윗을 수행한 날짜를 정렬의 기준으로 삼아야 합니다. 그러면 정렬을 할 때마다 조건문으로 리트윗인지 아닌지를 체크하고 그만큼 시간을 소요하게 됩니다.

그래서 결국 타임라인 컬렉션을 직접 만들고 여기에는 사용자의 아이디와 트윗 목록, 마음에 들어요 목록을 넣게 되는데, 각 목록에는 트윗의 아이디와 타임라인에 등록한 시간 그리고 리트윗인지 아닌지 여부를 담는 객체입니다. 이렇게 하면 다음과 같은 절차를 수행하게 됩니다.

1. 로그인한 사용자의 타임라인 데이터를 가져옵니다. tweet_list 에 담긴 tweet_id 를 통해 트윗의 정보를 가져옵니다.
2. 팔로워 목록의 user_id 를 이용해 각 사용자의 타임라인을 가져옵니다. 마찬가지로 tweet_id 로 트윗의 정보를 가져옵니다.
3. 1번과 2번에서 가져온 트윗들을 등록한 시간순으로 내림차순하여 타임라인을 만듭니다.

따라서 하나의 쿼리문으로 완성할 수 있어서 시간 순으로 정렬하기 수월해졌고, 트윗의 작성일 대신 타임라인에 등록한 날짜를 등록하게 되어 리트윗에도 유연하게 대응할 수 있게 되었습니다.

### 로그인 여부 인증하기

트윗의 작성, 리트윗 등 로그인을 해야만 사용할 수 있는 기능들이 있습니다. 로그인 인증을 위한 미들웨어가 필요한데, passport 에서 지원하는 isAuthenticated() 메소드를 이용했습니다.

isAuthenticated() 는 `req.isAuthenticated()`로 실행하는데, 작동원리는 passport 를 이용해 로그인을 했을 때 request 영역에 생성되는 user 객체를 실제로 가지고 있는지를 확인하는 것입니다.

```javascript
// Passport 모듈의 lib/http/request.js 에서 가져왔습니다.

/**
 * Test if request is authenticated.
 *
 * @return {Boolean}
 * @api public
 */
req.isAuthenticated = function () {
  var property = this._userProperty || "user";
  return this[property] ? true : false;
};

/**
 * Test if request is unauthenticated.
 *
 * @return {Boolean}
 * @api public
 */
req.isUnauthenticated = function () {
  return !this.isAuthenticated();
};
```

특이하게도 Passport 의 공식문서에는 isAuthenticated 대한 설명이 없지만 [Node.js 교과서](http://www.yes24.com/product/goods/62597864)를 그 존재를 알았고, 덕분에 간편하게 인증 미들웨어를 생성할 수 있었습니다.

(참고로 깃허브 이슈에 이에 대한 [문의사항](https://github.com/jaredhanson/passport/issues/683)이 있는데, 메소드에 대한 설명을 풀 리퀘스트를 보냈지만 아직도 머지를 하지 않았다고 합니다.)

## 4. 빌드 후 배포하기

마지막으로 프로젝트를 빌드해서 실제 사이트에 배포하는 과정에서 겪은 과정을 설명드리고자 합니다. 저는
헤로쿠(Heroku)를 이용했습니다. 헤로쿠는 앱의 배포와 실행을 돕는 플랫폼으로 무료로 이용할 수 있다는 점이 큰 장점입니다. Node 뿐만 아니라 Ruby, Java, PHP, Python. Go 등 다양한 코드를 지원합니다.

여기서 저는 Node 를 선택해서 배포를 진행했는데, 그 과정에서 여러 문제를 겪었습니다.

- `npm start` 스크립트를 실행하지 못하는 문제

깃허브 레포지토리를 연결한 후 처음으로 배포를 진행했는데 다음과 같은 에러가 발생했습니다.

```
Failed at the twitter_clone_server@1.0.0 start script.
This is probably not a problem with npm. There is likely additional logging output above.
```

스크립트를 실행하질 못했다는 내용인데, [헤로쿠에서는 devDependencies 에 있는 의존성을 설치하지 않는데](https://stackoverflow.com/a/68676440), `npm start` 스크립트에 개발 의존성으로 설치했던 `cross-env`를 작성한 것이 원인임을 찾아냈습니다. 즉, 헤로쿠에서 `cross-env` 모듈을 설치하지 않아 스크립트에 있는 cross-env 를 인식하지 못해서 생긴 것이었습니다.

따라서 스크립트의 내용을 `node dist/index.js`으로 수정해서 오직 "node 로 빌드한 폴더 dist 안의 index.js 를 실행하라"는 명령만 내리도록 했습니다.

- process.env 환경 변수 설정

개발을 할 때는 `dotenv`라는 모듈을 이용해서 민감한 설정들을 .env 파일에 저장해서 사용해왔습니다. 하지만 헤로쿠에서는 Config Vars 설정에서 환경변수를 등록해야 합니다.

- PORT 설정

AWS Beanstalk 에서는 배포를 할 때의 포트를 8080으로 지정합니다. 그래서 처음에는 헤로쿠도 배포용 포트를 따로 지정해야 하는줄 알았습니다. 하지만 stack overflow 의 [Setting the port for node.js server on Heroku](https://stackoverflow.com/a/51572239)을 통해 헤로쿠는 PORT 환경변수로 동적으로 포트를 할당하고 있다는 것을 알게 되었고, 따라서 아래처럼 포트의 설정을 수정했습니다.

```javascript
private PORT = process.env.PORT || 5000;
```

## 5. 글을 마치며

백엔드와 클라이언트, 그리고 앱 배포까지 모든 과정을 수행하면서 그동안 배워왔던 내용을 점검하며 배울 수 있었습니다. 그리고 기능을 만들기 위해 트위터에 접속해 살펴보는 일이 많았는데, 직접 글을 쓰고 리트윗을 하면서 간단하게 동작하는 기능들 하나하나가 사실은 많은 고민과 논의를 거쳐 나온 것임을 다시금 깨달을 수 있었습니다. 그러면서 트위터나 인스타그램같은 대규모 시스템은 어떻게 데이터베이스를 설계하고 각 기능들을 설계했는지, 설계하면서 겪은 문제가 무엇일지에 대한 궁금증이 생겼습니다.
