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

[주소](https://witterclone.herokuapp.com/)

## 백엔드 프로젝트의 구조, 설계

각 기능 단위로 구분해서 폴더에 저장하고, 요청과 응답을 처리하는 `컨트롤러`, 컨트롤러의 요청을 받아 실제 데이터베이스와 연동하여 기능을 수행하고 컨트롤러로 요청에 맞는 데이터를 전달하는 `서비스`, 그리고 컨트롤러 또는 서비스 클래스의 인터페이스를 모은 `인터페이스`로 구분해서 작성했습니다.

기능 단위로 폴더를 구분하는 작업은 이전부터 해왔지만, 맡고 있는 역할에 따라서 컨트롤러와 서비스, 인터페이스로 분류하는 작업은 처음이었습니다. 그리고 예전에는 함수 단위로 export 를 해서 라우터에 연결했다면, 이번에는 각 컨트롤러, 서비스를 하나의 클래스로 만들어서 작업했습니다.

클래스로 만들면서 주의한 점은 되도록 하나의 함수에는 하나의 기능만을 담는 것으로, 서비스 메소드에는 요청에 맞는 그 기능만을 작성하고, 그 외 부수적인 기능은 private 접근 제한을 걸어 클래스 내에서만 사용하도록 했습니다. 이를 통해 코드의 가독성과 재사용성을 높이고, 해당하는 요청 이외의 메소드의 사용을 제한할 수 있었습니다.

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

다음으로 주의한 점은 서비스 클래스에 대한 인터페이스를 되도록 꼼꼼하게 작성하는 것입니다. 각 메소드의 매개변수와 리턴의 타입을 명확하게 정의함으로써 잘못된 값을 가져오거나 리턴하지 않도록 하는 한편, 나중에 코드를 다시 읽었을 때 이 메소드는 이런 매개변수를 사용해서 이런 결과값을 리턴하는 의도로 작성한 것이라는 것을 빠르게 파악할 수 있다는 장점이 있었습니다. 부수적으로 인터페이스를 먼저 작성한 후 클래스를 선언할 때 vscode 의 자동완성 기능을 이용해서 조금이나마 수고를 덜 수 있었습니다.

## heroku 로 배포하기
