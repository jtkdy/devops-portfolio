---
title: 핵심 서비스 CloudWatch 대시보드와 Grafana Alert 설계
date: 2026-06-01
tags: [monitoring, grafana, cloudwatch, alert, slack]
status: done
updated: 2026-10-21
---

## 배경

회사 핵심 서비스(위성 이미지 처리/검색/AI 서비스)는 ECS, ALB, RDS, Redis, S3, ACM 등 여러 AWS 리소스로 구성돼 있는데, 장애가 나기 전까지는 어느 리소스가 임계치에 가까워지고 있는지 확인할 방법이 없는 상태
CloudWatch 콘솔을 리소스별로 하나씩 열어보는 방식으로는 전체 상태를 한눈에 파악하기 어려웠고, 장애 발생 시 Slack으로 자동 알림이 오는 체계도 부재
대시보드와 Alert를 함께 구축해서 이 공백을 메우기로 결정

## 접근

CloudWatch를 Grafana의 데이터소스로 직접 연결하고, 리소스군별로 Row를 나눠 대시보드를 구성하는 방향으로 결정
Alert 임계치를 잡으려면 기준선이 필요했는데, 기존 응답시간·에러율 실측 데이터도, 서비스팀과 합의한 목표치도 없는 상태
그래서 가용성·에러율·응답시간 SLI 패널부터 만들고, SLO 수치는 일반적인 웹 서비스 수준을 참고한 임시 기준으로 우선 설정
Alert Rule 임계치도 이 임시 기준에서 출발했고, 실측이 쌓이면 다시 조정하는 전제로 진행
처음에는 대시보드 구성에 집중, 이후 별도 작업으로 Slack Alert 연동과 IAM 인증 트러블슈팅을 이어서 진행

## 구현

### 대시보드 구성

CloudWatch를 IAM Role Assume 방식으로 연결하고, 서비스 리소스군별로 Row 구분

| Row | 대상 | 핵심 패널 |
|---|---|---|
| ECS | 서비스 7개(이미지처리/AI·LLM/검색·구매/프론트엔드 등) | CPU/Memory Utilization, Running Task Count, Network In/Out |
| ALB | 일반 API용 / LLM API용 ALB 2개 | Request Count, 4xx/5xx Error Rate, Latency(p50/p90/p99), Connection Count |
| RDS | 메인 RDS, pgvector용 Aurora | CPU, DB Connections, Read/Write IOPS·Latency, Free Storage, Burst Balance |
| Redis | ElastiCache 클러스터 | Freeable Memory, Swap Usage, CPU, Cache Hit Ratio, Evictions |
| S3 | 팀 데이터 버킷 | Bucket Size, Object Count, Put/Get Requests, 4xx/5xx Errors |
| ACM | 인증서 | Days to Expiry (45일 이상 Safe / 30일 이하 Yellow / 14일 이하 Red) |

SLI Row는 가용성(`(A-B)/(A+0.001)*100`), 에러율, Latency를 Expression으로 별도 계산해서 리소스 지표와 나란히 배치

![[sli-dashboard.png]]
*SLI Row 대시보드 (최근 24시간) — 가용성 99.7%, Error Rate 0.263%, p50/p90/p99 응답시간. 이 구간엔 5xx 스파이크가 섞여 있어 임시 기준(99.9%) 대비 소폭 미달로 잡힘*

### SLO 임시 기준

실측 데이터나 서비스팀 합의 없이 잡은 출발점
에러 버짓 운영, 기준 위반 시 대응 절차까지는 정의하지 않음

| 구분 | 가용성 | Error Rate | p50 | p90 | p99 |
|---|---|---|---|---|---|
| 일반 API | 99.9% 이상 | 0.1% 이하 | 500ms 이하 | 1s 이하 | 3s 이하 |
| LLM API | 99.9% 이상 | 1% 이하 | 3s 이하 | 10s 이하 | 30s 이하 |

LLM API는 응답 생성 시간이 길어 응답시간·에러율만 일반 API보다 느슨하게 잡음, 이 역시 실측 근거 없는 값
위성 데이터 파이프라인은 SQS 메시지 대기 시간 5분 이하, DLQ 메시지 수 0 유지를 임시 기준으로 적어둠(SQS 모니터링 패널은 아직 미구현)

### Slack Alert 연동

Alert Rule 하나당 severity 라벨은 하나만 가능해서, 같은 메트릭이라도 Warning/Critical을 별도 Rule로 분리
구간 중복을 막기 위해 Warning은 `IS WITHIN RANGE`, Critical은 `IS ABOVE` 조건으로 나눠 잡는 패턴으로 정착
`ServiceName = *` 같은 와일드카드 Dimension을 쓰면 Rule 하나로 여러 인스턴스를 동시에 감지 가능해 ECS 서비스별로 Rule을 복제할 필요 없음

Alert 메시지 템플릿에서 동적 수치를 표시할 때 `$values.A.Value`, `$values.C.Value` 같은 키는 `nil`이나 깨진 값으로 출력
`$values.reducer.Value`가 Reduce 단계 결과값을 정상적으로 참조하는 키라는 걸 확인하고 나서야 템플릿이 의도대로 동작

```
`{{ $labels.ServiceName }}` Memory 90% 초과 위험 (현재 `{{ $values.reducer.Value | printf "%.2f" }}`%)
```

### 주요 Alert Rule

| 대상 | Rule | 조건 | Severity |
|---|---|---|---|
| ECS | Service Down | RunningTaskCount < 1 | critical |
| ECS | CPU Warning/Critical | 80~89% / 90% 초과 | warning/critical |
| ALB | 5xx Warning/Critical | 10~49건 / 50건 초과 | warning/critical |
| ALB | Latency Warning/Critical | p99 3~5s / 5s 초과 | warning/critical |

RDS/Redis Alert Rule 생성 완료

![[alert-rule-critical.png]]
*Critical Alert Rule 목록 (Warning은 별도 폴더로 분리) — Service Down, ECS/RDS/Redis CPU, ALB 5xx/Latency 등 운영 중*

### IAM AssumeRole 반복 실패 트러블슈팅

- **문제** — Grafana → AWS CloudWatch 연동 도중 `AccessDenied: sts:AssumeRole` 에러가 반복 발생
- **원인** — Grafana Cloud가 내부적으로 쓰는 IAM User가 주기적으로 로테이션되는데, Trust Policy에 특정 User ARN을 고정해뒀던 것 (상세 원인 분석은 [[01-Grafana 통합 모니터링 스택 구축]] 참고)
- **해결** — Trust Policy Principal을 개별 User 대신 Grafana Cloud AWS 계정의 `root`로 변경

재발 시 점검 순서는 4단계로 정리

1. 에러 메시지에서 User ARN 추출
2. Trust Policy Principal 비교
3. `root` 여부 확인
4. 콘솔에서 직접 수정

## 결과

- 핵심 서비스 리소스(ECS/ALB/RDS/Redis/S3/ACM) CloudWatch 대시보드 신규 구축, Alert Rule 약 26개 운영 중 (Grafana Free 플랜 한도 500개 대비 여유)
- ECS/ALB Alert를 Slack으로 실시간 전송, Critical 알람은 멘션으로 즉시 알림
- IAM AssumeRole 403 에러 재발 없이 안정화

## 회고

대시보드보다 Alert 설계에 시간이 더 소요
특히 Warning/Critical을 하나의 Rule로 합치려다가 severity 라벨 제약 때문에 막혔던 부분, 템플릿 변수 문법이 문서화가 부실해서 직접 하나씩 찍어보고 확인해야 했던 부분이 기억에 남음
SLO는 실측 기간 없이 숫자부터 정한 게 아쉬운 점
기준선이 없으니 99.9%가 이 서비스에 맞는 목표인지, Alert 임계치가 너무 민감하거나 둔한지 판단 불가
순서를 다시 잡는다면 SLI 패널로 2~4주 실측 → 분포 기준으로 SLO 초안 → 서비스팀과 합의 → Alert 임계치 반영

남은 과제는 세 가지로 정리

- **SLO 재설정** — SLI 실측값 기반으로 임시 기준 재검토
- **SQS 모니터링** — 패널 자체가 아직 미구현
- **GPU 클러스터 Container Insights** — 활성화 여부 결정 필요

## 관련 문서
- [[01-Grafana 통합 모니터링 스택 구축]]
- [[06-Grafana Alert AI 조치 가이드 자동화]]
- [[04-ECS 로그 Grafana Loki 수집]]
- [[02-GPU 딥러닝 모델 서빙 ECS 이관|GPU 딥러닝 서빙 ECS 이관 (GPU Container Insights 미구현 항목의 출처)]]
