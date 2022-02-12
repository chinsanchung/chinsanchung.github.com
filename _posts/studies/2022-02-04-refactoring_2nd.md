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

{:start='2'}

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

{:start='3'}

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

{:start='4'}

4. 그리고 play 매개변수를 삭제합니다.

{:start='5'}

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

{:start='2'}

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

{:start='3'}

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

{:start='4'}

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

### 1.6 계산 단계와 포맷팅 단계 분리하기

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

{:start='2'}

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

{:start='3'}

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

{:start='4'}

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

{:start='5'}

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

1. 공연료 계산기 만들기

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

{:start='2'}

2. 함수들을 계산기로 옮기기

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

{:start='3'}

3. 공연료 계산기를 다형성 버전으로 만들기

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

## 2. 리팩터링 원칙

### 2.1 리팩터링 정의

저자는 리팩터링을 "소프트웨어의 겉보기 동작은 그대로 유지한 채, 코드를 이해하고 수정하기 쉽도록 내부 구조를 변경하는 기법", 그리고 "소프트웨어의 겉보기 동작은 그대로 유지한 채, 여러 가지 리팩터링 기법을 적용해서 소프트웨어를 재구성하는 것"으로 명사와 동사형으로 정의를 구분하고 있습니다.

### 2.2 두 개의 모자

개발의 목적을 "기능 추가" 또는 "리팩터링"으로 명확히 구분하라는 언급하고 있습니다. 비록 짧은 작업일지라도 구분하는 것이 개발의 흐름을 끊지 않아 능률적으로 작업할 수 있다는 생각이 듭니다.

### 2.3 리팩터링하는 이유

1. 소프트웨어 셜계가 좋아집니다. 코드의 양이 줄어들수록 코드를 수정하거나 설계를 파악하기 쉬워집니다.
2. 소프트웨어를 이해하기 쉬워집니다. 동작하는 데에만 신경을 쓰게 된다면, 앞으로의 유지 보수가 어려워질 가능성이 높습니다.
3. 버그를 쉽게 찾을 수 있습니다.
4. 프로그래밍 속도를 높일 수 있습니다. 꾸준한 리팩터링으로 내부 설계를 잘 유지한다면 새로운 기능을 추가할 지점 또는 수정할 지점을 쉽게 찾을 수 있습니다. 저자의 말대로 처음부터 완벽한 설계는 불가능하지만, 리팩터링을 통해 언제든 개선할 수 있습니다.

### 2.4 언제 리팩터링해야 할까?

1. 준비를 위한 리팩터링: 기능을 쉽게 추가하기

> 리팩터링하기 가장 좋은 시점은 코드베이스에 기능을 새로 추가하기 직전이다. 이 시점에 현재 코드를 살펴보면서, 구조를 살짝 바꾸면 다른 작업을 하기가 훨씬 쉬워질 만한 부분을 찾는다. - p86

{:start='2'}

2. 이해를 위한 리팩터링: 코드를 이해하기 쉽게 만들기

변수의 이름을 바꾸거나 긴 함수를 나누는 작업으로, 코드를 정리하여 설계를 더 깊이 이해할 수 있도록 할 수 있습니다.

{:start='3'}

3. 쓰레기 줍기 리팩터링

쓸데없이 복잡하거나 중복인 코드를 간단하게나마 수정하고, 시간이 걸리는 일을 메모를 남긴 후에 일을 마치고 나면 처리하는 방식을 뜻합니다.

{:start='4'}

4. 계획된 리팩터링과 수시로 하는 리팩터링

저자는 계획을 세우는 방법 대신, 기능을 추가하거나 버그를 잡는 동안에 리팩토링을 함께 하는 방법을 선호한다고 합니다. 저 자신도 되도록 계획된 리팩토링을 할 일을 줄이고, 기회가 될 때마다 하는 것이 더 좋다고 생각합니다. 다만 리팩터링 커밋, 기능 추가 커밋을 분리하는 방식은 맥락을 생각했을 때 그리 선호하지 않는다고 언급하고 있습니다.

{:start='5'}

5. 오래 걸리는 리팩터링

팀 전체가 오랫동안 투자해야 할 대규모 리팩터링일지라도, 몇 주에 걸쳐 조금씩 해결하는 것이 좋습니다. 리팩터링의 특징인 "겉보기 동작을 그대로 유지"를 이용하는 것입니다.

{:start='6'}

6. 코드 리뷰에 리팩터링 활용하기

코드 리뷰를 통해 서로 다른 사람의 관점에서 코드를 바라보는 과정을 거치면서, 아이디어를 얻거나 지식을 전파할 수 있습니다. 여기서 리팩터링을 통해 코드 리뷰의 결과를 더 구체적으로 도출할 수 있습니다.

{:start='7'}

7. 리팩터링하지 말아야 할 때

지저분한 코드일지라도 굳이 수정할 필요가 없거나, 아예 처음부터 새로 작성하는 것이 더 쉬울 때에는 리팩토링을 하지 않는 편이라고 합니다. 다만, 이것은 많은 경험을 통해 판단할 수 있는 것이라고 명시하고 있습니다.

### 2.5 리팩터링 시 고려할 문제

1. 새 기능의 개발 속도 저하

상황에 따라 리팩터링을 할지 말지를 조율해야 합니다. 그리고 리팩터링의 본질은 코드베이스를 꾸미는 것이 아니라 "경제적인 이유"로 하는 것으로, 기능의 추가 시간과 버그 수정 시간을 줄이는 것이 목적입니다.

{:start='2'}

2. 코드 소유권

지금까지 혼자 개발하고 리팩터링하는 일이 많아 몰랐는데, 코드의 소유자가 다른 팀일 때의 리팩터링은 고려한 적이 없었습니다. 저자는 이러한 제약 사항을 기존의 함수를 유지한 채 **함수 이름 바꾸기**를 적용하고, 새로운 함수를 호출하도록 수정하는 형식으로 클라이언트에 영향을 주지 않으려 노력하는 편이라고 언급합니다.

{:start='3'}

3. 브랜치

각 기능 단위로 브랜치를 분리하여 팀원과 함께 작업하는 방식은, 독립 브랜치로 작업하는 기간이 길어질수록 마스터로 통합하기 어려워진다는 단점이 있습니다.
따라서 통합 주기를 2~3일 단위로 짧게 괸리하는 지속적 통합(Continuous Integration) 방식을 권장합니다. 최소한 하루에 한 번 마스터와 통합하여 머지의 복잡도를 낮춥니다.

지금까지 혼자서 리팩토링을 수행했기 때문에 이런 문제가 생길 가능성이 있다는 것을 처음으로 인식하였습니다. 확실히 풀타임으로 개발을 한다면, 기능별 브랜치로 리팩터링을 하는 것은 자칫하면 꼬일 수 있기에 부담스러울 것입니다. 저자의 말대로 CI 를 수행하되, 만약 어렵다면 되도록 통합 주기만큼은 빨리 잡는 것이 좋다고 생각합니다.

{:start='4'}

4. 테스팅

리팩터링 과정에서의 오류를 빨리 잡기 위해서 테스트 스위트(test suite)가 필요합니다. 즉 리팩터링을 위해선 자가 테스트 코드(스스로 성공과 실패를 판단하는 테스트)를 갖춰야 합니다.
또한, 자가 테스트 코드는 통합 과정에서 발생하는 의미 충돌을 잡는 매커니즘으로 사용하여, 지속적 배포(Continuous Delievery)를 수행할 수 있게 해줍니다.

{:start='5'}

5. 레거시 코드

레거시 코드를 리팩터링하려면 테스트 코드의 도움이 필요합니다. 확실해 다른 사람이 오래 전에 작성한 레거시 코드를 테스트 없이 리팩터링하는 것은 막막할 것이라 생각합니다. 저자의 말대로 한 번에 해결하려 하지 말고 눈에 띄는 부분, 서로 관련된 부분부터 조금씩 개선해가는 것이 그나마 유일한 해결책이라 느낍니다.

{:start='6'}

6. 데이터베이스

저자가 언급하는 데이터베이스 리팩터링은 "커다란 변경들을 쉽게 조합하고 다룰 수 있는 데이터 마이그레이션 스크립트를 작성하고, 접근 코드와 데이터베이스 스키마에 대한 구조적 변경을 이 스크립트로 처리하게끔 통합하는" 것입니다. 또한 전체 변경 과정을 작고 독립적인 단계로 쪼개야 합니다.

예시로, 이전의 코드를 유지한 채로 새로운 필드를 추가하여 커밋을 한 후에 새로운 필드를 적용하면서 테스트 및 오류 수정을 실시합니다. 교체를 완료하면 이전의 필드를 삭제합니다.

### 2.6 리팩터링, 아키텍처, 애그니(YAGNI)

처음부터 이용자의 요구사항을 이해하기 어렵기에, 첫 코드를 완벽하게 작성하는 것은 불가능한 일입니다. 하지만 리팩터링을 통해 아키텍처를 수정하면 요구사항의 변화에 대처할 수 있습니다.

여기서 나오는 것이 간결한 설계 점진적 설계, 애그니(YAGNI)라고 부르는 방식입니다.

- 우선 현재까지의 요구사항만을 해결하도록 작성합니다.
- 진행하면서 요구사항을 깊이 이해하게 되면 아키텍처도 그에 맞게 리팩터링합니다. 소프트웨어의 복잡도에 지장을 주지 않는 메커니즘은 부담없이 넣지만, 복잡도를 높일 수 있는 유연성 메커니즘은 검증을 거친 후에 추가합니다.

### 2.7 리팩터링과 소프트웨어 개발 프로세스

리팩터링의 첫 번째 토대는 자가 테스트 코드입니다. 또한 지속적 통합으로 팀원 각자의 작업을 방해하지 않으면서 수행한 리팩토링을 빠르게 공유할 수 있어야 합니다.

자가 테스트 코드, 지속적 통합, 리팩터링을 적용하여 YAGNI 설계를 수행한다면 소프트웨어의 신뢰성을 높이고 언제든 릴리스할 수 있는 상태를 유지하도록 해줍니다.

### 2.8 리팩터링과 성능

> 리팩터링은 성능 좋은 소프트웨어를 만드는 데 기여한다. 단기적으로 보면 리팩터링 단계에서는 성능이 느려질 수도 있다. 하지만 최적화 단계에서 코드를 튜닝하기 훨씬 쉬워지기 때문에 결국 더 빠른 소프트웨어를 얻게 된다. - p.106

## 3. 코드에서 나는 악취

이번 장에서는 어떠한 상황일 때 리팩터링을 수행해야 하는지를 정리하고 있습니다.

### 3.1 기이한 이름

> 함수, 모듈, 변수, 클래스 등은 그 이름만 보고도 각각이 무슨 일을 하고 어떻게 사용해야 하는지 명확히 알 수 있도록 엄청나게 신경 써서 이름을 지어야 한다. - p.114

이런 상황에서는 **함수 선언 바꾸기**, **변수 이름 바꾸기**, **필드 이름 바꾸기**처럼 이름을 바꾸는 리팩터링을 적용합니다.

### 3.2 중복 코드

예시로 두 함수(메소드)가 같은 표현식을 쓴다면 **함수 추출하기**를, 비슷하지만 완전히 같지는 않다면 **문장 슬라이드하기**로 비슷한 부분을 모아 **함수 추출하기**를 쉽게 할 수 있는지를 파악합니다. 같은 부모로부터 파생된 서브 클래스의 코드가 중복이면 **메소드 올리기**로 부모로 옮깁니다.

### 3.3 긴 함수

> 간접 호출의 효과, 즉 코드를 이해하고, 공유하고, 선택하기 쉬워진다는 장점은 함수를 짧게 구성할 때 나오는 것이다. - p.115

함수의 이름에 동작 방식이 아닌 '의도'(목적)을 드러내야 하는 점은 계속 의식하고 있습니다. 다만 "'무엇을 하는지'를 코드가 잘 설명해주지 못할수록 함수로 만드는 게 유리하다"는 부분은 아직 잘 와닿지가 않습니다.

---

**함수 추출하기**로 함수의 길이를 줄이고, 만약 매개변수의 수가 많아 추출 작업에 방해가 된다면 **매개변수 객체 만들기**와 **객체 통째로 넘기기**로 매개변수의 수를 줄입니다. 또한 **임시 변수를 질의 함수로 바꾸기**로 임시 변수의 수를 줄일 수 있습니다. 위의 방법으로도 매개변수와 임시 변수가 많다면 **함수를 명령으로 바꾸기**를 고려합니다.

**조건문 분해하기**, switch 문의 경우 각 케이스마다 **함수 추출하기**를 적용합니다. 같은 조건에 대한 switch 문이 여러 개일 경우 **조건문을 다형성으로 바꾸기**를 적용합니다.
반복문의 경우, 코드를 추출해 독립된 함수로 만들고, 만약 여러 작업이 겹쳐 있다면 **반복문 쪼개기**를 적용하여 작업을 분리합니다.

### 3.4 긴 매개변수 목록

다른 매개변수에서 값을 얻어올 수 있는 매개변수는 **매개변수를 질의 함수로 바꾸기**로 제거합니다. 데이터 구조에서 일부를 추출하여 매개변수로 전달한다면 **객체 통째로 넘기기**를 이용해 원본을 그대로 전달합니다. 항상 함께 전달되는 매개변수들은 **매개변수 객체 만들기**로 하나로 묶습니다. 함수의 동작 방식을 정하는 플래그 역할의 매개변수(예시: 불린 값으로 조건문을 적용하여 동작을 구분하는 경우)는 **플래그 인수 제거하기**로 없앱니다.(플래그 인수 대신 코드를 분리하여 새로운 함수를 작성합니다.)

여러 개의 함수가 특정 매개변수의 값을 공통으로 사용하면 **여러 함수를 클래스로 묶기**를 통해 공통 값들을 클래스의 필드로 정의합니다.

### 3.5 전역 데이터

되도록 전역 데이터는 **변수 캡슐화하기**를 통해 함수로 감싸야 합니다. 또한 접근자 함수들을 클래스나 모듈에 넣고, 그 안에서만 사용할 수 있도록 접근 범위를 최소로 줄이는 것도 좋습니다.

### 3.6 가변 데이터

함수형 프로그래밍에서는 데이터를 변경할 경우 원본은 그대로 두고, 복사본을 만들어서 변경을 수행하는 것을 기본 개념으로 하고 있습니다. 함수형 언어가 아니라면 아래의 방식들로 위험을 줄일 수 있습니다.

- **변수 캡슐화하기**로 정해진 함수를 거쳐야만 값을 수정할 수 있도록 합니다. 값을 감시하거나 코드를 개선하기 쉬워집니다.
- 하나의 변수에 용도가 다른 값들을 저장하기 위해 값을 갱신한다면 **변수 쪼개기**로 각 용도별로 독립 변수에 저장합니다.
- **문장 슬라이드하기**, **함수 추출하기**로 갱신하는 코드로부터 부작용이 없는 코드를 분리합니다.
- API를 만들 때는 **질의 함수와 변경 함수 분리하기**로 꼭 필요한 경우가 아니라면 부작용이 있는 코드를 호출할 수 없게 합니다.
- **세터 제거하기**로 변수의 유효범위를 줄입니다.
- **파생 변수를 질의 함수로 바꾸기**를 적용합니다.
- **여러 함수를 클래스로 묶기**, **여러 함수를 변환 함수로 묶기**로 변수를 갱신하는 유효범위를 제한합니다.
- 내부 필드에 데이터를 담고 있는 변수라면 **참조를 값으로 바꾸기**로 구조체를 통째로 교체하도록 합니다.

### 3.7 뒤엉킨 변경

> 뒤엉킨 변경은 단일 책임 원칙(Single Responsibility Principle)이 제대로 지켜지지 않을 때 나타난다. 즉, 하나의 모듈이 서로 다른 이유들로 인해 여러 가지 방식으로 변경되는 일이 많을 때 발생한다. - p.119

서로 다른 맥락의 로직을 순차적으로 실행하는 경우, **단계 쪼개기**로 분리합니다. 처리 과정에서 각기 다른 맥락의 함수를 호출하는 빈도가 높다면, **함수 옮기기**로 처리 과정을 맥락별로 구분합니다. 모듈이 클래스일 경우 **클래스 추출하기**를 사용합니다.

### 3.8 산탄총 수술

"코드를 변경할 때마다 자잘하게 수정해야 하는 클래스가 많을 때" 발생하는 문제입니다.

함께 변경되는 대상들을 **함수 옮기기**와 **필드 옮기기**로 한 모듈로 묶습니다. 비슷한 데이터를 다루는 함수가 많으면 **여러 함수를 클래스로 묶기**를 적용합니다. 데이터 구조를 변환 또는 보강하는 함수들은 **여러 함수를 변환 함수로 묶기**를 적용합니다. 그리고 **단계 쪼개기**로 묶은 함수들의 출력 결과를 묶어 다음 단계 로직으로 전달합니다.

어설프게 분리된 로직은 **함수 인라인하기**, **클래스 인라인하기**로 하나로 합칩니다.

### 3.9 기능 편애

> 기능 편애는 흔히 어떤 함수가 자기가 속한 모듈의 함수나 데이터보다 다른 모듈의 함수나 데이터와 상호작용할 일이 더 많을 때 풍기는 냄새다. - p.121

**함수 옮기기**로 해당 함수를 상호작용하는 대상으로 옮깁니다. 만약 일부분만 편애할 경우, 그 부분만 독립 함수로 추출한 다음 함수 옮기기를 수행합니다. 만약 어디로 옮겨야 할 지 명확하지 않다면, 가장 많은 데이터를 포함한 모듈로 옮깁니다.

### 3.10 데이터 뭉치

데이터 뭉치는 일부 값을 삭제했을 때 나머지 데이터만으로는 의미가 없는 것을 뜻합니다. 이것을 **클래스 추출하기**로 하나의 객체로 만들고, **매개변수 객체 만들기**나 **객체 통째로 넘기기**를 적용하여 매개변수의 수를 줄입니다.

### 3.11 기본형 집착

저자의 말은 정수나 부동소수점 수 등 다양한 기본형을, 단순히 문자열 집합처럼 다른 방식으로 표현하는 것을 "기본형 집착"이라 부르는 것 같습니다. **기본형을 객체로 바꾸기**를 적용하고, 기본형 코드가 조건부 동작을 제어할 경우 **타입 코드를 서브클래스로 바꾸기**와 **조건부 로직을 다형성으로 바꾸기**를 차례로 적용합니다. 또한 기본형 그룹은 데이터 뭉치처럼 **클래스 추출하기**, **매개변수 객체 만들기**를 적용합니다.

### 3.12 반복되는 switch 문

switch 문에 다형성을 적용하기 시작하면서, 이제는 switch 문을 무조건적으로 검토 대상이라 보는 대신 동일한 조건부 로직이 여러 곳에서 반복되는 코드를 주시하는 것이 좋다고 언급하고 있습니다.

### 3.13 반복문

**반복문을 파이프라인으로 바꾸기**를 적용합니다. 필터, 맵 등의 파이프라인 연산으로 원소의 처리 과정을 쉽게 파악할 수 있습니다.

### 3.14 성의 없는 요소

함수(메소드), 클래스, 인터페이스 등 코드의 구조를 잡을 때 활용되는 요소를 **함수 인라인하기**, **클래스 인라인하기**로 제거합니다. 상속을 사용했으면 **계층 합치기**를 적용합니다.

### 3.15 추측성 일반화

나중에 필요할지도 모른다는 생각으로 미래를 대비하여 작성한 코드는, 실제로 사용하지 않을 거라면 제거하는 것이 좋습니다. 추상 클래스라면 **계층 합치기**를, 의미없이 위임하는 코드는 **함수 인라인하기** 또는 **클래스 인라인하기**로 삭제합니다. 본문에서 사용하지 않는 매개변수는 **함수 선언 바꾸기**로 없앱니다.

### 3.16 임시 필드

코드를 이해하기 어렵게 만드는 임시 필드는 **클래스 추출하기**, **함수 옮기기**로 임시 필드와 그에 관련된 코드를 새 클래스로 옮깁니다. 임시 필드의 유효성을 검증하는 조건부 로직이 있다면, **특이 케이스 추가하기**로 유효하지 않을 때를 위한 대안 클래스를 만들고 그 로직을 제거합니다.

### 3.17 메시지 체인

메시지 체인(다른 객체를 요청하는 작업이 연쇄적으로 이어지는 코드)는 한 부분을 수정하려면 다른 영역도 수정해야 한다는 문제가 있습니다.

```js
// 메시지 체인의 예시
managerName = aPerson.department.manager.name;
```

이를 서버 자체에 위임 메소드를 만들어 위임 객체의 존재를 숨기는 **위임 숨기기**로 해결합니다.

```js
// 위임 숨기기 예시
managerName = aPerson.department.managerName; // manager 객체를 숨김
managerName = aPerson.manager.name; // department 객체를 숨김
managerName = aPerson.managerName; // manager, department 객체 모두를 숨김
```

### 3.18 중개자

캡슐화를 위한 위임을 지나치게 사용할 경우, **중개자 제거하기**로 위임 대신 직접 소통하도록 만듭니다. 위임 메소드를 제거한 후 남는 일이 거의 없다면 호출하는 쪽으로 인라인합니다.(**함수 인라인하기**)

### 3.19 내부자 거래

모듈 사이의 데이터 거래를 최소로 줄이고 투명하게 처리할 필요가 있습니다. **함수 옮기기**, **필드 옮기기**로 모듈들을 떼어놓아 사적으로 처리하는 부분을 줄입니다.
관심사를 공유한다면 새로운 모듈을 만들거나 **위임 숨기기**로 다른 모듈이 중간자 역할을 하도록 합니다.

### 3.20 거대한 클래스

> 한 클래스가 너무 많은 일을 하려다 보면 필드 수가 상당히 늘어난다. 그리고 클래스에 필드가 너무 많으면 중복 코드가 생기기 쉽다. - p.128

**클래스 추출하기**로 서로 연괸성이 있는 필드들을 묶어줍니다. 또는 **슈퍼클래스 추출하기**, **타입 코드를 서브클래스로 바꾸기**를 적용합니다.

### 3.21 서로 다른 인터페이스의 대안 클래스들

특정 클래스를 다른 클래스로 교체하기 위해선 두 클래스의 인터페이스가 같아야 합니다. **함수 선언 바꾸기**나 **함수 옮기기**로 인터페이스를 동일하게 만들어줍니다. 중복이 생긴다면 **슈퍼클래스 추출하기**를 고려합니다.

### 3.22 데이터 클래스

데이터 필드와 게터/세터 메소드로만 구성된 데이터 클래스에 대하여, public 필드를 **레코드 캡슐화하기**로 제거하고 **세터 제거하기**로 데이터에 대한 접근을 막습니다.

만약 다른 클래스에서 데이터 클래스의 게터/세터를 사용한다면 **함수 옮기기** 또는 **함수 추출하기**로 해당 클래스의 일부를 데이터 클래스로 옮길 수 있는지 확인합니다.

### 3.23 상속 포기

만약 서브클래스가 부모의 동작은 필요로 하는 대신 인터페이스를 포기하고 싶어한다면, **서브클래스를 위임으로 바꾸기**나 **슈퍼클래스를 위임으로 바꾸기**로 상속 메커니즘에서 벗어나도록 할 수 있습니다.

### 3.24 주석

> 주석을 남겨야겠다는 생각이 들면, 가장 먼저 주석이 필요 없는 코드로 리팩터링해본다. - p.131

무엇을 해야 하는지 모르는 상황, 현재의 진행 상황, 불확실한 부분, 작성 이유에 대한 설명이라면 주석으로 달아두는 것이 좋습니다.
