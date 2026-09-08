---
title: Lambda + EventBridge로 인프라 데일리/먼슬리 리포트 자동화
date: 2026-06-25
tags: [monitoring, lambda, slack, aws]
status: done
---

## 배경

Grafana 대시보드는 실시간 상태 확인에는 좋지만, 매일 아침 팀 전체가 "어제 무슨 일이 있었는지"를 한눈에 훑어볼 수 있는 요약본은 부재
비용 급등, 인프라 변경, 보안 이벤트를 각자 확인하는 방식은 놓치기 쉬운 구조
그래서 매일 아침 정해진 시각에 Slack으로 인프라 현황 요약을 자동 전송하는 리포트 시스템을 제작

## 접근

Lambda + EventBridge 스케줄 조합으로 데일리/먼슬리 리포트를 분리해서 구성
데일리는 평일 09:00 KST, 먼슬리는 매월 3일 09:00 KST로 설정, 먼슬리를 1일이 아닌 3일로 잡은 이유는 전월 비용 데이터가 완전히 집계되는 시점을 기다려야 했기 때문
수집 항목은 네 가지로 결정

- **비용** — Cost Explorer 집계 기준
- **인프라 변경 이력** — CloudTrail 이벤트
- **GuardDuty 보안 이벤트**
- **Wazuh 보안 이벤트**

## 구현

### 리소스 구성

| 리소스 | 역할 |
|---|---|
| Lambda (데일리) | 평일 09:00 리포트 생성/전송 |
| Lambda (먼슬리) | 매월 3일 09:00 리포트 생성/전송 |
| EventBridge Rule 2개 | 각 Lambda 트리거 스케줄 |
| Slack 채널 | 모니터링 알람 전용 채널 |

Lambda는 Python 3.12, Timeout 90초, Memory 128MB로 경량 구성, Slack Webhook URL은 환경변수로 주입

### 데일리 리포트 수집 항목

- **비용 현황** — 전전날 기준(Cost Explorer 집계 지연 감안), 총 비용/전주 동일 요일 대비 증감/Top 5 서비스 표시, 전주 대비 20% 이상 급등 시 경고
- **인프라 변경 이력**
  - 대상 — 최근 24시간 CloudTrail 이벤트(ECS 생성/수정, EC2 생성/종료, RDS 생성/삭제/수정, IAM Role 생성 등)
  - 조회 방식 — `ReadOnly=false` 필터 대신 EventName별 개별 조회(`CreateSession`이 24시간 내 1000건+로 MaxItems 한도를 초과해서 우회)
- **GuardDuty 보안 이벤트** — 최근 24시간, severity 2 이상, High/Medium은 최대 5건 상세 표시, Low는 건수와 설명만 표시
- **Wazuh 보안 이벤트** — 최근 24시간, level 4 이상, CloudWatch Logs로 전달된 Wazuh 로그를 조회해서 Critical(12+)/High(8-11)/Medium(4-7)으로 분류

### 먼슬리 리포트 수집 항목

전월 총 비용, 전전월 대비 증감(금액+비율), 일평균 비용, Top 10 서비스(비율 포함)를 정리해서 전송

### IAM 권한

Cost Explorer 조회, CloudTrail 이벤트 조회, GuardDuty Finding 조회, Wazuh 로그 그룹 읽기 권한만 인라인 정책으로 최소 부여

## 결과

- 평일 오전 자동으로 비용/인프라변경/보안이벤트 요약이 Slack에 게시
- 전주 대비 20% 이상 비용 급등 시 경고 표시로 이상 조기 발견 가능
- 먼슬리 리포트로 월별 비용 추이를 별도 조회 없이 확인 가능

## 회고

CloudTrail `ReadOnly=false` 필터가 MaxItems 한도에 걸린다는 걸 미리 몰랐다면 리포트가 중간에 끊기는 문제를 겪었을 상황
EventName별 개별 조회로 바꾼 게 우회책이지만 근본적으로는 더 세밀한 이벤트 필터링 방식을 고민해볼 필요
알려진 한계도 정리

- **Wazuh 커버리지 부족** — 보안 모니터링 서버 1대만 커버, 다른 서버는 에이전트 미설치
- **ConsoleLogin 이벤트 누락** — CloudTrail 탐지 범위에서 빠짐
- **ECS 상태 미포함** — 이 리포트가 아닌 Grafana Alert로 대체 중

다음에는 ConsoleLogin 이벤트 포함 여부와 Wazuh 에이전트 확대 설치를 검토 예정

## 관련 문서
- [[01-overview-grafana-monitoring-stack]]
- [[04-CloudWatch 연동 및 데일리 리포트|Wazuh 이벤트가 이 리포트에 합류하는 지점]]
- [[온프렘 보안 모니터링 스택 구축|프로젝트 전체 타임라인]]
