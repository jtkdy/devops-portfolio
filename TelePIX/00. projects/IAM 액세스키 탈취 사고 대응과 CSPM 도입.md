---
title: IAM 액세스키 탈취 사고 대응과 CSPM 도입
date: 2026-08-27
tags: [index, moc, project, security, incident, cspm]
status: active
---
GuardDuty가 severity 9.0 알림을 잡아낸 사고 대응이, 2주 뒤 CSPM(Cloud Security Posture Management) 도구 도입으로 이어진 흐름
사고 대응 문서와 그 결과로 만들어진 인프라 문서가 서로 다른 도메인 폴더(incidents, security)에 있어서 하나로 묶었음

## 타임라인

| 순서 | 날짜 | 문서 | 도메인 | 내용 |
|---|---|---|---|---|
| 1 | 2026-08-12 | [[IAM 액세스키 탈취 인시던트 대응]] | incidents | GuardDuty 탐지, 액세스키 탈취 대응. AWS 설정 감사 수단이 아예 없었다는 게 대응 과정에서 드러남 |
| 2 | 2026-08-24 | [[01-Prowler Self-Hosted 구축]] | security | 이 사고를 계기로 CSPM 도구(Prowler) Self-Hosted 구축. 최초 스캔에서 Critical 33건 포함 1,838건 FAIL 확인 |
| 3 | 2026-08-25 | [[02-Prowler 보안 대시보드 연동]] | security | Prowler 스캔 결과를 사내 개발자 포탈 보안 탭에 연동해서 상시 확인 가능하게 함 |

## 프로젝트 요약

- **왜**: 액세스키 탈취 사고 대응 중 "AWS 설정이 잘못돼 있어도 아무도 모르는 상태"라는 근본 공백이 드러남 (S3 데이터 이벤트 로깅 부재 문제와는 별개로, 설정 자체를 점검하는 수단이 없음)
- **핵심 판단**: 사고 재발 방지를 사후 대응(로깅 강화)뿐 아니라 사전 예방(설정 상시 감사)까지 같이 가져가기로 함
- **지금 상태**: Prowler로 매일 계정 전체를 스캔 중이고 결과를 개발자 포탈에서 바로 확인 가능 — High 527건 중 실제 조치/Mute 분류 작업은 아직 진행 중

## 관련 문서
- [[jtkdy/TelePIX/00. projects/00-index|프로젝트 인덱스]]
