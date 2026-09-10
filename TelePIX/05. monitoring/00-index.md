---
title: 05. monitoring 인덱스
date: 2026-08-27
tags: [index, moc, monitoring, grafana]
status: active
---
Grafana Cloud 기반 통합 모니터링 스택 구축 시리즈
전체 개요를 먼저 보고, 이후 영역별 구축기와 확장 기능 순으로 읽으면 흐름 연결

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [[01-Grafana 통합 모니터링 스택 구축]] | 전체 개요 — 왜 Grafana Cloud로 통합했고 Cross-Account IAM 인증을 어떻게 풀었는지 (기반 문서) |
| 2 | [[02-AWS 비용 모니터링 구축]] | CUR 2.0 + Athena로 AWS 비용을 서비스별/리소스별로 쪼개보기 (Glue Table 자동생성 함정, Athena 인용규칙 포함) |
| 3 | [[03-온프렘 서버 모니터링 구축]] | Grafana Alloy + Node Exporter로 온프렘 서버 4대 통합 |
| 4 | [[04-ECS 로그 Grafana Loki 수집]] | ECS 서비스 로그를 Loki로 실시간 수집 |
| 5 | [[05-핵심 서비스 CloudWatch-Grafana Alert 설계]] | 핵심 서비스 CloudWatch 대시보드 + SLO 기반 Alert 설계 |
| 6 | [[06-Grafana Alert AI 조치 가이드 자동화]] | (05 확장) Grafana Alert에 OpenAI 조치 가이드 자동 응답 붙이기 |
| 7 | [[07-인프라 데일리-먼슬리 리포트 자동화]] | Lambda + EventBridge로 인프라 현황 데일리/먼슬리 리포트 자동 전송 |
| 8 | [[08-Sentry 애플리케이션 에러 모니터링 Slack 연동]] | Sentry로 애플리케이션 에러 트래킹, coreservice/Keycloak 서비스별 Slack 알림 채널 분리 |

## 관련 문서
- [[jtkdy/TelePIX/00-index|jtkdy 전체 인덱스]]
- [[jtkdy/TelePIX/04. 개발자 포탈/00-index|04. 개발자 포탈 인덱스]] (06번 장애 관리 자동화가 이 알림 체계를 참조)
