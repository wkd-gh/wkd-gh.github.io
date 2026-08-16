---
title: Sensor, BranchOperator 그리고 HiveOperator vs PrestoOperator
date: 2026-08-16 15:00:00 +0900
categories: [Data Engineering, Airflow]
tags: [Airflow, Hive, Presto]
image: thumbnail.png
media_subpath: /assets/img/posts/2026-08-16-airflow-sensor-branch-hive-presto-operator/
---

> Airflow DAG를 작성하다 보면 자주 쓰게 되는 Sensor, BranchOperator, 그리고 헷갈리기 쉬운 HiveOperator와 PrestoOperator의 차이를 정리해본 글입니다.

이 글에서 예시로 드는 파이프라인은 대략 아래와 같은 구조다. 상류 DAG을 기다렸다가(Sensor), 조건에 따라 분기하고(Branch), Hive로 무거운 적재를 한 뒤 Presto로 빠르게 검증하고, 두 분기가 결과를 각각 다시 하나의 task로 합류(trigger_rule)하는 흐름이다.

![Airflow Sensor → Branch → Hive/Presto 예시 파이프라인](pipeline_dag.svg){: width="1220" height="420" }
_Sensor → Branch → Hive/Presto Operator → trigger_rule 합류까지 이어지는 예시 파이프라인_

### <i class="fas fa-clock fa-fw"></i> **Sensor: 조건이 될 때까지 기다리기**

Sensor는 특정 조건이 충족될 때까지 대기하다가, 조건이 만족되면 성공 처리되는 특수한 Operator다. 상류 시스템의 파일 도착, 다른 DAG의 완료 등을 기다릴 때 쓴다.

동작 방식은 `mode` 파라미터로 결정된다.

- **`poke` 모드(기본값)** : 워커 슬롯을 계속 점유한 채로 주기적으로 조건을 확인한다. 대기 시간이 길어지면 워커 리소스를 낭비하게 된다.
- **`reschedule` 모드** : 조건이 충족되지 않으면 워커 슬롯을 반납하고, 다음 체크 시점에 다시 스케줄링된다. 대기 시간이 긴 센서라면 이쪽이 훨씬 효율적이다.

```python
from airflow.sensors.external_task import ExternalTaskSensor

wait_for_upstream = ExternalTaskSensor(
    task_id="wait_for_upstream",
    external_dag_id="etf_price_info",
    external_task_id="load_to_databricks",
    mode="reschedule",
    timeout=60 * 60,
    poke_interval=60,
)
```

### <i class="fas fa-code-branch fa-fw"></i> **BranchPythonOperator: 조건 분기**

파이프라인 중간에 조건에 따라 서로 다른 downstream task를 태우고 싶을 때 쓴다. 함수가 반환하는 `task_id`(또는 `task_id` 리스트)를 따라 그 경로만 실행되고, 나머지 분기는 자동으로 `skipped` 처리된다.

```python
from airflow.operators.python import BranchPythonOperator

def choose_branch(**context):
    row_count = context["ti"].xcom_pull(task_ids="check_row_count")
    return "load_partition" if row_count > 0 else "skip_notification"

branch = BranchPythonOperator(
    task_id="branch_by_row_count",
    python_callable=choose_branch,
)
```

여기서 주의할 점은 **trigger_rule**이다. 분기 이후 다시 하나의 task로 합류하는 지점이 있다면, 기본값인 `all_success` 때문에 스킵된 분기가 있는 순간 downstream이 전부 `upstream_failed`가 될 수 있다. 이럴 땐 합류 지점 task의 `trigger_rule`을 `none_failed_min_one_success`로 바꿔줘야 스킵된 분기를 무시하고 정상적으로 진행된다.

### <i class="fas fa-database fa-fw"></i> **HiveOperator vs PrestoOperator**

이름만 보면 둘 다 "쿼리를 실행하는 Operator"라 비슷해 보이지만, 내부적으로 호출하는 엔진의 성격이 완전히 다르다.

| 항목 | HiveOperator | PrestoOperator (Trino 포함) |
| --- | --- | --- |
| 실행 엔진 | MapReduce / Tez / Spark | MPP 기반 분산 쿼리 엔진 |
| 처리 방식 | 디스크 기반, 단계별(stage) 처리 | 인메모리 기반, 파이프라인 처리 |
| 적합한 워크로드 | 대용량 배치 ETL, 무거운 조인/집계 | 짧고 빠른 대화형 쿼리, 검증성 조회 |
| 장애 내성 | 중간 단계 재시도에 강함 | 쿼리 하나가 실패하면 처음부터 재시도 |
| 지연 시간 | 상대적으로 김 (수 분~수십 분) | 상대적으로 짧음 (초~수 분) |

```python
from airflow.providers.apache.hive.operators.hive import HiveOperator

load_partition = HiveOperator(
    task_id="load_partition",
    hive_cli_conn_id="hive_default",
    hql="""
        INSERT OVERWRITE TABLE etf_price_info PARTITION (basDt='{{ ds_nodash }}')
        SELECT * FROM stg_etf_price_info WHERE basDt = '{{ ds_nodash }}';
    """,
)
```

```python
from airflow.providers.presto.operators.presto import PrestoOperator

validate_row_count = PrestoOperator(
    task_id="validate_row_count",
    presto_conn_id="presto_default",
    sql="SELECT count(*) FROM etf_price_info WHERE basDt = '{{ ds_nodash }}'",
)
```

Connection 설정도 다르다. HiveOperator는 HiveServer2/Metastore Thrift 연결을, PrestoOperator는 Presto/Trino 클러스터의 HTTP(S) 엔드포인트를 사용한다. 그래서 같은 DAG 안에서도 목적에 따라 두 Connection을 함께 등록해두는 경우가 많다.

### <i class="fas fa-scale-balanced fa-fw"></i> **그래서 언제 무엇을 쓰는지?**

내가 배치를 짤 때 기준으로 삼는 건 이렇다.

- **야간에 대용량 데이터를 적재/변환하는 무거운 배치** → `HiveOperator`. 실행 시간이 길어도 상관없고, 중간 stage 실패 시 재시도 내성이 중요한 작업.
- **적재 이후 결과를 검증하거나, 짧은 집계로 상태를 확인하는 작업** → `PrestoOperator`. 응답이 빨라야 DAG 전체 실행 시간에 영향을 덜 준다.

같은 파이프라인 안에서도 "적재는 Hive, 검증은 Presto"처럼 엔진을 나눠 쓰는 패턴이 자주 나오는 이유가 여기에 있다.

### <i class="fas fa-flag-checkered fa-fw"></i> **정리**

Sensor와 BranchOperator는 파이프라인의 흐름 제어를, HiveOperator와 PrestoOperator는 어떤 엔진으로 쿼리를 실행할지를 정하는 역할이다. 네 가지 모두 자주 쓰이는 만큼, 각각이 내부적으로 무엇을 기다리고 어떤 엔진을 호출하는지 정확히 이해하고 쓰는 것이 불필요한 재시도나 리소스 낭비를 줄이는 데 도움이 될 것 같다.
