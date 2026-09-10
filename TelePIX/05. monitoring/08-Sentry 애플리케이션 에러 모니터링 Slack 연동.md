---
title: Sentry 애플리케이션 에러 모니터링 Slack 연동
date: 2026-09-08
tags: [monitoring, sentry, slack, error-tracking]
status: draft
---
## 배경

애플리케이션 레벨 에러를 파악하는 속도가 느려, 장애 발생 후 원인 파악까지 지연되는 문제 있음
기존 모니터링 스택([[01-Grafana 통합 모니터링 스택 구축|Grafana 기반 통합 모니터링]])은 인프라 지표 중심이라 애플리케이션 에러 자체를 추적하기엔 한계

## 접근

Sentry 도입해 애플리케이션 에러 트래킹, Slack 연동으로 실시간 알림 체계 구축
프로젝트별로 알림 채널 분리 — coreservice, Keycloak 각각 별도 채널로 라우팅해 담당 영역별로 확인하도록 구성

## 구현

- coreservice, Keycloak 서비스에 각각 Sentry 프로젝트 연동
- 서비스별 Slack 알림 채널 분리 구성

## 결과

- 장애 인지 시간 단축
- 개발자 간 에러 공유 속도 개선

## 회고



## 관련 문서
- [[00-index|05. monitoring 인덱스]]
- [[01-Grafana 통합 모니터링 스택 구축]]
