---
title: 왜 다들 파케이, 파케이 하는 걸까
date: 2026-08-20 20:00:00 +0900
categories: [Data Engineering, Data]
tags: [Parquet, CSV, JSON, ORC, Arrow, File Format]
image: thumbnail.png
media_subpath: /assets/img/posts/2026-08-20-why-parquet/
---

> 데이터 관련 직군이라면 한 번쯤은 들어봤을 파일 포맷, Parquet. 성능을 따지기 시작하면 왜 항상 파케이가 정답처럼 나오는지 정리해본 글입니다.

### <i class="fas fa-question fa-fw"></i> **왜 다들 파케이를 쓸까**

데이터 엔지니어링 쪽 채용 공고나 기술 블로그를 보면 십중팔구 Parquet이 등장한다. CSV나 JSON으로 시작한 프로젝트도 어느 정도 규모가 커지면 결국 Parquet으로 옮겨가는 경우가 많다. 이유는 결국 하나로 모인다 — **저장 방식 자체가 분석 워크로드에 최적화되어 있기 때문**이다.

### <i class="fas fa-table-columns fa-fw"></i> **Row 기반 vs Column 기반**

CSV, JSON, XML은 전부 **Row 기반(row-oriented)** 포맷이다. 한 행(row)의 모든 컬럼값이 디스크에 연달아 저장된다. 반면 Parquet은 **Column 기반(column-oriented)** 포맷이라, 같은 컬럼의 값들끼리 모아서 저장한다.

![Row-oriented vs Column-oriented 저장 방식](row_vs_column_storage.svg){: width="1000" height="560" }
_같은 데이터를 Row 기반과 Column 기반으로 각각 저장했을 때의 차이_

이 차이 하나가 아래에서 설명할 성능·용량 차이 대부분의 원인이 된다.

### <i class="fas fa-gauge-high fa-fw"></i> **그래서 뭐가 좋은데**

**1. 압축 효율이 훨씬 좋다**

같은 타입, 심지어 비슷한 값들끼리 연달아 저장되기 때문에 압축 알고리즘이 훨씬 잘 먹힌다. 이름 컬럼이면 이름끼리, 날짜 컬럼이면 날짜끼리 모여 있으니 반복되는 패턴을 찾기가 쉬워진다. 여기에 Parquet은 Dictionary Encoding(반복되는 값을 정수 인덱스로 치환), RLE(Run-Length Encoding, 같은 값의 반복 횟수만 저장) 같은 컬럼형 인코딩까지 기본으로 적용해서, 같은 데이터라도 CSV보다 파일 크기가 훨씬 작아진다.

**2. 필요한 컬럼만 읽는다 (Projection Pushdown)**

CSV나 JSON은 컬럼 하나만 조회하려 해도 파일 전체를 처음부터 끝까지 읽어야 한다. Parquet은 컬럼별로 물리적으로 분리 저장돼 있어서, 쿼리에 필요한 컬럼의 데이터만 골라 읽고 나머지는 아예 디스크 I/O 자체가 발생하지 않는다.

**3. 필요 없는 데이터 블록은 아예 건너뛴다 (Predicate Pushdown)**

Parquet은 컬럼 값의 최소/최대값 같은 통계를 메타데이터에 저장해둔다. `WHERE age > 30` 같은 조건이 들어오면, 통계상 조건을 만족할 수 없는 블록은 실제로 읽지도 않고 통째로 건너뛴다.

실제로 AWS가 공개한 벤치마크를 보면 체감이 확실하다. 1TB짜리 CSV 데이터를 Athena로 스캔하면 1.15TB를 읽고 약 $6이 청구되는데, 같은 데이터를 Parquet으로 변환하면 파일 크기는 130GB로 줄고 실제 스캔량은 2.72GB, 비용은 $0.03 수준까지 떨어진다. 스캔량 기준 99% 이상 절감이다 ([AWS Big Data Blog 참고](https://www.cloudforecast.io/using-parquet-on-athena-to-save-money-on-aws/)).

### <i class="fas fa-folder-tree fa-fw"></i> **Parquet 파일 내부 구조**

Parquet 파일 하나는 이런 계층 구조로 되어 있다.

![Parquet 파일 내부 구조](parquet_file_structure.svg){: width="900" height="560" }
_Row Group → Column Chunk → Page, 그리고 이걸 가리키는 Footer_

- **Row Group** : 일정 개수의 row를 묶은 단위. 파일 하나에 여러 개 존재한다.
- **Column Chunk** : 하나의 Row Group 안에서, 컬럼 하나에 해당하는 데이터 뭉치.
- **Page** : Column Chunk를 더 잘게 쪼갠 실제 압축·인코딩 단위.
- **Footer** : 파일 맨 끝에 스키마 정보와 각 Row Group/Column Chunk의 통계(min/max, null count), 오프셋 위치가 담겨 있다. 엔진은 파일을 열 때 이 Footer부터 읽어서 "어디를 읽고 어디를 건너뛸지" 미리 계획을 세운다.

### <i class="fas fa-scale-balanced fa-fw"></i> **CSV / JSON / XML / ORC와 비교하면**

| 항목 | CSV | JSON | XML | ORC | Parquet |
| --- | --- | --- | --- | --- | --- |
| 저장 방식 | Row 기반 | Row 기반 | Row 기반 | Column 기반 | **Column 기반** |
| 스키마 | 없음 (전부 문자열) | 느슨함 (타입은 있지만 강제 안 됨) | 있음 (태그 기반) | 있음, 파일에 내장 | **있음, 파일에 내장 + 강타입** |
| 압축 효율 | 낮음 | 낮음 | 매우 낮음 (태그 오버헤드) | 높음 | **높음** |
| 컬럼 일부만 조회 | 불가 (전체 스캔) | 불가 | 불가 | 가능 | **가능** |
| 사람이 직접 읽기 | 쉬움 | 쉬움 | 쉬움 | 어려움 (바이너리) | 어려움 (바이너리) |
| 대표 사용처 | 엑셀 내보내기, 간단한 로그 | API 응답, 설정 파일 | 레거시 시스템 연동 | Hive/Hadoop 생태계 | 데이터 웨어하우스, 범용 분석 워크로드 |

표로 정리해놓고 보면 명확하다. CSV/JSON/XML은 **"사람이 읽기 좋고, 시스템 간 주고받기 편한"** Row 기반 포맷이고, ORC와 Parquet은 **"기계가 대량으로 분석하기 좋은"** Column 기반 포맷이다. 애초에 설계 목적 자체가 다르기 때문에, 사실 이 비교는 Parquet 입장에서 좀 불공평한 승부다.

진짜 라이벌은 따로 있다 — 같은 컬럼 기반인 ORC다.

### <i class="fas fa-people-arrows fa-fw"></i> **그럼 ORC랑 붙으면? Arrow는 뭔데?**

**ORC (Optimized Row Columnar)** 도 Parquet와 마찬가지로 컬럼 기반 저장, 컬럼 통계 기반 Predicate Pushdown, 압축·인코딩을 전부 지원한다. 원래 Hive 성능 개선을 위해 만들어진 포맷이라, 태생부터 Hive/Hadoop 생태계에 가깝다.

실제 벤치마크들을 찾아보면 우열이 뚜렷하지 않다. 처리 엔진에 따라 갈리는데, Hive 위에서는 ORC가, SparkSQL 위에서는 Parquet가 더 좋은 성능을 보이는 경향이 있다. 압축률도 조건에 따라 뒤집힌다 — ZLIB 압축을 쓴 ORC가 풀 스캔 기준 최대 60% 수준까지 개선되는 경우가 있는 반면, Parquet은 기본 압축(Snappy) 기준 개선 폭이 7% 정도에 그친다는 벤치마크도 있다. 반대로 Parquet의 Dictionary Encoding이 더 공격적이라 파일 크기가 더 작게 나온 사례도 있다. 즉 "Parquet가 저장 방식 자체 때문에 압도적으로 이긴다"는 CSV/JSON/XML 비교와는 다르게, ORC와는 진짜 대등한 싸움이라는 게 맞는 표현이다.

그럼 실무에서는 왜 Parquet 쪽으로 더 많이 쓰일까. 성능보다는 **생태계** 문제에 가깝다. Parquet은 Spark의 기본 저장 포맷이고, Delta Lake·Iceberg 같은 오픈 테이블 포맷의 기본 파일 포맷이며, BigQuery·Snowflake·Athena·Redshift Spectrum 등 대부분의 클라우드 데이터 웨어하우스가 네이티브로 지원한다. ORC는 여전히 강력하지만 상대적으로 Hive/Hadoop 생태계 안에서 더 존재감이 크다.

**Apache Arrow**는 아예 다른 카테고리다. Arrow는 애초에 **디스크에 압축해서 저장하기 위한 포맷이 아니라, 메모리 위에서 여러 프로세스/언어가 데이터를 복사 없이 주고받기 위한 인메모리(in-memory) 포맷**이다. Parquet·ORC가 "디스크에 최대한 작게, 효율적으로 저장하기"가 목표라면, Arrow는 "메모리에 올라온 데이터를 디코딩 과정 없이 최대한 빠르게 처리하기"가 목표다. 그래서 Arrow 파일은 압축을 거의 하지 않고, 디스크 상의 표현과 메모리 상의 표현이 동일하게 설계되어 있다.

실제로는 둘이 경쟁하기보다 함께 쓰인다. Spark·DuckDB·Polars 같은 엔진들이 디스크의 Parquet 파일을 읽어서, 실제 연산은 메모리에 Arrow 포맷으로 올려놓고 처리하는 식이다. "저장은 Parquet, 연산은 Arrow"인 셈이라, 애초에 같은 문제를 놓고 겨루는 관계가 아니다.

### <i class="fas fa-circle-question fa-fw"></i> **그럼 CSV·JSON은 언제 쓰나**

Parquet가 항상 정답인 건 아니다.

- **사람이 직접 열어봐야 하는 경우** : 엑셀로 열어서 확인하거나 공유해야 한다면 CSV가 압도적으로 편하다.
- **API 요청/응답, 설정 파일** : 구조가 자주 바뀌고 사람이 직접 다뤄야 하는 상황에는 JSON이 낫다.
- **데이터 양이 아주 작은 경우** : 몇백 줄짜리 로그 파일에 컬럼형 저장의 이점은 거의 없고, 오히려 바이너리라 디버깅만 불편해진다.
- **레거시 시스템 연동** : XML 기반 인터페이스를 쓰는 오래된 시스템과 맞춰야 한다면 선택지가 없는 경우도 많다.

반대로 **적재 후 반복적으로 분석 쿼리를 돌리는 데이터 웨어하우스, 데이터 레이크 환경**이라면 Parquet 쪽이 거의 항상 유리하다. 한 번 쓰고 여러 번 읽는(write-once, read-many) 워크로드일수록 그 격차가 커진다.

### <i class="fas fa-flag-checkered fa-fw"></i> **정리**

결국 핵심은 하나다. Row 기반 포맷은 "한 건을 통째로 다루기"에 최적화돼 있고, Column 기반인 Parquet은 "특정 컬럼들을 대량으로 훑기"에 최적화돼 있다. 분석 워크로드는 거의 항상 후자에 해당하기 때문에, 데이터 엔지니어링 생태계에서 Parquet가 사실상 기본값이 된 거라고 생각한다.
