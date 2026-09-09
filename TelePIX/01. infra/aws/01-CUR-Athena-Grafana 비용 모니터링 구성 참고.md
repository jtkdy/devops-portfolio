---
title: CUR-Athena-Grafana 비용 모니터링 구성 참고
date: 2026-06-22
updated: 2026-08-05
tags: [monitoring, grafana, athena, cost, cur, reference]
status: active
---
## 아키텍처

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

## 구성 리소스

| 리소스          | 값                                                                                                                                  |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| S3 Prefix    | `cur/`                                                                                                                             |
| Athena 결과 저장 | 별도 결과 전용 Prefix (`athena-results/`)                                                                                                |
| 형식           | Parquet, Daily granularity, Resource ID 포함, Athena integration ON                                                                  |
| 리전           | us-east-1 (Export API 자체가 us-east-1 고정 — 빌링은 IAM/Route53처럼 글로벌 서비스 취급이라 S3/Glue/Athena도 이 리전에 맞춰야 함<br>파티션 Lambda만 ap-northeast-2) |

## IAM 권한

### Grafana Role
인라인 정책으로 Athena/Glue/S3 최소 권한 추가:
- `athena:StartQueryExecution`, `GetQueryExecution`, `GetQueryResults`, `StopQueryExecution`, `GetWorkGroup`, `ListWorkGroups`
- `glue:GetDatabase`, `GetDatabases`, `GetTable`, `GetTables`, `GetPartitions`
- `s3:GetObject`, `ListBucket`, `PutObject` — 비용 데이터 버킷 한정

### 파티션 자동화 Lambda Role
- `athena:StartQueryExecution`, `GetQueryExecution`
- `glue:GetTable`, `GetPartitions`, `BatchCreatePartition`
- `s3:GetObject`, `ListBucket`, `PutObject`, `GetBucketLocation` — 비용 데이터 버킷 한정

### S3 버킷 정책
- `billingreports.amazonaws.com`, `bcm-data-exports.amazonaws.com` → PutObject 허용
- 파티션 자동화 Lambda Role → S3 읽기/쓰기 허용

## 파티션 자동화

| 항목 | 내용 |
|---|---|
| 트리거 | EventBridge `cron(0 1 1 * ? *)` (매월 1일 10:00 KST) |
| 동작 | 당월 `billing_period` 파티션 자동 추가 (`ADD IF NOT EXISTS`) |

| 파티션 | 등록 방법 |
|---|---|
| 2026-06 | MSCK REPAIR TABLE (수동, 최초 등록) |
| 2026-07 | ALTER TABLE ADD PARTITION (수동) |
| 2026-08 | Lambda 테스트 실행 (자동화 검증) |
| 2026-09~ | Lambda 자동 실행 |

## S3 Lifecycle 정책

| Prefix | 보존 기간 | 이유 |
|---|---|---|
| `athena-results/` | 7일 | 쿼리 결과 임시 파일, 재실행 시 재생성 |

## Athena 쿼리 주의사항

- Athena Engine v3(Trino): `CREATE/DROP TABLE`, `CREATE DATABASE`, `ALTER TABLE ... ADD PARTITION`은 백틱, `SELECT` 등 DML은 쌍따옴표
- `MSCK REPAIR TABLE` 미지원 → 파티션 등록 시 `ALTER TABLE ... ADD PARTITION` 사용
- 파티션 컬럼: DDL에서는 대문자로 선언했더라도 SELECT에서는 소문자 `billing_period`로 참조 (Trino가 컬럼명을 소문자로 정규화, 형식 `'2026-06'`)
- 파티션 미등록 시 `WHERE` 필터 무관하게 `COUNT(*) = 0` 반환 → 데이터 없음이 아니라 메타데이터 미등록 신호
- CUR Export가 자동 생성해주는 `create-table.sql`을 그대로 실행하면 `STORED AS INPUTFORMAT/OUTPUTFORMAT` 절이 빠져있어서 텍스트 포맷으로 잡히고 Parquet 데이터를 못 읽음 — Parquet용 InputFormat/OutputFormat/SerDe를 명시한 DDL로 다시 생성 필요

## 검증 쿼리

```sql
-- 파티션 확인
SHOW PARTITIONS <db>.<table>;

-- row 수 확인
SELECT COUNT(*) AS row_count
FROM <db>.<table>
WHERE billing_period = '2026-07';

-- 서비스별 비용
SELECT
  line_item_product_code AS service,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost_usd
FROM <db>.<table>
WHERE billing_period = '2026-07'
GROUP BY line_item_product_code
ORDER BY cost_usd DESC;

-- 리소스 ID별 비용
SELECT
  line_item_resource_id AS resource_id,
  line_item_product_code AS service,
  ROUND(SUM(line_item_unblended_cost), 2) AS cost_usd
FROM <db>.<table>
WHERE billing_period = '2026-07'
GROUP BY line_item_resource_id, line_item_product_code
ORDER BY cost_usd DESC;
```

## Grafana 대시보드 구성

| 패널 | 기준 컬럼 |
|---|---|
| 이번 달 총 비용 | `SUM(line_item_unblended_cost)` |
| 일별 비용 추이 | `line_item_usage_start_date` |
| 서비스별 비용 | `line_item_product_code` |
| 리소스 ID별 비용 | `line_item_resource_id` |
| 이상 감지 | anomaly_count |
| CPU 대비 비용 | CloudWatch CPUUtilization 오버레이 |

## 관련 문서
- [[jtkdy/TelePIX/01. infra/00-index|01. infra 인덱스]]
- [[02-AWS 비용 모니터링 구축|왜/어떻게 만들었는지 서술형 기록]]
- [[01-Grafana 통합 모니터링 스택 구축|Grafana 모니터링 스택 전체 개요]]
