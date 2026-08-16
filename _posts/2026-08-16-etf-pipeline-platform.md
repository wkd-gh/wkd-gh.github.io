---
title: ETF 데이터 파이프라인 플랫폼
date: 2026-08-16 12:00:00 +0900
categories: [Project, Side Project]
tags: [ETF, Airflow, Databricks]
image: thumbnail.png
media_subpath: /assets/img/posts/2026-08-16-etf-pipeline-platform/
---

> 본 프로젝트는 Databricks라는 플랫폼에 대해 이해하고 실무에 직접 적용해보기 위해 개인적으로 진행한 PoC 프로젝트입니다.

![System Architecture Diagram](system_architecture.png){: width="724" height="388" }
_System Architecture Diagram_

#### <i class="fas fa-diagram-project fa-fw"></i> **프로젝트 개요**

평소 실무에서 접해보지 못했던 Databricks를 직접 다뤄보고 싶어서 혼자 기획부터 구현까지 진행한 사이드 프로젝트다. 공공데이터포털의 ETF/주식 시세 데이터를 수집·적재하고, Databricks Genie를 통해 자연어로 조회(Text2SQL)까지 해볼 수 있는 작은 데이터 플랫폼을 만드는 게 목표였다.

위에 있는 System Architecture Diagram도 직접 구상하고 그린 것으로, 수집부터 적재, 분석, 서비스 노출까지 전체 흐름을 end-to-end로 설계해보았다.

관련 레포는 아래 세 개다.

- [airflow-etl-pipeline](https://github.com/wkd-gh/airflow-etl-pipeline) : 공공데이터포털 API 수집 및 Databricks 적재 Airflow DAG
- [genie-slack-bot](https://github.com/wkd-gh/genie-slack-bot) : Databricks Genie를 슬랙에서 호출하는 Cloud Run 서비스
- [genie-web-app](https://github.com/wkd-gh/genie-web-app) : Genie를 웹/모바일 환경에서 사용할 수 있는 FastAPI 기반 서비스

#### <i class="fas fa-server fa-fw"></i> **전체 아키텍처**

크게 세 흐름으로 나뉜다.

- **수집/적재** : 공공데이터포털(금융위원회 증권상품정보) API를 Airflow DAG가 매일 새벽 배치로 호출하고, 원본 데이터를 GCS에 스테이징한 뒤 Databricks Delta Lake 테이블로 적재한다.
- **분석/조회** : Databricks 플랫폼의 Dashboard와 Genie(Text2SQL)를 통해 적재된 데이터를 시각화하고, 자연어 질의로 조회 및 분석할 수 있게 한다.
- **서비스** : Genie API를 `genie-slack-bot`(Cloud Run)이 호출하여 슬랙에서 바로 질의응답이 가능하도록 연동했고, `genie-web-app`으로 웹/모바일 환경에서도 같은 경험을 제공한다.

Airflow는 별도의 관리형 서비스 대신 GCE VM 위에 Docker Compose로 직접 올렸고, VM 상태는 Google Cloud Monitoring으로 주시하다가 설정한 임계치 도달 시 슬랙으로 알림이 오도록 구성했다. 배포는 Cloud Build에서 GitHub `main` 브랜치 push를 감지하여 자동으로 처리한다.

#### <i class="fas fa-cloud fa-fw"></i> **사용한 클라우드/기술 스택**

- **Databricks (Free Edition)** : Delta Lake 테이블 저장 및 Genie(Text2SQL) 분석
- **GCP (Free Credit)** : GCE VM(Airflow 호스팅), Cloud Build(CI/CD), Cloud Run(Slack Bot·Web App 서버리스 배포), Secret Manager(환경변수/시크릿 관리), Cloud Monitoring(VM 상태 모니터링), GCS(원본 데이터 스테이징)
- **Airflow** : 배치 오케스트레이션, Docker Compose 기반 셀프 호스팅
- **Slack** : Airflow 실행 결과 및 인프라 알림, Genie 질의응답 채널

민감한 값(API 키, Access Token 등)은 전부 GCP Secret Manager에 등록해두고, Airflow Connection과 Cloud Run 환경변수에서 참조하는 방식으로 코드에 직접 노출되지 않게 했다.

#### <i class="fas fa-database fa-fw"></i> **Databricks 테이블 설계**

공공데이터포털에서 제공하는 증권상품정보 API 기준으로 아래 6개 테이블을 Delta Lake에 구성했다.

- `item_info` : KRX 상장종목정보
- `etf_price_info` : ETF 가격정보
- `stock_price_info` : 주식 가격정보
- `securities_price_info` : 증권 가격정보
- `preemptive_right_certificate_price_info` : 신주인수권증서 가격정보
- `preemptive_right_securities_price_info` : 신주인수권증권 가격정보

우선은 스키마 변경에 유연하게 대응하기 위해 모든 컬럼을 `STRING` 타입으로 적재하고, 이후 조회 패턴이 안정화되면 타입을 최적화하기로 했다.

```sql
CREATE TABLE IF NOT EXISTS money_digger.equity_derivative.item_info (
  basDt    STRING, -- 기준일자
  srtnCd   STRING, -- 단축코드
  isinCd   STRING, -- ISIN코드
  mrktCtg  STRING, -- 시장구분 (KOSPI/KOSDAQ)
  itmsNm   STRING, -- 종목명
  crno     STRING, -- 법인등록번호
  corpNm   STRING, -- 법인명
  _loaded_at STRING -- 적재일자
)
USING DELTA;
```

#### <i class="fas fa-robot fa-fw"></i> **파이프라인 구현**

##### DAG 확장

처음 `item_info` DAG을 만들 때 API 호출 → GCS 스테이징 → Delta 테이블 UPSERT로 이어지는 패턴을 잡아두고, 이후 5개 API(ETF/주식/증권/신주인수권 가격정보)를 추가할 때는 이 패턴을 그대로 재사용했다.

반복적인 보일러플레이트를 손으로 다시 짜는 대신, `CLAUDE.md`에 프로젝트 컨벤션을 정리해두고 Claude에게 API URL·테이블명·UPSERT 키·스키마만 넘겨서 `dags/{테이블명}_dag.py`와 `include/utils/{테이블명}_.py`를 한 번에 생성하도록 했다. 실제로 사용한 프롬프트는 대략 이런 형태다.

```bash
CLAUDE.md와 기존 코드 패턴을 참고해서 아래 API들에 대한 DAG을 추가해줘.

1. etf_price_info
   - API URL: https://apis.data.go.kr/1160100/service/GetSecuritiesProductInfoService/getETFPriceInfo
   - 테이블명: etf_price_info
   - UPSERT 키: ["isinCd", "basDt"]
   - GCS_PREFIX: GetSecuritiesProductInfoService
   - 스키마: basDt, srtnCd, isinCd, itmsNm, clpr, vs, fltRt, nav, mkp, hipr, lopr, trqu, trPrc, ...

(이하 stock_price_info 등 나머지 4개 API도 동일한 형식으로 명시)

각 API마다 dags/{테이블명}_dag.py, include/utils/{테이블명}_.py 두 파일을 생성해줘.
item_info 패턴을 그대로 따라야 하고 add_new_dag.md 체크리스트 항목을 모두 충족해야 해.
```

신규 DAG 추가 시 지켜야 할 항목들을 `add_new_dag.md` 체크리스트로 만들어두니, AI가 생성한 코드도 사람이 짠 것과 동일한 구조·네이밍·에러 처리 방식을 따르게 되어 리뷰 부담이 크게 줄었다.

##### genie-web-app 구현

`genie-slack-bot`으로 슬랙 연동을 끝낸 뒤, 같은 Genie 기능을 웹/모바일에서도 쓸 수 있게 하고 싶어서 요구사항을 정리해 프롬프트로 넘겼다.

```bash
genie-slack-bot이 슬랙에서 지니를 사용하는 거라고 한다면
genie-web-app은 웹이나 모바일 환경에서 지니를 사용하는건데
백엔드는 FastAPI를 사용할 것 같고 배포는 동일하게 Cloud Run으로 서버리스 환경에서 할 것 같아.
genie-web-app을 작성해줄 수 있을까?

나의 요구사항은 아래와 같아.
- 모바일에서도 사용이 가능해야 하고 반응형 구조이어야 함, 웹도 반응형이어야 함
- 서버는 FastAPI 사용, 배포는 Cloud Run으로 할거고 genie-web-app 레포가 main에 푸쉬되면
  자동으로 Cloud Run에 배포가 되도록 할거야
- 필요한 환경변수 관리는 genie-slack-bot과 동일하게 GCP Secret Manager 서비스를 이용할거야
- 회원가입, 로그인, 로그아웃, 회원탈퇴 같은 기능들도 있어야 해
- 가능하다면 대시보드 제작하고 관리하는 기능도 추가되면 좋겠어
- 지금까지 지니와 대화했던 채팅내역들도 히스토리로 남을 수 있는 기능도 있으면 좋겠어

기술스택: Cloud SQL, Cloud Run, Cloud Build, FastAPI 등
```

기능 명세를 최대한 구체적으로 적어서 넘기니, 인증(회원가입/로그인/로그아웃/회원탈퇴)과 Genie 대화 히스토리 기능까지 포함된 서비스 뼈대를 빠르게 만들 수 있었고, `airflow-etl-pipeline`과 마찬가지로 `main` 푸시 시 Cloud Build가 자동으로 Cloud Run에 배포하도록 붙였다.

#### <i class="fas fa-gear fa-fw"></i> **GCE VM 인프라**

Airflow는 `e2-standard-2`(vCPU 2, 메모리 8GB) 스펙의 GCE VM 위에 Docker Compose로 올렸다.

![GCE VM Computing Spec](vm_computing_spec.png){: width="2000" height="923" }
_GCE VM Computing Spec_

```bash
# Docker 설치
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker

# repo clone
git clone https://github.com/wkd-gh/airflow-etl-pipeline.git
cd airflow-etl-pipeline

# .env 파일 생성 (Secret Manager 값 기반)
nano .env

# 실행
docker compose up -d
```

`AIRFLOW_UID`, `FERNET_KEY`처럼 Airflow 부트스트랩 시점에 Docker Compose가 직접 읽어야 하는 값들만 `.env`에 남겨두고, 나머지 API 키·Databricks 토큰 등은 Secret Manager에서 관리한다.

#### <i class="fas fa-bell fa-fw"></i> **모니터링 및 알림**

VM 리소스 모니터링 도구로 Grafana 대신 **Google Cloud Monitoring**을 선택했다. Grafana를 쓰려면 결국 같은 VM 위에 Docker Compose로 하나 더 올려야 하는데, Airflow DAG이 몰리는 피크 타임(23:59)에 메모리 경합으로 OOM이 날 수 있다고 판단해서다.

구성한 내용은 다음과 같다.

1. VM에 Cloud Ops Agent 설치 (메모리/디스크 메트릭 수집)
2. GCP Monitoring에 Slack Notification Channel 등록
3. Airflow API 서버 대상 Uptime Check 생성 (`/api/v2/monitor/health`, 5분 간격)
4. Alert Policy 4종 생성 — CPU 과부하(85%), 메모리 부족(85%), 디스크 부족(80%), Airflow 다운(Uptime Check 실패)

임계치를 넘으면 아래처럼 슬랙으로 바로 알림이 온다.

![VM CPU 사용률 경고 Slack Alert](vm_cpu_alert_slack.png){: width="1988" height="642" }
_VM CPU 사용률 경고 Slack Alert_

VM 리소스뿐 아니라 DAG 단위 성공/실패도 슬랙으로 알림이 오도록 붙여서, 배치가 조용히 실패하고 지나가는 상황이 없게 했다.

![Airflow DAG 성공 Slack 알림](airflow_dag_success_slack.png){: width="1566" height="994" }
_Airflow DAG 성공 Slack 알림_

![Airflow DAG 실패 Slack 알림](airflow_dag_failed_slack.png){: width="1482" height="992" }
_Airflow DAG 실패 Slack 알림_

위 실패 알림은 실제로 Databricks SQL Warehouse가 자동 중지(idle timeout)된 상태에서 배치가 돌면서 발생했던 케이스인데, Error Log와 링크까지 슬랙에 바로 남으니 원인 파악이 훨씬 빨랐다.

#### <i class="fas fa-comments fa-fw"></i> **Databricks Genie 연동**

`genie-slack-bot`을 통해 슬랙에서 `@Databricks`를 멘션하면 Genie가 적재된 Delta 테이블을 대상으로 자연어 질의를 SQL로 변환해서 답변해준다.

![Genie 답변 예시-1](genie_slack_qna_question.png){: width="1430" height="1524" }
_Genie 질문 예시-1_

예를 들어 "레버리지·인덱스·선물이 포함된 종목을 제외하고 추세가 가장 좋은 ETF를 1개 뽑고 이유도 제시해달라"고 물으면, Genie가 조건에 맞는 SQL을 직접 생성해서 실행하고 결과와 근거를 함께 돌려준다.

![Genie 답변 예시-2](genie_slack_qna_answer.png){: width="1422" height="694" }
_Genie 답변 예시-2_

생성된 SQL과 근거 지표(등락률 등)를 함께 보여주기 때문에, 단순 조회를 넘어서 "왜 이 종목인지"까지 답을 받을 수 있다는 점이 인상 깊었다.

#### <i class="fas fa-code-branch fa-fw"></i> **CI/CD**

`airflow-etl-pipeline`, `genie-slack-bot`, `genie-web-app` 모두 `main` 브랜치에 푸시되면 Cloud Build가 자동으로 빌드·배포하도록 구성했다.

![Cloud Build 배포 성공 Slack 알림](cloudbuild_deploy_slack.png){: width="1452" height="1044" }
_Cloud Build 배포 성공 Slack 알림_

배포 성공/실패 여부도 슬랙으로 알림이 오도록 해서, 로컬에서 커밋만 올리면 실제 서비스에 반영됐는지 콘솔을 따로 열어보지 않아도 확인할 수 있다.

#### <i class="fas fa-flag-checkered fa-fw"></i> **회고 및 앞으로의 계획**

관리형 서비스 없이 GCE VM 한 대에 Airflow를 직접 올리고, 모니터링부터 알림, 배포까지 전체 라이프사이클을 처음부터 끝까지 혼자 설계하고 구성해보았는데, 아무것도 없는 상태에서 하나씩 쌓아 배포까지 이어가는 과정 자체가 꽤나 재밌었다.

특히 Genie로 자연어 질의가 실제로 동작하는 걸 확인했을 때, PoC 수준으로 짧게 구현해본 것치고는 결과물이 꽤 그럴듯하게 나와 신기했고 Databricks라는 플랫폼 자체가 생각보다 꽤 괜찮다는 인상을 많이 받았다. Genie 외에 다른 기능들도 하나씩 공부하고 실습해봐야할 것 같다.

추후에는 아래 방향으로 조금씩 더 다듬어볼 생각이다.

- `genie-web-app`의 대시보드 관리 기능 고도화
- Delta Lake 테이블 컬럼 타입 최적화 (현재는 전부 `STRING`)
- Airflow 인프라 이중화 및 장애 대응 시나리오 보강

#### <i class="fas fa-book fa-fw"></i> **참고 자료**

- [공공데이터포털 - 금융위원회 상장기업정보 (getItemInfo)](https://www.data.go.kr/data/15094775/openapi.do)
- [공공데이터포털 - 금융위원회 증권상품시세정보 (getETFPriceInfo)](https://www.data.go.kr/data/15094806/openapi.do)
- [공공데이터포털 - 금융위원회 주식시세정보](https://www.data.go.kr/data/15094808/openapi.do)
- [Databricks Genie Conversation API](https://docs.databricks.com/aws/en/genie/conversation-api)
