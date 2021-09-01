---
title: '머신러닝 공부 - 군집 알고리즘의 개요과 K-평균'
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
description: 비지도 학습 중 하나인 군집 알고리즘과 K-평균에 대해 공부한 내용을 정리합니다.
tags:
  - machine_learning
---

비지도 학습이란 무엇인지 간략하게 개요를 잡고, 그리고 비지도 학습에 사용하는 알고리즘인 K-평균에 대해서 정리합니다.

## 비지도 학습이란

머신러닝 알고리즘 중에서 타깃에 대한 정보가 없을 때 데이터를 학습하여 예측을 수행하는 알고리즘입니다. 오직 입력 데이터로만 예측을 도출하는 것이 지도 학습과의 차이점입니다.

### 비지도 학습의 종류

비지도 학습은 크게 비지도 변환(unsupervised transformation)과 군집(clustering)으로 구분합니다.

**비지도 변환**은 데이터를 새롭게 표현하여 사람이나 다른 머신러닝 알고리즘이 원래 데이터보다 쉽게 해석할 수 있도록 만드는 알고리즘입니다. 고차원 데이터의 특성을 줄여(이 방법을 "차원 축소"라고 부릅니다.) 꼭 필요한 특징을 가진 데이터만 표현하도록 하거나, 또는 데이터를 구성하는 단위나 성분을 찾는 데 활용합니다.

**군집**은 간단히 말해 데이터를 비슷한 것끼리 그룹으로 묶는 것입니다. 여러 각도에서 찍은 사진에서 패턴을 찾아 비슷한 샘플끼리 묶어 같은 과일 또는 사람이라고 분류하는 것을 예시로 들 수 있습니다.

## K-평균

### K-평균 알고리즘이란?

K-평균은 간단하고 널리 사용하는 군집 알고리즘입니다. 작동 방식은 다음과 같습니다.

1. 무작위로 클러스터 중심은 데이터의 어떤 영역을 대표하는 클러스터 중심을 k 개 찾습니다.
2. 각 샘플에서 가장 가까운 클러스터 중심을 찾아 해당 클러스터의 샘플로 지정합니다.
3. 클러스터에 속한 샘플의 평균값으로 클러스터 중심을 변경합니다.
4. 클러스터 중심에 변화가 없을 때까지 2, 3번 과정을 반복합니다.

여기서 K 를 몇 개로 지정할 것인지 결정하는 방법은 시각화를 해서 확인하기, 해당 데이터에 대한 도메인 지식으로 판별하기 등이 있습니다.

### 최적의 클러스터 개수(K)를 찾기

K-평균의 단점 중 하나는 클러스터의 개수를 사전에 지정해야 한다는 것인데, 실제 세계의 데이터는 수많은 속성과 대용량인 경우가 많아 위에 언급한 시각화, 지식으로 확인하기 어려울 때가 많습니다. 이럴 떄는 적절한 K 값을 찾는 도구를 사용해야 합니다.

1. Elbow

클러스터의 개수를 X 축으로, 클러스터 중심과 클러스터에 속한 샘플 사이의 거리를 제곱해서 합한 *이너셔*를 Y 축으로 하는 선 그래프를 그리면, 그래프가 꺾이는 지점이 있는데, 거기를 **엘보**라 부르며 그것을 클러스터의 개수 k 로 지정하는 방법입니다.

![elbow](https://www.scikit-yb.org/en/latest/_images/elbow-1.png)
[이미지 출처](https://www.scikit-yb.org/en/latest/api/cluster/elbow.html)

사이킷런의 KMeans 클래스는 자동으로 이너셔를 계산해서 `inertia_`속성으로 제공하고 있어서 직접 확인하실 수 있습니다.

2. 실루엣 계수

Elbow 보다 계산 비용이 많이 들지만 더 정확합니다. a 는 동일한 클러스터의 다른 샘플까지의 평균 거리(=클러스터 내부의 평균 거리), b 는 가장 가까운 클러스터까지의 평균 거리(=가장 가까운 클러스터의 샘플까지의 평균 거리. 여기서 샘플과 가장 가까운 클러스터는 자신이 속한 클러스터를 제외하고 b 가 최소인 클러스터입니다.)를 뜻합니다.

실루엣 점수는 -1 ~ 1 사이의 값으로, 1에 가까울수록 자신의 클러스터 안에 잘 속해 있고 다른 클러스터와는 멀리 떨어져 있습니다.

$$\text{silhouette} = (b - a) / max(a, b)$$

사이킷런의 `silhouette_score()`함수로 실루엣 계수를 구할 수 있습니다.

```python
from sklearn.metrics import silhouette_score
silhouette_score(X, kmeans.labels_)
```

### 특성 스케일링

실제 세계의 데이터에서는 각 특성마다 다른 범위(scale)을 가지는 경우가 많습니다. 따라서 별 다른 처리 없이 데이터를 이용해 군집 알고리즘을 수행하면 아래와 같은 상황이 나올 수 있습니다.

![스케일링 전의 데이터](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-08-29-Clustering_raw_data.PNG?raw=true)

데이터를 변환하는 방법은 크게 두 가지가 있습니다.

- 정규화 또는 최대-최소 스케일링은 데이터의 값을 0 ~ 1 사이로 변환합니다.
- 표준화 또는 Z-Score 스케일링은 데이터의 값을 평균이 0, 표준편차가 1이 되도록 변환합니다.

사이킷런의 `MinMaxScaler`, `StandardScaler`으로 데이터 스케일링을 할 수 있습니다.

### K-평균의 한계

K-평균은 속도가 빠르고 확장이 용이하지만, 최적이 아닌 솔수션을 피하기 위해 여러 번 실행해야 하며, 클러스터의 개수를 미리 지정해야 합니다. 또한 클러스터의 크기가 큰 경우, 밀집도가 서로 다른 경우, 또는 클러스터의 형태가 원형이 아닌 경우 잘 작동하지 않습니다.

![K-평균의 한계](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-08-29-Clustering_k_means_cons.PNG?raw=true)

### 사이킷런에서 K-평균 수행하기

[사이킷런 공식 문서](https://scikit-learn.org/stable/modules/generated/sklearn.cluster.KMeans.html)에서 더 자세한 내용을 확인하실 수 있습니다.

```python
from sklearn.cluster import KMeans

model = KMeans(n_clusters=3)
preds = model.fit_predict(data)
```

## 참고

[Intro to Machine Learning with TensorFlow](https://www.udacity.com/course/intro-to-machine-learning-with-tensorflow-nanodegree--nd230)

[혼자 공부하는 머신러닝+딥러닝 - 6-2 K-평균](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=257932080)

[파이썬 라이브러리를 활용한 머신러닝(번역개정판) - 3장 비지도 학습과 데이터 전처리](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=186846299)

[핸즈온 머신러닝 - 1부 9장 비지도 학습](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=237677114)
