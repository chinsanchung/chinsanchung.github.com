---
title: 'Introduction to Machine Learning with TensorFlow - Identify Customer Segments'
layout: single
author_profile: false
read_time: false
comments: false
share: true
related: true
categories:
  - project
toc: true
toc_sticky: true
toc_labe: 목차
description: Introduction to Machine Learning with TensorFlow 의 세 번째 프로젝트 "Identify Customer Segments"를 어떻게 작성했는지 설명합니다.
tags:
  - nanodegree
---

**Introduction to Machine Learning with TensorFlow** 의 세 번째 프로젝트 "Identify Customer Segments"를 진행하면서, 주어진 문제의 정답을 왜 이렇게 작성했는지를 정리하고자 합니다.

## 개요: 프로젝트에 대한 설명

이번 프로젝트는 Bertelsmann 파트너사인 AZ Direct 와 Arvato Finance Solution 에서 제공한 현실의 데이터로 작업을 합니다. 이 데이터는 독일에서 통신 판매를 수행하는 회사에 대한 것으로, 그들이 원하는 것은 *우편 발송 캠페인을 위해 제품을 구매할 가능성이 큰 사람들을 식별하는 것*입니다. 현실 세계의 데이터인만큼, EDA 를 통해 데이터를 확인, 전처리하고 정규화나 표준화를 진행하는 것이 필요합니다.

### 데이터의 구성

- `Udacity_AZDIAS_Subset.csv`: 독일인의 인구 통계 데이터로, 891,211명의 행과 85개의 특성이 있습니다.
- `Udacity_CUSTOMERS_Subset.csv`: 메일 주문 회사의 고객에 대한 인구 통계 데이터입니다. 191,652명의 행과 85개의 특성이 있습니다.
- `Data_Dictionary.md`: 데이터셋의 특성을 설명합니다.
- `AZDIAS_Feature_Summary.csv`: 인구 통계 데이터의 특성에 대한 속성을 요약했습니다.
- `Identify_Customer_Segments.ipynb`: 제출해야 할 노트북으로, 프로젝트에 대한 가이드와 설명이 담겨 있습니다.

Arvato Bartlesmann 와의 협약으로 인해 프로젝트가 완성하면 반드시 로컬 컴퓨터에서 위의 파일들을 지워야합니다.(또한 프로젝트 이외의 목적으로 데이터를 활용하면 안됩니다.) 그에 따라 이번 프로젝트는 주피터 노트북으로 깃허브 레퍼지토리에 올리는 대신 이 문서에 작성하겠습니다.

### 진행 절차

1. 데이터 전처리

When you start an analysis, you must first explore and understand the data that you are working with. In this (and the next) step of the project, you’ll be working with the general demographics data. As part of your investigation of dataset properties, you must attend to a few key points:

- How are missing or unknown values encoded in the data? Are there certain features (columns) that should be removed from the analysis because of missing data? Are there certain data points (rows) that should be treated separately from the rest?
- Consider the level of measurement for each feature in the dataset (e.g. categorical, ordinal, numeric). What assumptions must be made in order to use each feature in the final analysis? Are there features that need to be re-encoded before they can be used? Are there additional features that can be dropped at this stage?

You will create a cleaning procedure that you will apply first to the general demographic data, then later to the customers data.

2. 특성 변환

Now that your data is clean, you will use dimensionality reduction techniques to identify relationships between variables in the dataset, resulting in the creation of a new set of variables that account for those correlations. In this stage of the project, you will attend to the following points:

- The first technique that you should perform on your data is feature scaling. What might happen if we don’t perform feature scaling before applying later techniques you’ll be using?
- Once you’ve scaled your features, you can then apply principal component analysis (PCA) to find the vectors of maximal variability. How much variability in the data does each principal component capture? Can you interpret associations between original features in your dataset based on the weights given on the strongest components? How many components will you keep as part of the dimensionality reduction process?

You will use the sklearn library to create objects that implement your feature scaling and PCA dimensionality reduction decisions.

3. 군집화

Finally, on your transformed data, you will apply clustering techniques to identify groups in the general demographic data. You will then apply the same clustering model to the customers dataset to see how market segments differ between the general population and the mail-order sales company. You will tackle the following points in this stage:

- Use the k-means method to cluster the demographic data into groups. How should you make a decision on how many clusters to use?
- Apply the techniques and models that you fit on the demographic data to the customers data: data cleaning, feature scaling, PCA, and k-means clustering. Compare the distribution of people by cluster for the customer data to that of the general population. Can you say anything about which types of people are likely consumers for the mail-order sales company?

sklearn will continue to be used in this part of the project, to perform your k-means clustering. In the end, you will export the completed notebook with your work as an HTML file, which will serve as a report documenting your approach and findings.

https://review.udacity.com/#!/rubrics/1973/view

## 데이터 전처리

전처리 전에 데이터를 불러옵니다.

```python
azdias = pd.read_csv('Udacity_AZDIAS_Subset.csv', sep=';')
feat_info = pd.read_csv('AZDIAS_Feature_Summary.csv', sep=';')
```

### 누락 데이터에 접근하기

#### 누락 값을 NaN 으로 변경하기

feat_info 의 `missing_or_unknown` 열에는 azdias 데이터의 값들 중에서 누락 값이 무엇인지를 문자열로 변환한 리스트로 알려줍니다. 우선 누락 값들을 전부 NaN 으로 변경합니다.

리스트를 문자열로 변환했던 `missing_or_unknown` 값들을 [list_eval() ](https://stackoverflow.com/questions/23111990/pandas-dataframe-stored-list-as-string-how-to-convert-back-to-list을 이용해 원래대로 되돌립니다.

1. `name 'X' is not defined` 라는 에러가 발생했는데, 열이 "[-1, XX]" 으로 생긴 것이라서 생긴 문제. 그래서 replace 두 번해서 괄호를 지우고 split(',')으로 나눴습니다.

`value_counts()`으로 어떤 값들이 있는지 파악한 후 함수를 만들어서 조건문 형식으로 값을 변환합니다.

```python
def convert_to_list(val):
    if val == '[]':
        return []
    elif val == '[-1,X]':
        return [-1,'X']
    elif val == '[XX]':
        return ['XX']
    elif val == '[-1,XX]':
        return [-1,'XX']
    else:
        converted_list = val.replace('[','').replace(']','').split(',')
        return [int(x) for x in converted_list]

feat_info['missing_or_unknown'] = feat_info['missing_or_unknown'].apply(convert_to_list)
```

2. 데이터셋에 변환한 값을 비교해야 함. 리스트 형태를 반복문을 돌려서? 시리즈로 뽑은 후에 apply 돌려서? apply 를 돌릴 때 열의 이름을 알고, 그래서 fea 에 연결시킨다음에 for 문 돌리기..?

```python
for attribute in feat_info['attribute']:
    missing_or_unknown = feat_info.loc[feat_info['attribute'] == attribute]['missing_or_unknown'].values[0]
    azdias[attribute] = azdias[attribute].apply(lambda x: np.nan if x in missing_or_unknown else x)
```

#### 각 열의 누락 데이터를 확인하기

1. 누락 데이터가 얼마나 있는지를 확인합니다.

```python
nan_count_list = azdias.isnull().sum()
total_data_size = azdias.shape[0]

fig, axs = plt.subplots(17, 5, figsize=(20, 30))
fig.tight_layout()
x_labels = ['NaN', 'Not NaN']
idx = 0
for i in range(17):
    for j in range(5):
        weights = [nan_count_list[idx], total_data_size - nan_count_list[idx]]
        axs[i][j].hist(x_labels, weights=weights)
        axs[i][j].set_title(feat_info['attribute'].iloc[idx])
        idx += 1
```

```python
nan_count_percentage = (nan_count_list / total_data_size) * 100

plt.figure(figsize=(20, 10))
percentage_hist = plt.hist(feat_info['attribute'], weights=nan_count_percentage, bins=85)
plt.xticks(rotation=90)
plt.show()
```

2. 각 열의 누락 값의 양에 따른 패턴을 조사합니다.

결측치가 있는 비율을 계산한 `nan_count_percentage`을 살펴봤을 때 'AGER_TYP', 'GEBURTSJAHR', 'TITEL_KZ', 'ALTER_HH','KK_KUNDENTYP','KBA05_BAUMAX' 이 여섯 열이 가장 높은 비율을 보였습니다.

- 'AGER_TYP' 최고 연령 유형. 76.9%
- 'GEBURTSJAHR' 생년월일. 44%
- 'TITEL_KZ' 학술 타이틀 플래그. 99.7%
- 'ALTER_HH' 세대주의 생년월일. 34.8%
- 'KK_KUNDENTYP' 지난 12개월 동안의 소비자 패턴. 65.5%
- 'KBA05_BAUMAX' 마이크로셀 내에서 가장 일반적인 건물 유형. 53%

이번에는 `Data_Dictionary.md`을 참고해서 9개의 주요 특성들 중에서 결측치의 비율이 10% 이상인 열이 몇 개가 있는지를 살펴봤습니다.

- Person-level features: 10 / 38
- Household-level features: 3 / 4
- Building-level features: 6 / 7
- RR4 micro-cell features: 3 / 3
- RR3 micro-cell features: 6 / 6
- Postcode-level features: 3 / 3
- RR1 neighborhood features: 4 / 5
- PLZ8 macro-cell features: 8 / 8
- Community-level features: 3 / 3

Building-level features, RR3 micro-cell features, RR1 neighborhood features, PLZ8 macro-cell features 은 결측치가 10% 이상인 경우가 많았습니다. 결측치를 처리하기 위한 과정이 필요해 보입니다.

3. 결측치를 데이터셋에서 삭제하기

6개의 열 'AGER_TYP', 'GEBURTSJAHR', 'TITEL_KZ', 'ALTER_HH','KK_KUNDENTYP','KBA05_BAUMAX'을 삭제한 DataFrame 을 `azdias_without_outlier` 에 할당했습니다.

```python
azdias_without_outlier = azdias.drop(columns=['AGER_TYP', 'GEBURTSJAHR', 'TITEL_KZ', 'ALTER_HH','KK_KUNDENTYP','KBA05_BAUMAX'])
```

#### 각 행의 누락 데이터를 확인하기

1. 각 행에 얼마나 많은 결측치가 있는지 확인합니다.

우선 각 행에 결측치가 몇 개가 있는지를 계산한 데이터프레임을 만들었습니다. [참고한 코드](https://datascience.stackexchange.com/questions/12645/how-to-count-the-number-of-missing-values-in-each-row-in-pandas-dataframe)

```python
isna_azdias = azdias.isna()
nan_count = isna_azdias.sum(axis=1)
nan_count_df['nan_count'].value_counts().sort_values(ascending=False)
```

한 행의 전체 열 중에서 결측치가 얼마나 있는지를 계산한 `nan_count`의 값을 정렬했을 때, 개수가 3, 2, 4인 행이 전체 데이터의 51.4%를 차지하고 있었습니다.

```python
nan_count_df.loc[nan_count_df['nan_count'] > 10].describe()
```

2. 누락된 데이터의 양에 따라 데이터셋을 두 그룹으로 나눕니다.

우선 원래의 데이터프레임을 복사한 후 거기에 `nan_count`를 열으로 추가한 데이터프레임을 만듭니다. 그리고 위의 수치를 토대로 결측치의 개수가 20미만인 것과 이상인 것으로 데이터를 분리합니다.

```python
azdias_with_nan_count_under_five = azdias_with_nan_count.loc[azdias_with_nan_count['nan_count'] < 20]
azdias_with_nan_count_more_five = azdias_with_nan_count.loc[azdias_with_nan_count['nan_count'] >= 20]
```

3. 두 그룹을 비교하면서, 결측치가 많은 데이터와 없는 데이터의 특성의 분포가 서로 유사한지를 확인합니다.

각 그룹에서 5개를 추출한 후, 결측치가 없는 열의 데이터로 그래프를 그려 비교합니다.

```python
# 결측치가 없는 데이터 5개를 추출합니다.
under_nan_head = azdias_with_nan_count_under.sort_values(by='nan_count').head()
# 결측치가 많은 데이터 5개를 추출합니다.
more_nan_head = azdias_with_nan_count_more.sort_values(by='nan_count', ascending=False).head()
# 비교를 위해서 열 중에서 결측치가 없는 데이터만을 저장합니다.
non_nan_columns = np.array(nan_count_percentage.loc[nan_count_percentage == 0].index)
under_nan_head = under_nan_head[non_nan_columns]
more_nan_head = more_nan_head[non_nan_columns]
# 결측치가 많은 그래프, 결측치가 없는 그래프를 그립니다.
fig, axs = plt.subplots(6, 4, figsize=(20, 30))
fig.tight_layout()
x_labels = np.arange(5)
idx = 0
for i in range(6):
    for j in range(4):
        axs[i][j].plot(x_labels, under_nan_head[non_nan_columns[idx]], label='under_nan_head')
        axs[i][j].plot(x_labels, more_nan_head[non_nan_columns[idx]], label='more_nan_head')
        axs[i][j].set_title(non_nan_columns[idx])
        axs[i][j].set_xticks(x_labels)
        axs[i][j].legend(loc='center right')
        idx += 1
```

## 특성 변환

## 군집화 알고리즘을 적용하기
