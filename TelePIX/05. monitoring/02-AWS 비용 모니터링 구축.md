---
title: CUR 2.0 + Athena + Grafana로 AWS 비용 모니터링 구축
date: 2026-06-22
tags: [monitoring, grafana, athena, cost, cur]
status: done
---
## 배경

AWS 비용을 매번 Cost Explorer 콘솔에 들어가서 확인하는 방식은 서비스별/리소스별로 세밀하게 쪼개보기 어려웠고, 팀 대시보드에 자연스럽게 얹기도 힘든 구조
비용 이상 징후를 더 빠르게 캐치하고, 다른 인프라 지표와 같은 화면에서 보고자 CUR(Cost and Usage Report) 기반 파이프라인을 새로 구축

## 접근

AWS Cost Explorer API를 직접 붙이는 방법도 검토
결정적으로 리소스 레벨(인스턴스 ID 단위) 비용 데이터는 Cost Explorer에서 최근 14일치만 제공, 그마저도 별도 활성화 필요라는 제약
어떤 EC2 인스턴스가 비용을 얼마나 쓰는지 드릴다운하려는 목적 자체가 Cost Explorer로는 애초에 불가능한 요구
그래서 CUR 2.0(Data Exports)을 S3로 매일 내려받고 Athena로 SQL 조회하는 구조로 결정
Athena는 CUR용 Glue Table을 자동 생성해주는 Athena Integration 옵션이 있어서 스키마 관리 부담이 적을 것으로 기대했으나, 실제로는 자동 생성된 DDL을 그대로 사용 불가 — 아래 트러블슈팅 참고
마지막 단계는 Grafana Cloud의 Athena 데이터소스로 연결해서 CloudWatch 데이터와 같은 화면에 배치하는 작업

## 구현

### 아키텍처

```
CUR 2.0 (Data Exports)
↓ Daily 자동 전송
S3 (비용 데이터 버킷) /cur/
↓ Glue Table (자동 생성, Athena integration ON)
Glue Database
↓ SQL 쿼리
Athena (us-east-1, CUR 리전 고정)
↓ Grafana Assume Role
Grafana Cloud Athena Datasource → 대시보드
```

### 구성 리소스

| 리소스 | 내용 |
|---|---|
| S3 Prefix | `cur/` |
| Athena 결과 저장 | 별도 결과 전용 Prefix |
| Data Export 형식 | Parquet, Daily granularity, Resource ID 포함 |
| Glue DB/Table | CUR 전용 |
| 리전 | us-east-1 (Export API 자체가 us-east-1 고정 — 빌링은 IAM/Route53처럼 리전 구분 없는 글로벌 서비스로 취급돼서, 실제 S3/Glue/Athena도 이 리전에 맞춰야 한다) |

IAM 권한은 Grafana Role과 파티션 자동화용 Lambda Role 두 곳에 각각 최소 권한으로 분리 부여
Grafana Role에는 Athena 쿼리 실행/Glue 메타데이터 조회/S3 결과 버킷 읽기쓰기만, Lambda Role에는 파티션 추가에 필요한 Glue/S3 권한만 인라인 정책으로 부여

### Glue Table 자동 생성의 함정

- **문제** — CUR Export를 켜면 AWS가 Glue Table 생성용 DDL(`create-table.sql`)을 S3에 같이 떨궈주는데, 이걸 그대로 실행하면 안 되는 함정
- **원인** — `STORED AS INPUTFORMAT/OUTPUTFORMAT` 절이 빠져있어서, 실행 시 테이블이 기본값인 텍스트 포맷으로 잡혀 실제 데이터(Parquet) 조회 불가
- **해결** — Parquet용 InputFormat/OutputFormat/SerDe를 명시적으로 추가한 DDL로 재생성

```sql
CREATE EXTERNAL TABLE `cost-export-db`.cost_export_table(...)
PARTITIONED BY (BILLING_PERIOD STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
STORED AS INPUTFORMAT 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION 's3://.../data/'
TBLPROPERTIES ('classification'='parquet');
```

식별자 인용 규칙도 작업 종류마다 달라서 초반에 혼선
DB/테이블명에 하이픈이 들어가다 보니 작업에 따라 인용 부호가 다르면 "Queries of this type are not supported"라는 알기 어려운 에러 발생

| 작업 | 인용 방식 | 이유 |
|---|---|---|
| `CREATE DATABASE`/`TABLE`, `DROP TABLE` | 백틱 `` ` `` | Hive DDL 파서 |
| `ALTER TABLE ... ADD PARTITION` | 백틱 `` ` `` | Hive 메타스토어 |
| `SELECT` 등 모든 DML | 큰따옴표 `"` | Trino/Presto ANSI 파서 |

파티션 컬럼명도 DDL에서는 대문자(`BILLING_PERIOD`)로 선언했더라도 SELECT에서는 소문자(`billing_period`)로 참조 필요 — Trino 엔진이 컬럼명을 소문자로 정규화하기 때문

### 파티션 자동화

Athena 파티션은 매월 늘어나야 하는데 수동 관리는 누락 위험이 있어 Lambda로 자동화
EventBridge `cron(0 1 1 * ? *)` (매월 1일 10시 KST)에 맞춰 당월 `billing_period` 파티션을 `ADD IF NOT EXISTS`로 추가하는 방식
첫 두 달은 `MSCK REPAIR TABLE`과 `ALTER TABLE ADD PARTITION`으로 수동 등록, 세 번째 달부터 Lambda 자동 실행으로 검증

겪은 함정 두 가지

- DDL은 백틱, DML은 쌍따옴표를 써야 하고, `MSCK REPAIR TABLE`이 지원되지 않아 파티션 등록에는 `ALTER TABLE ... ADD PARTITION`을 써야 한다는 점을 삽질 끝에 확인 (Athena Engine v3/Trino 기준)
- 파티션이 등록 안 된 상태에서는 `WHERE` 조건과 무관하게 `COUNT(*) = 0`이 나오는데, 이게 "데이터 없음"이 아니라 "메타데이터 미등록" 신호라는 점이 혼동 포인트

### S3 Lifecycle

Athena 쿼리 결과 Prefix는 재실행 시 다시 생성되는 임시 파일이라 7일 보존으로 Lifecycle 정책을 걸어 비용 절감

## 결과

- **최초 검증** — 해당 월 누적 1.8만 건 이상 row 정상 집계, 서비스별 비용 1위 EC2로 기존 Cost Explorer 조사 결과와 방향 일치 → 파이프라인 신뢰도 확인
- 서비스별/리소스별 비용을 Grafana 대시보드에서 CloudWatch 지표와 나란히 조회 가능
- 파티션 등록을 매월 1일 Lambda 자동 실행으로 전환, 수동 등록 누락 리스크 제거
- **대시보드 패널 구성**
  - 이번 달 총 비용
  - 일별 비용 추이
  - 서비스별/리소스 ID별 비용
  - 이상 감지
  - CPU 대비 비용 오버레이

![[aws-cost-dashboard.png]]
*비용 모니터링 대시보드 — 이번 달 총 비용/일별 추이/리소스ID·서비스별 비용/이상탐지 (계정ID·리소스명은 마스킹)*

## 회고

CUR 기반 파이프라인은 처음 세팅 비용(Glue/Athena 스키마 이해, 파티션 개념)이 있지만 한번 잡아두면 SQL로 원하는 만큼 세밀하게 비용을 쪼개볼 수 있어 만족스러운 결과
아쉬운 점은 Athena 쿼리 실행 비용이 스캔 데이터량에 비례하기 때문에, 파티션 프루닝을 제대로 안 걸면 비용 모니터링 자체가 비용을 발생시키는 역설이 생길 수 있다는 것 — 앞으로 쿼리 작성 시 `billing_period` 파티션 필터를 항상 강제하는 방향으로 대시보드 쿼리 점검 예정

## 관련 문서
- [[01-Grafana 통합 모니터링 스택 구축]]
