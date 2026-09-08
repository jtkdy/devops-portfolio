---
title: ECS 서비스 로그를 Grafana Loki로 실시간 수집하기
date: 2026-07-27
tags: [grafana, loki, lambda, cloudwatch, ecs, monitoring]
status: done
---

## 배경

ECS에서 돌아가는 여러 서비스(이미지 처리, AI/LLM, 미디어 처리, SSO, GPU 추론)의 로그를 확인하려면 매번 AWS 콘솔에 들어가서 CloudWatch Logs를 서비스별로 뒤져야 하는 번거로움
팀원들이 AWS 콘솔 권한 없이도 실시간으로 로그를 볼 수 있게 하려는 목적, 장애 대응 시 여러 서비스 로그를 한 화면에서 비교하려는 필요도 함께 존재

## 접근

CloudWatch Logs를 Grafana에서 직접 조회하는 방법도 있었으나, 실시간성과 쿼리 편의성 면에서 Loki로 로그를 전송하는 방식이 낫다고 판단
AWS가 공식 제공하는 lambda-promtail(Go 런타임)을 활용해 CloudWatch Logs Subscription Filter → Lambda → Loki Push 파이프라인을 구성
보관 기간은 구간별로 다르게 설계

- **실시간~14일** — Loki
- **14일~90일** — CloudWatch Logs Insights
- **90일 이후** — 추후 S3+Athena로 확장할 계획으로 남겨둠

당장 붙인 건 앞의 두 구간, 마지막 구간은 다음 과제로 이월

## 구현

### 아키텍처

```
ECS 서비스 (이미지처리/미디어처리/SSO/GPU)
    │
    ▼
CloudWatch Log Groups (90일 보관)
    │ Subscription Filter (로그 발생 즉시 트리거)
    ▼
Lambda (lambda-promtail)
    │ Loki API Push
    ▼
Grafana Cloud Loki (14일 보관)
    │
    ▼
Grafana 대시보드 (팀원 접근)
```

### 구성 상세

| 항목 | 내용 |
|---|---|
| Lambda 런타임 | Go, lambda-promtail v1.0.1 |
| 배포 방식 | CloudFormation 스택 (Grafana Labs 공식 템플릿) |
| ExtraLabels | `env,prod` → Loki 레이블 `__extra_env=prod` |
| ReservedConcurrency | 2 |
| IAM 권한 | CloudWatch Logs 읽기(해당 로그 그룹 한정) + Lambda 기본 실행 권한 |

Subscription Filter는 대상 로그 그룹마다 개별 등록해야 하고, **설정 시점 이후 로그부터만 Loki로 전송된다는 제약**이 있어 소급 적용 불가
서비스군별로 로그 그룹을 정리해 순차적으로 등록

- 핵심 서비스(이미지 처리/AI·LLM/검색·구매/프론트엔드) — 6개
- 미디어 처리 서비스 — 3개
- SSO 인증 — 1개
- GPU 추론(변화 탐지/객체 탐지/특정 객체 탐지) — 3개

신규 로그 그룹 생성 시 Subscription Filter를 놓치면 로그가 아예 안 잡히므로 CLI 스크립트로 등록 절차 표준화

```bash
aws logs put-subscription-filter \
  --log-group-name "/ecs/신규/로그그룹" \
  --filter-name "lambda-promtail" \
  --filter-pattern "" \
  --destination-arn "$LAMBDA_ARN" \
  --profile devops \
  --region ap-northeast-2
```

### Loki 쿼리 설계

Loki 레이블은 아래 세 가지를 기본으로 사용

- `__aws_cloudwatch_log_group` — 로그 그룹명
- `__extra_env` — 환경
- `detected_level` — Loki가 로그 내용을 분석해 자동 부여하는 레벨

`detected_level`은 감지에 실패하면 `unknown`으로 남는데, 이 경우를 감안해 "INFO 제외" 쿼리(`detected_level!="info"`)를 실시간 모니터링용 기본 패턴으로 잡아 헬스체크 노이즈 필터링

대시보드는 서비스군별 섹션(핵심 서비스/미디어 처리/SSO/GPU)으로 나누고, 각 섹션에 ERROR/WARN 건수 Stat 패널, 서비스별 ERROR 추이 Timeseries, 실시간 로그 Logs 패널을 배치하는 구조로 통일

## 결과

- ECS 서비스 로그 15개 로그 그룹을 Loki로 실시간 수집
- 팀원들이 AWS 콘솔 권한 없이 브라우저에서 실시간 로그 조회 가능
- 로그 레벨(ERROR/WARN) 기준 대시보드로 이상 징후 즉시 확인 가능

## 회고

Subscription Filter가 소급 적용되지 않는다는 제약을 미리 몰랐다면 "왜 과거 로그가 하나도 안 보이지"로 헤맸을 상황
신규 서비스 추가 시 로그 그룹 생성과 Subscription Filter 등록을 세트로 묶는 체크리스트를 남겨둔 게 실질적으로 도움
아직 90일 이후 장기 보관(Kinesis Firehose → S3)과 ERROR/WARN Grafana Alert 연동은 미완, 다음 우선순위로 남음

## 관련 문서
- [[01-overview-grafana-monitoring-stack]]
- [[05-satchat-alert-integration]]
- [[02-GPU 딥러닝 모델 서빙 ECS 이관|GPU 딥러닝 서빙 ECS 이관 (이 로그 그룹들이 편입된 이관 작업)]]
