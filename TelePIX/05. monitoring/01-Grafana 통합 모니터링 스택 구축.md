---
title: Grafana 기반 통합 모니터링 스택 구축
date: 2026-07-06
tags: [monitoring, grafana, aws, onprem]
status: done
---

## 배경

인프라를 운영하면서 AWS 리소스, 비용, 온프렘 서버 상태가 흩어져 있어 한 곳에서 확인할 필요
CloudWatch 콘솔, Cost Explorer, 서버 SSH 접속을 오가며 상태를 확인하는 방식은 장애 초기 대응 속도를 떨어뜨렸고, 팀원 전체가 접근하기도 불편
그래서 Grafana Cloud를 중심으로 AWS 인프라, AWS 비용, 온프렘 서버를 하나의 대시보드 체계로 묶는 작업을 진행

## 접근

Grafana Cloud를 선택한 이유는 무료 티어에서도 Alert Rule 500개, 3명까지 무료 사용자를 지원해서 초기 비용 부담 없이 시작 가능했기 때문
데이터소스는 영역별로 다르게 구성

- AWS 인프라 — CloudWatch 직접 쿼리
- AWS 비용 — CUR(Cost and Usage Report) 2.0을 Athena로 조회
- 온프렘 서버 — Grafana Alloy와 Node Exporter 조합으로 메트릭 원격 전송

가장 까다로웠던 부분은 Grafana Cloud가 AWS 리소스에 접근하는 인증 방식
Cross-Account IAM Role을 Assume하는 구조인데, 처음에는 Trust Policy의 Principal에 Grafana 측 특정 IAM User ARN을 지정
그런데 이 방식은 곧 문제로 이어짐

## 구현

### 데이터소스 구성

| 영역 | 데이터소스 | 대상 |
|---|---|---|
| AWS 인프라 | CloudWatch | ECS, ALB, RDS, ElastiCache, S3, ACM |
| AWS 비용 | Athena (CUR 2.0) | 서비스별/리소스별 비용 |
| 온프렘 서버 | Grafana Alloy + Node Exporter | 서버 4대 |

Grafana Cloud 스택 수집 방식 통일

- **Metrics** — Prometheus/Mimir 호환 remote_write
- **Logs** — Loki 호환 remote_write

### IAM Cross-Account Role 인증 문제

- **원인** — Grafana Cloud가 AWS 계정에 CloudWatch 조회 권한으로 Assume하는 IAM Role의 Trust Policy에 Grafana 측 IAM User ARN을 직접 지정했는데, Grafana Cloud가 이 내부 IAM User를 주기적으로 로테이션한다는 걸 나중에 확인 → User가 바뀔 때마다 403 AccessDenied 재발
- **해결** — Trust Policy의 Principal을 특정 User ARN 대신 Grafana Cloud AWS 계정의 `root`로 설정, 어떤 User로 로테이션되든 같은 계정 루트 권한으로 커버되어 재발 없음
- **미설정** — External ID 조건도 추가 시도했으나, 이 환경에서는 매칭 오류가 발생해 결국 미사용으로 원복

## 결과

- CloudWatch, 비용, 온프렘 서버 모니터링을 Grafana Cloud 하나로 통합
- IAM Trust Policy를 `root` Principal로 바꾼 뒤 AssumeRole 403 에러 재발 없음
- 팀원 전체가 브라우저로 대시보드 접근 가능 (기존에는 콘솔 접근 권한자만 확인 가능)

## 회고

Cross-Account Role 인증에서 특정 User ARN을 Trust Policy에 지정하는 방식은 SaaS 서비스 연동에서 흔히 저지르는 실수라는 걸 체감
SaaS 제공자 쪽 내부 자격증명이 바뀔 수 있다는 전제를 깔고, 계정 단위(root)로 신뢰 범위를 잡는 게 맞는 방향
다음에 다른 SaaS를 AWS와 연동할 때도 이 패턴을 먼저 확인할 계획
External ID는 보안상 더 안전한 선택지인데 이번 환경에서는 적용하지 못한 점이 아쉬움, 추후 재검증 예정

## 관련 문서
- [[02-AWS 비용 모니터링 구축]]
- [[03-온프렘 서버 모니터링 구축]]
- [[05-핵심 서비스 CloudWatch-Grafana Alert 설계]]
- [[06-Grafana Alert AI 조치 가이드 자동화]]
