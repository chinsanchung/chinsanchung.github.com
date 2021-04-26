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

그리고 결제입니다. [아임포트 docs](https://docs.iamport.kr/)

### 2. 일반 결제, 정기 결제 요청

## 3. 백엔드에서 결제 서비스

### 1. 기본적인 설정

### 2. 일반 결제

### 3. 정기 결제

## 4. 설명을 마치며