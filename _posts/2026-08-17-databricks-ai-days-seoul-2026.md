---
title: Databricks AI Days Seoul 2026 참관 후기
date: 2026-08-17 00:30:00 +0900
categories: [Daily Life, Review]
tags: [Databricks, AI Days Seoul, Review]
image: thumbnail.png
media_subpath: /assets/img/posts/2026-08-17-databricks-ai-days-seoul-2026/
---

> Databricks에서 주최한 AI Days Seoul 2026에 다녀와 당시에 적어뒀던 메모들을 개인적인 생각으로 정리한 글입니다.

![AI Days Seoul 2026 전체 아젠다](event_agenda.png){: width="2000" height="1732" }
_AI Days Seoul 2026 전체 아젠다_

오전 키노트(Democratizing Data+AI)를 시작으로, 오후에는 Track별 고객사례 세션과 Analytics/AI Agent/Apps 기술 세션이 이어지는 하루짜리 컨퍼런스였다. 이 글에는 그중 인상 깊었던 고객사례 세션 위주로 정리했다.

#### <i class="fas fa-bag-shopping fa-fw"></i> **카카오스타일 — 두집살림으로 천천히 이관하기**

카카오스타일의 기존 아키텍처는 전체적으로 AWS 기반이었다. Airflow는 EKS 위에서 돌아가고, Apache Hudi로 CDC를 처리하며, 파이썬 스크립트는 JupyterHub에서 개발·운영하는 구조였다.

![카카오스타일 기존 아키텍처](kakaostyle_as_is.jpg){: width="1600" height="900" }
_카카오스타일 기존(AS-IS) 아키텍처_

인상 깊었던 부분은 to-be 전략이었다. 한 번에 Databricks로 마이그레이션하기엔 리스크가 크다고 판단해서, 기존 스택과 Databricks를 **두집살림 체제로 병행**하면서 천천히 이관하기로 했다고 한다. 대시보드는 앞으로도 redash를 메인으로 쓸 예정이라고 하는데, 마침 최근 Databricks가 redash를 인수하면서 이 선택이 자연스럽게 로드맵 안에 들어오게 된 점도 흥미로웠다.

![카카오스타일 목표(TO-BE) 아키텍처](kakaostyle_to_be.jpg){: width="1600" height="900" }
_카카오스타일 목표(TO-BE) 아키텍처, Databricks와 기존 AWS 스택을 병행_

#### <i class="fas fa-layer-group fa-fw"></i> **NOL 유니버스 — Databricks 없이 만들었다면**

NOL 유니버스 세션은 "Databricks 없이 이 시스템을 만들어야 했다면?"이라는 질문으로 시작했다. 실제로 Databricks 없이 동일한 데이터 플랫폼을 구성하려면, 모델 학습·배치·공통 인프라·런타임 영역마다 SageMaker, EMR, MWAA, Athena, Lambda, API Gateway, Secrets Manager처럼 서로 다른 서비스를 각각 학습하고 운영해야 한다.

![Databricks 없이 이 시스템을 만들었다면](nol_universe_without_databricks.jpg){: width="1600" height="900" }
_Databricks 없이 이 시스템을 만들었다면 필요했을 서비스들_

소수 인원으로 운영하는 조직 입장에서는 이 정도 서비스를 각각 다 학습하고 운영할 리소스가 부족할 수밖에 없다는 게 요지였다. 반면 Databricks를 쓰면 Unity Catalog 안에 데이터를 모아두고, Workflow로 Feature·Segment·Machine Learning·Statistics 단계를 원하는 대로 조합해서 Segment API와 추천 모델까지 이어지는 구조를 훨씬 단순하게 구현할 수 있었다고 한다.

![NOL 유니버스 Overall Architecture](nol_universe_architecture.jpg){: width="1600" height="900" }
_Unity Catalog + Workflow 기반 Self Segmentation 아키텍처_

#### <i class="fas fa-shirt fa-fw"></i> **무신사 — Genie 정확도를 끌어올리는 데이터마트 설계**

무신사 세션은 Genie(Text2SQL)를 실제 운영에 붙이는 과정에서 얻은 노하우가 핵심이었다. 전사 지표 모니터링 대시보드 옆에 Genie Agent가 분석 프레임 기반으로 지표 보고를 자동화하고, Genie Chat으로 실시간 Ad-hoc 질의까지 받는 구조를 보여줬다.

![무신사 Genie 활용 지표 모니터링 자동화](musinsa_genie_dashboard.jpg){: width="1600" height="900" }
_Genie를 활용한 지표 모니터링 자동화 (Dashboard · Genie Agent · Genie Chat)_

가장 실질적으로 도움이 된 내용은 "Genie 성능은 결국 데이터 구조와 컨텍스트 품질에서 결정된다"는 데이터마트 설계 원칙이었다.

- **테이블·컬럼 구조 단순화** : 테이블 수를 최소화하고 컬럼명을 비즈니스 기준으로 직관적으로 정의 → 어디를 조회해야 하는지 빠르게 판단
- **SQL 생성 단순화** : Join·Aggregation 구조를 단순화하고, 분석용 OBT(One Big Table) 혹은 Single Layer로 구성 → 안정적인 SQL 생성
- **Metrics 정의 일관성** : 주요 지표를 사전 계산·고정해서 Roll-up/Drill-down 시에도 동일 기준 유지 → 잘못된 계산 없이 일관된 결과
- **자연어-데이터 매핑** : 비즈니스 용어와 컬럼/테이블을 직접 매핑 → 자연어를 정확하게 데이터로 변환

![무신사 데이터마트 설계 및 구축 원칙](musinsa_datamart_design.jpg){: width="1600" height="900" }
_데이터마트 설계 및 구축 4원칙_

그리고 이 원칙들이 필요한 이유를 보여주는 벤치마크 결과도 인상적이었다. 여러 시도 끝에 Genie가 반복적으로 틀리는 대표 실수 유형 5가지를 정리했다고 한다.

- Wrong Table/Column — 잘못된 테이블·컬럼 선택 (데이터 구조 이해 오류)
- Wrong Filter — 잘못된 조건으로 필터링 (질문 해석 오류)
- Wrong Metric/Formula — 잘못된 비즈니스 지표 계산 (비즈니스 로직 이해 실패)
- Wrong Grain & Aggregation — 잘못된 Grouping/Aggregation (집계 수준 오류)
- Hallucinated Column/Join — 존재하지 않는 컬럼·Join 생성 (스키마 기반 추론 실패)

![Genie Benchmark 설계 - 대표 실패 유형](musinsa_genie_benchmark.jpg){: width="1600" height="900" }
_Genie Benchmark 설계, 대표적인 Error Type 기반으로 실패 유형을 구조적으로 검증_

결국 이 5가지 유형만 의도적으로 막아도 훨씬 나은 결과를 얻을 수 있었다는 게 무신사 팀의 결론이었다. Genie 도입을 고민하고 있다면 이 데이터마트 설계 기준과 실수 유형 리스트만 챙겨도 시행착오를 꽤 줄일 수 있을 것 같다.

#### <i class="fas fa-carrot fa-fw"></i> **한살림 — 수요예측 시스템**

한살림은 온프레미스 DB(ERP)에서 DW를 거쳐 Databricks로 데이터를 가져오는 수요예측 시스템 구성도를 공유했다. AutoML로 모델을 학습하고, Databricks Notebook에서 Python 스크립트를 운영하며, Job Scheduler·Job Monitoring까지 Databricks Workflow로 관리하는 구조였다.

![한살림 수요예측 시스템 구성도](hansalim_architecture.jpg){: width="1600" height="900" }
_한살림 수요예측 시스템 구성도 (현재)_

솔직히 말하면 한살림 세션은 이 구성도 하나 말고는 크게 새로 얻어간 내용은 없었다. 그래도 온프레미스 DW와 클라우드 데이터 플랫폼을 JDBC/DBLink로 연동하는 하이브리드 구조 자체는 참고할 만했다.

#### <i class="fas fa-tv fa-fw"></i> **LG전자 — 5명이 200명 넘는 사용자를 지원하는 법**

개인적으로 가장 밀도 있게 들었던 세션이다. LG전자도 Airflow를 EKS 위에서 쓰고 있고, Databricks Delta Lakehouse를 Bronze → Silver → Gold 3단으로 나눠 운영하고 있었다(굳이 비유하자면 GCP의 GCS+BigQuery 조합과 비슷한 역할이라고 설명했다). 발표 중간중간 AWS S3를 상당히 강하게 칭찬하셨는데, "Databricks 행사니까 AWS 얘기는 최대한 자제하겠다"며 구체적인 이유는 아쉽게도 넘어가셨다.

![LG전자 플랫폼 아키텍처](lg_platform_architecture.jpg){: width="1600" height="900" }
_LG전자 데이터 플랫폼 아키텍처, 저장소/컴퓨팅 분리 + Delta Lakehouse_

가장 흥미로웠던 부분은 조회 성능 전략이었다. 처음엔 리퀴드 클러스터링을 고려했지만, 자체 벤치마크 결과 파티셔닝이 더 낫다고 판단해서 최종적으로는 파티셔닝 전략으로 갔다고 한다. 다루는 데이터 크기가 워낙 크다 보니 스토리지 비용도 무시할 수 없어서, 주기적인 VACUUM 실행과 S3 Inventory 기반 메타데이터-실파일 비교 모니터링으로 저장소를 절약하고 있었다.

![LG전자 성능과 비용 전략](lg_performance_cost.jpg){: width="1600" height="900" }
_성능과 비용 — 조회 속도, 저장 구조, 데이터 삭제, 저장소 절약, 그리고 검토했지만 적용하지 않은 것들(ZORDER, Bloom Filter, Liquid Cluster)_

검토했지만 적용하지 않은 것들을 명시적으로 공유해준 점도 좋았다. ZORDER는 컴팩션 비용이 과도했고, Bloom Filter는 대부분의 테이블 사용 패턴에 맞지 않았고, Liquid Cluster는 OPTIMIZE 시점을 통제할 수 없어서 비용 발생 시점 제어가 어렵다는 이유로 제외했다고 한다.

모니터링은 Databricks Workflow 기본 알림만으로는 불특정 다수가 쓰는 환경에서 확인·대응에 한계가 있다고 판단해서, 자체 모니터링 RDS와 사내 메신저(Teams)를 연동한 실시간 알림 체계를 별도로 구축했다.

![LG전자 모니터링 체계](lg_monitoring.jpg){: width="1600" height="900" }
_실시간 알림 · 비용 모니터링 · 성능 모니터링 · 데이터 품질 모니터링_

마지막으로 가장 인상적이었던 숫자는 **운영자 3~5명이 200명 넘는 사용자를 지원**하고 있다는 부분이었다. 반복적인 운영 작업을 자동화하고, 분석용 sandbox 생성이나 월간 보안점검 리포트 같은 관리 체계를 자동화했고, Self-service 분석이 어려운 사용자를 위한 데이터 공급 서비스까지 운영해서 운영 업무 리드타임을 20~30%까지 줄였다고 한다. Databricks 교육 6회, 데이터 분석 개론 3회, 부서별 맞춤 교육 3회처럼 사용자 교육에도 꾸준히 투자해서, ad-hoc 업무를 줄이고 데이터 민주화를 실질적으로 추진하고 있다는 인상을 받았다.

![LG전자 운영과 확산](lg_operation_scale.jpg){: width="1600" height="900" }
_운영과 확산 — 3→5명의 운영자로 200명 이상의 사용자를 지원_

#### <i class="fas fa-flag-checkered fa-fw"></i> **마무리**

이번 행사에서 가장 크게 남은 인상은, 같은 Databricks를 쓰더라도 회사가 처한 상황에 따라 접근 방식이 이렇게까지 달라질 수 있다는 점이었다. 리스크를 줄이려고 기존 스택과 한동안 병행하는 곳이 있는가 하면, Unity Catalog 안에 아예 다 몰아넣고 과감하게 통합해버리는 곳도 있었다. 정답이 하나로 정해져 있다기보다는, 조직 규모와 감내할 수 있는 리스크 수준에 따라 전략 자체가 달라진다는 걸 여러 회사의 사례로 직접 확인할 수 있었던 자리였다.

특히 무신사의 데이터마트 설계 원칙·Genie 실패 유형 5가지, LG전자의 파티셔닝·VACUUM 운영 전략은 지금 진행 중인 프로젝트에도 바로 적용해볼 만한 내용이라 따로 정리해뒀다. 다음에 비슷한 행사가 열리면 이번처럼 메모만 남기지 말고, 실제로 적용해본 결과까지 후속 글로 남겨보고 싶다.