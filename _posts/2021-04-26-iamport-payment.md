---
title: "아임포트 결제 기능 도입기"
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
description: 아임포트 결제 기능을 개발해온 과정을 설명합니다.
tags:
  - javascript
  - express
---

현재 AWS S3(객체 스토리지 서비스)에 음원 및 비디오 파일을 업로드한 후 Elemental MediaConvert(비디오 트랜스코딩 서비스), Elastic Transcoder(음원 미디어 트랜스코딩 서비스)으로 인코딩해서 매장에 스트리밍하는 판뮤직이란 서비스를 외주받아 개발하고 있습니다. 저는 미디어 변환 대신 아임포트 결제 기능을 제작하고 있는데, 개발해온 과정을 정리하고자 글을 작성합니다.
주로 [아임포트 docs](https://docs.iamport.kr/)를 참고해서 진행했고, 클라이언트는 create-react-app, 백엔드는 express 를 이용해 개발하고 있습니다.

## 1. 들어가기에 앞서

우선 아임포트 관리자에서 테스트 모드 설정을 해야 합니다. [테스트모드 설정하기](https://docs.iamport.kr/admin/test-mode)를 참고하셔서 작업하시기 바랍니다. 참고로 일반 결제와 정기 결제의 PG 설정이 다르다는 점에 유의하셔야 합니다.
글의 진행 방식은 크게 기본적인 설정, 일반 결제, 정기 결제, 환불으로 진행하겠습니다.

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

## 3. 일반 결제

### 클라이언트

일반 결제를 시작합니다. 일반 결제를 위해선 결제 방법을 미리 설정해둬야 진행할 수 있습니다. 카드, 휴대폰, 계좌이체 등 사용하고자 하는 결제 방법으로 진행합니다.

```javascript
const onStartCommon = useCallback(() => {
  const param = {
    pg: "html5_inicis",
    pay_method: 'card',
    merchant_uid: 'merchant_123',
    name: '결제할 상품 이름',
    amount: 10000,
    buyer_email: '이메일',
    buyer_name: '이름',
    buyer_tel: '01012345678',
    digital: false,
    m_redirect_url: '모바일용 리다이렉트 url'
  };
  window.IMP.request_pay(param, async (response) => {
        if (response.success) {
          const { imp_uid, merchant_uid } = response;
          try {
            await axios.post('/payment', { imp_uid, merchant_uid, user_id });
            return;
          } catch (error) {
            console.log('등록 에러')
            return;
          }
            .then(() => {
              console.log('결제 성공')
            }, [])
            .catch((error) => {
              console.log('DB 등록 에러:', error?.response?.data);
              return openModal({ body: error?.response?.data });
            });
        } else {
          console.log('결제 실패: ', response.error_msg);
          return openModal({ body: response.error_msg });
        }
      });
}, []);
```

이번에는 백엔드 영역입니다.

```javascript
const onStartCommonPyment = async () => {};
```

## 4. 정기 결제

## 5. 환불 과정
