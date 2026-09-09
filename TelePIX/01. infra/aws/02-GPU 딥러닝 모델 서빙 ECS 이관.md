---
title: 온프렘 GPU 딥러닝 서빙을 ECS로 이관하며 Grafana 검증까지 붙이기
date: 2026-06-09
updated: 2026-06-30
tags: [ecs, gpu, deeplearning, aws, monitoring, troubleshooting]
status: done
---
## 배경

핵심 서비스의 딥러닝 추론 3종(객체 탐지, 변화 탐지, 맹그로브 탐지)이 온프렘 GPU 서버에서 사람이 직접 실행하는 방식으로 운영 중
문제는 네 가지로 정리

- **이중화 부재** — 서버가 죽으면 서비스가 그대로 멈춤
- **배포 자동화 없음** — 수동 접속 후 직접 실행
- **모니터링 없음** — VRAM OOM(메모리 초과) 같은 이상 상황을 감지할 수단 자체가 없음
- **가중치 관리 비표준** — bind-mount·자동 다운로드·NAS로 서비스마다 제각각 관리

이 중 특히 모니터링 부재가 이번 이관의 방향을 결정

이번 작업의 핵심은 단순히 서버를 옮기는 게 아님
온프렘에서 겪던 "모니터링 없음" 문제를 그대로 ECS로 옮겨가면 이관의 의미가 반으로 줄어든다고 판단
그래서 처음부터 이관 계획 안에 Grafana로 GPU 사용률·VRAM 사용량·OOM 알람까지 확인하는 검증 단계를 포함

## 접근

기존 ECS 클러스터에는 Triton 기반 서빙 서비스가 이미 있었지만 실제 실행 태스크는 0개, 사실상 미사용 상태
대안을 네 가지로 놓고 비교

| 옵션                 | 장점                  | 단점                                       |
| ------------------ | ------------------- | ---------------------------------------- |
| 온프렘 유지             | 변경 없음               | 이중화/모니터링/배포 자동화 없음                       |
| ECS + Triton 유지    | 기존 인프라 재활용          | config.pbtxt 복잡도 높음, 현재 FastAPI 코드와 미스매치 |
| ECS + FastAPI (신규) | 코드 구조 그대로 이식, 운영 단순 | 초기 인프라 구축 필요                             |
| ECS + SageMaker    | 완전 관리형              | 비용 프리미엄, 오버스펙                            |

세 서비스 모두 이미 FastAPI 기반으로 작성돼 있어서, Triton으로 갈아타며 config.pbtxt를 새로 익히고 코드를 재작성하는 비용이 이관의 이점을 깎아먹는다고 판단
기존 Triton 서비스가 running task 0이라는 것 자체가 이미 답

인스턴스 구성은 GPU 메모리(VRAM) 실측치를 기준으로 판단 (`nvidia-smi`)

- 객체 탐지 ~4.4GB, 변화 탐지 ~2.9GB, 맹그로브 탐지 ~0.3GB에 CUDA 오버헤드 ~1GB를 더해도 합계 **8.6GB 수준**
- T4 16GB 단일 인스턴스(g4dn.xlarge)에 3개 서비스를 같이 올려도 여유가 있다고 판단
- 다만 동시 요청이 몰리면 OOM 리스크가 있어서, 이 지점을 사람이 매번 확인하는 대신 Grafana VRAM 알람으로 상시 모니터링하기로 결정 — 이게 이번 이관에서 "그냥 옮기기"와 가장 다른 지점

가중치 파일은 컨테이너 이미지에 포함하지 않고 S3로 분리
이미지에 포함하면 객체 탐지 기준 30GB가 넘어가서 ECR push/pull 비용과 시간이 과도해지고, 가중치를 업데이트할 때마다 이미지를 재빌드해야 하는 것도 비효율적
S3로 분리하면 가중치 버전 관리도 이미지와 독립적으로 가능

## 구현

### 전체 흐름

7단계로 나눠 진행

1. **사전 준비** — 온프렘 서버(GPU 추론 서버, 이미지 처리 서버)에 흩어져 있던 가중치 파일(~3.4GB)을 AI팀 리소스 버킷으로 업로드, AI팀 의존 항목(베이스 이미지, 레포명, `/health` 엔드포인트, 환경변수 키 목록)을 블로커로 별도 추적
2. **ECR 레포 생성** — 서비스별 레포 4개(베이스 이미지 포함)를 Terraform으로 작성
3. **이미지 빌드 & ECR Push**
   - 계획 — Dockerfile에 `entrypoint.sh` 추가, 컨테이너 기동 시 S3에서 가중치 자동 다운로드
   - 변경 — GPU 노드에 인스턴스 스토어(NVMe)를 마운트해 가중치를 미리 내려받고, 컨테이너엔 볼륨으로 읽기 전용 마운트
   - 이유 — 기동마다 수 GB씩 받는 것보다 노드에 상주시켜 여러 서비스가 공유하는 쪽이 기동 시간·네트워크 비용 모두에서 유리
4. **Terraform ECS 인프라 작성**
   - Task Definition/Service/IAM Role/CloudWatch 로그 그룹을 서비스 3개분 작성
   - `sharedMemorySize` — 기존 서비스가 Integer 최대값 오류로 방치돼 있던 걸 보고 8192(8GB)로 명확히 고정
   - GPU 리소스 요구사항(`resourceRequirements: GPU 1`) — 애초 계획에 포함했지만, 서비스 3개를 GPU 1개에 동시 배치하지 못하게 막는다는 걸 배포 중 확인하고 제거 (아래 트러블슈팅 참고)
5. **기존 미사용 서비스 폐기** — desired count 0으로 내리고 태스크 정의 비활성화
6. **배포 및 검증** — ECS 태스크 기동 확인, 헬스체크, 추론 테스트에 이어 **Grafana에서 GPU 사용률·VRAM 사용량·OOM 알람을 확인**하는 단계를 명시적으로 포함
7. **CI/CD 구성** — 레포별 Bitbucket Pipelines(PR 시 빌드 검증 → main 시 ECR push + ECS 롤링 배포), 이 단계는 실제로는 아직 착수하지 못함

### Task Definition 최종 설정

```json
{
  "linuxParameters": { "sharedMemorySize": 8192 },
  "environment": [
    { "name": "NVIDIA_VISIBLE_DEVICES", "value": "all" },
    { "name": "NVIDIA_DRIVER_CAPABILITIES", "value": "compute,utility" }
  ],
  "logConfiguration": {
    "logDriver": "awslogs",
    "options": { "awslogs-group": "/ecs/satchat/object-detection", "awslogs-region": "ap-northeast-2" }
  }
}
```

`resourceRequirements`로 GPU를 예약하는 대신, Docker의 `default-runtime`을 nvidia로 바꾸고 `NVIDIA_VISIBLE_DEVICES=all` 환경변수로 GPU에 접근하는 방식으로 최종 정리

### 실행하며 부딪힌 문제들

- **ECS GPU 리소스 예약 충돌**
  - 증상 — `resourceRequirements: GPU 1` + 서비스 3개 동일 인스턴스 배치 → `insufficient GPU resource available` (ECS가 GPU를 논리적 카운팅, VRAM 여유와 무관)
  - 해결 — `resourceRequirements` 제거 → `default-runtime: nvidia` + 환경변수 방식으로 전환
- **Python 패키지 editable install ↔ pip install 충돌**
  - 증상 — object-detection에서 `mmcv`/`mmrotate` ModuleNotFoundError
  - 원인 — 정반대(`mmcv`는 PYTHONPATH 누락, `mmrotate`는 PYTHONPATH가 정상 pip 설치본 `site-packages`를 가림)
  - 교훈 — PYTHONPATH 진단은 설치 방식 확인부터
- **IAM Task Role 자격증명 미인식**
  - 증상 — mangrove 서비스 `KeyError: 'AWS_ACCESS_KEY_ID'`
  - 원인 — 코드가 `os.environ["AWS_ACCESS_KEY_ID"]` 직접 참조, ECS는 자격증명을 IAM Task Role로 주입(환경변수 아님)
  - 해결 — `os.getenv()`로 변경, boto3가 자격증명 없으면 IAM Role 자동 사용
- **ALB 헬스체크 타임아웃** — 신규 타겟그룹이 `Target.Timeout` unhealthy, 원인: 보안그룹 인바운드에 서비스 포트(8765/8767/8032) 미개방, 해결: ALB 전용 보안그룹과 GPU 노드 보안그룹 인바운드 규칙 정리

### 모니터링을 이관 계획에 내장시킨 지점

이 CloudWatch 로그 그룹들은 이후 [[04-ECS 로그 Grafana Loki 수집|ECS 로그를 Loki로 실시간 수집하는 작업]]에서 GPU 서비스군으로 그대로 편입
이관 단계에서 로그 그룹 이름과 구조를 미리 잡아둔 덕분에, 로그 모니터링 구축 시 별도 작업 없이 바로 Subscription Filter 적용 가능

GPU 사용률/VRAM 대시보드와 OOM Alert는 [[05-핵심 서비스 CloudWatch-Grafana Alert 설계|핵심 서비스 CloudWatch 대시보드]] 쪽 작업과 이어지는데, 해당 문서에는 "GPU 클러스터 Container Insights 활성화 여부 결정 필요"가 미구현 항목으로 남음
즉 이관 계획에서 의도한 Grafana 검증 체계가 아직 완전히 붙지 않은 상태로, 이 이관 작업의 다음 단계로 이어짐

## 결과

- **ECS 전환** — object-detection/change-detection/mangrove 3개 서비스 정상 운영 전환, VRAM 사용량 ~6.6GB/15.4GB
- **ALB 라우팅** — 도메인 3개(`satchat-ml-*.telepix.ai`) 라우팅 구성, 헬스체크 전부 healthy, 단 이 시점 ALB/타겟그룹/리스너룰은 콘솔 수동 생성 → Terraform state 밖, 편입은 후속 작업 ([[03-ALB 타겟그룹 Terraform 편입]])
- **보안그룹 분리** — ALB 전용 보안그룹과 GPU 노드 보안그룹 분리, 네트워크 경계 명확화
- **온프렘 병행 유지** — LLM 서버 URL 전환 확인 완료 전까지 온프렘 서비스 유지 (트래픽 유실 방지)
- **CI/CD 미착수** — Phase 7(파이프라인 구성) 계획만 존재, 현재 수동 배포로 운영

## 회고

단일 T4 인스턴스에 3개 서비스를 몰아넣는 구조는 비용/구성 단순성 면에서는 맞는 선택이지만, 인스턴스 장애 시 3개 서비스가 동시에 죽는다는 트레이드오프를 그대로 안고 가는 결정
이 리스크를 줄이는 유일한 수단이 결국 모니터링이라, 이관 계획 초기부터 Grafana 검증을 Phase 6에 명시적으로 박아둔 게 맞는 순서였다고 판단

계획과 실제가 갈린 지점이 두 군데 있었는데, 둘 다 "설계 단계에서는 안 보이던 것들"이라 인상 깊음

- **가중치 로딩 방식** — 컨테이너 기동 시 S3에서 받아오는 `entrypoint.sh` 방식은 설계상 깔끔했지만, 실제로 여러 서비스를 같은 노드에 올리고 나니 인스턴스 스토어에 미리 받아두고 공유하는 쪽이 기동 시간·트래픽 면에서 더 유리
- **GPU 리소스 예약** — `resourceRequirements: GPU 1`도 VRAM 여유 계산만 보면 문제없어 보였지만, ECS 스케줄러가 GPU를 물리적 여유가 아니라 논리적 개수로 카운팅한다는 건 실제로 배포해보기 전까지 드러나지 않는 제약

인프라 설계에서 "자원 여유"와 "스케줄러가 그 자원을 어떻게 셈하는지"는 별개의 문제라는 걸 이번에 체감

남은 과제도 정리해두면 다음과 같음

- **AI팀 의존 항목** — 베이스 이미지, 레포명, 헬스체크 경로가 블로커로 남았던 게 가장 아쉬운 지점, 인프라 쪽 준비가 끝나도 다른 팀 산출물이 없으면 다음 단계로 못 넘어가는 구조라, 다음엔 이런 교차 팀 의존성을 Phase 1보다 앞단에서 확인할 필요
- **CI/CD 미착수** — 계획에는 넣어놓고 실제로는 끝까지 못 간 숙제
- **T4 성능 미검증** — 기존 RTX 3090 Ti/4080 대비 추론 성능이 낮다는 점을 실측 검증 없이 배포부터 진행, Grafana 지표로 계속 확인 필요

## 관련 문서
- [[jtkdy/TelePIX/01. infra/00-index|01. infra 인덱스]]
- [[GPU 딥러닝 서빙 마이그레이션|프로젝트 전체 타임라인]]
- [[04-ECS 로그 Grafana Loki 수집|ECS 로그 모니터링 (GPU 로그 그룹 편입)]]
- [[05-핵심 서비스 CloudWatch-Grafana Alert 설계|핵심 서비스 대시보드/Alert (GPU Container Insights 미구현 항목)]]
- [[03-ALB 타겟그룹 Terraform 편입|ALB/타겟그룹 Terraform 편입 (후속 작업)]]
- [[satchat-gpu CloudFormation 이중소유 정리 사고|CloudFormation 이중소유 정리 사고 (후속 작업 중 발생)]]
