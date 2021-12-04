---
title: "아임포트 결제 기능 도입기"
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
description: 아임포트 결제 기능을 개발해온 과정을 정리합니다.
excerpt: 아임포트 결제 기능을 개발해온 과정을 정리합니다.
tags:
  - javascript
  - express
  - iamport
---

현재 AWS S3(객체 스토리지 서비스)에 음원 및 비디오 파일을 업로드한 후 Elemental MediaConvert(비디오 트랜스코딩 서비스), Elastic Transcoder(음원 미디어 트랜스코딩 서비스)으로 인코딩해서 매장에 스트리밍하는 판뮤직이란 서비스를 외주받아 개발하고 있습니다. 저는 미디어 변환 대신 아임포트 결제 기능을 제작하고 있는데, 개발해온 과정을 정리하고자 글을 작성합니다.

주로 [아임포트 docs](https://docs.iamport.kr/)를 참고해서 진행했고, 클라이언트는 create-react-app, 백엔드는 express 를 이용해 개발하고 있습니다.

## 1. 들어가기에 앞서

우선 아임포트 관리자에서 테스트 모드 설정을 해야 합니다. [테스트모드 설정하기](https://docs.iamport.kr/admin/test-mode)를 참고하셔서 작업하시기 바랍니다. 참고로 일반 결제와 정기 결제의 PG 설정이 다르다는 점에 유의하셔야 합니다.
글의 진행 방식은 크게 기본적인 설정, 일반 결제, 정기 결제, 모바일 콜백으로 진행하겠습니다.

## 2. 기본 설정

### 클라이언트에서의 아임포트 cdn 설정

클라이언트에서는 우선 라이브러리를 가져온 후, 그것으로 아임포트를 실행하는 버튼을 만들어야 합니다. 그리고 아임포트에서 결제를 하면서 서버 함수를 연결시킵니다.

처음에는 index.html 의 script 에 라이브러리를 가져왔지만, 지금은 버튼이 있는 컴포넌트가 마운트됐을 때 라이브러리를 실행하도록 변경했습니다. 그래서 처음부터 전부 불러오지 않고, 필요할 때만 라이브러리를 가져온다는 장점이 있습니다.

```javascript
// import 과정은 생략합니다.
function IamportLibrary() {
  // 라이브러리 가져오기
  useEffect(() => {
    const loadSdk = () => {
      const jqueryPromise = new Promise((res, rej) => {
        const library = document.querySelector("#jquery-sdk");
        if (library) {
          res();
        } else {
          const jquery = document.createElement("script");
          jquery.id = "jquery-sdk";
          jquery.src = "//code.jquery.com/jquery-3.6.0.min.js";
          jquery.onload = res;
          document.body.appendChild(jquery);
        }
      });
      const iamport = new Promise((res, rej) => {
        const library = document.querySelector("#iamport-sdk");
        if (library) {
          res();
        } else {
          const iamport = document.createElement("script");
          iamport.id = "iamport-sdk";
          iamport.src = "//cdn.iamport.kr/js/iamport.payment-1.1.8.js";
          iamport.onload = res;
          document.body.appendChild(iamport);
        }
      });
      return Promise.all([jqueryPromise, iamport]);
    };
    const unloadSdk = () => {
      const jqueryPromise = new Promise((res, rej) => {
        const library = document.querySelector("#jquery-sdk");
        if (library) {
          library.parentNode.removeChild(library);
        } else {
          res();
        }
      });
      const iamport = new Promise((res, rej) => {
        const library = document.querySelector("#iamport-sdk");
        if (library) {
          library.parentNode.removeChild(library);
        } else {
          res();
        }
      });

      return Promise.all([jqueryPromise, iamport]);
    };

    loadSdk();

    return function cleanup() {
      unloadSdk();
    };
  }, []);

  // ...
}
```

### 백엔드 설정

백엔드에서는 기본적으로 아임포트에 요청하기 위해 엑세스 토근이 필요합니다. 이 토큰을 헤더의 authorization 에 담아 전달합니다.

```javascript
const getAccessToken = async () => {
  try {
    const response = await axios({
      url: "https://api.iamport.kr/users/getToken",
      method: "post",
      headers: { "Content-Type": "application/json" },
      data: {
        imp_key: "impKey",
        imp_secret: "impSecret",
      },
    });

    const { access_token } = response.data.response;
    return access_token;
  } catch (error) {
    throw error;
  }
};
```

그 다음으로 중요한 것이 웹훅입니다. 웹훅은 1. 결제를 했을 때(일반 또는 정기 예약), 2. 환불을 했을 때 호출합니다.

웹훅에는 `imp_uid`, `merchant_uid`를 가져오는데, 이것을 활용해 아임포트로부터 엑세스 토큰을 얻어 결제 정보를 찾아 갱신합니다.

저는 `merchant_uid`를 크게 일반 결제용 "merchant_000", 정기 결제용 "schedule_000", 그리고 정기 결제를 위한 빌링키 발급용 "billing_key_000"으로 구분해서 웹훅을 이용했습니다. 각 결제에 맞춰 웹훅을 쪼개 설명하고, 마지막에 코드 전부를 보여드리겠습니다.

```javascript
const iamportWebhook = async (req, res) => {
  const { imp_uid, merchant_uid } = req.body;
  if (merchant_uid.indexOf('billing_key_') >= 0) {
    // 빌링키 발급
  } else if ((merchant_uid.indexOf('schedule_') >= 0) {
    // 정기 결제로 예약했던 결제를 진행한 후 호출됩니다.
  } else {
    // 일반 결제입니다.
  }
}
```

## 3. 일반 결제

### 클라이언트

일반 결제를 시작합니다. 일반 결제를 위해선 결제 방법을 미리 설정해둬야 진행할 수 있습니다. 카드, 휴대폰, 계좌이체 등 사용하고자 하는 결제 방법으로 진행합니다.

아임포트 공식 문서에 따르면, 우선 결제 프로세스 `window.IMP.request_pay()`를 실행하기 전에 데이터베이스에 주문 정보를 저장하고, `window.IMP.request_pay()`를 실행해 결제를 진행, 마지막으로 결제의 위변조 여부를 서버에서 점검하고 갱신하는 것입니다.

```javascript
const startIamportPayment = useCallback(async () => {
  console.log("1. 우선 DB에 결제 정보를 저장합니다.");
  try {
    const merchant_uid = await axios.post("payment", {
      user_id: "사용자 고유 아이디",
      name: "상품 이름",
      amount: 10000,
    });
    console.log("생성 결과에는 merchant_uid 가 담겨있습니다.");
    await commonPayment(merchant_uid.data);
    return;
  } catch (error) {
    console.log("서버에서의 결제 등록 에러", error);
    alert(error?.response?.data);
    return;
  }
}, []);
const commonPayment = useCallback((merchant_uid) => {
  const param = {
    pg: "html5_inicis",
    pay_method: "card",
    merchant_uid,
    name: "상품 이름",
    amount: 10000,
    buyer_email: "이메일",
    buyer_name: "이름",
    buyer_tel: "01012345678",
    digital: false,
    m_redirect_url: "모바일용 리다이렉트 url",
  };

  console.log("2. 아임포트에서 결제를 진행합니다.");

  window.IMP.request_pay(param, async (response) => {
    if (response.success) {
      const { imp_uid, merchant_uid } = response;
      try {
        console.log("결제 결과를 DB와 비교해 위변조를 검증합니다.");
        const response = await axios.post("/payment-complete", {
          imp_uid,
          merchant_uid,
          user_id: "가상계좌일 경우 사용자아이디",
        });
        if (response.data.status === "paid") {
          alert("결제를 성공했습니다.");
        }
        if (response.data.status === "vbank") {
          alert(response.data.text);
        }
      } catch (error) {
        console.log("서버에서의 결제 검증 에러 또는 실패", error);
        alert(error?.response?.data);
      }
    } else {
      console.log("아임포트에서의 결제 실패: ", response.error_msg);
      alert(response);
    }
  });
}, []);
```

### 백엔드

이번에는 백엔드 영역입니다. 우선 처음 결제를 생성하는 함수입니다.

```javascript
const createPayment = async (req, res) => {
  console.log("결제를 시작하기 전 DB에 저장합니다.");
  const { name, amount, user_id } = req.body;
  try {
    const merchant_uid = `merchant_${new Date.getTime()}`;
    await Order.create({
      name,
      amount,
      user_id,
      merchant_uid,
      order_date: new Date(),
    });
    res.send(merchant_uid);
  } catch (error) {
    res.status(500).send("저장 에러");
  }
};
app.post("/payment", createPayment);
```

다음으로 아임포트서 결제를 완료한 후 데이터베이스와 검증하는 함수입니다.

```javascript
const verifyCommonPayment = async (req, res) => {
  const { imp_uid, merchant_uid, user_id } = req.body;
  try {
    const order = await Order.findOne({ merchant_uid });

    const accessToken = await getAccessToken();
    const response = await axios.get(
      `https://api.iamport.kr/payments/${imp_uid}`,
      { headers: { Authorization: accessToken } }
    );
    const paymentData = response.data.response;
    const { amount, status } = paymentData;

    if (amount === order.amount) {
      console.log("결과가 일치합니다.");
      if (status === "ready") {
        console.log(
          "가상계좌 발급입니다. 해당 정보를 클라이언트로 전달합니다."
        );
        const { vbank_num, vbank_date, vbank_name } = paymentData;
        await User.updateOne(
          { _id: user_id },
          { $set: { vbank_num, vbank_date, vbank_name } }
        );
        res.send({
          status: "vbank",
          text: `가상계좌 발급. ${vbank_name}은행 ${vbank_num}, 기한: ${vbank_date}`,
        });
      } else if (status === "paid") {
        res.send({ status: "paid", text: "일반 결제 성공" });
      }
    }
  } catch (error) {
    console.log("에러 발생", error);
    res.status(500).json(error);
  }
};
app.post("/payment-complete", verifyCommonPayment);
```

마지막으로 웹훅에서 데이터를 동기화합니다.

```javascript
const iamportWebhook = async (req, res) => {
  const { imp_uid, merchant_uid } = req.body;
  // ...
  const accessToken = await getAccessToken();
  const response = await axios.get(
    `https://api.iamport.kr/payments/${imp_uid}`,
    { headers: { Authorization: accessToken } }
  );
  const paymentData = response.data.response;
  const { merchant_uid, amount, status } = paymentData;
  const order = await Order.findOne({ merchant_uid });

  if (amount === order.amount) {
    console.log("결제 정보를 갱신합니다.");
    await Order.updateOne({ merchant_uid }, { $set: paymentData });
  } else {
    res.status(400).send("결제 금액 불일치");
  }
};
```

## 4. 정기 결제

### 클라이언트

일반 결제와는 달리, 정기 결제에서는 빌링키를 우선 발급받고 결제를 진행해야합니다. 빌링키는 "구독형 정기결제, 종량제 과금결제 등 원하는 시점에 재결제를 진행할 수 있는 결제용 암호화 키"로 빌링키와 1:1 대응을 하는 `customer_uid`를 통해 향후 정기 결제 예약을 진행합니다.

REST API, 일반결제창으로 구분하는데 저는 일반결제창을 이용했습니다.

```javascript
const startIamportPayment = useCallback(async () => {
  try {
    console.log("1. 우선 DB에 결제 정보를 저장합니다.");
    const merchant_uid = await axios.post("payment", {
      user_id: "사용자 고유 아이디",
      name: "상품 이름",
      amount: 10000,
    });
    console.log("생성 결과에는 merchant_uid 가 담겨있습니다.");
    await seasonPayment(merchant_uid.data);
    return;
  } catch (error) {
    console.log("서버에서의 결제 등록 에러", error);
    alert(error?.response?.data);
    return;
  }
}, []);
const seasonPayment = useCallback(async () => {
  const user_id = "사용자아이디로 빌링키와 1:1대응합니다";
  const parameter = {
    pg: "html5_inicis",
    pay_method: "card",
    merchant_uid: `billing_key_${merchantUid}`,
    customer_uid: user_id,
    name: "최초인증결제",
    amount: 0, // 빌링키 발급을 위해 0으로 진행합니다.
    buyer_email: user?.user_id,
    buyer_name: user?.name,
    buyer_tel: user?.tel,
    m_redirect_url: `${baseUrl}/payment/complete/${ticketInfo?._id}`,
  };
  window.IMP.request_pay(parameter, async (response) => {
    console.log("최초인증결제 결과: ", response);
    if (response.success) {
      const { imp_uid, merchant_uid } = response;
      console.log("빌링키 발급 후 결제를 진행합니다.");
      try {
        const response = await axios.post("/season-payment", {
          imp_uid,
          merchant_uid,
          user_id,
        });
      } catch (error) {
        console.log("서버에서의 결제 실패", error);
        alert(error?.response?.data);
      }
    } else {
      console.log("결제 실패: ", response.error_msg);
      alert(response.error_msg);
      return;
    }
  });
}, []);
```

### 백엔드

일반결제와 마찬가지로 결제 정보를 저장합니다.

```javascript
const createPayment = async (req, res) => {
  console.log("결제를 시작하기 전 DB에 저장합니다.");
  const { name, amount, user_id } = req.body;
  try {
    const merchant_uid = `merchant_${new Date.getTime()}`;
    await Order.create({
      name,
      amount,
      user_id,
      merchant_uid,
      order_date: new Date(),
    });
    res.send(merchant_uid);
  } catch (error) {
    res.status(500).send("저장 에러");
  }
};
app.post("/payment", createPayment);
```

다음으로 일반결제와 다르게 백엔드에서 아임포트에 결제를 요청합니다. 또한 결제와 동시에 다음 결제를 예약합니다.

```javascript
const orderAndSchedule = async (req, res) => {
  const { merchant_uid } = req.body;
  try {
    const accessToken = await getImportAccessToken();
    const order = await Order.findOne({ merchant_uid });
    if (!paymentFromDB) {
      console.log("데이터베이스에 결제 정보가 없습니다.");
      res.status(400).send("데이터베이스에 결제 정보가 없습니다.");
    } else {
      console.log("정기권 결제를 시작합니다.");
      const newOrder = await axios.post(
        "https://api.iamport.kr/subscribe/payments/again",
        {
          customer_uid: "빌링키 등록에 사용한 uid",
          merchant_uid,
          amount: order?.amount,
          name: order?.title,
          buyer_email: "사용자로부터 정보를 가져와서 기입",
          buyer_name: "사용자로부터 정보를 가져와서 기입",
          buyer_tel: "사용자로부터 정보를 가져와서 기입",
        },
        { headers: { Authorization: accessToken } }
      );
      const { code, message } = paymentResult.data;

      if (code === 0) {
        console.log("결제 성공. 다음 결제를 예약합니다.");
        const nextMerchantUid = `schedule_${new Date().getTime()}`;
        const schedule_at = "Unix Time Stamp 를 기입하셔야 합니다.";
        await axios.post(
          "https://api.iamport.kr/subscribe/payments/schedule",
          {
            customer_uid: "빌링키 등록에 사용한 uid",
            schedules: [
              {
                merchant_uid: nextMerchantUid,
                schedule_at,
                amount: order?.amount,
                name: order?.title,
                buyer_email: "사용자로부터 정보를 가져와서 기입",
                buyer_name: "사용자로부터 정보를 가져와서 기입",
                buyer_tel: "사용자로부터 정보를 가져와서 기입",
              },
            ],
          },
          { headers: { Authorization: accessToken } }
        );
        console.log("또한 새로운 merchant_uid 를 현재 결제에 저장합니다.");
        console.log(
          "다음 결제를 newMerchantUid 로 진행할 때 가격이나 이름 등을 가져오기 위해서입니다."
        );
        await Order.updateOne(
          { merchant_uid },
          { $set: { next_merchant_uid: nextMerchantUid } }
        );
      } else {
        console.log(
          "고객 카드 한도초과, 거래정지카드, 잔액부족 등으로 실패했습니다."
        );
        res
          .status(409)
          .send(
            "고객 카드 한도초과, 거래정지카드, 잔액부족 등으로 실패했습니다."
          );
      }
    }
  } catch (error) {
    console.log("정기 결제에 에러가 발생했습니다.", error);
    res.status(500).send("정기 결제에 에러가 발생했습니다.");
  }
};
```

첫 정기결제는 `merchant_uid` 를 `merchant_`으로 시작했기에, 웹훅에서 데이터를 갱신할 때는 일반 결제의 방식을 따릅니다. 하지만 `schedule_`로 시작하는 다음 결제는 갱신에 추가로 다음 결제를 예약하는 과정을 추가합니다.

```javascript
const iamportWebhook = async (req, res) => {
  const { imp_uid, merchant_uid } = req.body;
  if (merchant_uid.indexOf("billing_key_") >= 0) {
    console.log("빌링키 결제는 데이터베이스에 저장하지 않습니다.");
    res.send("billing");
  } else if (merchant_uid.indexOf("schedule_") >= 0) {
    console.log(
      "이전 예약을 아임포트에서 결제했습니다. 해당 정보를 데이터베이스에 생성합니다."
    );
    try {
      const accessToken = await getAccessToken();
      const response = await axios.get(
        `https://api.iamport.kr/payments/${imp_uid}`,
        { headers: { Authorization: accessToken } }
      );
      const paymentData = response.data.response;
      const { status, customer_uid } = paymentData;
      if (status === "paid") {
        console.log(
          "예약 결제 성공으로, 해당 데이터를 데이터베이스에 저장합니다."
        );
        console.log("결제에 대한 정보를 불러옵니다.");
        const prevOrder = await Order.findOne({
          next_merchant_uid: merchant_uid,
        }).lean();
        const nextMerchantUid = `schedule_${new Date().getTime()}`;
        await Order.create({
          name: prevOrder.name,
          merchant_uid,
          amount: prevOrder.amount,
          user_id: customer_uid,
          order_date: new Date(),
          next_merchant_uid: nextMerchantUid,
          payment_detail: paymentData,
        });
        console.log("이제 다음 결제를 예약합니다.");

        const schedule_at = "밀리초로 작성하셔야 합니다.";
        await axios.post(
          "https://api.iamport.kr/subscribe/payments/schedule",
          {
            customer_uid,
            schedules: [
              {
                merchant_uid: nextMerchantUid,
                schedule_at,
                amount: prevOrder.amount,
                name: prevOrder.name,
              },
            ],
          },
          { headers: { Authorization: accessToken } }
        );
      } else if (status === "cancelled") {
        console.log("결제를 환불했습니다.");
        await Order.updateOne(
          { merchant_uid },
          { $set: { payment_detail: paymentData } }
        );
        res.send("결제 환불");
      }
    } catch (error) {
      console.log("아임포트 정기 결제 갱신 에러",, error);
      res.send("아임포트 정기 결제 갱신 에러가 발생했습니다.");
    }
  }
};
```

### 예약 해지

서버에서 예약을 취소하는 방법을 작성해봅니다.

```javascript
const cancelSchedule = async () => {
  const { customer_uid, merchant_uid, next_merchant_uid } = req.body;
  try {
    const accessToken = await getImportAccessToken();
    const response = await axios.post(
      "https://api.iamport.kr/subscribe/payments/unschedule",
      { customer_uid, merchant_uid: [next_merchant_uid] },
      { headers: { Authorization: accessToken } }
    );
    const { code, message } = response.data;
    if (code === 0) {
      console.log(
        "해지 성공. 이전 예약에 해지한 예약에 대한 정보를 수정합니다."
      );
      await Order.updateOne(
        { merchant_uid },
        { $set: { next_merchant_uid: "" } }
      );
      res.send("해지 성공");
    } else {
      console.log("해지 실패", message);
      res.status(409).send(message);
    }
  } catch (error) {
    console.log("예약 해지에 에러가 발생했습니다.", error);
    res.send("예약 해지에 에러가 발생했습니다.");
  }
};
app.post("/cancel-schedule", cancelSchedule);
```

## 5. 모바일 콜백

공식 문서에 따르면 "KG이니시스, LG U+, NHN KCP, 나이스정보통신, JTNet 그리고 KICC는 PC 환경과는 다르게 각 PG사의 웹사이트로 리디렉션되어 결제가 진행됩니다. 따라서 IMP.request_pay(param, callback)를 호출해 결제시, 결제 프로세스 완료 후 callback 함수가 실행되지 않습니다." 라고 하며, 따라서 콜백을 실행할 클라이언트 페이지를 따로 생성해야합니다.

아임포트에서 콜백의 url 에 정보를 담는데 아래와 같습니다.

- 성공 : `URL주소?imp_uid=아임포트_번호&merchant_uid=가맹점_고유_주문번호&imp_success=true`
- 실패 : `URL주소?imp_uid=아임포트_번호&merchant_uid=가맹점_고유_주문번호&imp_success=false&error_code=에러_코드&error_msg=에러_메시`

여기에 저는 정기권인지 아닌지 여부를 params 에 담아 구분하도록 했습니다.

```javascript
import React, { useEffect, useCallback } from "react";
import axios from "axios";
import { useHistory, useLocation, useRouteMatch } from "react-router-dom";

function IamportCallback() {
  const match = useRouteMatch();
  const location = useLocation();
  const history = useHistory();

  useEffect(() => {
    const query = location.search.replace("?", "").split("&");
    const isSeason = match.params.season;
    const imp_uid = query[0].split("=")[1];
    const merchant_uid = query[1].split("=")[1];

    if (query.length === 3) {
      console.log("성공한 경우");
      if (isSeason === "true") {
        console.log("정기권");
        axios
          .post("/season-complete", { merchant_uid })
          .then(() => {
            alert("결제 성공");
            history.push("이전 페이지로 이동합니다.");
          })
          .catch((error) => {
            alert("걸제 실패");
            history.push("이전 페이지로 이동합니다.");
          });
      } else {
        axios
          .post("/payment/common-complete", { imp_uid, merchant_uid })
          .then(() => {
            alert("결제 성공");
            history.push("이전 페이지로 이동합니다.");
          })
          .catch((error) => {
            alert("걸제 실패");
            history.push("이전 페이지로 이동합니다.");
          });
      }
    } else {
      const message = query[4].split("=")[1];
      console.log("실패", message);
      alert(message);
    }
  }, []);

  return <></>;
}

export default IamportCallback;
```

## 6. 그 외 참고사항

정기결제를 위한 Unix Time Stamp 값을 계산할 때 [moment-timezone](https://www.npmjs.com/package/moment-timezone)으로 날짜를 구해서 계산했습니다. 그 이유는 AWS Beanstalk 에 배포했을 때 Date 객체가 한국 시간(UTC+09:00) 대신 UTC+0:00으로 시간을 계산하는 문제가 있었고, 또한 시간을 문자열로 가공하거나 계산하기가 간편해서입니다.

```javascript
import moment from "moment-timezone";

const getMilliSecondFromToday = () => {
  console.log("정기 결제의 밀리초를 구하기위해 사용한 방법입니다.");
  const duration = 30; //30일로 가정합니다.
  const today = moment().tz("Asia/Seoul");
  const target = today.add(duration, "day");

  return target.unix();
};
```

타임존에 대해 더 자세한 설명은 [자바스크립트에서 타임존 다루기 (1)](https://meetup.toast.com/posts/125), [자바스크립트에서 타임존 다루기 (2)](https://meetup.toast.com/posts/130)에서 읽어보실 수 있습니다.
