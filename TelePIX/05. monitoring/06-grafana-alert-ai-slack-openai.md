---
title: Grafana Alert에 OpenAI 기반 AI 조치 가이드 자동 응답 붙이기
date: 2026-06-11
tags: [runbook, grafana, lambda, slack, openai, api-gateway]
status: done
---

## 배경

Grafana Alert가 Slack으로 날아와도 "무슨 조치를 해야 하는지"는 결국 담당자가 알람 내용을 보고 직접 판단해야 하는 구조
온콜 대응 초기에 조치 방향을 빠르게 잡는 데 시간이 소요된다는 문제, 알람 메시지에 상황별 대응 가이드를 자동으로 붙여주면 초기 대응 속도를 줄일 수 있다고 판단해 AI 기반 조치 가이드 자동 응답 파이프라인을 제작

## 접근

기존 Slack 알람 메시지(Contact Point 기반)는 그대로 유지하면서, 같은 Contact Point에 Webhook 통합을 추가하는 방식으로 결정
Lambda가 Webhook으로 알람을 받아서 OpenAI를 호출해 조치 가이드를 생성하고, 원본 알람이 올라온 Slack 스레드에 Reply로 붙이는 구조
기존 알람 흐름을 건드리지 않고 얹는 방식이라 롤백 부담 적음

## 구현

### 아키텍처

```
Grafana Cloud Alert (firing)
↓
Contact Point
├── Slack 통합 → 원본 알람 메시지 (기존)
└── Webhook 통합 → API Gateway POST /webhook
↓
Lambda (1차 호출) → 즉시 200 응답 + 자기 자신 비동기(Event) 재호출
↓
Lambda (2차, 비동기)
↓
원본 알람 스레드 탐색 (conversations_history)
↓
OpenAI (gpt-4.1-mini) 호출 → 조치 가이드 생성
↓
Slack 스레드에 AI 조치 가이드 Reply
```

### 구성

| 항목 | 내용 |
|---|---|
| Lambda Runtime | Python 3.12, x86_64 |
| Timeout / Memory | 30초 / 256MB |
| API Gateway | HTTP API, $default stage |
| AI 모델 | gpt-4.1-mini (OpenAI) |
| 시크릿 관리 | Secrets Manager (OpenAI API Key, Slack Bot Token 분리 저장) |
| Slack Bot 권한 | `chat:write`, `channels:history`, `groups:history` |

IAM 인라인 정책은 Secrets Manager 조회와 Lambda 자기 자신 재호출(비동기 패턴용) 권한만 최소로 부여
`AWSLambdaBasicExecutionRole` 관리형 정책도 별도로 붙여야 CloudWatch Logs가 남는다는 걸 트러블슈팅 과정에서 확인

### 핵심 코드 동작

Grafana Webhook의 응답 대기 시간이 짧다는 제약 때문에, 1차 호출에서는 즉시 200을 반환하고 실제 처리는 Lambda가 자기 자신을 비동기(Event)로 재호출해서 처리하는 패턴 채택

```python
def lambda_handler(event, context):
    if event.get("_async_invoke"):
        return process_alert(event.get("body", "{}"))

    get_lambda_client().invoke(
        FunctionName=context.invoked_function_arn,
        InvocationType="Event",
        Payload=json.dumps({"_async_invoke": True, "body": event.get("body", "{}")}).encode(),
    )
    return {"statusCode": 200, "body": json.dumps({"message": "accepted"})}
```

### 트러블슈팅

구축 과정에서 4가지 문제를 순서대로 처리

1. **로그가 아예 안 남음** — Grafana Test 시 500 에러가 났는데 CloudWatch 로그 그룹 자체가 부재, 원인: Lambda는 정상 실행되고 있었지만 IAM Role에 `AWSLambdaBasicExecutionRole`이 없어서 로그 전송 권한 자체가 없었음, 진단 팁: curl로 엔드포인트를 직접 호출해서 200과 정상 응답을 받으면 "Lambda는 정상, 로그 권한만 문제"로 좁힐 수 있음
2. **Slack `missing_scope` 에러** — `conversations_history` 호출 시 채널이 Private인데 `groups:history` 스코프가 없어서 발생, Public 채널은 `channels:history`로 충분하지만 Private은 별도 스코프 필요, Slack App에 스코프를 추가한 뒤 **Reinstall to Workspace**를 해야 기존 토큰에 반영, Secrets Manager 토큰도 함께 갱신해야 한다는 점이 놓치기 쉬운 포인트
3. **`AttributeError: 'NoneType' object has no attribute 'items'`** — Grafana의 "Test notification"이 `"values": null`로 전송하는데, 실제 알람은 값이 있어서 테스트에서만 터지는 버그, 해결: `(alert["values"] or {}).items()` 형태로 None-safe 처리
4. **Lambda는 성공(200)인데 Grafana는 실패로 표시** — OpenAI 응답 시간(~20초)이 Grafana Webhook 테스트 타임아웃(추정 10~15초)보다 길어서 Grafana가 먼저 타임아웃 처리하는 문제, 이게 바로 "즉시 200 + 비동기 재호출" 패턴을 도입한 배경

## 결과

- Grafana Alert 발생 시 Slack 알람 스레드에 AI 조치 가이드가 자동으로 첨부
- 월 예상 비용은 알람 100건 기준 약 $0.13, 1,000건 기준 약 $1.30 (gpt-4.1-mini 기준) + Secrets Manager 고정비 약 $0.80/월
- 기존 Slack 알람 흐름 유지, 부가 기능만 추가 → 운영 중단 없이 배포

## 회고

"즉시 200 + 자기 자신 비동기 재호출" 패턴은 Webhook 타임아웃 문제를 우회하는 실용적인 해법이었으나, Lambda 재귀 호출 자체가 잘못 설계되면 무한 루프로 이어질 수 있는 위험한 패턴이라 `_async_invoke` 플래그로 명확히 분기 필요
Slack 스코프 변경 후 Reinstall이 필요하다는 걸 몰라서 한동안 헤맴, Slack App 권한 변경 작업은 항상 "스코프 추가 → Reinstall → 토큰 갱신"을 세트로 기억할 계획
다음 단계로는 버튼 클릭 시에만 AI를 호출하는 방식(현재는 알람 발생 시 무조건 호출)과 Slack Signing Secret 검증 추가를 계획 중

## 관련 문서
- [[05-satchat-alert-integration]]
- [[01-overview-grafana-monitoring-stack]]
