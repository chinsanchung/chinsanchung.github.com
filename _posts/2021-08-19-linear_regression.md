---
title: '머신러닝 공부 - 선형 회귀와 다항 회귀'
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
description: 선형 회귀와 다항 회귀에 대해 공부한 내용을 정리합니다.
tags:
  - machine_learning
---

대표적인 회귀 알고리즘인 **선형 회귀**, 그리고 **다항 회귀**에 대해 그동안 읽어왔던 교재를 기반으로 정리하고자 합니다. 언어는 파이썬, 주요 라이브러리는 numpy, pandas, scikit-learn 을 사용합니다.

## 선형 회귀란?

**선형 회귀**는 널리 사용되는 대표적인 회귀 알고리즘으로, 비교적 간단하고 성능이 뛰어나기 때문에 맨 처음 배우는 머신러닝 알고리즘 중 하나입니다.[^linear01]

구하고자 하는 직선의 방정식은 아래와 같습니다.

$$y = ax + b$$

여기서 a 는 기울기(=계수 coefficient, 가중치 weight), b 는 절편(=y_intercept, 편향 offset)이라고 합니다. 학습을 통해 최적의 기울기와 절편을 찾는 것이 선형 회귀 모델의 목적입니다.

아래 코드는 scikit-learn 을 이용한 선형 회귀 모델의 예시입니다.

```python
# 데이터셋을 훈련용과 테스트용으로 나눠 훈련과 평가에 사용합니다.
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

model = LinearRegression()
model.fit(X_train, y_train)

print('학습으로 도출한 최적의 기울기 a: ', model.coef_, ' 절편 b: ', model.intercept_)
print('훈련 세트의 점수', model.score(X_train, y_train))
print('테스트 세트의 점수', model.score(X_test, y_test))
```

선형 회귀는 예측과 훈련 세트에 있는 타깃 y 사이의 **평균제곱오차**(Mean Squared Error)를 최소화하는 파라미터 w 와 b 를 찾습니다. 평균제곱오차는 예측값과 타깃 값의 차이를 제곱하여 더한 후에 샘플의 개수로 나눈 것입니다.[^linear02]

- 평균제곱오차: 평균절대오차(Mean Absolute Error) 와 함께 회귀의 손실 함수 중 하나입니다. 이 값을 최소화하는 모델이 좋은 모델이라 할 수 있습니다. 다음은 평균제곱오차의 공식입니다.
- `model.score()` 로 구하는 값은 $R^2$ 로

$$Error = \frac{1}{2m}\displaystyle\sum_{i=1}^m (y - \hat{y})^2$$

## 다항 회귀

원본 특성의 다항식을 추가하는 방식입니다. 특성 x 가 주어지면 x ** 2, x ** 3, x \*\* 4 등을 시도해볼 수 있습니다.[^polynomial01] 다음은 2개의 특성을 만들어 2차방정식으로 만든 예시입니다.

$$y = a_{1}x^{1} + a_{2}x^{2} + b$$

```python
X = np.array([[2],[3],[4],[5],[6]])
target = np.array([5.9, 32.0, 40.0, 51.5, 70.0])
# 선형 모델은 자동으로 절편을 포함하므로 include_bias 값을 False 로 합니다..
poly = PolynomialFeatures(degree=2, include_bias=False)
# 새로 만들 특성의 조합을 찾습니다.
poly.fit(X)
# 실제로 데이터를 변환합니다.
poly.fit(X)
X_poly = poly.transform(X)
# 새로 만든 특성의 이름을 확인할 수 있는 메소드입니다.
print(poly.get_feature_names() # ['x0', 'x0^2']

model = LinearRegression()
model.fit(X_poly, target)
```

다항식 특성은 1차원 데이터셋에서도 매우 부드러운 곡선을 만듭니다. 그러나 고차원 다항식은 데이터가 부족한 영역에서 너무 민감하게 동작합니다.[^polynomial02] 그리고 너무 많은 특성의 다항식이면 과대 적합이 발생합니다.

- 참고: `PolynomialFeatures`는 scikit-learn 에서 제공하는 변환기입니다. `PolynomialFeatures` 클래스는 기본적으로 각 특성을 제곱한 항을 추가하고 특성끼리 서로 곱한 항을 추가합니다.[^polynomial03]

## 다중 회귀

현실의 데이터는 위처럼 하나의 특성을 사용하지 않고 여러 개의 특성을 사용하는데, 이러한 회귀를 **다중 회귀**라고 부릅니다. 다중 회귀에서는 특성이 늘어난만큼 구해야 할 가중치가 늘어납니다. 다음은 n 개의 특성이 있다고 가정할 때의 다중 회귀 방정식입니다.

$$\hat{y} = a_{1}x_{1} + a_{2}x_{2} + ... + a_{n}x_{n} + b$$

## 릿지와 라쏘 회귀

선형 회귀 모델에 규제를 추가한 모델입니다. 규제는 머신러닝 모델이 훈련 세트를 너무 과도하게 학습하지 못하도록 훼방하는 것을 말합니다. 선형 회귀 모델의 경우 특성에 곱해지는 계수(또는 기울기)의 크기를 작게 만드는 일입니다.[^regularization01]

릿지와 라쏘 두 모델은 규제를 하는 방법이 다르며, 일반적으로 릿지를 더 선호합니다.

### 릿지 회귀

릿지 회귀에서는 L2 규제를 사용합니다. 선형 회귀에 비해 규제가 있기 때문에 훈련 세트에서의 성능은 낮아지지만, 그만큼 과대적합이 줄어듭니다.

평균제곱오차 식에 $\alpha\displaystyle\sum_{i=1}^{n}w_{i}^{2}$ 항이 추가됩니다. $\alpha$ 를 크게 하면 패널티의 효과가 커지고(가중치 감소), $\alpha$ 를 작게 하면 그 반대가 됩니다.[^ridge01] (${w_i}$ 가 가중치입니다.)

```python
ridge = Ridge(alpha=5).fit(X, y)
print(ridge.score(X_train, y_train)))
```

### 라쏘 회귀

릿지 회귀에서는 L1 규제를 사용합니다. 평균제곱오차 식에 $\alpha\displaystyle\sum_{i=1}^{n} \lvert w_{i}\rvert$ 항을 추가합니다.

L1 규제는 L2 규제와 달리 유용하지 않은 특성의 가중치를 0으로 만들어 모델에서 완전히 제외하기도 합니다. 여기서도 $\alpha$ 파라미터를 조절해서 규제의 정도를 조절할 수 있습니다.

```python
# 반복 횟수를 충분히 지정하지 않으면 경고를 띄웁니다.
model = Lasso(alpha=5, max_iter=10000)
```

## 선형 회귀의 장단점

선형 모델은 학습 속도가 빠르고 예측도 빠릅니다. 매우 큰 데이터셋과 희소한 데이터셋에도 잘 작동합니다. 샘플에 비해 특성이 많을 때 잘 작동합니다. 선형 모델의 또 하나의 장점은 회귀와 분류에서 본 공식을 사용해 예측이 어떻게 만들어지는지 비교적 쉽게 이해할 수 있다는 것입니다. 하지만 계수의 값들이 왜 그런지 명확하지 않을 때가 종종 있습니다. 특히 데이터셋의 특성들이 서로 깊게 연관되어 있을 때 그렇습니다.[^proscons01]

## 주석

[^linear01] 혼자 공부하는 머신러닝+딥러닝. 3.2 선형 회귀. 135쪽
[^linear01] 파이썬 라이브러리를 활용한 머신러닝(번역개정판). 2. 지도 학습. 76쪽
[^polynomial01] 파이썬 라이브러리를 활용한 머신러닝(번역개정판). 4. 데이터 표현과 특성 공학. 293쪽
[^polynomial02] 파이썬 라이브러리를 활용한 머신러닝(번역개정판). 4. 데이터 표현과 특성 공학. 295쪽
[[^polynomial03] 혼자 공부하는 머신러닝+딥러닝. 3.3 특성 공학과 규제. 155쪽
[^regularization01] 혼자 공부하는 머신러닝+딥러닝. 3.3 특성 공학과 규제. 159쪽
[^ridge01] 파이썬 라이브러리를 활용한 머신러닝(번역개정판). 2. 지도 학습. 78쪽
[^proscons01] 파이썬 라이브러리를 활용한 머신러닝(번역개정판). 2. 지도 학습. 99쪽
