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

## 데이터 전처리

전처리 전에 데이터를 불러옵니다.

```python
azdias = pd.read_csv('Udacity_AZDIAS_Subset.csv', sep=';')
feat_info = pd.read_csv('AZDIAS_Feature_Summary.csv', sep=';')
```

### 누락 데이터에 접근하기

#### 누락 값을 NaN 으로 변경하기

feat_info 의 missing_or_unknown 열에는 azdias 데이터의 값들 중에서 누락 값이 무엇인지를 문자열로 변환한 리스트로 알려줍니다. 우선 누락 값들을 전부 NaN 으로 변경합니다.

리스트를 문자열로 변환했던 missing_or_unknown 열의 값들을 [list_eval() ](https://stackoverflow.com/questions/23111990/pandas-dataframe-stored-list-as-string-how-to-convert-back-to-list을 이용해 원래대로 되돌립니다.

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

2. feat_info 의 missing_or_unknown 열과 azdias 데이터프레임을 비교해서 결측치인 값들을 NaN 으로 바꾸는 작업을 수행합니다.

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

각 행에 결측치가 몇 개가 있는지를 계산한 시리즈를 만들었습니다. [참고한 코드](https://datascience.stackexchange.com/questions/12645/how-to-count-the-number-of-missing-values-in-each-row-in-pandas-dataframe)

```python
isna_azdias = azdias.isna()
nan_count = isna_azdias.sum(axis=1)
```

각 행에 결측치가 몃 개가 있는지를 계산한 pandas Series 를 만들었습니다. 그리고 `value_counts()`, `sort_values(ascending=False)`를 이용해 각 행의 결측치의 개수를 기준으로 그룹으로 묶은 후, 묶은 행의 개수만큼 오름차순으로 정렬했습니다. 그 결과 결측치가 3개인 행이 가장 많으며, 상위 3개의 비중이 전체의 51.4%, 상위 5개의 비중이 71.8%를 차지합니다.

```python
nan_count.value_counts().sort_values(ascending=False)
```

3 189850
2 135292
4 133473
5 110108
6 71615

```

```

이번에는 위의 결과에서 결측치가 가장 많은 행을 기준으로 내림차순으로 정렬했습니다. 상위 10개 모두 결측치가 40개를 초과하고 있으며, 전체의 8.3%를 차지하고 있습니다.

```python
nan_count.value_counts().sort_index(ascending=False)
```

54 1
53 45579
51 483
50 105
49 27357

```

```

전체 데이터에서 결측치가 한 자리수인 행의 비중은 80.8%으로, 나머지 19.2% 를 주의해서 데이터 전처리를 수행하면 되겠습니다.

2. 누락된 데이터의 양에 따라 데이터셋을 두 그룹으로 나눕니다.

결측치가 한 자리수인 행이 80.8%를 차지하는 것을 보며, 결측치가 10이상인지 아닌지를 기준으로 데이터를 나눴습니다.

```python
azdias_with_nan_count_under = azdias_with_nan_count.loc[azdias_with_nan_count['nan_count'] < 10]
azdias_with_nan_count_more = azdias_with_nan_count.loc[azdias_with_nan_count['nan_count'] >= 10]
```

3. 두 그룹을 비교하면서, 결측치가 많은 데이터와 없는 데이터의 특성의 분포가 서로 유사한지를 확인합니다.

각 그룹에서 20개를 추출한 후, 결측치가 없는 열의 데이터로 그래프를 그려 비교합니다. 그래프를 그리기 전, 그래프로 표현하기 위해 결측치가 없는 특성만을 선택했습니다.

- 결측치가 없는 특성들은 전부 Person-level features 에 속하며,

```python
# 결측치가 없는 데이터 20개를 추출합니다.
under_nan_head = azdias_with_nan_count_under.sort_values(by='nan_count').head()
# 결측치가 많은 데이터 20개를 추출합니다.
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

![결측치가 20이상 그리고 20미만인 데이터셋 비교](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-09-03-Identify-Customer-Segments_compare_nan.png?raw=true)

`more_nan_head` 의 값이 높은 경우는 14개, `under_nan_head`가 높은 경우는 8개입니다. `under_nan_head`는 남성이, `more_nan_head`는 여성이 더 많았습니다.

그 외 특이한 점은 `more_nan_head`에서는 5개 모두 동일한 값을 가진 특성들이 많았습니다. "SEMIO\_"로 시작하는 항목의 값은 5 이상인 경우가 많았는데, Data_Dictionary 에 따르면 low affinity, very low affinity 를 의미합니다. 따라서 결측치가 많은 항목이면 affinity 가 낮을 가능성이 높다고도 할 수 있습니다.

### 특성 선택 후 다시 인코딩하기

우선 특성이 어떻게 구성되어있는지 확인합니다.

```python
azdias.info()
```

범주형 특성의 클래스들을 확인합니다.

```python
azdias.describe(include=np.object)
```

#### 범주형 특성을 재인코딩하기

1. 범주형 특성이 어떤 것이 있는지 확인합니다.

```python
azdias.describe(include=np.object)
```

또한 int64 형식의 특성도 숫자로 표현된 범주형 특성이라 할 수 있습니다.

```python
azdias.describe(include=np.int64)
```

2. 범주형 특성을 다시 인코딩합니다.

결측치가 가장 많은 6개 특성을 제거한 데이터에서 우선 범주형 특성의 결측치를 최빈값으로 변경합니다. 우리는 이미 `azdias.describe(include=np.object)`을 통해 최빈값이 무엇인지를 알고 있습니다.

```python
azdias_without_outlier['OST_WEST_KZ'] = azdias_without_outlier['OST_WEST_KZ'].fillna('W')
azdias_without_outlier['CAMEO_DEUG_2015'] = azdias_without_outlier['CAMEO_DEUG_2015'].fillna('8')
azdias_without_outlier['CAMEO_DEU_2015'] = azdias_without_outlier['CAMEO_DEU_2015'].fillna('6B')
azdias_without_outlier['CAMEO_INTL_2015'] = azdias_without_outlier['CAMEO_INTL_2015'].fillna('51')
```

숫자형인 범주형 특성의 결측치도 최빈값으로 변경합니다.

```python
mode_of_int_azdias = azdias_without_outlier[['ANREDE_KZ', 'FINANZ_MINIMALIST', 'FINANZ_SPARER', 'FINANZ_VORSORGER',
       'FINANZ_ANLEGER', 'FINANZ_UNAUFFAELLIGER', 'FINANZ_HAUSBAUER',
       'FINANZTYP', 'GREEN_AVANTGARDE', 'SEMIO_SOZ', 'SEMIO_FAM', 'SEMIO_REL',
       'SEMIO_MAT', 'SEMIO_VERT', 'SEMIO_LUST', 'SEMIO_ERL', 'SEMIO_KULT',
       'SEMIO_RAT', 'SEMIO_KRIT', 'SEMIO_DOM', 'SEMIO_KAEM', 'SEMIO_PFLICHT',
       'SEMIO_TRADV', 'ZABEOTYP']].mode()

for idx in np.arange(24):
    feature_name = mode_of_int_azdias.columns[idx]
    mode = mode_of_int_azdias.values[0][idx]
    azdias_without_outlier[feature_name] = azdias_without_outlier[feature_name].fillna(mode)
```

마지막으로 숫자형 특성의 결측치를 최빈값으로 변경합니다.

```python
mode_of_float_azdias = azdias_without_outlier[azdias_without_outlier.describe().columns]

for idx in np.arange(75):
    feature_name = mode_of_float_azdias.columns[idx]
    mode = mode_of_float_azdias.values[0][idx]
    azdias_without_outlier[feature_name] = azdias_without_outlier[feature_name].fillna(mode)
```

우선 데이터는 가장 결측치가 많던 6개의 특성을 제외한 것을 사용했습니다. 그리고 범주형 특성은 4개, 숫자 형태의 범주형 특성은 24개가 있다는 것을 `describe()` method 로 확인했습니다.

다음에는 결측치를 각 특성의 최빈값으로 변경했습니다.

#### PRAEGENDE_JUGENDJAHRE, CAMEO_INTL_2015 특성을 조사하고 다시 설계하기

1. PRAEGENDE_JUGENDJAHRE 특성은 10년에 걸친 세대의 움직임(mainstream 또는 avantgarde), 위치(동쪽 또는 서쪽)에 대한 3차원의 정보를 결합한 것입니다. 이 특성을 10년에 대한 간격과 세대의 움직임 두 특성으로 분리해야합니다.

딕셔너리 값은 세대의 간격의 앞 글자(40년대는 4, 50년대는 5)와 움직임(0은 mainstream, 1은 avantgarde)을 뜻합니다.

```python
praegende_jugendjahre = {
    "1": [4, 0],
    "2": [4, 1],
    "3": [5, 0],
    "4": [5, 1],
    "5": [6, 0],
    "6": [6, 1],
    "7": [6, 1],
    "8": [7, 0],
    "9": [7, 1],
    "10": [8, 0],
    "11": [8, 1],
    "12": [8, 0],
    "13": [8, 1],
    "14": [9, 0],
    "15": [9, 1],
}
generation = []
movement = []

azdias_without_outlier['PRAEGENDE_JUGENDJAHRE'].apply(create_new_feature_with_praegende_jugendjahre)

azdias_without_outlier['GENERATION'] = generation
azdias_without_outlier['MOVEMENT'] = movement
```

"GENERATION"은 어느 세대인지 연도의 앞 글자를 따와서 저장한 숫자 형태의 범주형 feature 입니다. "MOVEMENT"은 범주형이지만 향후 예측에 활용하기 위해서 mainstream 일 경우 0, avantgarde 일 때는 1으로 설정했습니다.

2. CAMEO_INTL_2015 특성은 부와 삶의 단계를 포함하고 있습니다. 10의 자리는 부, 1의 자리를 삶으로 구분하고 있으며, 이 특성 또한 10의 자리와 1의 자리로 분리해야합니다.

```python
wealth_list = []
life_list = []


def create_new_feature_with_cameo_intl_2015(value):
    wealth_list.append(value[0])
    life_list.append(value[1])

azdias_without_outlier['CAMEO_INTL_2015'].apply(create_new_feature_with_cameo_intl_2015)

azdias_without_outlier['WEALTH'] = wealth_list
azdias_without_outlier['LIFE'] = life_list
```

"CAMEO_INTL_2015" 값의 앞자리를 "WEALTH" 그리고 뒷자리를 "LIFE"로 구분해서 새로운 특성을 만들었습니다.

새로운 특성을 만들기 앞서 "PRAEGENDE_JUGENDJAHRE", "CAMEO_INTL_2015"의 결측치를 최빈값으로 변경했기 때문에 에러가 발생하지는 않았습니다. 다만 "PRAEGENDE_JUGENDJAHRE"의 10, 12 그리고 11, 13은 변환했을 때 같은 값이 나오게 됩니다. 이 부분을 염두에 둬야 할 것입니다.

#### 필요한 특성만을 선택하기

데이터프레임에 필요한 열만 있는지를 확인합니다. 여기서는 변환을 완료한 "PRAEGENDE_JUGENDJAHRE", "CAMEO_INTL_2015" 특성을 삭제했습니다.

```python
azdias_without_outlier = azdias_without_outlier.drop(columns=['PRAEGENDE_JUGENDJAHRE'])
azdias_without_outlier = azdias_without_outlier.drop(columns=['CAMEO_INTL_2015'])
```

그리고 범주형 특성을 원-핫 인코딩을 통해 인코딩했습니다. 저는 [get_dummies](https://pandas.pydata.org/docs/reference/api/pandas.get_dummies.html)함수를 이용해 범주형 특성을 자동으로 변환했습니다.

```python
azdias_without_outlier = pd.get_dummies(azdias_without_outlier)
```

### 데이터를 클리닝하는 함수 만들기

지금까지 수행했던 데이터 전처리 과정을 하나의 함수로 만드는 과제입니다.

1. feature_info 에서 제시한 결측치를 활용해 실제 데이터의 결측치를 NaN 으로 바꿉니다.
2. 결측치가 가장 높은 열 6개를 제거합니다.
3. 범주형 특성, 숫자형 특성의 결측치를 최빈값으로 바꿉니다.
4. 2개 또는 3개의 정보를 한번에 포함하고 있는 PRAEGENDE_JUGENDJAHRE 특성과 CAMEO_INTL_2015 특성을 각각 두 개의 특성으로 나누고 원래 특성을 제거합니다.
5. 전처리를 완료한 데이터프레임을 반환합니다.

```python
def clean_data(df):
    """
    Perform feature trimming, re-encoding, and engineering for demographics
    data

    INPUT: Demographics DataFrame
    OUTPUT: Trimmed and cleaned demographics DataFrame
    """
    target_df = df.copy()
    # Put in code here to execute all main cleaning steps:
    # convert missing value codes into NaNs, ...
    ## 1. Convert missing or unknown values to NaN
    for attribute in feat_info['attribute']:
        missing_or_unknown = feat_info.loc[feat_info['attribute'] == attribute]['missing_or_unknown'].values[0]
        target_df[attribute] = target_df[attribute].apply(lambda x: np.nan if x in missing_or_unknown else x)

    print("1 ", target_df.shape)
    # remove selected columns and rows, ...
    ## 2. Drop columns which have missing values over 50%
    target_df = target_df.drop(columns=['AGER_TYP', 'GEBURTSJAHR', 'TITEL_KZ',
                                        'ALTER_HH','KK_KUNDENTYP','KBA05_BAUMAX'])
    print("2 ", target_df.shape)
    # select, re-encode, and engineer column values.
    ## 3. Fill NaN value to the mode
    ### 3-1. Categorical features
    mode_of_categorical = customers.describe(include=np.object).iloc[2]
    target_df['OST_WEST_KZ'] = target_df['OST_WEST_KZ'].fillna(mode_of_categorical[0])
    target_df['CAMEO_DEUG_2015'] = target_df['CAMEO_DEUG_2015'].fillna(mode_of_categorical[1])
    target_df['CAMEO_DEU_2015'] = target_df['CAMEO_DEU_2015'].fillna(mode_of_categorical[2])
    target_df['CAMEO_INTL_2015'] = target_df['CAMEO_INTL_2015'].fillna(mode_of_categorical[3])
    ### 3-2. Numeric but categorical features
    mode_of_int_azdias = target_df[['ANREDE_KZ', 'FINANZ_MINIMALIST', 'FINANZ_SPARER', 'FINANZ_VORSORGER',
       'FINANZ_ANLEGER', 'FINANZ_UNAUFFAELLIGER', 'FINANZ_HAUSBAUER',
       'FINANZTYP', 'GREEN_AVANTGARDE', 'SEMIO_SOZ', 'SEMIO_FAM', 'SEMIO_REL',
       'SEMIO_MAT', 'SEMIO_VERT', 'SEMIO_LUST', 'SEMIO_ERL', 'SEMIO_KULT',
       'SEMIO_RAT', 'SEMIO_KRIT', 'SEMIO_DOM', 'SEMIO_KAEM', 'SEMIO_PFLICHT',
       'SEMIO_TRADV', 'ZABEOTYP']].mode()
    for idx in np.arange(24):
        feature_name = mode_of_int_azdias.columns[idx]
        mode = mode_of_int_azdias.values[0][idx]
        target_df[feature_name] = target_df[feature_name].fillna(mode)
    ### 3-3. Numeric features
    mode_of_float_azdias = target_df[target_df.describe().columns]
    for idx in np.arange(75):
        feature_name = mode_of_float_azdias.columns[idx]
        mode = mode_of_float_azdias.values[0][idx]
        target_df[feature_name] = target_df[feature_name].fillna(mode)
    print("3-3 ", target_df.shape)
    ## 4. Dividing PRAEGENDE_JUGENDJAHRE into two features
    ### 4-1. PRAEGENDE_JUGENDJAHRE => GENERATION | MOVEMENT
    praegende_jugendjahre = {
        "1": [4, 0],
        "2": [4, 1],
        "3": [5, 0],
        "4": [5, 1],
        "5": [6, 0],
        "6": [6, 1],
        "7": [6, 1],
        "8": [7, 0],
        "9": [7, 1],
        "10": [8, 0],
        "11": [8, 1],
        "12": [8, 0],
        "13": [8, 1],
        "14": [9, 0],
        "15": [9, 1],
    }
    generation = []
    movement = []

    def create_new_feature_with_praegende_jugendjahre(value):
        value_astype_str = str(int(value))
        generation.append(praegende_jugendjahre[value_astype_str][0])
        movement.append(praegende_jugendjahre[value_astype_str][1])

    target_df['PRAEGENDE_JUGENDJAHRE'].apply(create_new_feature_with_praegende_jugendjahre)
    print("4-1 GENERATION length: ",target_df.shape[0], len(generation))
    print("4-1 MOVEMENT length: ",target_df.shape[0], len(movement))
    target_df['GENERATION'] = generation
    target_df['MOVEMENT'] = movement
    ### 4-2. CAMEO_INTL_2015 => WEALTH | LIFE
    wealth_list = []
    life_list = []
    def create_new_feature_with_cameo_intl_2015(value):
        wealth_list.append(value[0])
        life_list.append(value[1])

    target_df['CAMEO_INTL_2015'].apply(create_new_feature_with_cameo_intl_2015)

    target_df['WEALTH'] = wealth_list
    target_df['LIFE'] = life_list
    ### 4-3. Drop original features, PRAEGENDE_JUGENDJAHRE and CAMEO_INTL_2015.
    target_df = target_df.drop(columns=['PRAEGENDE_JUGENDJAHRE'])
    target_df = target_df.drop(columns=['CAMEO_INTL_2015'])
    ## 5. Applying one-hot-encoding to categorical features
    target_df = pd.get_dummies(target_df)
    # Return the cleaned dataframe.
    return target_df
```

## 특성 변환

### 특성 스케일링을 수행하기

우선 결측치가 남아있는지를 확인합니다. 결측치가 없는 것을 확인했고 이제 스케일링을 수행할 수 있습니다.

```python
azdias_without_outlier.isna().sum().loc[isna_sum_series != 0]
```

```python
from sklearn.preprocessing import StandardScaler
scaler = StandardScaler()
X_scaled = scaler.fit_transform(azdias_without_outlier)
```

결측치가 가장 많은 6개의 특성을 제거하고, 남은 결측치를 최빈값으로 교체를 해 두었기 때문에 `StandardScaler`를 실행할 때 에러가 발생하지 않았습니다. feature scaling 으로 891221개의 행과 141개의 열을 가진 ndarray `X_scaled`를 얻었습니다.

### 차원 축소를 수행하기

차원 축소 알고리즘 중 하나인 주성분 분석(PCA)를 이용해 데이터의 분산이 최대인 벡터를 찾습니다.

우선 특성 전부를 주성분으로 분석해서 최적의 n_components 를 찾아봅니다.

```python
from sklearn.decomposition import PCA

pca = PCA()
pca.fit(X_scaled)
X_pca = pca.transform(X_scaled)

print("original data.shape: ", X_scaled.shape)
print("reduced data.shape: ", X_pca.shape)
print("PCA.components._shape: ", pca.components_.shape)
# 설명된 분산의 비율을 그래프로 그려 적절한 주성분의 개수를 확인합니다.
plt.plot(pca.explained_variance_ratio_)
plt.show()
```

확인한 결과 20개의 주성분이 대부분의 분산을 표현하고 있었습니다. 이번에는 n_components 를 20으로 지정하고 PCA 를 실행하겠습니다.

```python
pca = PCA(n_components=20)
pca.fit(X_scaled)

X_pca = pca.transform(X_scaled)

print('분산을 얼마나 유지하고 있는지의 비율: ', np.sum(pca.explained_variance_ratio_)) # 0.53
```

### 주성분을 해석하기

주성분 3개의 가중치를 확인하고 정렬하여 관찰해봅니다. 주성분의 가중치를 특성의 이름과 연결시킨 데이터프레임을 만드는 함수를 작성하고, 주성분 3개를 입력해 결과를 얻습니다.

```python
def create_pca_weights_df(pca_idx):
    weights_df = pd.DataFrame(pca.components_[pca_idx], index=azdias_without_outlier.columns,
                             columns=['weights'])
    weights_df.sort_values(ascending=False, by='weights', inplace=True)
    return weights_df

first_weights = create_pca_weights_df(0)
second_weights = create_pca_weights_df(1)
third_weights = create_pca_weights_df(2)
```

여기서 상위 10개의 가중치를 출력해 비교를 했습니다.

```python
first_weights.iloc[:10]
second_weights.iloc[:10]
third_weights.iloc[:10]
```

첫 번째 주성분의 가중치는 상위 10개 모두 0.13 ~ 0.19 사이로 고른 분포를 보인 반면, 나머지 두 주성분의 가중치는 일부만 큰 가중치를 보이며 나머지는 0.1 미만을 가지고 있었습니다. 가중치의 분포를 비교했을 때, 오직 "PLZ8_ANTG4"만 세 주성분 모두 가지고 있는 feature 였습니다. 그리고 두 주성분이 가지고 있는 feature 는 "PLZ8_BAUMAX", "ORTSGR_KLS9", "FINANZ_HAUSBAUER", "HH_EINKOMMEN_SCORE" 였습니다. 그만큼 세 주성분이 중요하다고 생각하는 특성은 달랐습니다.

```python
print('first - (> 0, <= 0)',
    (len(first_weights.loc[first_weights['weights'] > 0]), len(first_weights.loc[first_weights['weights'] <= 0])))
print('first - (> 0, <= 0)',
    (len(second_weights.loc[second_weights['weights'] > 0]), len(second_weights.loc[second_weights['weights'] <= 0])))
print('first - (> 0, <= 0)',
    (len(third_weights.loc[third_weights['weights'] > 0]), len(third_weights.loc[third_weights['weights'] <= 0])))
```

세 주성분 모두 positive weight 와 negative weight 의 개수가 비슷합니다. 즉 모든 특성들 사이에 공통의 상호관계가 있는지, 그리고 각 특성들이 어떤 관계성을 띄고 있는지를 명확히 설명하기가 어렵다는 뜻입니다.

## 군집 알고리즘

### 인구 통계 데이터에 군집 알고리즘을 적용하기

k-평균 알고리즘을 이용해서 PCA 로 변환한 데이터에 대해 군집화를 시도합니다.

우선 최적의 클러스터 수(n_clusters)를 얻기 위해, 반복문으로 10개 ~ 30개 사이의 n_clusters 로 k-평균 알고리즘의 점수를 구합니다.

```python
cluster_range = np.arange(10, 30)

inertia = []
total_score = []

for k in range(10, 30):
    km = KMeans(n_clusters=k, random_state=0)
    km.fit(X_pca)
    inertia.append(km.inertia_)
    total_score.append(km.score(X_pca))

print("average within-cluster distances: ", total_score / len(total_score))
```

**이너셔**를 그래프로 그려 최적의 클러스터 수를 찾아봅니다. 이너셔는 클러스터 중심과 클러스터에 속한 샘플 사이의 거리의 제곱 합입니다. 이너셔로 클러스터에 속한 샘플이 얼마나 가깝게 모여있는지를 알 수 있으며, 이를 통해 최적의 클러스터를 찾을 수 있습니다.

클러스터의 개수 k 를 X 축, 이너셔를 Y 축으로 잡은 그래프에서 선이 꺾이기 시작하는 지점을 최적의 클러스터 개수로 잡고, k-means 모델을 만들어 예측을 수행했습니다.

```python
plt.figure(figsize=(15, 6))
plt.plot(range(10, 30), inertia)
plt.title('Inertia of K-Means')
plt.xlabel('K')
plt.ylabel('inertia')
plt.show()
```

기울기가 꺾이는 부분인 0이 최적의 클러스터 개수입니다. 이제 n_clusters 를 0개로 해서 모델을 만들고 인구 통게 데이터에 대해 예측을 수행합니다.

```python
km = KMeans(n_clusters=k, random_state=0)
km.fit(X_pca)
km_pred = km.predict(X_pca)
print(km_pred)
```

### 고객 데이터에 모든 과정을 수행하기

Udacity_CUSTOMERS_Subset.csv 에서 예측을 수행하기 위한 고객 데이터를 불러온 후, 학습했던 k-평균 모델으로 예측을 수행합니다.

```python
customers = pd.read_csv('Udacity_CUSTOMERS_Subset.csv', sep=';')
costomers_df = clean_data(customers)

km_customer_pred = km_customer.predict(costomers_df.to_numpy())
print("Prediction for the customer demographics data:\n", km_customer_pred)
```

### 고객 데이터와 인구 통계 데이터를 비교하기

이번에는 독일의 인구 통계 데이터와 메일-주문 회사의 고객 데이터 두 개의 군집을 비교해서 회사에게 가장 강력한 고객의 기반이 어디인지를 확인합니다.

1. 고객 데이터와 인구 통계 데이터에 대해 각각의 클러스터 데이터 비율을 비교해봅니다. `countplot`으로 전체 개수를 보여주는 대신, 클러스터가 데이터에서 차지하는 비중을 퍼센트르 보여주는 `barplot`으로 작성했습니다. 참고한 코드는 [Add percentages instead of counts to countplot #1027](https://github.com/mwaskom/seaborn/issues/1027#issuecomment-250456545)에서 확인하실 수 있습니다.

```python
df_azdias_pred = pd.DataFrame(dict(x=km_azdias_pred))
df_customer_pred = pd.DataFrame(dict(x=km_customer_pred))

fig, axs = plt.subplots(1, 2, figsize=(17, 12))

sns.barplot(x='x', y='x', data=df_azdias_pred, ax=axs[0],
           estimator=lambda x: len(x) / len(df_azdias_pred) * 100)
axs[0].set_title("Propotion of data for general population")

sns.barplot(x='x', y='x', data=df_customer_pred, ax=axs[1],
           estimator=lambda x: len(x) / len(df_customer_pred) * 100)
axs[1].set_title("Propotion of data for customers")
plt.show()
```

2. 인구 통계 데이터에 비해 고객 데이터에서 과대평가되는 클러스터에 속한 사람들이 누구인지를 찾아봅니다.

인구 통계 데이터에 비해 고객 데이터에서 확연히 높은 부분은 3번 클러스터입니다. 우선 이 클러스터를 `inverse_transform()`으로 복원합니다. 그 다음 복원한 ndarray 로 데이터프레임을 만들고, 3번 클러스터에 대한 정보만을 추출하여 `describe()`로 요약된 정보를 얻습니다.

```python
reversed_azdias = pca.inverse_transform(X_pca)
reversed_customer = pca.inverse_transform(target_pca)

reversed_azdias_df = pd.DataFrame(reversed_azdias, columns=azdias_without_outlier.columns)
reversed_customer_df = pd.DataFrame(reversed_customer, columns=azdias_without_outlier.columns)

cluster_3_azdias_describe = reversed_azdias_df.iloc[cluster_3_azdias_idx].describe()
cluster_3_customers_describe = reversed_customer_df.iloc[cluster_3_customers_idx].describe()
```

마지막으로 산점도를 그리는 함수를 작성해서 평균과 표준편차에 대한 그래프를 얻습니다.

```python
def create_cluster_info_scatter(title, first_y, second_y):
    plt.figure(figsize=(20,7))
    plt.title(title)
    plt.scatter(x=azdias_without_outlier.columns, y=first_y)
    plt.scatter(x=azdias_without_outlier.columns, y=second_y)
    plt.xticks([])
    plt.xlabel('features')
    plt.legend(['General population', 'Customers'])
    plt.show()

create_cluster_info_scatter("Overpresented Cluster 3 - Feature's mean",
                           cluster_3_azdias_describe.iloc[1], cluster_3_customers_describe.iloc[1])
```

![클러스터 3: 평균의 산점도](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-09-03-Identify-Customer-Segments_cluster3_mean.png?raw=true)

평균의 산점도를 보면 전반부에서는 간격이 좁게 뭉쳐있지만, 후반으로 갈수록 넓게 퍼지는 양상을 띄고 있습니다. 두 데이터를 비교했을 때 전체적인 흐름은 비슷하지만, 고객 데이터는 인구 통계 데이터에 비해 각 점들 사이의 거리가 더 가깝습니다.

```python
create_cluster_info_scatter("Overpresented Cluster 3 - Feature's std",
                           cluster_3_azdias_describe.iloc[2], cluster_3_customers_describe.iloc[2])
```

![클러스터 3: 표준편차의 산점도](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-09-03-Identify-Customer-Segments_segments_cluster3_std.png?raw=true)

표준편차의 산점도에서는 고객의 데이터는 인구 통계 데이터보다 전반적으로 아래에 있는 경우가 많았고, 간격의 높낮이도 인구 통계 데이터보다 완만합니다.

3. 인구 통계 데이터에 비해 고객 데이터에서 과도하게 저평가되는 클러스터에 속한 사람들이 누구인지를 찾아봅니다.

2, 5, 9, 11에서 가장 차이가 크게 나는 것은 9번 클러스터로 9번 클러스터에 대한 평균의 산점도, 표준편차의 산점도를 그려봅니다.

```python
create_cluster_info_scatter("Overpresented Cluster 9 - Feature's mean",
                           cluster_9_azdias_describe.iloc[1], cluster_9_customers_describe.iloc[1])
```

![클러스터 9: 평균의 산점도](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-09-03-Identify-Customer-Segments_cluster9_mean.png?raw=true)

3번 클러스터와 달리 전반적으로 고르게 분포된 모습을 보입니다. 또한 고객의 데이터가 인구 통계 데이터보다 점 사이의 간격이 좁고 뭉쳐있다는 점 역시 비슷합니다.

```python
create_cluster_info_scatter("Overpresented Cluster 3 - Feature's std",
                           cluster_9_azdias_describe.iloc[2], cluster_9_customers_describe.iloc[2])
```

![클러스터 9: 표준편차의 산점도](https://github.com/chinsanchung/chinsanchung.github.com/blob/master/assets/images/2021-09-03-Identify-Customer-Segments_cluster9_std.png?raw=true)

표준편차의 산점도 역시 전반적인 흐름은 3번 클러스터와 비슷하지만, 9번 클러스터의 고객 데이터의 표준편차는 3번보다 가파른 높낮이를 보이고 있습니다.

## 프로젝트를 마치며

지금까지 했던 프로젝트와 달리 실제 독일의 인구 통계 데이터로 전처리에서부터 예측까지 모든 과정을 거치며 비지도 학습을 수행할 수 있었습니다. 어려웠던 점은 우선 책이나 강의에서 다뤘던 알기 쉽고 수가 적었던 특성들이, 이번 데이터에서는 특성이 85개에 되도록 많은 정보를 담기 위해 한 특성에 두 세개의 정보를 압축해 이해하기 어려웠던 것입니다. 이해하기 어려운 상태에서 전처리를 하고 비교를 하며 난감함을 느꼈지만, 실무에서는 다루는 독립 변수만 몇 백개가 넘는 경우가 많다고 들었기에 실무에 대해 조금이나마 겉핥기를 했다고 생각합니다. 비록 인구 통계에 대한 전문 용어나 지식은 부족했던 점, 그리고 통계학적인 지식이 부족해 전처리와 데이터 간 비교를 매끄럽게 수행하지 못해 아직 많이 부족하다는 것을 느꼈습니다.
