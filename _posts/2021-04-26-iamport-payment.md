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

## 2. 클라이언트에서의 아임포트 cdn 설정

클라이언트에서는 우선 라이브러리를 가져온 후, 그것으로 아임포트를 실행하는 버튼을 만들어야 합니다. 그리고 아임포트에서 결제를 하면서 서버 함수를 연결시킵니다.

### 1. cdn 가져오기 및 시작 버튼

처음에는 index.html 의 script 에 라이브러리를 가져왔지만, 지금은 버튼이 있는 컴포넌트가 마운트됐을 때 라이브러리를 실행하도록 변경했습니다. 그래서 처음부터 전부 불러오지 않고, 필요할 때만 라이브러리를 가져온다는 장점이 있습니다.

```javascript
// import 과정은 생략합니다.
function IamportLibrary() {
  // 라이브러리 가져오기
  useEffect(() => {
    const loadSdk = () => {
      const jqueryPromise = new Promise((res, rej) => {
        const library = document.querySelector('#jquery-sdk');
        if (library) {
          res();
        } else {
          const jquery = document.createElement('script');
          jquery.id = 'jquery-sdk';
          jquery.src = '//code.jquery.com/jquery-3.6.0.min.js';
          jquery.onload = res;
          document.body.appendChild(jquery);
        }
      });
      const iamport = new Promise((res, rej) => {
        const library = document.querySelector('#iamport-sdk');
        if (library) {
          res();
        } else {
          const iamport = document.createElement('script');
          iamport.id = 'iamport-sdk';
          iamport.src = '//cdn.iamport.kr/js/iamport.payment-1.1.8.js';
          iamport.onload = res;
          document.body.appendChild(iamport);
        }
      });
      return Promise.all([jqueryPromise, iamport]);
    };
    const unloadSdk = () => {
      const jqueryPromise = new Promise((res, rej) => {
        const library = document.querySelector('#jquery-sdk');
        if (library) {
          library.parentNode.removeChild(library);
        } else {
          res();
        }
      });
      const iamport = new Promise((res, rej) => {
        const library = document.querySelector('#iamport-sdk');
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

### 2. 일반 결제, 정기 결제 요청

[아임포트 docs](https://docs.iamport.kr/)에 따르면, 일반 결제와 정기 결제를 구분해서 진행해야 합니다. 편의를 위해 한 함수에 전부 넣지 않고 기능별로 나눠서 작성했습니다.

우선 일반 결제부터입니다. 일반 결제는 결제 수단을 선택한 후에 결제 창을 호출해야 합니다. 따라서 수단을 지정할 버튼 등을 추가하시면 됩니다.

```javascript
function IamportLibrary() {
  // ... 라이브러리 임포트
  const onClickIamportButton = useCallback(async () => {
    // 우선 아임포트 화면을 출력합니다.
    window.IMP.init('아임포트 키')
    // merchant_uid(주문번호)는 아임포트에서 검증을 위해 활용합니다.
    const merchantUid = 'merchant_' + new Date().getTime();
    await startIamportPay(merchantUid);
  }, []);
  const startIamportPay = useCallback(async() => {
    // 일반결제인지 아닌지를 구분해서 별도로 진행합니다.
    if (item.type === 'normal') {
      await normalPay({ merchantUid, item });
    } else {
      await subscribePay({ merchantUid, item });
    }
  }, []);
  const normalPay = useCallback(async({ merchantUid, item }) => {
    const {name, amount} = item
    const requestBody = {
      pg: 'html5_inicis',
      pay_method: payMethod,
      merchant_uid: merchantUid,
      name,
      amount,
      buyer_name: '사용자 이름',
      buyer_email: '이메일',
      buyer_tel: '전화번호',
      // 결제할 상품이 콘텐츠인지 여부. 휴대폰 소액결제 시 필수
      digital: false,
      m_redirect_url: '모바일용 콜백'
    }
    window.IMP.request_pay(requestBody, async (response) => {
      if (response.success) {
        const { imp_uid, merchant_uid } = response;
        await axios.post('/payment/normal-pay', {
          imp_uid, 
          merchant_uid, 
          userId: '사용자 아이디',
          name, 
          amount,
        }).then(() => alert('결제를 처리했습니다.'))
        .catch(error => {
          console.log('서버에서의 결제 오류', error)l
          alert('일반 결제 실패')
        })
      } else 
    })
  }, [payMethod]);
}
```

한 가지 주목할 점은 `m_redirect_url`입니다. 모바일 화면에서는 `request_pay` 의 콜백 함수를 실행하지 않는다고 합니다. 따라서 콜백을 실행하는 별도의 클라이언트 페이지를 작성하셔야 합니다. 콜백 페이지는 아래에서 설명드리겠습니다.

다음으로 정기 결제 함수입니다. 정기 결제는 우선 빌링키를 발급받고, 그 후에 빌링키와 1:1로 대응하는 customer_uid 를 통해 서버에서 결제를 진행합니다. 현재 이니시스로 테스트하고 있어서 일반결제창을 활용했습니다. 카드로만 결제하며, 빌링키는 0원으로 입력합니다. 

```javascript
function IamportLibrary() {
  // ... 라이브러리 임포트
  // ... 아임포트 버튼 실행
  // ... 일반결제
  const subscribePay = useCallback(async({ merchantUid, item }) => {
    const {name, amount} = item
    const requestBody = {
      pg: 'html5_inicis',
      pay_method: 'card',
      merchant_uid: `billing_${merchantUid}`,
      customer_uid: '사용자의 고유한 키(예: ObjectId',
      name,
      amount: 0,
      buyer_name: '사용자 이름',
      buyer_email: '이메일',
      buyer_tel: '전화번호'
      digital: false,
      m_redirect_url: '모바일용 콜백'
    }
    window.IMP.request_pay(requestBody, async (response) => {

    })
  }, [payMethod]);
```

빌링키의 `merchant_uid`를 일반 결제와 다르게 작성한 이유는 일반 결제, 정기 결제, 빌링키를 구분하는 게 서버에서 작업할 때 도움이 되기 때문입니다.

### 3. 모바일 콜백 페이지
## 3. 백엔드에서 결제 서비스

### 1. 기본적인 설정

### 2. 일반 결제

### 3. 정기 결제

## 4. 설명을 마치며