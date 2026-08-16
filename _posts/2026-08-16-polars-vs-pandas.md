---
title: pandas 대신 Polars, 써볼 만할까?
date: 2026-08-16 13:00:00 +0900
categories: [Data Engineering, Python]
tags: [Python, Pandas, Polars]
image: thumbnail.png
media_subpath: /assets/img/posts/2026-08-16-polars-vs-pandas/
---

> 최근 커뮤니티에서 유독 자주 보이는 Polars, 실제로 Pandas와 뭐가 어떻게 다른건지 정리해본 글입니다.

#### <i class="fas fa-question fa-fw"></i> **왜 지금 Polars인가**

Pandas는 오래 써왔지만, 데이터가 커지면 단일 스레드 기반 eager 실행 방식 때문에 체감 속도가 뚝뚝 떨어지는 경험을 다들 한 번쯤 해봤을 것이다. Polars는 이 지점을 정면으로 겨냥한 라이브러리다.

- **Rust로 작성** : GIL의 영향을 받지 않고 진짜 멀티스레드 병렬 처리가 가능하다.
- **Apache Arrow 기반 컬럼형 메모리 포맷** : 캐시 친화적인 메모리 레이아웃으로 집계·필터 연산이 빠르다.
- **Lazy evaluation 지원** : 연산을 즉시 실행하지 않고 쿼리 플랜을 세운 뒤 최적화해서 한 번에 실행할 수 있다.

#### <i class="fas fa-bolt fa-fw"></i> **핵심 차이**

| 항목 | Pandas | Polars |
| --- | --- | --- |
| 실행 모델 | Eager (즉시 실행) | Eager + Lazy 둘 다 지원 |
| 병렬 처리 | 기본적으로 단일 스레드 | 기본 멀티스레드 |
| 메모리 포맷 | NumPy 기반 | Apache Arrow 기반 |
| Null 처리 | `NaN`(실수 컬럼에 한정) | 타입 무관 `null` |
| API 스타일 | 인덱스 기반 접근, 다양한 문법 | 메서드 체이닝 중심, 일관된 표현식(Expression) API |

가장 크게 체감되는 차이는 **인덱스의 유무**다. Pandas는 행 인덱스를 항상 신경 써야 하지만, Polars는 인덱스 개념이 없고 컬럼 중심으로만 다룬다. 그만큼 `reset_index()` 같은 코드가 사라진다.

#### <i class="fas fa-code fa-fw"></i> **코드로 비교하기**

동일한 작업(조건 필터 → 그룹별 집계)을 두 라이브러리로 각각 짜보면 차이가 명확하다.

```python
# Pandas
import pandas as pd

df = pd.read_csv("etf_price_info.csv")
result = (
    df[df["fltRt"] > 0]
    .groupby("mrktCtg")["trPrc"]
    .sum()
    .reset_index()
)
```

```python
# Polars (Eager API)
import polars as pl

df = pl.read_csv("etf_price_info.csv")
result = (
    df.filter(pl.col("fltRt") > 0)
    .group_by("mrktCtg")
    .agg(pl.col("trPrc").sum())
)
```

문법 자체는 크게 낯설지 않다. `pl.col(...)`이라는 표현식(Expression)을 통해 연산을 선언한다는 점이 Pandas와의 가장 큰 문법적 차이다.

#### <i class="fas fa-gauge-high fa-fw"></i> **진짜 강점은 Lazy API**

Polars의 진가는 `scan_*` 계열 함수와 `.lazy()`를 쓸 때 드러난다. Pandas는 한 줄씩 즉시 실행되지만, Polars는 전체 쿼리를 계획으로 모아뒀다가 최적화 후 한 번에 실행한다.

![Pandas Eager vs Polars Lazy 실행 흐름](eager_vs_lazy.svg){: width="980" height="560" }
_Pandas(Eager) vs Polars(Lazy) 실행 흐름 비교_

```python
result = (
    pl.scan_csv("etf_price_info.csv")   # 파일을 즉시 읽지 않고 쿼리 플랜만 구성
    .filter(pl.col("fltRt") > 0)
    .group_by("mrktCtg")
    .agg(pl.col("trPrc").sum())
    .collect()                          # 이 시점에 최적화된 플랜으로 한 번에 실행
)
```

`.collect()`를 호출하기 전까지는 실제 연산이 일어나지 않는다. 그 사이에 Polars 옵티마이저가 **predicate pushdown**(필터를 최대한 앞당겨서 읽어야 할 데이터 자체를 줄임), **projection pushdown**(실제로 쓰는 컬럼만 읽음) 같은 최적화를 자동으로 적용한다. `.explain()`으로 최적화된 실행 계획을 직접 확인할 수도 있다.

#### <i class="fas fa-triangle-exclamation fa-fw"></i> **아직은 고려해야 할 점**

- **생태계 성숙도** : `matplotlib`, `scikit-learn` 등 기존 데이터 분석 생태계 상당수가 Pandas DataFrame을 전제로 만들어져 있다. Polars 결과를 넘길 땐 `.to_pandas()` 변환이 필요한 경우가 여전히 많다.
- **팀 러닝커브** : 인덱스가 없고 표현식 기반이라는 점이 Pandas에 익숙한 팀원에게는 낯설 수 있다.
- **일부 기능 공백** : Pandas만큼 오래된 라이브러리가 아니다 보니, 아주 지엽적인 함수는 아직 없는 경우도 있다.

#### <i class="fas fa-flag-checkered fa-fw"></i> **정리**

데이터가 메모리에 여유 있게 들어가는 작은 규모의 탐색적 분석이라면 굳이 Pandas를 버릴 이유는 없다. 하지만 수백만 행 이상의 배치 집계, 혹은 파이프라인 안에서 반복적으로 도는 무거운 전처리 로직이라면 Polars의 Lazy API로 옮겨볼 만한 가치가 충분하다고 생각한다. 다음에는 실제 배치 파이프라인 하나를 Pandas → Polars로 교체하면서 처리 시간이 얼마나 줄어드는지 직접 재보고 싶다.
