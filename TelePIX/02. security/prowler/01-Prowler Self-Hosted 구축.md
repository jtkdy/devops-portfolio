---
title: Prowler Self-Hosted 구축
date: 2026-08-24
tags: [security, prowler, cspm, aws, docker]
status: done
---
## 배경

보안 스택은 Wazuh와 Suricata로 온프렘 이벤트와 네트워크 트래픽은 커버하고 있었지만, AWS 인프라 설정 자체가 잘못되어 있는 경우를 탐지하는 수단은 부재
IAM 권한이 과도하게 열려있거나, S3 버킷이 퍼블릭으로 노출되어 있거나, 암호화가 빠진 리소스가 있어도 아무도 모르는 상태

8월에 겪은 [[IAM 액세스키 탈취 인시던트 대응|IAM 액세스키 탈취 인시던트]]에서 이 공백이 더 명확해짐
사고 대응 과정에서 AWS 설정 감사 자체가 없어 공격자가 어디까지 접근했는지 파악하는 데 시간 소요
CSPM(Cloud Security Posture Management, 클라우드 인프라 설정을 자동으로 점검해주는 도구) 도입이 필요하다는 결론에 도달, 오픈소스인 Prowler를 Self-Hosted로 구축하기로 결정

## 접근

처음에는 기존 온프렘 서버에 Docker로 올리는 방안을 검토
이미 Wazuh가 돌고 있는 서버라 추가 인프라 없이 빠르게 붙일 수 있다는 장점
하지만 Prowler Local Server는 Django, Neo4j, PostgreSQL, Valkey, Celery 등 컨테이너 8개가 떠야 하는 구조라 Wazuh와 리소스를 나눠 쓰기엔 부담

결국 기존 VPC에 전용 EC2를 분리해서 올리는 방향으로 결정
IAM Role도 기존 것과 섞지 않고 Prowler 전용으로 신규 생성
용도가 다른 서비스가 같은 Role을 공유하면 CloudTrail에서 어떤 서비스가 어떤 API를 호출했는지 추적이 어려워지기 때문

UI 접근 방식도 고민 필요
Prowler EC2가 배치된 VPC는 온프렘에서 직접 라우팅이 안 되는 구조라 처음에는 퍼블릭 서브넷 배치를 고려했지만, Prowler 스캔 결과에 AWS 취약점 전체 목록이 담겨있어 외부 노출은 절대 불가로 판단해 SSM 포트포워딩 방식으로 확정

## 구현

### AWS 리소스 구성

EC2는 t3.large(vCPU 2, 8GB)로 생성
처음엔 t3.medium(4GB)으로 시작했는데 api 컨테이너(Django)와 neo4j가 각각 1.3GB 이상을 차지하면서 메모리 여유가 없어지고 SSM 세션이 불안정해지는 현상 반복
t3.large로 업그레이드한 뒤 전체 메모리 사용률이 약 50% 수준으로 안정

| 항목 | 값 |
|---|---|
| 용도 | Prowler 전용 EC2 (SSM Only 접속) |
| Instance Type | t3.large (vCPU 2, 8GB) |
| Private IP | 10.9.x.x |
| VPC | 공유 인프라 VPC |

IAM Role은 Prowler EC2 전용으로 생성하고 `SecurityAudit`, `ViewOnlyAccess`, `AmazonSSMManagedInstanceCore` 세 가지 정책 부착
첫 스캔 이후 `logs:FilterLogEvents`, `lambda:GetFunction`, `apigateway:GET` 등 SecurityAudit으로 커버되지 않는 권한이 확인되어, 인라인 정책으로 추가 권한 부착

Security Group은 인바운드를 완전히 차단
SSM 포트포워딩 방식은 EC2 내부에서 루프백으로 연결되기 때문에 인바운드 규칙 없이도 접근 가능

### Docker Compose 구성

컨테이너 구성은 아래와 같음

| 컨테이너 | 이미지 | 포트 |
|---|---|---|
| prowler-api | prowlercloud/prowler-api:stable | 8080 |
| prowler-ui | prowlercloud/prowler-ui:stable | 3000 |
| prowler-worker / worker-beat | prowlercloud/prowler-api:stable | - |
| prowler-mcp-server | prowlercloud/prowler-mcp:stable | 8000 |
| prowler-postgres | postgres:16-alpine | 5432 |
| prowler-valkey | valkey/valkey:8-alpine | 6379 |
| prowler-neo4j | graphstack/dozerdb:5.26.27.0 | 7687 |

`.env`에서 중요한 설정은 `AUTH_URL`
SSM 포트포워딩으로 로컬 커스텀 포트(`xxxx`)를 쓰기 때문에 기본값인 `localhost:3000`에서 `localhost:xxxx`로 변경 필요
이 값은 `restart`로는 반영되지 않고 `--force-recreate`로 재생성해야 적용

```bash
AUTH_URL=http://localhost:xxxx
API_BASE_URL=http://prowler-api:8080/api/v1
```

EC2 재부팅 시 자동 기동을 위해 systemd 서비스로 등록
docker-compose 실행 파일 경로가 `/usr/local/bin/docker-compose`라는 점 주의 필요
`/usr/bin`으로 잘못 지정하면 서비스가 즉시 종료

```ini
[Unit]
Description=Prowler Local Server
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/root/prowler
ExecStart=/usr/local/bin/docker-compose up
ExecStop=/usr/local/bin/docker-compose down
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### AWS Provider 연동

Prowler UI에서 AWS Provider를 추가할 때 `AWS SDK Default` 방식을 선택
EC2 Instance Profile로 크레덴셜을 자동 감지하되, Prowler UI가 해당 Role을 Assume할 수 있도록 신뢰 정책에 Principal과 ExternalId 조건 추가

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::<AWS 계정>:role/<Prowler EC2용 IAM Role>" },
  "Action": "sts:AssumeRole",
  "Condition": { "StringEquals": { "sts:ExternalId": "<마스킹>" } }
}
```

### UI 접근 방법

로컬 PC에서 SSM 포트포워딩을 실행한 뒤 브라우저에서 로컬 커스텀 포트로 접속하는 방식
터미널을 닫으면 터널이 끊기므로 접속하는 동안 유지 필요
매번 명령어를 입력하지 않도록 연결 스크립트로 제작

## 결과

최초 스캔(2026-08-24)에서 641개 체크 항목으로 1,292개 리소스를 점검했고 총 1,838건의 FAIL 확인

| Severity | 건수 |
|---|---|
| Critical | 33 |
| High | 527 |
| Medium | 652 |
| Low | 626 |

Critical 33건 중 즉시 조치 가능한 항목은 당일 처리

- NAT 인스턴스 Security Group에 열려있던 DB 포트 인바운드 제거
- 한 번도 사용된 적 없으면서 AdministratorAccess를 보유하고 있던 미사용 Role 삭제
- 퍼블릭 쓰기가 허용되어 있던 미사용 S3 버킷 제거

NAT 노드 전체 포트 오픈으로 인한 `ec2_instance_port_*` 14건과 Virtual MFA로 인한 `iam_root_hardware_mfa_enabled`는 오탐으로 판단해 Mute 처리

## 회고

t3.medium으로 시작했다가 메모리 부족으로 hang이 반복됐던 게 아쉬운 점
Neo4j와 Django가 각각 1.3GB 이상을 쓴다는 걸 미리 파악했더라면 처음부터 t3.large로 시작했을 것
Prowler Local Server를 올리려면 Django + Neo4j 합산으로 최소 4GB는 필요하니, 여유 메모리를 감안해 8GB 이상 스펙으로 시작하는 게 적절

SSM 포트포워딩 방식은 보안적으로는 최선이지만 터미널을 계속 열어둬야 한다는 불편함 존재
사내 개발자 포탈에 Prowler API를 연동해서 대시보드로 바로 볼 수 있게 되면 이 불편함이 해소될 것 같아, 후속 작업으로 이어서 진행([[02-Prowler 보안 대시보드 연동]])

스캔 결과 1,838건의 FAIL이 나왔지만 상당수는 오탐이거나 의도된 설정
High 527건을 하나씩 검토해서 실제 조치 대상과 Mute 대상을 분류하는 작업이 남음

## 관련 문서
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
- [[02-Prowler 보안 대시보드 연동]]
- [[IAM 액세스키 탈취 인시던트 대응]]
- [[IAM 액세스키 탈취 사고 대응과 CSPM 도입|프로젝트 전체 타임라인]]
