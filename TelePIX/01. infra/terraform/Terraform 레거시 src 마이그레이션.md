---
title: Terraform 레거시 src/ 마이그레이션
date: 2026-07-13
tags: [terraform, migration, legacy, infra]
status: done
---
## 배경

`src/projects/*` 하위에 2026-04-22 이후 변경 이력이 없는 레거시 구조가 그대로 잔존
단순히 오래된 코드가 쌓여있는 줄 알았는데 state를 직접 뜯어보니 일부는 실제로 운영 중인 인프라를 관리 중
그냥 지웠다가는 VPC부터 RDS까지 같이 날아갈 수 있는 상황

처음엔 state 파일 크기만 보고 180B짜리 `t_apply`는 "빈 state니까 삭제 가능"으로 판정했으나 이 레포 전체의 backend(S3 버킷 + DynamoDB 락 테이블)를 선언 중
state가 비어있어도 코드와 실제 리소스가 전부 active 상태

## 접근

조사를 두 번 진행
첫 번째는 state 파일만 확인했고 두 번째는 `.tf` 소스 코드와 AWS 실제 리소스 조회 결과를 같이 대조

state에 남아있는 리소스가 `managed`인지 `data`(조회 전용)인지 구분하는 게 핵심
`aws_vpc`, `aws_route53_zone` 같은 건 state에 있어도 destroy 대상이 아니고 그냥 조회만 하는 거라 건드리면 안 되는 대상

정리 순서는 리스크 기준으로 설정:

1. 실제 리소스가 이미 없는 drift 항목 → `terraform state rm`으로 state에서만 제거
2. 독립적이고 단순한 리소스부터 `infra/aws`로 이관 (Tier 1)
3. 민감정보 포함되거나 여러 리소스와 얽힌 것 (Tier 2)
4. 공유 VPC 네트워크 전체 (Tier 3)
5. satchat GPU 인프라 (Tier 4)

## 구현

| 단계 | 대상 | 방식 |
| --- | --- | --- |
| 1. drift 정리 | state에만 남은 삭제 리소스 9건 (`satchat` EFS 관련 3건, `t_common` DocumentDB 관련 5건) | `terraform state rm`으로 state에서만 제거 |
| 2. t_common 이관 | KMS key/SG 4개/RDS parameter·subnet group(Tier 1) → RDS 인스턴스/ElastiCache/NAT·bastion/route(Tier 2) | `terraform import`로 순차 편입 |
| 3. t_base(VPC 네트워크) 이관 | VPC/subnet 8개/IGW/route table + S3 Gateway Endpoint(조회 중 신규 발견) | `terraform import`로 편입 |
| 4. satchat GPU 인프라 이관 | ECS 클러스터/ASG/ALB/리스너/target group/서비스 3개 | `terraform import`로 순차 편입 |
| 5. 마무리 | `t_apply` | `bootstrap/`으로 이름·위치 변경 |

이관 과정에서 발견한 것들

- **t_common**
  - legacy 코드에 없던 ElastiCache `log_delivery_configuration`, bastion EC2 수동 부착 IAM instance profile 등 drift 확인 후 코드에 반영
  - RDS 비밀번호는 placeholder+`ignore_changes`로 분리
  - 이관 대상 자체가 없던 ElastiCache user는 삭제
- **t_base** — `terraform import`가 원격 state를 즉시 바꾸는 특성 때문에 머지 전 VPC 삭제 시도 near-miss 발생(최소권한 원칙 덕에 실패로 그침), route table import는 VPC ID 기준이라는 함정 확인
- **satchat GPU** — 콘솔 수동 생성 리소스(리스너 규칙·target group 3개씩)를 실물 조회로 확인 후 편입, 미사용 `ai-service`는 destroy
- **마무리** — state key는 디렉토리 경로와 무관해 파일 이동만으로 실 인프라 영향 없이 종료

## 결과

- 약 2주에 걸쳐 레거시 `src/` 전체 종료
- drift 9건 정리, 레거시 프로젝트 5개 중 4개 이관 또는 삭제 완료
- `t_apply` → `bootstrap/`으로 정리되면서 pre-2026-04-22 세대 코드 마이그레이션 전부 종료

## 회고

이번 작업에서 얻은 건 legacy 코드를 그대로 믿으면 안 된다는 것
plan 자체가 안 돌아가는 구식 provider 때문에 수년간 수동으로 변경된 것들이 코드에 반영되지 않고 drift로 축적
security group 규칙, ElastiCache 로그 설정, EC2 IAM profile 등 실제 리소스와 코드가 다른 경우가 계속 발생, 매번 AWS 실제 리소스를 직접 조회해서 대조하는 게 필수

`terraform import`가 원격 shared state를 즉시 바꾼다는 것도 위험 포인트
VPC 삭제 시도 near-miss가 없었다면 이 작업 방식이 얼마나 위험한지 몰랐을 것으로 추정

## 관련 문서
