---
title: "Mongo DB 적응기"
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - experience
toc: true
toc_sticky: true
toc_labe: 목차
description: NoSQL인 몽고를 처음 접하고, 사용해왔던 과정을 기록했습니다.
excerpt: NoSQL인 몽고를 처음 접하고, 사용해왔던 과정을 기록했습니다.
tags:
  - database
  - mongo
  - NoSQL
---

express 기반 백엔드 서버를 구축할 때 사용하는 mongoDB에 대해서, 제가 어떤 방식으로 사용하는 지를 정리하고자 글을 작성했습니다.

## 1. express 서버에 mongo atlas 연결하기

mongoDB 클라우드 매니지 서비스인 [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)에서 우선 프로젝트를 생성한 후, 클러스터를 생성해 그것을 연결시키는 작업을 진행했습니다. 여기서 mongoDB 객체 모델링 툴인 [mongoose](https://www.npmjs.com/package/mongoose) 덕분에, 수월하게 연결할 수 있었습니다.
연결하는 코드는 [노마드 코더](https://nomadcoders.co/)에서의 받은 강의에서 배운 대로 작성했습니다.

```javascript
// src/database.js
const mongoUrl = "MongoDB Atlas에서 가져온 connection string";
mongoose.connect(mongoUrl, { useNewUrlParser: true, useFindAndModify: false });
const database = mongoose.connection;

const handleOpen = () => console.log("Connected to Database");
const handleError = (error) =>
  console.log(`Error on Database Connection: ${error}`);

database.once("open", handleOpen);
database.on("error", handleError);
```

## 2. 컬렉션 스키마, 모델 설정하기

연결을 완료한 후, 실제 데이터를 작성하기 위해 스키마와 모델을 작성해야합니다. 예시로 자유게시판의 게시글 컬렉션의 모델을 작성했습니다.

```javascript
// src/model/Board.js
const schema = new mongoose.Schema({
  title: { type: String, required: true },
  contents: { type: String, required: true },
  registered: { type: Date, default: Date.now },
  writerId: { type: mongoose.Types.ObjectId, ref: "users", required: true },
});

const model = mongoose.model("boards", schema);

export default model;
```

각 항목마다 타입을 정하고, 필수인지(required), 아니면 기본값을 지정할 것인지(default)를 설정할 수 있습니다.
`writerId`는 회원 컬렉션에서의 회원과 연결되는 외래키입니다. 처음에는 회원의 이름만 필요하니까 `writerName: {type: String, required: true}`로 하면 되지 않을까 생각했지만, 만약 회원이 이름을 바꿀 경우 거기에 맞춰 해당 회원의 게시글들을 수정하는 번거로움이 생긴다는 것을 깨달았습니다.
지금 작성한 모델을 이용해 컬렉션에 대해 생성, 읽기, 갱신, 삭제를 수행할 수 있습니다.

## 2. 컬렉션에서 원하는 데이터를 가져오기

모델을 이용해 정보를 가져올 수 있게 되었습니다. 예시로 게시글 하나를 불러오는 코드를 작성했습니다. MongoDB를 처음 접할 때 작성한 코드입니다.

```javascript
// src/controller/Board.js
import Board from "./../model/Board.js";
import User from "./../model/User.js";

const getOne = async (req, res) => {
  try {
    const { id } = req.params;
    const boardInfo = await Board.findById(id);
    const userInfo = await User.findById(boardInfo.writerId);

    res.json({ ...boardInfo, writerName: userInfo.name });
  } catch (error) {
    console.log("board - getOne error", error);
    res.status(500).send("getOne error");
  }
};
```

이 방식은 번거롭고, 두 번 값을 호출하는 만큼 속도도 느립니다. 방법을 찾던 중 aggregate를 알게 되었고, 이를 활용하기로 생각했습니다.

```javascript
// 처음 도입부, 함수 선언은 생략합니다.
const { id } = req.params;
const aggregateOption = [
  { $match: { _id: mongoose.Types.ObjectId(id) } },
  {
    $lookup: {
      from: "users",
      let: { userId: "$writerId" },
      pipeline: [
        { $match: { $expr: { $eq: ["$_id", "$$userId"] } } },
        { $project: { _id: 1, name: "$name" } },
      ],
      as: "userInfo",
    },
  },
  { $unwind: "$userInfo" },
  {
    $set: {
      writerName: "$userInfo.name",
      registered: {
        $dateToString: {
          format: "%Y-%m-%d",
          timezone: "+09:00",
          date: "$registered",
        },
      },
    },
  },
];

const boardInfo = await Board.aggregate(aggregateOption);
```

aggregate의 장점은 타 컬렉션과의 연결뿐만이 아닙니다. 다른 예시로 학생의 점수를 보여주는 aggregate를 작성하겠습니다.

```javascript
const aggregateOption = [
  { $match: { _id: mongoose.Types.ObjectId(id) } },
  {
    $set: {
      totalScore: {
        $add: ["$korean", "$math", "$english", "$society", "$science"],
      },
      testedAt: {
        $dateToString: {
          format: "%Y-%m-%d",
          timezone: "+09:00",
          date: "$testedAt",
        },
      },
    },
  },
  {
    $project: {
      _id: 1,
      name: 1,
      class: 1,
      testedAt: 1,
      totalScore: 1,
      meanScore: { $divide: ["$totalScore", 5] },
      isPassed: { $cond: [{ $gte: ["$totalScore", 350] }, "통과", "재시험"] },
    },
  },
];
```

국어, 수학, 영어, 사회, 과학의 총점을 구한 후, 그것을 통해 평균 점수와 통과 여부를 aggregate에서 계산하도록 만들었습니다. 데이터 가공을 aggregate 영역에서 수행함으로써, 백엔드 또는 클라이언트 영역에서 데이터를 가공하는 수고가 줄었습니다. 이러한 장점으로 인해, 저는 현재도 find(), findOne() 보다는 aggregate를 보다 적극적으로 사용하고 있습니다.

## 3. 플러그인을 추가하기

저는 하나의 document(데이터)를 작성할 때마다 숫자 인덱스를 추가하는 절차가 필요했습니다. 이전까지는 숫자 값만을 저장하는 컬렉션을 임의로 만든 후, 거기서 숫자를 불러와서 인덱스로 추가하는 과정을 수작업으로 했었습니다. 하지만 [mongoose-auto-increment](https://www.npmjs.com/package/mongoose-auto-increment) 플러그인 덕분에 이 과정을 자동으로 수행하도록 만들었습니다.

```javascript
// src/model/Board.js
import autoIncrement from "mongoose-auto-increment";

const schema = new mongoose.Schema({
  serial: Number,
  title: { type: String, required: true },
  contents: { type: String, required: true },
  registered: { type: Date, default: Date.now },
  writerId: { type: mongoose.Types.ObjectId, ref: "users", required: true },
});

autoIncrement.initialize(mongoose.connection);
schema.plugin(autoIncrement.plugin, {
  model: "boards",
  field: "serial",
  startAt: 1,
  increment: 1,
});

const model = mongoose.model("boards", schema);

export default model;
```

mongoose-auto-increment는 자동으로 identitycounters 컬렉션을 생성해 count값들을 저장합니다. 이제 게시글을 만들 때마다 serial값을 자동으로 추가해줍니다.
