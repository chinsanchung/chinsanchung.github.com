---
title: "(책) 리팩터링 2판"
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - study
toc: true
toc_sticky: true
toc_labe: 목차
description: 마틴 파울러의 『리팩터링 2판』을 읽으며 내용을 정리합니다.
excerpt: 마틴 파울러의 『리팩터링 2판』을 읽으며 내용을 정리합니다.
tags:
  - book
---

마틴 파울러의 『리팩터링 2판』에 대해 정리하는 목적의 글입니다.

## 1. 리팩터링 - 첫 번째 예시

### 1.2 예시 프로그램을 본 소감

> 프로그램이 새운 기능을 추가하기에 편한 구조가 아니라면, 먼저 기능을 추가하기 쉬운 형태로 리팩터링하고 나서 원하는 기능을 추가한다. - p.27

처음 작성할 때 그 당시의 상황만 고려하여 함수를 작성하는 방식은 이후에 새로운 요구사항이 발생할 경우 전체를 수정할 가능성이 있어 비효율적입니다.

### 1.3 리팩터링의 첫 단계

> 리팩터링의 첫 단계는 항상 똑같다. 리팩터링할 코드 영역을 꼼꼼하게 검사해줄 테스트 코드들부터 마련해야 한다. - p.28

저자는 여러 가지 사례를 미리 작성한 후, 성공과 실패를 스스로 판단하는 자가진단 테스트로 만들어 테스트를 수행한다고 합니다. 또한 리팩터링을 테스트에 상당히 의지하여 수행하고, 테스트를 통해 버그의 가능성을 줄인다고 언급합니다. 이 점은 저도 공감하는 부분인데, 이전에 실패의 조건을 놓친 채로 성급하게 커밋을 올려 후회한 경험이 있기 때문입니다. 지금은 테스트 케이스를 먼저 작성한 후에 초기 코드를 리팩터링하는 식으로 기능을 구현하는데, 이전보다 확실히 실수가 줄었음을 채감하고 있습니다.

### 1.4 statement() 함수 쪼개기

> statement()처럼 긴 함수를 리팩터링할 때는 먼저 전체 동작을 각각의 부분으로 나눌 수 있는 지점을 찾는다. - p.29

우선 switch 문을 하나의 함수로 추출합니다. 내부에서 사용하는 변수 중에서 값이 바뀌지 않는 것을 매개변수로, 값이 바뀌는 것은 내부의 지역 변수로 선언합니다. 이렇게 코드의 조각을 함수를 추출하는 과정을 저자는 **함수 추출하기**라는 이름을 붙이고 있습니다.

조금식 변경하고 매번 테스트하는 것이 리팩터링의 핵심이며 한 번에 너무 많이 수정하려다 실수를 저지르면 디버깅이 어려워진다는 글을 읽고, 저 자신은 후자의 방식으로 진행하고 있음을 깨달았습니다. 작은 기능을 리팩터링할 때마다 테스트를 수행하는 습관을 들여야겠습니다.

추출한 함수 `amountFor`의 변수, 인수의 이름을 수정합니다. 반환 값을 result 라 하는 부분은 괜찮은 방법이라 느꼈습니다. 매개변수 이름에 접두어로 타입 이름을 적는 방식은 아직은 잘 이해가 가질 않습니다. 여기서는 `perf`를 `aPerformance`로 바꿨는데, 매개변수의 역할이 뚜렷하지 않아 부정 관사(a/an)을 붙인 것이라 합니다.

```javascript
function statement(invoice, plays) {
  function amountFor(aPerformance, play) {
    let result = 0;
    switch (play.type) {
      default:
      // ...
    }
    return result;
  }
}
```

---

이번에는 임시 변수 `play`를 제거합니다. 이러한 임시 변수를 최대한 제거하지 않으면, 로컬 범위에 존재하는 이름이 늘어나 추출 작업이 복잡해진다고 합니다. 이러한 리팩터링을 **임시 변수를 질의 함수로 바꾸기**라고 부릅니다.

1. 질의 함수 만들기

```javascript
function statement(invoice, plays) {
  function playFor(aPerformance) {
    return plays[aPerformance.playID];
  }
}
```

statement 함수에서의 `play` 변수의 값을 질의 함수로 불러오도록 수정합니다.

```javascript
function statement(invoice, plays) {
  // ...
  for (let perf of invoice.performances) {
    const play = playFor(perf);
  }
  // ...
}
```

2. **변수 인라인하기**

```javascript
function statement(invoice, plays) {
  // ...
  for (let perf of invoice.performances) {
    let thisAmount = amountFor(perf, playFor(perf));
    // ...
    totalAmount += thisAmount;
  }
  // ...
}
```

3. **함수 선언 바꾸기**

```javascript
function statement(invoice, plays) {
  function amountFor(aPerformance, play) {
    let result = 0;
    // ...
    switch (playFor(aPerformance).type) {
      // ...
      default: {
        throw new Error(playFor(aPerformance).type);
      }
    }
    return result;
  }
}
```

4. 그리고 play 매개변수를 삭제합니다.

5. `amountFor()`를 받는 thisAmount 를 제거하고 **변수 인라인하기**를 적용합니다.

```javascript
function statement(invoice, plays) {
  let result = `청구 내역 (고객명: ${invoice.custormer})\n`;
  // ...
  for (let perf of invoice.performances) {
    // let thisAmount = amountFor(perf);
    // ...
    result = `${playFor(perf).name}: ${format(amountFor(perf) / 100)} (${
      perf.audience
    }석)\n`;
    totalAmount += amountFor(perf);
  }
}
```

저자는 유효 범위를 신경 써야 할 대상이 줄어들어 추출 작업이 쉬워진다는 장점이 있다고 언급했습니다. 저는 지금까지 지역 변수를 자주 사용했는데, 이러한 방법을 통해 추출 작업이 쉬워진다면 앞으로 이 방법으로 리팩터링을 적용해야겠습니다.

---

이번에는 적립 포인트를 계산하는 `volumeCredits`의 연산을 추출합니다.

```javascript
function statement(invoice, plays) {
  function volumeCreditsFor(perf) {
    let volumeCredits = 0;
    // ... 적립 포인트 계산에 대한 모든 연산을 추출합니다.
    return volumeCredits;
  }
}
```

```javascript
function statement(invoice, plays) {
  // ...
  for (let perf of invoice.performances) {
    volumeCredits += volumeCreditsFor(perf);
    // ...
  }
}
```

테스트, 커밋 후에 변수의 이름을 수정합니다.

```javascript
function statement(invoice, plays) {
  function volumeCreditsFor(aPerformance) {
    let result = 0;
    // ...
    return result;
  }
}
```

---

최상위 함수 statement() 에서 임시 변수 중 하나인 format 변수를 제거합니다. 작가는 직접 선언하여 사용하는 형식을 선호한다고 합니다.

> 임시 변수는 자신이 속한 루틴에서만 의미가 있어서 루틴이 길고 복잡해지기 쉽다. - p.42

format 기능을 함수로 직접 선언해 사용하도록 바꿉니다.

```javascript
// 이전
function statement(invoice, plays) {
  // ...
  const format = new Intl.NumberFormat("en-US", {
    // ...
  }).format(aNumber);
  // ...
  result += `${playFor(perf).name}: ${format(amountFor(perf) / 100)}`;
  // ...
}
```

처음에는 이름을 format 으로 동일하게 했지만, 함수의 역할을 더 잘 설명하는 이름으로 변경합니다. **함수 선언 바꾸기**

```javascript
function statement(invoice, plays) {
  function usd(aNumber) {
    return new Intl.NumberFormat("en-US", {
      // ...
    }).format(aNumber / 100);
    // 여러 차례 등장하는 단위 변환 로직도 여기로 옮깁니다.
  }
}
```

---

volumeCredits 변수를 제거합니다.

이 변수는 반복문을 한 바퀴 돌 때마다 값을 누적하기에, **반복문 쪼개기**로 우선 누적되는 부분을 따로 뺍니다.

1. 반복문 쪼개기

```javascript
function statement(invoice, plays) {
  // ...
  let volumeCredits = 0;

  for (let perf of invoice.performances) {
    // ...
  }
  // volumeCredits 를 별도의 반복문으로 분리
  for (let perf of invoice.performances) {
    volumeCredits += volumeCreditsFor(perf);
  }
  // ...
}
```

2. 문장 슬라이드하기

이어서 **문장 슬라이드**로 volumeCredits 변수의 선언을 반복문 바로 앞으로 옮깁니다. 저는 이전의 변수 선언은 최상위로 배치하고, 연산을 그 아래에 작성했는데 지금의 방식이 흐름을 이해하기에 더 수월하다고 생각합니다.

```javascript
function statement(invoice, plays) {
  // ...

  let volumeCredits = 0;
  for (let perf of invoice.performances) {
    volumeCredits += volumeCreditsFor(perf);
  }
  // ...
}
```

3. 함수 추출하기

이렇게 volumeCredits 와 관련된 문장을 모으면 임시 변수를 질의 함수로 바꾸는 작업이 수월해진다고 합니다. **함수로 추출**하는 작업을 합니다.

```javascript
function statement(invoice, plays) {
  function totalVolumeCredits() {
    let volumeCredits = 0;
    for (let perf of invoice.performances) {
      volumeCredits += volumeCreditsFor(perf);
    }
    return volumeCredits;
  }
  // ...
}
```

4. 변수 인라인하기

마지막으로 임시 변수 volumeCredits 를 **인라인**합니다. 변수를 선언하지 않고, 활용하는 부분에 직접 호출하여 활용하는 것입니다.

```javascript
function statement(invoice, plays) {
  // ...
  result += `적립 포인트: ${totalVolumeCredits()}`;
}
```

반복문을 쪼개어 두 번 사용했지만, 이 정도의 중복은 연산 속도에 영향을 주지 않는 경우가 많다고 합니다. 따라서 특별한 경우가 아니라면 일단 무시하고 리팩터링을 하는 것이 좋다고 말하고 있습니다.

이번에는 4개의 과정으로 나누어 리팩터링, 테스트, 커밋을 진행했습니다. 코드가 복잡할수록 단계를 작게 나누면 작업 속도가 빨라집니다.

---

임시 변수 totalAmount 도 volumeCredits 와 같은 방식으로 제거합니다.

```javascript
function statement(invoice, plays) {
  // 임시 이름
  function appleSauce() {
    let totalAmount = 0;
    for (let perf of invoice.performances) {
      totalAmount += amountFor(perf);
    }
    return totalAmount;
  }
  let totalAmount = appleSauce();
  // ...
}
```

그리고 함수의 이름, 추출한 함수 안의 이름을 수정합니다. (`totalAmount`, `totalVolumeCredits` 함수의 임시 변수를 result 로 수정)

### 1.5 중간 점검: 난무하는 중첩 함수

```js
function statement(invoice, plays) {
  let result = "청구 내역";
  for (let perf of invoice.performances) {
    result += `${playFor(perf).name}: ${usd(amountFor(perf))} (${
      perf.audience
    })\n`;
  }
  result += `총액: ${usd(totalAmount())}\n`;
  result += `적립 포인트: ${totalVolumeCredits()}\n`;
  return result;

  function totalAmount() {}
  // 중첩 함수 시작
  function totalVolumeCredits() {}
  function usd(aNumber) {}
  function volumeCreditsFor(aPerformance) {}
  function playFor(aPerformance) {}
  function amountFor(aPerformance) {}
}
```

### 1.7 계산 단계와 포맷팅 단계 분리하기

이제 statement() 함수의 HTML 버전을 만들어봅니다. 문제는 최상단 코드만 HTML 로 변환하면 되는데, 분리한 계산 함수가 중첩 함수로 들어 있습니다.

저자는 **단계 쪼개기**를 이용하고 있습니다. statement() 함수에 필요한 데이터를 처리하기, 그리고 처리한 결과를 텍스트나 HTML 로 표현하는 것입니다.

1. 두 번째 단계가 될 코드를 **함수 추출하기**로 뽑아냅니다.

```js
function statement(invoice, plays) {
  return renderPlainText(invoice, plays);
}

function renderPlainText(invoice, plays) {
  let result = "청구 내역";
  for (let perf of invoice.performances) {
    result += `${playFor(perf).name}: ${usd(amountFor(perf))} (${
      perf.audience
    })\n`;
  }
  result += `총액: ${usd(totalAmount())}\n`;
  result += `적립 포인트: ${totalVolumeCredits()}\n`;
  return result;

  function totalAmount() {}
  // 중첩 함수 시작
  function totalVolumeCredits() {}
  function usd(aNumber) {}
  function volumeCreditsFor(aPerformance) {}
  function playFor(aPerformance) {}
  function amountFor(aPerformance) {}
}
```

2. 두 단계 사이의 중간 데이터 구조 역할을 하는 객체를 만들어 renderPlainText() 에 인수로 전달합니다.

```js
function statement(invoice, plays) {
  const statementData = {};
  return renderPlainText(statementData, invoice, plays);
}

function renderPlainText(data, invoice, plays) {
  // ...
}
```

renderPlainText() 의 invoice 와 plays 로 전달되는 데이터를 data 로 옮깁니다. 계산 관련 코드는 statement(), data 매개변수로 전달된 데이터를 처리하는 것은 renderPlainText() 가 맡도록 만들 수 있습니다.

```js
function statement(invoice, plays) {
  const statementData = {};
  statementData.customer = invoice.customer;
  return renderPlainText(statementData, invoice, plays);
}

function renderPlainText(data, invoice, plays) {
  let result = `고객명: ${data.customer}`;
  // ...
}
```

공연 정보도 중간 데이터 구조(`data`)로 옮기면 renderPlainText() 의 invoice 매개변수를 삭제할 수 있습니다.

그리고 연극 제목도 중간 데이터 구조에서 가져옵니다.

```js
function statement(invoice, plays) {
  const statementData = {};
  statementData.customer = invoice.customer;
  statementData.performances = invoice.performances.map(enrichPerformance);
  return renderPlainText(statementData, plays);

  function enrichPerformance(aPerformance) {
    const result = Object.assign({}, aPerformance); // 얕은 복사
    return result;
  }
}
```

`Object.assign`으로 공연 객체를 복사한 이유는 함수로 건넨 데이터 `aPerformance`를 수정하지 않기 위해서입니다. 저자는 데이터를 최대한 불변으로 취급한다고 합니다.

3. 실제 데이터를 담아봅니다.

**함수 옮기기**를 이용해서 renderPlainText() 의 중첩 함수였던 playFor() 함수를 statement() 로 옮깁니다.

```js
function statement(invoice, plays) {
  // ...
  function enrichPerformance(aPerformance) {
    const result = Object.assign({}, aPerformance); // 얕은 복사
    result.play = playFor(result);
    return result;
  }

  function playFor(aPerformance) {}
}
```

renderPlainText() 안에서 playFor 를 호출하던 부분을 중간 데이터 `data`를 사용하도록 바꿉니다.

```js
function renderPlainText(data, plays) {
  // ...
  for (let perf of data.performances) {
    result += `${perf.play.name} ...`;
  }

  function volumeCreditsFor(aPerformance) {
    // ...
    if ("comedy" === aPerformance.play.type) {
      // ...
    }
  }

  function amountFor(aPerformance) {
    // ...
    switch (aPerformance.play.type) {
      // ...
      default:
        throw new Error(aPerformance.play.type);
    }
  }
}
```

가격(amount) amountFor(), 적립 포인트(volumeCredits) volumeCreditsFor(), 총합(totalAmount, totalVolumeCredits) totalAmount()와 totalVolumeCredits() 도 비슷한 방법으로 옮깁니다.

```js
function statement(invoice, plays) {
  const statementData = {};
  statementData.customer = invoice.customer;
  statementData.performances = invoice.performances.map(enrichPerformance);
  statementData.totalAmount = totalAmount(statementData);
  statementData.totalVolumeCredits = totalVolumeCredits(statementData);
  return renderPlainText(statementData, plays);

  function enrichPerformance(aPerformance) {
    const result = Object.assign({}, aPerformance);
    result.play = playFor(result);
    result.amount = amountFor(result);
    result.volumeCredits = volumeCreditsFor(result);
    return result;
  }
  // ...
}
```

**반복문을 파이프라인으로 바꾸기**를 적용합니다.

```js
function renderPlainText(data, plays) {
  // ...
  function totalAmount(data) {
    return data.performances.reduce((total, p) => total + p.amount, 0);
  }
  function totalVolumeCredits(data) {
    return data.performances.reduce((total, p) => total + p.volumeCredits, 0);
  }
}
```

4. 첫 단계인 "statment()에 필요한 데이터 처리"에 해당하는 코드를 모두 별도 함수로 뺍니다.

```js
function statement(invoice, plays) {
  return renderPlainText(createStatementData(invoice, plays));
}

function createStatementData(invoice, plays) {
  const statementData = {};
  // ...
  return statementData;
}
```

이제 createStatementData()를 별도의 파일로 빼고, createStatementData()에서 반환할 변수 이름을 result 로 변경합니다.

```js
export default function createStatementData(invoice, plays) {
  const result = {};
  // ...
  return result;

  function enrichPerformance(aPerformance) {}
  function playFor(aPerformance) {}
  function amountFor(aPerformance) {}
  function volumeCreditsFor(aPerformance) {}
  function totalAmount(data) {}
  function totalVolumeCredits(data) {}
}
```

5. 마지막으로 HTML 버전을 작성합니다.

```js
import createStatementData from "./createStatementData";

function statement(invoice, plays) {
  return renderPlainText(createStatementData(invoice, plays));
}

function renderPlainText(data, plays) {}

function htmlStatement(invoice, plays) {
  return renderHtml(createStatementData(invoice, plays));
}

function renderHtml(data) {}

// renderHtml()에서 사용하도록 usd()를 최상위로 옮깁니다.
function usd(aNumber) {}
```

코드량은 늘어났지만, 그 덕분에 전체 로직의 요소를 뚜렷하게 구분하고 계산과 출력을 분리할 수 있었습니다. 무조건 간결하게만 압축하는 것을 생각했는데, 저자가 말한대로 간결함보다는 명료함을 더 중시해아 한다는 것을 위의 코드를 통해 이해하게 되었습니다.

> 모듈화하면 각 부분이 하는 일과 그 부분들이 맞물려 돌아가는 과정을 파악하기 쉬워진다. - p.63

### 1.8 다형성으로 계산 코드 재구성하기

연극 장르를 추가하고 장르마다 공연료와 적립 포인트 계산을 다르게 지정합니다.

amountFor()의 조건부 로직(연극 장르에 따라 계산 방식이 다름)은 코드를 수정할수록 대응하기 어려워집니다. 저자는 객체지향의 핵심 특성인 다형성을 활용합니다.

목표는 상속 계층을 구성해서 각 서브클래스가 각자의 구체적인 계산 로직을 정의하는 것입니다. 여기서 핵심적으로 사용하는 리팩터링 기법이 **조건부 로직을 다형성으로 바꾸기**입니다.

---

상속 계층부터 정의합니다. (공연료와 적립 포인트 계산 함수를 담을 클래스가 필요합니다.)

#### 1. 공연료 계산기 만들기

enrichPerformance() 함수의 amountFor(), volumeCreditsFor()를 전용 클래스 `PerformanceCalculator`로 옮깁니다.

우선 연극 레코드부터 시작합니다. 클래스의 생성자에 **함수 선언 바꾸기**를 적용하여 공연할 연극을 계산기로 전달합니다.

```js
export default function createStatementData(invoice, plays) {
  function enrichPerformance(aPerformance) {
    const calculator = new PerformanceCalculator(
      aPerformance,
      playFor(aPerformance)
    );
    const result = Object.assign({}, aPerformance);
    result.play = calculator.play;
    result.amount = amountFor(result);
    result.volumeCredits = volumeCreditsFor(result);
    return result;
  }
}

class PerformanceCalculator {
  constructor(aPerformance, aPlay) {
    this.performance = aPerformance;
    this.play = aPlay;
  }
}
```

#### 2. 함수들을 계산기로 옮기기

**함수 옮기기** 리팩터링을 단계별로 진행합니다. 우선 공연료 계산 코드를 클래스 안으로 복사합니다.

```js
class PerformanceCalculator {
  constructor(aPerformance, aPlay) {
    this.performance = aPerformance;
    this.play = aPlay;
  }

  // amountFor() 함수를 복사한 것입니다.
  get amount() {
    // aPerformance.audience -> this.performance 로 수정합니다.
    // aPerformance.play -> this.play 로 수정합니다.
  }
}
```

컴파일-테스트-커밋 이후, 원본 함수의 작업을 위임합니다.

```js
function amountFor(aPerformance) {
  return new PerformanceCalculator(aPerformance, playFor(aPerformance)).amount;
}
```

그리고 원래의 **함수를 인라인**하여 새 함수를 직접 호출하도록 합니다. 적립 포인트를 계산하는 함수도 같은 방법으로 옮깁니다.

```js
export default function createStatementData(invoice, plays) {
  function enrichPerformance(aPerformance) {
    const calculator = new PerformanceCalculator(
      aPerformance,
      playFor(aPerformance)
    );
    const result = Object.assign({}, aPerformance);
    result.play = calculator.play;
    result.amount = calculator.amount;
    result.volumeCredits = calculator.volumeCredits;
    // ...
  }
}

class PerformanceCalculator {
  constructor(aPerformance, aPlay) {
    this.performance = aPerformance;
    this.play = aPlay;
  }

  // volumeCreditsFor() 함수를 복사한 것입니다.
  get volumeCredits() {
    // aPerformance -> this.performance 로 수정합니다.
  }
}
```

#### 3. 공연료 계산기를 다형성 버전으로 만들기

타입 코드 대신 서브클래스로 사용하도록 변경합니다. (**타입 코드를 서브클래스로 바꾸기**) PerformanceCalculator 의 서브클래스를 사용하게 만들고, 서브클래스를 사용하기 위해 생성자 함수 대신 함수를 호출하도록 바꿔야 합니다. (자바스크립트는 생정자가 서브클래스의 인스턴스를 반환할 수 없다고 합니다. 그래서 **생성자를 팩토리 함수로 바꾸기**를 적용합니다.)

```js
export default function createStatementData(invoice, plays) {
  function enrichPerformance(aPerformance) {
    const calculator = createPerformanceCalculator(
      aPerformance,
      playFor(aPerformance)
    );
    // ...
  }
}

function createPerformanceCalculator(aPerformance, aPlay) {
  return new PerformanceCalculator(aPerformance, aPlay);
}
```

함수 형식으로 하면 서브클래스를 선택해서 반환할 수 있습니다.

```js
function createPerformanceCalculator(aPerformance, aPlay) {
  switch (aPlay.type) {
    case "tragedy":
      return new TragedyCalculator(aPerformance, aPlay);
    case "comedy":
      return new ComedyCalculator(aPerformance, aPlay);
    default:
      throw new Error("알 수 없는 장르");
  }
}

class TragedyCalculator extends PerformanceCalculator {}
class ComedyCalculator extends PerformanceCalculator {}
```

**조건부 로직을 다형성으로 바꾸기**를 적용합니다. 슈퍼클래스 PerformanceCalculator 의 `get amount()`를 서브클래스로 옮기고, 슈퍼클래스의 함수는 에러 처리를 합니다.

```js
class PerformanceCalculator {
  // ...
  get amount() {
    throw new Error("subclass responsiblity");
  }
}

class TragedyCalculator extends PerformanceCalculator {
  get amount() {}
}
class ComedyCalculator extends PerformanceCalculator {
  get amount() {}
}
```

volumeCredits() 의 경우 일부 장르만 약간 다르고 대다수는 관객 수를 검사하는데, 일반적인 경우를 슈퍼클래스에 놓고 달라지는 부분은 오버라이드합니다.

```js
class PerformanceCalculator {
  // ...
  get volumeCredits() {
    return Math.max(this.performance.audience - 30, 0);
  }
}

class ComedyCalculator extends PerformanceCalculator {
  get volumeCredits() {
    return super.volumeCredits + Math.floor(this.performance.audience / 5);
  }
}
```

### 1.9 상태 점검: 다형성을 활용하여 데이터 생성하기

이전과의 차이점은 장르별 계산 코드를 함께 묶어뒀기 때문에, 만약 새로운 장르를 추가한다면 createPerformanceCalculator() 에 새로운 조건문을 추가하고 새로운 서브클래스를 작성하면 됩니다. 슈퍼클래스, 서브클래스, 그리고 팩토리 함수를 이용해 다형성을 구현하는 것을 보고, 조건부 로직을 구현하는 방법을 이렇게 구현할 수 있다는 것을 알았습니다.
