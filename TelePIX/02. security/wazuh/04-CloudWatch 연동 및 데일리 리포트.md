---
title: Wazuh → CloudWatch Logs 연동 및 데일리 리포트
date: 2026-06-25
tags: [wazuh, cloudwatch, lambda, slack, iam-roles-anywhere, daily-report]
status: active
---
## 배경

Wazuh Dashboard에서 실시간으로 이벤트를 볼 수 있지만, 매일 아침 대시보드를 열어서 확인하는 건 비현실적
팀 전체가 인지하려면 Slack으로 요약 리포트가 와야 함
Wazuh 이벤트를 CloudWatch Logs로 푸시하고, Lambda로 집계해서 매일 아침 Slack에 리포트를 보내는 파이프라인 구축

## 접근

Wazuh 자체 email alert 기능도 있지만, 이미 [[07-daily-monthly-report-automation|CloudTrail/GuardDuty 이벤트를 Lambda로 집계해서 Slack에 보내는 구조]] 존재
Wazuh 이벤트도 같은 파이프라인에 녹여 넣는 게 관리 포인트를 줄이는 방향으로 판단

Wazuh → CloudWatch 푸시는 [[01-온프렘 PKI 기반 IAM Roles Anywhere 구축|CA 발급 관리 계정]]이 전담
IAM Roles Anywhere로 AWS 자격증명을 획득하고, `docker exec`으로 Wazuh 컨테이너 내 알림 로그(`alerts.json`)에 접근해서 5분마다 CloudWatch에 배치 전송
Lambda는 매일 06:00 KST에 실행되어 전날 24시간치 이벤트를 집계하고 Slack으로 전송

Lambda 코드를 처음 작성할 때 `get_log_events` API를 쓰면서 스트림명을 하드코딩
실제 CloudWatch 스트림명은 `2026/06/29/[$LATEST]abc123...` 형태라 매일 빈 결과를 반환하며 조용히 종료
나중에 `filter_log_events`로 교체해서 해결

## 구현

### 전체 흐름

```
Wazuh Manager (Docker)
└─ /var/ossec/logs/alerts/alerts.json
↓ docker exec + cron 5분 주기 (CA 발급 관리 계정)
wazuh_to_cloudwatch.sh
└─ IAM Roles Anywhere → 온프렘 자격증명용 IAM Role
↓ aws logs put-log-events
CloudWatch Logs (/wazuh)
↓ EventBridge (매일 06:00 KST)
Lambda (데일리 리포트)
└─ filter_log_events → level 기준 집계
↓
Slack 데일리 리포트
```

### Wazuh → CloudWatch 푸시

5분마다 실행하는 스크립트
상태 파일에 마지막 전송 timestamp를 저장해서 중복 전송 방지

```bash
*/5 * * * * /bin/bash /usr/local/bin/wazuh_to_cloudwatch.sh >> wazuh_cw.log 2>&1
```

동작 순서는 상태 파일에서 마지막 전송 timestamp를 읽고, `docker exec`으로 `alerts.json`을 읽어서 `rule.level >= 4` 필터링 후 마지막 전송 시각 이후 이벤트만 추출해 CloudWatch에 배치 전송

`~/.aws/config`에서 `credential_process` 값은 반드시 한 줄로 작성 필요
`\` 이어쓰기를 지원하지 않아 멀티라인으로 작성하면 credential-process 실행 오류 발생

| 항목 | 값 |
|---|---|
| CloudWatch Log Group | `/wazuh` |
| 리전 | ap-northeast-2 |
| 보존 기간 | 90일 |
| 필터링 기준 | `rule.level >= 4` |

### Lambda 리포트 구조

Wazuh CloudWatch 로그는 JSON 형태로 축적
Lambda에서 `filter_log_events`로 전날 24시간치를 가져와서 `rule.level` 기준으로 분류

```
📋 인프라 데일리 리포트 | 2026-07-13

🔴 Critical (level 11+) 0건
🟠 High (level 8-10) 0건
🟡 Warning (level 5-7) 279건

🔑 SSH 인증 실패 — 3건
📁 FIM 변경 감지 — 12건
⚙️ Systemd 서비스 오류 — 4건
```

[[02-FIM 노이즈 제거]]와 [[03-커스텀 룰 관리]]를 거치면서 High 건수가 2,725건에서 0건으로 감소
이제 리포트에서 실제로 확인이 필요한 이벤트만 노출

## 회고

CloudTrail/GuardDuty 이벤트와 Wazuh 이벤트를 같은 Lambda 하나에서 집계하는 구조가 깔끔
매일 아침 Slack 한 곳에서 인프라 전체 상황 확인 가능

`get_log_events` 스트림 하드코딩 실수로 초기에 Wazuh 섹션이 항상 빈 결과로 노출
Lambda 실행 로그에 로깅이 없어 문제를 뒤늦게 발견
처음부터 주요 단계마다 로깅을 넣었으면 바로 잡을 수 있었을 것

S3 데이터 이벤트 로깅이 아직 없어 파일 레벨 접근 추적 불가
[[01-MikroTik-Suricata-Wazuh 온프렘 IDS 구축|작년 크립토마이닝 미탐지 사고]] 때도 S3 오브젝트 접근 주체를 특정하지 못한 게 이 때문
비용 검토 후 활성화 고려 중 (이 공백은 이후 [[IAM 액세스키 탈취 인시던트 대응|8월 IAM 액세스키 탈취 인시던트]]에서도 그대로 재발)

## 관련 문서
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
- [[01-에이전트 설치 및 구성]]
- [[02-FIM 노이즈 제거]]
- [[03-커스텀 룰 관리]]
- [[01-온프렘 PKI 기반 IAM Roles Anywhere 구축]]
- [[07-daily-monthly-report-automation]]
- [[온프렘 보안 모니터링 스택 구축|프로젝트 전체 타임라인]]
