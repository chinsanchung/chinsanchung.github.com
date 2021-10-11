---
title: '트위터의 일부 기능을 클론 코딩한 twitter_clone 제작기'
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
description: 트위터의 일부 기능을 클론 코딩한 twitter_clone 프로젝트를 제작하고 heroku 에 배포했던 경험을 작성합니다.
tags:
  - toy_project
---

[주소](https://witterclone.herokuapp.com)

## 백엔드 프로젝트의 구조와 설계

실제 트위터에는 다양한 기능들이 있지만, 저는 주로 4가지 기능에 중점을 두었습니다. 로그인, 회원가입, 로그아웃같은 사용자 인증 기능, 트윗의 CRD(Create, Read, Delete)와 트윗과 관련된 기능들(리트윗, 마음에 들어요), 트윗의 목록(타임라인)을 보여주기, 마지막으로 사용자의 정보를 확인하고 갱신하는 기능을 각각 `auth`, `tweet`, `reading`, `user` 폴더로 구분했습니다.

각 폴더에는 요청과 응답을 처리하는 `컨트롤러`, 컨트롤러의 요청을 받아 실제 데이터베이스와 연동하여 기능을 수행하고 컨트롤러로 요청에 맞는 데이터를 전달하는 `서비스`, 그리고 컨트롤러 또는 서비스 클래스의 인터페이스를 모은 `인터페이스`를 작성했습니다.

한 가지 생각한 점은 컨트롤러와 서비스는 모두 클래스 기반으로 만들었는데, 만드는 과정에서 두 가지를 염두에 두고 작업했습니다.

1. 하나의 함수에는 하나의 기능만을 담는 것으로, 서비스 메소드에는 요청에 맞는 그 기능만을 작성하고, 그 외 부수적인 기능은 private 접근 제한을 걸어 클래스 내에서만 사용하도록 했습니다. 이를 통해 코드의 가독성과 재사용성을 높이고, 해당하는 요청 이외의 메소드의 사용을 제한할 수 있었습니다.

다음은 그런 방식으로 작성한 코드의 예시입니다. 사용자의 정보를 가져오는 aggregate 쿼리문을 리턴하는 `getUserInfoQuery`, 그리고 특정 트윗과 그에 대한 답글들을 출력하는 `getTweets`메소드입니다. (트윗의 작성자 아이디를 매개변수로 받아 특정 사용자의 정보를 가져오기 위해 aggregate 쿼리문을 메소드로 만들었습니다.)

```javascript
export default class ReadingService implements IReadingService {
    private getUserInfoQuery = (writer_id: string) => {
    return {
      $lookup: {
        from: 'users',
        let: { writer_id: `$${writer_id}` },
        pipeline: [
          { $match: { $expr: { $eq: ['$user_id', '$$writer_id'] } } },
          {
            $project: {
              _id: 0,
              name: '$name',
              user_id: '$user_id',
              profile_color: '$profile_color',
              description: '$description',
              follower: '$follower',
              following: '$following',
              follower_count: { $size: '$follower' },
              following_count: { $size: '$following' },
            },
          },
        ],
        as: 'user',
      },
    };
  };
  // ...
    getTweets = async (tweet_id: number): Promise<IGetTweetsResponse> => {
    try {
      const originalTweet = await TweetModel.aggregate([
        { $match: { tweet_id } },
        this.getUserInfoQuery('user_id'),
        ...this.defaultSettingQuery,
        { $unwind: '$user' },
      ]);
      if (originalTweet.length > 0 && originalTweet[0].is_active) {
        const comments = await TweetModel.aggregate([
          { $match: { tweet_id: { $in: originalTweet[0].comments } } },
          this.getUserInfoQuery('user_id'),
          ...this.defaultSettingQuery,
          { $unwind: '$user' },
          { $sort: { create_date: 1 } },
        ]);
        return {
          origin: originalTweet[0],
          comments,
        };
      } else {
        throw createError(404, '존재하지 않는 트윗입니다.');
      }
    } catch (error) {
      throw error;
    }
  };
}
```

2. 다음으로 주의한 점은 서비스 클래스에 대한 인터페이스를 되도록 꼼꼼하게 작성하는 것입니다. 각 메소드의 매개변수와 리턴의 타입을 명확하게 정의함으로써 잘못된 값을 가져오거나 리턴하지 않도록 하는 한편, 나중에 코드를 다시 읽었을 때 이 메소드는 이런 매개변수를 사용해서 이런 결과값을 리턴하는 의도로 작성한 것이라는 것을 빠르게 파악할 수 있다는 장점이 있었습니다. 부수적으로 인터페이스를 먼저 작성한 후 클래스를 선언할 때 vscode 의 자동완성 기능을 이용해서 조금이나마 수고를 덜 수 있었습니다.

## 트위터의 기능 구현하기

트위터의 일부 기능들을 구현하면서 기억에 남는 점 몇 가지를 설명하고자 합니다.

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
  var property = this._userProperty || 'user';
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

## 빌드 후 배포하기

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
