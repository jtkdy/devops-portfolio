---
title: 콘솔에서 수동으로 만든 ALB를 Terraform으로 편입하기
date: 2026-07-20
tags: [terraform, aws, alb, ecs, gpu, drift]
status: done
---
## 배경

[[02-GPU 딥러닝 모델 서빙 ECS 이관|GPU 딥러닝 서빙 ECS 이관]] 당시 급하게 트래픽부터 받아야 해서 ALB, 타겟그룹, 리스너룰을 AWS 콘솔에서 수동으로 생성
문제는 이 리소스들이 Terraform state 밖에 있다는 것
다른 인프라는 전부 코드로 관리되는데 이 부분만 콘솔에서 직접 만질 수 있는 상태로 남아있으면, 누군가 실수로 콘솔에서 설정을 바꾸는 순간 코드와 실제 상태가 어긋나는 드리프트가 발생하고 다음 `terraform plan`에서 예상치 못한 변경이 튀어나올 위험
신규 모델 서비스가 하나둘 늘어나기 전에, 이 리소스들을 Terraform으로 편입해서 코드가 유일한 소스가 되도록 정리하기로 결정

## 접근

새로 만들지 않고, 있는 걸 그대로 코드로 가져오기
신규 리소스를 생성하면 트래픽이 잠깐이라도 끊길 수 있는데, 이미 정상 운영 중인 ALB를 굳이 재생성할 이유가 없음
`terraform import`로 기존 리소스를 state에 편입시키고, 코드가 실제 설정과 완전히 일치하는지 `terraform plan`으로 확인하는 순서로 진행

타겟그룹에 GPU 노드를 등록하는 방식이 고민 됐는데 ECS 서비스의 `load_balancer` 블록을 쓰면 배포할 때마다 자동으로 타겟 등록/해제가 됨
이 클러스터는 EC2 launch type에 인스턴스가 1대뿐이라 자동 등록 방식이 잘 맞지 않음
대신 GPU 노드(EC2 인스턴스) ID와 포트를 직접 지정하는 `aws_lb_target_group_attachment`로 수동 등록하는 방식을 채택하여 노드가 교체되면 attachment도 같이 갱신해야 한다는 제약이 남지만, 지금 규모(단일 노드)에서는 이 편이 오히려 더 명시적이고 예측 가능하다고 판단

## 구현

- `infra/aws/load_balancer.tf` — 레거시 ALB(`coreservice-ai-alb`) Terraform 코드로 이관
- `infra/aws/lb_listener.tf` — HTTPS(443) 리스너 이관, default action 404 fixed-response
- `infra/aws/target_group.tf` — 서비스별 타겟그룹 3개 이관 (`target_type=instance`, 헬스체크 `/health`)
- `infra/aws/lb_listener_rule.tf` — host-header 리스너룰 3개(priority 10/11/12) 코드화 후 import

| 서비스 | 타겟그룹명 | 포트 |
|---|---|---|
| object-detection | coreservice-object-detection-tg | 8765 |
| change-detection | coreservice-change-detection-tg | 8767 |
| mangrove | coreservice-mangrove-tg | 8032 |

- 레거시 `ai-service` 전용 미사용 타겟그룹 제거
- `project/coreservice/gpu/ecs_service.tf` — `aws_lb_target_group_attachment`로 GPU 노드-타겟그룹 연결 구성 (인스턴스 ID + 포트 방식)

## 결과

- **편입 완료** — ALB 1개/리스너/타겟그룹 3개/리스너룰 3개, 신규 생성 없이 Terraform 관리 범위로 편입, 트래픽 경로·설정 변경 없음
- **드리프트 방지** — 이후 변경은 Terraform 필수 경유, 콘솔 직접 수정 시 다음 plan에서 드리프트로 감지
- **레포 분리** — `infra/aws/`(ALB/타겟그룹/리스너/리스너룰) ↔ `project/coreservice/gpu/`(타겟그룹 attachment), 공용/서비스전용 레포 계층 구조 유지

## 회고

수동으로 만들어둔 리소스를 나중에 IaC로 편입하는 작업은 신규로 뭔가를 만드는 것보다 오히려 어려운데 실수하면 "없던 걸 만드는" 게 아니라 "이미 트래픽을 받고 있는 걸 건드리는" 것이기 때문
`import` 전에 반드시 `plan`으로 실제 상태와 코드가 완전히 일치하는지 확인하는 습관이 이번에 확실히 자리 잡음

`aws_lb_target_group_attachment`를 인스턴스 ID로 고정하는 방식은 지금 규모에서는 괜찮지만, GPU 노드를 오토스케일링하거나 여러 대로 늘리는 순간 이 방식은 한계에 부딪힘
지금은 명시적인 게 낫다고 판단했지만, 노드가 늘어나는 시점에는 ECS 서비스의 `load_balancer` 블록 기반 자동 등록으로 다시 전환 필요

이 작업을 진행하면서 예상치 못한 걸 하나 더 발견
`coreservice-gpu-cluster`의 ECS 서비스 3개가 Terraform뿐 아니라 콘솔(ECS Console V2)이 자동 생성한 CloudFormation 스택에도 동시에 소유된 상태
이 이중 소유권을 정리하는 과정에서 실제로 서비스 하나가 삭제·재생성되는 사고까지 발생 ([[coreservice-gpu CloudFormation 이중소유 정리 사고|별도 문서]] 정리)

## 관련 문서
- [[jtkdy/TelePIX/01. infra/00-index|01. infra 인덱스]]
- [[GPU 딥러닝 서빙 마이그레이션|프로젝트 전체 타임라인]]
- [[02-GPU 딥러닝 모델 서빙 ECS 이관]]
- [[coreservice-gpu CloudFormation 이중소유 정리 사고]]
