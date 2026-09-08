---
title: GPU 클러스터 CloudFormation 이중소유 정리 사고
date: 2026-07-23
tags: [aws, cloudformation, ecs, terraform, incident]
status: done
---
## 배경

[[03-ALB 타겟그룹 Terraform 편입|ALB를 Terraform으로 편입하는 작업]]을 하다가 예상 못 한 사안을 하나 더 발견
GPU 클러스터의 ECS 서비스 3개(object-detection, change-detection, mangrove)가 Terraform뿐 아니라 CloudFormation 스택에도 동시에 소유된 상태
AWS 콘솔(ECS Console V2)에서 서비스를 만지면 `ECS-Console-V2-Service-*`라는 이름의 CloudFormation 스택이 자동으로 생성돼서 그 서비스를 추적하기 시작하는데, 과거에 누군가 콘솔에서 이 서비스들을 편집하면서 같은 물리 리소스(ARN)를 CloudFormation도 동시에 소유하게 된 상태
정리하지 않으면 Terraform과 CloudFormation이 같은 리소스를 서로 다르게 "내 것"이라고 주장하는 구조가 계속 남는 셈

세 스택의 상태부터 제각각
change-detection 스택은 `CREATE_COMPLETE`로 정상 종료됐지만, mangrove와 object-detection은 배포 서킷 브레이커가 걸려서 `CREATE_FAILED` 상태 — 그런데 이 두 서비스는 실제로는 정상 동작 중
스택은 실패했는데 실물은 멀쩡한, 다뤄야 하는 상태가 스택 상태와 다른 까다로운 상황

## 접근

`CREATE_COMPLETE`인 change-detection 스택은 간단
`DeletionPolicy: Retain`을 걸어두고 스택만 지우면, 실제 ECS 서비스는 그대로 남고 CloudFormation의 소유권만 소멸
문제는 `CREATE_FAILED` 상태인 나머지 둘
이 상태의 스택은 `DeletionPolicy`를 걸려면 `update-stack --disable-rollback`을 쓰는 것 외에 다른 경로가 없었는데, 이 명령이 정확히 무슨 부작용을 낼지 사전에 완전히 예측하지 못한 채로 mangrove부터 시도

## 구현

### mangrove 처리 중 발생한 사고

`update-stack --disable-rollback`을 실행했더니, CloudFormation이 "실패한 리소스만 재생성"하는 동작 방식 때문에 실제로 운영 중이던 mangrove ECS 서비스가 삭제된 뒤 재생성
몇 초간 서비스가 `DRAINING`(desiredCount 0) 상태로 빠졌다가 약 3분 뒤 자동으로 새 서비스가 `ACTIVE`로 복구됐지만, 문제는 여기서 끝나지 않음

- task definition revision 1(구버전)로 리셋
- 배포 설정도 콘솔 기본값으로 리셋 → Terraform 코드와 불일치 (`deployment_minimum_healthy_percent` 0→100, `deployment_circuit_breaker` false/false→true/true)
- revision 6 수동 재배포 시도 — `minimumHealthyPercent: 100` + GPU 인스턴스 1대 조건에서 신규/기존 태스크 포트(8032) 충돌로 배치 실패
- 서킷 브레이커 감지 → 자동 롤백, revision 1로 복귀

이 연쇄를 끊은 건 `terraform plan`으로 드리프트를 먼저 확인하고, `terraform apply`로 `minimum_healthy_percent`/`circuit_breaker` 설정을 코드값(0 / false,false)으로 되돌린 조치
설정을 원복한 뒤 revision 6 재배포를 다시 시도했더니 포트 충돌 없이 정상 완료

### 안전한 정리 절차로 재정립

이 사고를 겪고 나서, 스택 상태에 따라 절차 분리

**터미널 상태(`*_COMPLETE`)인 스택** — 무중단으로 정리 가능함을 확인
1. `create-change-set`으로 `DeletionPolicy: Retain` 추가만 미리보기 — `Scope: [DeletionPolicy]`, `Replacement: False`인지 반드시 확인해서 실제 리소스 속성 변경이 없는지 검증
2. `execute-change-set`으로 적용 (메타데이터만 바뀌고 리소스는 무변화)
3. `delete-stack` — `DeletionPolicy: Retain` 덕분에 스택만 삭제되고 실제 ECS 서비스는 보존됨
4. 서비스/타겟그룹 헬스 상태 재확인

**`CREATE_FAILED` 상태인 스택** — `update-stack --disable-rollback`이 유일한 경로이고, 서비스 삭제 후 재생성으로 인한 일시 중단이 불가피하다는 걸 인정할 수밖에 없었음
object-detection 처리할 때는 이 교훈을 반영해서, 진행 전에 현재 task definition revision과 배포 설정을 미리 기록해두고, 예상되는 일시 중단을 담당자에게 사전 고지한 뒤 진행
실제로 예상된 다운타임만 발생, 추가 사고 없음

### 무관한 프로젝트의 잔재 스택도 같이 발견됨

작업 도중 satchat-gpu와 무관한 다른 프로젝트(SSO 인증, 백오피스, 사내 다른 서비스 등) 소유의 실패한 CloudFormation 스택 6개도 확인
궁금해서 넘어가지 않고, 각 스택이 참조하는 리소스(TargetGroup, SecurityGroup 등)를 `describe-stack-resources`와 실제 AWS API(`elbv2`, `ec2`, `ecs`) 조회로 전수 확인
결과는 6개 모두 참조 리소스가 AWS 상에 전혀 남아있지 않은, 완전히 죽은 스택 — 일반 `delete-stack`(또는 콘솔 "Retry delete" → Force delete)으로 안전하게 삭제 가능한 상태
다만 이건 이번 작업 범위(satchat-gpu)를 벗어나는 다른 팀 소유 리소스라, 검토 결과만 남기고 실제 삭제는 각 서비스 담당자 승인을 받아 별도로 진행하도록 인계

## 결과

| 서비스 | 처리 방식 | 결과 |
|---|---|---|
| change-detection | Retain change-set → delete-stack (무중단) | Terraform 단독 관리 전환 완료 |
| mangrove | disable-rollback update(사고 발생) → 드리프트 복구 → 재배포 → delete-stack | Terraform 단독 관리 전환 완료 |
| object-detection | 사전 상태 기록 → disable-rollback update(예상된 다운타임) → 재배포 → delete-stack | Terraform 단독 관리 전환 완료 |

세 서비스 모두 CloudFormation 이중 소유에서 벗어나 Terraform 단독 관리로 정리, 무관한 프로젝트의 죽은 스택 6개도 안전 여부까지 확인된 상태로 각 담당자에게 인계

## 회고

`CREATE_FAILED` 상태의 스택이라고 해서 실제 리소스도 없을 거라고 가정한 게 이번 사고의 원인
스택 상태와 실제 리소스 상태는 별개라는 걸, 실제로 서비스 하나를 삭제·재생성시켜보고 나서야 확인
`update-stack --disable-rollback` 같은 명령을 실행하기 전에는 항상 `ecs describe-services` 같은 실물 조회로 먼저 확인하는 습관 정착

사고를 되짚어보면, 대응 자체는 나쁘지 않았다고 판단
`terraform plan`으로 드리프트를 명확히 짚어내고 코드값으로 원복하는 방식은 "일단 콘솔에서 손으로 되돌리자"는 유혹보다 훨씬 안전
이 경험을 바로 다음 서비스(object-detection) 처리에 반영해서 사전 상태 기록과 사전 고지라는 절차를 추가한 것도, 같은 실수를 반복하지 않는 방향으로 제대로 작동

무관한 6개 스택을 발견했을 때 그냥 지나치지 않고 안전성까지 확인해둔 건 잘한 선택으로 판단
다만 실제 삭제까지 내 선에서 처리하지 않고 각 담당자에게 넘긴 것도 맞는 판단 — 확인은 기술적으로 할 수 있어도, 다른 팀 소유 리소스를 삭제할 권한과 맥락은 갖고 있지 않았음

## 관련 문서
- [[jtkdy/TelePIX/03. incidents/00-index|03. incidents 인덱스]]
- [[GPU 딥러닝 서빙 마이그레이션|프로젝트 전체 타임라인]]
- [[03-ALB 타겟그룹 Terraform 편입]]
- [[02-GPU 딥러닝 모델 서빙 ECS 이관]]
