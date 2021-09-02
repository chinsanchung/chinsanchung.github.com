---
title: '머신러닝 공부 - 랜덤 투영과 ICA'
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
description: 차원 축소 기법인 랜덤 투영과 ICA, 그리고 언급하지 않은 차원 축소 알고리즘에 대해 공부한 내용을 정리합니다.
tags:
  - machine_learning
---

저번에는 차원 축소 기법이 무엇인지, 그리고 대표적인 차원 축소 기법인 주성분 분석(PCA)를 다뤘습니다. 이번에는 다른 차원 축소 기법인 랜덤 투영(Random Projection)과 ICA(Independent Component Analysis), 마지막으로 아직 언급하지 않은 차원 축소 기법들에 대해 정리합니다.

## 랜덤 투영(Random Projection)

랜덤 투영은 무작위로 선형 투영을 이용해서 데이터를 저차원 공간으로 투영하는 기법으로, 차원이 너무 많아서 PCA 로는 계산하기 어려운 데이터셋에서 주요 사용합니다. 고차원 데이터에서 PCA 보다 높은 성능을 제공하지만, 투영의 품질은 상대적으로 떨어집니다.

차원을 축소하는 품질은 샘플의 개수와 목표 차원 수에 따라 다르지만, 처음에 지정한 차원의 개수에는 의존적이지 않습니다.

### 사이킷런에서의 랜덤 투영

[SparseRandomProjection 공식문서](https://scikit-learn.org/stable/modules/generated/sklearn.random_projection.SparseRandomProjection.html)에서 더 자세한 내용을 읽으실 수 있습니다.

```python
from sklearn.random_projection import SparseRandomProjection
rp = SparseRandomProjection()
new_X = rp.fit_transform(X)
```

## ICA(Independent Component Analysis)

독립 구성요소 분석(ICA) 는 다변량 신호(예시 - 음원, 뇌파)를 최대한 독립적인 추가 구성요소로 분리합니다. 일반적으로 ICA 는 차원을 줄이는 데 사용하지 않고, 중첩된 신호를 분리하는 데 사용합니다.

항상 우리가 가지고 있는

### 사이킷런에서의 랜덤 투영

예시로, 한 공간에서 3명이 피아노, 첼로, TV 의 잡음을 녹음하고, 하나로 합쳐진 소리를 ICA 를 이용해서 다시 나누겠습니다. 사이킷런의 `FastICA`를 사용하여 진행합니다. 더 자세한 내용은 [공식 문서](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.FastICA.html)에서 확인하실 수 있습니다.

```python
import wave

# wav 파일을 읽어옵니다.
mix_1_wave = wave.open('ICA mix 1.wav','r')
# wave 파일에서 프레임을 추출해서 리스트에 저장합니다.
signal_1_raw =  mix_1_wave.readframes(-1)
# 정수 리스트로 변환합니다.
signal_1 = np.fromstring(signal_1_raw, 'Int16')

mix_2_wave = wave.open('ICA mix 2.wav','r')
signal_raw_2 = mix_2_wave.readframes(-1)
signal_2 = np.fromstring(signal_raw_2, 'Int16')

mix_3_wave = wave.open('ICA mix 3.wav','r')
signal_raw_3 = mix_3_wave.readframes(-1)
signal_3 = np.fromstring(signal_raw_3, 'Int16')

# 추출한 프레임 리스트를 하나의 데이터셋으로 만듭니다.
X = list(zip(signal_1, signal_2, signal_3))
```

ICA 를 사용해서 원래의 음원으로 복원합니다.

```python
from sklearn.decomposition import FastICA
# 피아노, 첼로, TV 3개의 컴포넌트가 있어 n_components 를 3으로 설정합니다.
ica = FastICA(n_components=3)
ica_result = ica.fit_transform(X)

# 음원1, 음원2, 음원3 으로 분리합니다.
result_signal_1 = ica_result[:,0]
result_signal_2 = ica_result[:,1]
result_signal_3 = ica_result[:,2]
```

분리한 독립 구성요소가 어떤 형태인지 그래프로 그려봅니다.

```
# Plot Independent Component #1
plt.figure(figsize=(12,2))
plt.title('Independent Component #1')
plt.plot(result_signal_1, c="#df8efd")
plt.ylim(-0.010, 0.010)
plt.show()

# Plot Independent Component #2
plt.figure(figsize=(12,2))
plt.title('Independent Component #2')
plt.plot(result_signal_2, c="#87de72")
plt.ylim(-0.010, 0.010)
plt.show()

# Plot Independent Component #3
plt.figure(figsize=(12,2))
plt.title('Independent Component #3')
plt.plot(result_signal_3, c="#f65e97")
plt.ylim(-0.010, 0.010)
plt.show()
```

이번에는 리스트 데이터를 다시 음원으로 변환해서 저장해봅니다. 먼저 정수로 변환을 한 후, 그 값을 int16 오디오의 범위 내로 매핑을 합니다. (참고로 100같은 값을 곱해 볼륨을 높일 수 있습니다.)

```python
from scipy.io import wavfile

# Convert to int, map the appropriate range, and increase the volume a little bit
result_signal_1_int = np.int16(result_signal_1*32767*100)
result_signal_2_int = np.int16(result_signal_2*32767*100)
result_signal_3_int = np.int16(result_signal_3*32767*100)


# Write wave files
wavfile.write("result_signal_1.wav", fs, result_signal_1_int)
wavfile.write("result_signal_2.wav", fs, result_signal_2_int)
wavfile.write("result_signal_3.wav", fs, result_signal_3_int)
```

## 그 외 다른 차원 축소 기법들

- 다차원 스케일링(MultiDimensional Scaling) : 샘플 간의 거리를 보존하면서 차원을 축소하는 기법입니다.
- Isomap : 각 샘플을 가장 가까운 이웃과 연결하는 형식으로 그래프를 만들고, 샘플 간에 지오데식 거리(예시 - 두 노드 사이의 지오데식 거리는 두 노드 사이의 최단 경로를 이루는 노드의 수를 뜻합니다.)를 유지하며 차원을 축소합니다.
- t-SNE : 비슷한 샘플은 가까이, 다른 샘플은 멀리 떨어지도록 하면서 차원을 축소합니다. 시각화에 많이 사용하고, 특히 고차원 공간에 있는 샘플의 군집을 시각화할 때 사용합니다.
- 선형 판별 분석(Linear Discriminant Analysis) : 사실 분류 알고리즘이지만 훈련 과정에서 클래스의 사이를 가장 잘 구분하는 축을 학습합니다. 이 축을 이용하면 투영을 통해 가능한 클래스를 멀리 떨어지게 유지시킬 수 있기 때문에, 다른 분류 알고리즘을 적용하기 전에 차원을 축소시키는 데 유용합니다.

## 참고

[Intro to Machine Learning with TensorFlow](https://www.udacity.com/course/intro-to-machine-learning-with-tensorflow-nanodegree--nd230)

[파이썬 라이브러리를 활용한 머신러닝(번역개정판) - 3장 비지도 학습과 데이터 전처리](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=186846299)

[핸즈온 머신러닝 - 1부 8장 차원 축소](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=237677114)

[사이킷런 공식 문서](https://scikit-learn.org/stable/index.html)
