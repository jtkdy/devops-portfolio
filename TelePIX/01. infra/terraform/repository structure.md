---
title: terraform 레포 구조
date: 2026-08-25
tags:
  - terraform
  - aws
  - infra
  - reference
status: done
---
Telepix 인프라를 관리하는 Terraform 모노레포. AWS 리소스를 공유 인프라(`infra`), 내부 서버(`internal`), 프로젝트별 리소스(`project`)로 계층화하여 관리한다. 레거시 루트 모듈(`src/`)은 `infra/aws` + `project/*`로 전량 이관 완료되어 더 이상 존재하지 않는다.

---

## 최상위 디렉토리

```
telepix-terraform/
├── aws/           # 재사용 가능한 Terraform 모듈 (로컬 모듈 라이브러리)
├── bootstrap/     # Terraform state/IAM 등 부트스트랩 리소스 (자체 workspace)
├── docs/          # Wiki 문서 (Obsidian)
├── infra/         # 공유 기반 인프라 (VPC, ECS 클러스터, ALB, RDS 등)
├── internal/      # 내부 서버 (개발자 포탈 등)
├── project/       # 프로젝트별 ECS 서비스 (coreservice, coreservice/gpu, mps, sso)
├── scripts/       # Terraform plan/apply 자동화 셸 스크립트
└── bitbucket-pipelines.yml  # CI/CD 파이프라인
```

---

## `aws/` — 로컬 Terraform 모듈

다른 workspace에서 `source = "../../../aws/..."` 형태로 호출하는 공용 모듈.

```
aws/
├── ec2/
│   ├── security_group/    # Security Group
│   └── load_balancer/
│       ├── (ALB 본체)
│       ├── listener/      # ALB Listener
│       └── target_group/  # Target Group
├── ecs/
│   └── task_definition/   # ECS Task Definition
├── route53/               # Route53 레코드
└── vpc/
    ├── subnet/            # Subnet
    └── route/             # Route Table
```

> 2026-07 dead code 정리([[Dead Code 제거 및 중복코드 검토]])에서 미참조 모듈(`ec2/asg`, `ec2/launch_template`, `ecs/service`, `load_balancer/listener/rule` 등)을 삭제했다. `source`로 참조되지 않는 모듈이 남아 있다면 우선 사용처를 확인할 것.

---

## `infra/aws/` — 공유 기반 인프라

Terraform workspace: `infra/aws`

VPC, IGW, NAT/bastion, 서브넷, 공용 ALB, ECS 클러스터(coreservice/product/mps/coreservice_gpu), RDS, ElastiCache, ECR, KMS, ASG, IAM 기반 역할 등 모든 프로젝트가 공유하는 리소스. 레거시 `src/t_base`, `src/t_common` 등에서 이관된 리소스도 모두 이 workspace에 통합되어 있다.

| 파일 | 주요 리소스 |
|------|------------|
| `vpc.tf` | VPC, IGW, S3 VPC Endpoint |
| `route_table.tf` | 퍼블릭/프라이빗 라우트 테이블, 라우트 |
| `ec2.tf` | NAT 인스턴스, Bastion 인스턴스 |
| `security_group.tf` | 공용 SG (RDS/NAT/bastion/ElastiCache 포함) |
| `load_balancer.tf` | 공용 ALB (coreservice-ai-alb 포함) |
| `lb_listener.tf` | ALB Listener (HTTP/HTTPS) |
| `lb_listener_rule.tf` | Listener Rule |
| `target_group.tf` | Target Group |
| `asg.tf` | Auto Scaling Group (coreservice_gpu 포함) |
| `launch_template.tf` | EC2 Launch Template |
| `capacity_provider.tf` | ECS Capacity Provider |
| `ecs.tf` | ECS 클러스터 (`coreservice_cluster`, `product_cluster`, `mps_cluster`, `coreservice_gpu_cluster`) |
| `cloudwatch_alarm.tf` | GPU ASG 스케일링 알람 |
| `rds.tf` | RDS 인스턴스, 파라미터/서브넷 그룹 |
| `kms.tf` | RDS 암호화용 KMS 키 |
| `elasticache.tf` | ElastiCache 클러스터, 서브넷 그룹, 로그 그룹 |
| `ecr.tf` | ECR 리포지토리 (coreservice, payment) |
| `s3.tf` | Terraform env 변수용 S3 버킷 |
| `iam.tf` | 공용 IAM 역할/정책 |
| `data.tf` | Data source (AMI, AZ 등) |
| `local.tf` | 공통 로컬 변수 (클러스터 이름, 서브넷 역할 등) |
| `provider.tf` | AWS provider 설정 |
| `variable.tf` | 입력 변수 |

---

## `internal/` — 내부 서버

### `internal/_modules/` — 내부 전용 재사용 모듈

```
_modules/
├── ec2_role/         # EC2 IAM Role + Instance Profile (공통 정책 + 서비스별 extra_policy_arns)
└── internal-server/  # 내부 서버 공통 패턴 (EC2 + SG + Route53)
```

`ec2_role`은 `aws_iam_role` + SSM/CloudWatch 공통 정책 attachment + `aws_iam_instance_profile`을 묶은 모듈로, `extra_policy_arns` 변수로 서비스별 추가 정책을 주입한다.

### `internal/groundctrl/` — 개발자 포탈 서버

Terraform workspace: `internal/groundctrl`
도메인: `<포탈 URL>`

내부 운영용 서버. EC2 인스턴스에 정적 EIP 부여, ALB를 통해 외부 노출. IAM Role은 `_modules/ec2_role` 모듈로 관리.

| 파일 | 주요 리소스 |
|------|------------|
| `ec2.tf` | EC2 인스턴스, EIP |
| `ebs_data_volume.tf` | EBS 데이터 볼륨 |
| `security_group.tf` | SG |
| `iam_role.tf` | `_modules/ec2_role` 호출 + 서비스별 IAM Policy(secrets/read/s3) |
| `lb_listener_rule.tf` | 공용 ALB에 개발자 포탈 경로 추가 |
| `target_group.tf` | Target Group |
| `route53.tf` | DNS 레코드 (`<포탈 URL>`) |
| `secrets_manager.tf` | Secrets Manager 시크릿 |
| `cloud_watch_log.tf` | CloudWatch Log Group (보존 기간 설정) |
| `data.tf` | Data source |
| `local.tf` | 로컬 변수 (도메인, 태그) |

---

## `project/` — 프로젝트별 ECS 서비스

각 프로젝트는 `shared` (IAM 역할/정책)와 `product` (실제 ECS 리소스)로 분리.

### `project/coreservice/`

위성 영상 기반 AI 챗봇 서비스. 도메인: `*.telepix.ai`

**`coreservice/shared/`** — IAM 공유 리소스

| 파일 | 내용 |
|------|------|
| `iam_role.tf` | ECS Task Execution Role |
| `iam_policy.tf` | 커스텀 IAM 정책 |
| `iam_role_restriction.tf` | 역할 제한 설정 |

**`coreservice/product/`** — 일반 ECS 서비스 (Fargate 기반, on-demand)

Terraform workspace: `project/coreservice/product`

서비스별로 `_<서비스명>.tf` 패턴으로 분리:

| 파일 | 서비스 | 서브도메인 |
|------|--------|-----------|
| `_backend.tf` | Backend API | `coreservice-api.telepix.ai` |
| `_front_client.tf` | 클라이언트 프론트엔드 | `coreservice.telepix.ai` |
| `_front_admin.tf` | 어드민 프론트엔드 | `coreservice-admin.telepix.ai` |
| `_llm.tf` | LLM 서비스 | `coreservice-llm.telepix.ai` |
| `_processing.tf` | 처리 서비스 | `coreservice-processing.telepix.ai` |
| `_exchange.tf` | 데이터 교환 | `coreservice-snp.telepix.ai` |
| `_tile.tf` | 타일 서비스 | `coreservice-tile.telepix.ai` |

공통 파일:

| 파일 | 내용 |
|------|------|
| `ecs_task_definition.tf` | (dead code 정리로 전체 삭제됨 — 각 `_*.tf`에서 개별 관리) |
| `data.tf` | 공유 인프라 data source |
| `route53.tf` | DNS 레코드 일괄 관리 |
| `logs.tf` | CloudWatch Log Group |
| `s3.tf` | S3 버킷 |
| `local.tf` | 컨테이너 설정, 도메인 맵 |

**`coreservice/gpu/`** — GPU 기반 AI 추론 서비스 (EC2 launch type)

Terraform workspace: `project/coreservice/gpu`

`infra/aws`의 `coreservice_gpu_cluster`(EC2 ASG + capacity provider) 위에서 동작하는 GPU 서비스 3종(`object_detection`, `mangrove`, `change_detection`). Task definition은 CI/CD가 별도 등록(data source로 참조), `desired_count`는 lifecycle에서 무시.

| 파일 | 내용 |
|------|------|
| `ecs_service.tf` | ECS Service 3종 + ALB target group attachment |
| `cloudwatch.tf` | 서비스별 CloudWatch Log Group |
| `data.tf` | 공유 인프라 / task definition data source |
| `local.tf` | 태그, cost category 매핑 |
| `variable.tf` / `provider.tf` | 입력 변수, provider 설정 |

---

### `project/mps/`

MPS(Media Processing Service) 서비스.

**`mps/shared/`** — IAM 공유 리소스 (coreservice/shared와 동일 패턴)

**`mps/product/`** — ECS 서비스

| 파일 | 내용 |
|------|------|
| `ecs_ondemand_service.tf` | ECS 온디맨드 서비스 (frontend/backend/process) — `task_definition`은 CI/CD drift 방지를 위해 lifecycle에서 ignore |
| `ecs_task_definition.tf` | Task Definition |
| `iam_task_execution_role.tf` | Task Execution Role |
| `cloud_watch_log.tf` | CloudWatch 로그 |
| `secrets_manager.tf` | Secrets |
| `route53.tf` | DNS |

---

### `project/sso/`

SSO(Single Sign-On) 서비스. `sso.telepix.ai`로 서빙 (레거시 `sso.telepix.net`은 중단).

**`sso/shared/`** — IAM 공유 리소스

| 파일 | 내용 |
|------|------|
| `iam_role.tf` | IAM Role |
| `iam_policy.tf` | IAM Policy |
| `iam_role_restriction.tf` | 역할 제한 |
| `terraform-iam-define-role.json` | IAM Trust Policy JSON |

**`sso/product/`** — ECS 서비스

| 파일 | 내용 |
|------|------|
| `ecs.tf` | ECS 서비스 + Task Definition — `task_definition`은 CI/CD drift 방지를 위해 lifecycle에서 ignore |
| `iam_task_execution_role.tf` | Task Execution Role |
| `logs.tf` | CloudWatch 로그 |
| `secrets_manager.tf` | Secrets |

---

## `bootstrap/` — Terraform 부트스트랩

State 관리·IAM 등 다른 모든 workspace가 의존하는 최초 리소스를 별도 workspace로 분리 관리 (`main.tf`, `locals.tf`, `provider.tf`, `variable.tf`). 자동 plan 커버리지 대상에서 제외되어 있어 변경 시 별도 검증 필요.

---

## `scripts/` — 자동화 스크립트

| 파일 | 역할 |
|------|------|
| `aws-auth.sh` | AWS CLI 인증 (assume role) |
| `tf-plan.sh` | `terraform plan` 실행 |
| `tf-plan-pr.sh` | PR용 plan (diff 출력) |
| `tf-apply.sh` | `terraform apply` 실행 |

---

## 레이어 구조 요약

```
┌─────────────────────────────────────────┐
│  project/{coreservice,coreservice/gpu,mps,sso}  │  ← 서비스 ECS 리소스
│  /product                                │
├─────────────────────────────────────────┤
│  project/{coreservice,mps,sso}/shared       │  ← 서비스 IAM
├─────────────────────────────────────────┤
│  internal/groundctrl                    │  ← 내부 운영 서버
├─────────────────────────────────────────┤
│  infra/aws                              │  ← VPC, ALB, ECS 클러스터, RDS 등
├─────────────────────────────────────────┤
│  aws/ (로컬 모듈), internal/_modules/    │  ← 공용 Terraform 모듈
└─────────────────────────────────────────┘
```

각 레이어는 Terraform `data` 소스로 상위 레이어의 리소스를 참조하며, state를 직접 공유하지 않는다 (`root 끼리는 참조 X` 원칙, `README.md` Policy 참고).

---