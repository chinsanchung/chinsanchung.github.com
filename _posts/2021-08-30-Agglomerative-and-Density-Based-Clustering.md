---
title: '머신러닝 공부 - 병합 군집과 밀도 군집'
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
description: 군집 알고리즘인 병합 군집과 밀도 군집을 정리합니다.
tags:
  - machine_learning
---

다양한 군집 알고리즘 중에서 이번에는 병합(Agglomerative) 군집, 밀도(Density) 군집 대해 정리했습니다.

## 병합 군집

병합 군집은 시작할 때 하나의 샘플과 인접한 샘플을 묶어 클러스터로 만들고, 이 과정은 지정한 개수만큼의 클러스터가 남을 때까지 계속 반복하며 클러스터를 합쳐나갑니다. 클러스터들의 묶음은 이진 트리와 같은 형태로, 계층 군집을 덴드로그램으로 그릴 수 있습니다.

![덴드로그램 예시](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-08-30-Agglomerative-and-Density_single_linkage.PNG?raw=true)

### linkage 종류

- ward 연결: 기본값입니다. 분산을 가장 작게 증가시키는 두 클러스터를 합쳐 비슷한 크기의 클러스터를 만듭니다. 대부분의 데이터셋에 알맞은 방법입니다. (다만 두 클러스터의 포인트 차이가 매우 크면 average 또는 complete 연결이 더 나을 수 있습니다.) ward 의 목적은 각 병합 단계에서 발생하는 분산을 최소화하는 것입니다.
- average 연결: 클러스터의 포인트 사이의 평균 거리가 가장 짧은 두 클러스터를 합칩니다.
- complete 연결: 클러스터 포인트 사이의 최대 거리가 가장 짧은 두 클러스터를 합칩니다.

### 사이킷런에서의 병합 군집

사이킷런의 `linkage`옵션에서 가장 비슷한 클러스터를 측정하는 방법으로 위에 언급한 ward 연결, average 연결, complete 연결을 지원합니다.

```python
from sklearn.cluster import AgglomerativeClustering
# ward
ward_clustering = AgglomerativeClustering(n_clusters=3)
ward_pred = ward_clustering.fit_predict(iris.data)
# complete
complete_clustering = AgglomerativeClustering(n_clusters=3, linkage='complete')
complete_pred = complete_clustering.fit_predict(iris.data)
# average
avg_clustering = AgglomerativeClustering(n_clusters=3, linkage='average')
avg_pred = avg_clustering.fit_predict(iris.data)
```

정규화한 X 를 이용해 덴드로그램을 그립니다. scipy 에서 관련 기능을 지원합니다.

```python
# 정규화하기
normalized_X = preprocessing.normalize(iris.data)
```

```python
from scipy.cluster.hierarchy import dendrogram, ward
from scipy.cluster.hierarchy import dendrogram

linkage_array = ward(normalized_X)
plt.figure(figsize=(22,18))
dendrogram(linkage_array)
plt.show()
```

### 병합 군집의 장단점

장점

- 기존처럼 시각화할 수 있을 뿐만 아니라 덴드로그램으로도 시각화가 가능합니다.
- 실제로 진화생물학같은 계층 관계가 있는 데이터셋에 유용합니다.
  단점
- 노이즈와 이상치에 민감합니다.
- 계산 집약적으로, 다차원의 대용량 데이터를 학습시키는데 오랜 시간이 걸립니다.

## 밀도(Density) 군집

### DBSCAN

DBSCAN 은 복잡한 형상을 찾거나, 어떤 클래스에도 속하지 않는 포인트를 구분하고, 그리고 클러스터에 개수를 미리 지정할 필요가 없습니다. 또한 병합 군집이나 K-평균보다는 느리지만 비교적 큰 데이터셋에도 적용할 수 있습니다.

#### DBSCAN 의 작동 방식

DBSCAN 의 아이디어는 데이터의 밀집 지역이 한 클러스터를 구성하며, 비교적 비어있는 지역을 경계로 다른 클러스터와 구분한다는 것입니다.

- 특성 공간에서 데이터가 많아 붐비는 **밀집 지역**을 찾습니다.
- 밀집 지역에 있는 포인트(**핵심 샘플**)을 사전에 입력한 파라미터 `min_samples`와 `eps`를 이용해 정의합니다. 만약 `eps` 거리 안의 포인트가 `min_samples`개만큼 들어 있다면, 이 포인트를 핵심 샘플으로 지정합니다. 반대로 `eps` 거리 안의 포인트가 `min_samples`개보다 적다면 그 포인트는 잡음(노이즈)로 레이블합니다. `eps`보다 가까운 핵심 샘플은 DBSCAN 에 의해 같은 클러스터로 합쳐집니다.

이 알고리즘은 시작 시 무작위로 포인트를 선정합니다. 그 다음 그 포인트에서 `eps` 거리 안의 모든 포인트를 찾고, `min_samples`보다 많은지 적은지에 따라 다르게 레이블합니다. 그 다음 `eps` 거리 안의 모든 이웃을 살핍니다. 아직 할당되지 않은 포인트이면 전에 만든 클러스터 레이블을 할당하고, 핵심 샘플이면 그 포인트의 이웃을 차례로 방문합니다. 방문을 하면:

- 아직 어떤 클러스터에도 할당되지 않은 포인트라면 바로 전에 만든 클러스터 레이블을 할당합니다.
- 만약 핵심 샘플이라면 그 포인트의 이웃들을 차례대로 방문합니다.

이렇게 계속해서 더 이상 `eps` 거리 안에 핵심 샘플이 없을 때까지 반복하고, 아직 방문하지 않은 포인트를 선택해 같은 과정을 반복합니다.

#### 사이킷런에서 DBSCAN 실행하기

```python
from sklearn import cluster
eps=1.32
min_samples=50
dbscan = cluster.DBSCAN(eps=eps, min_samples=min_samples)
clustering_labels_4 = dbscan.fit_predict(dataset_2)
```

`eps` 매개변수는 가까운 포인트의 범위를 결정합니다. 작게 하면 어떤 포인트도 핵심 포인트가 되지 못하고, 모든 포인트가 노이즈가 됩니다. 반대로 매우 크게 하면 하나의 클러스터에 모든 데이터가 들어갈 것입니다. 즉 간접적으로 몇 개의 클러스터를 만들지 제어합니다. 적절한 `eps`값을 얻기 위해 StandardScaler 또는 MinMaxScaler 로 모든 특성의 스케일을 비슷한 범위로 조절하면 좋습니다.

`min_samples` 설정은 덜 조밀한 지역의 포인트들을 노이즈로 할 것인지, 아니면 클러스터로 할 것인지를 결정합니다. 즉 클러스터의 최소 크기를 결정합니다.

## 참고

[Intro to Machine Learning with TensorFlow](https://www.udacity.com/course/intro-to-machine-learning-with-tensorflow-nanodegree--nd230)

[혼자 공부하는 머신러닝+딥러닝 - 6-2 K-평균](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=257932080)

[파이썬 라이브러리를 활용한 머신러닝(번역개정판) - 3장 비지도 학습과 데이터 전처리](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=186846299)

[핸즈온 머신러닝 - 1부 9장 비지도 학습](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=237677114)
