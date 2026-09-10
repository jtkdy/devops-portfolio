---
title: GPU 딥러닝 서빙 마이그레이션
date: 2026-08-27
tags: [index, moc, project, gpu, ecs]
status: active
---
## 타임라인

| 순서 | 날짜 | 문서 | 도메인 | 내용 |
|---|---|---|---|---|
| 1 | 2026-06-09 ~ 06-30 | [[02-GPU 딥러닝 모델 서빙 ECS 이관]] | infra | 온프렘 → ECS 이관 본작업. Grafana 검증을 계획에 내장, GPU 리소스 예약 충돌 등 실전 트러블슈팅 다수 |
| 2 | 2026-07-20 | [[03-ALB 타겟그룹 Terraform 편입]] | infra | 이관 때 급히 콘솔로 만든 ALB/타겟그룹을 Terraform으로 편입 |

## 참고 (이 프로젝트가 걸쳐있는 더 큰 문서)

GPU 서비스는 아래 두 모니터링 구축 작업에도 포함되어 있지만, 이 문서들 자체는 GPU 전용이 아니라 여러 서비스를 함께 다루는 더 큰 범위의 작업임

- [[04-ECS 로그 Grafana Loki 수집]] — ECS 로그 Loki 수집 작업에 GPU 서비스 로그 그룹도 포함됨
- [[05-핵심 서비스 CloudWatch-Grafana Alert 설계]] — GPU 클러스터 Container Insights는 아직 미구현 항목으로 남아있음

## 프로젝트 요약

- **왜**: 온프렘 GPU 서버는 이중화·모니터링·배포자동화가 전부 없는 상태
- **핵심 판단**: 단일 GPU 인스턴스에 서비스 3개를 VRAM 예산 맞춰 몰아넣고, 부족한 모니터링을 처음부터 계획에 포함
- **실행 중 배운 것**: 계획(VRAM 여유 계산) ≠ 실제(ECS 스케줄러 GPU 논리 카운팅) 가능성, 콘솔 급조 리소스는 기술 부채로 귀결 (ALB 수동 생성 → Terraform 편입 → CloudFormation 이중소유 사고)
- **지금 상태**: ECS 3개 서비스 정상 운영, ALB/보안그룹 분리 완료, Terraform 단독 관리로 정리 완료 — CI/CD 파이프라인과 GPU Container Insights는 아직 남은 과제

## 관련 문서
- [[jtkdy/TelePIX/00. projects/00-index|프로젝트 인덱스]]
