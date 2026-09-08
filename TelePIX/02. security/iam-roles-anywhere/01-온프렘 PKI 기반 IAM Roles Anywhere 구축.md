---
title: 온프렘 PKI 기반 IAM Roles Anywhere 구축
date: 2026-06-24
tags: [security, aws, iam, onprem, pki]
status: done
---
## 배경

온프렘 서버에서 S3 등 AWS 리소스에 접근해야 하는 상황이 생기기 시작했는데, 액세스키를 서버에 그대로 박아두는 방식은 쓰고 싶지 않음
액세스키는 유출되면 만료 시점까지 계속 유효하고, 로테이션도 사람이 챙겨야 함
그래서 X.509 인증서 기반으로 AWS 임시 자격증명을 발급받는 IAM Roles Anywhere로 방향 결정
앞으로 온프렘에서 AWS 리소스 접근이 늘어날 걸 감안해, PKI(공개키 기반구조) 체계와 계정 분리 설계를 이번에 제대로 잡기로 함

## 접근

가장 먼저 정한 원칙은 인증서 발급 권한과 실제 AWS 자격증명 실행 권한을 계정 단위로 분리하는 것
CA 인증서를 발급/갱신하는 계정과, 발급된 인증서로 실제 AWS 자격증명을 실행하는 계정을 나누면, 자격증명 실행 계정이 뚫려도 CA 자체를 새로 발급할 권한까지는 미노출
발급 관리 계정은 로그인이 가능한 계정으로, 자격증명 실행 계정은 로그인이 불가능한 서비스 계정으로 설계

CA 인증서 발급 계정에 대한 접근도 제한 필요
아무나 `su`로 전환해서 인증서를 발급할 수 있으면 계정 분리 의미 상실
별도 관리자 그룹을 만들고, 그 그룹 소속만 발급 계정으로 전환 가능하도록 PAM(Pluggable Authentication Modules, 리눅스 인증 모듈 체계)으로 제한하는 방향으로 결정

sudo 권한도 이 작업을 계기로 재점검
실제 업무상 sudo가 필요 없는 계정들이 그룹에 남아있는 걸 확인하고, 이번 기회에 최소권한 원칙에 맞게 정리

## 구현

### 계정 및 그룹 설계

| 계정 | 역할 | 비고 |
|---|---|---|
| CA 발급 관리 계정 | CA 인증서 발급/갱신 관리 | 로그인 가능, 관리자 그룹 소속 |
| AWS 자격증명 실행 계정 | AWS 자격증명 실행 전용 | 로그인 불가(nologin), 전용 그룹 소속 |

| 그룹 | 멤버 | 역할 |
|---|---|---|
| 관리자 그룹 | 운영자 2명 | CA 발급 계정 `su` 접근 허용 담당자 |
| AWS 자격증명 그룹 | CA 발급 계정, AWS 자격증명 실행 계정 | AWS 자격증명 파일 접근 권한 |

```bash
# AWS 자격증명 실행 계정 생성 (로그인 불가 서비스 계정)
sudo useradd \
  --uid 301 \
  --gid awsgrp \
  --no-create-home \
  --shell /usr/sbin/nologin \
  --comment "AWS credential service account" \
  awssvc

# CA 발급 계정을 자격증명 그룹에 보조그룹으로 추가
sudo usermod -aG awsgrp keymgr

# 관리자 그룹에 운영자 추가
sudo usermod -aG admgrp <운영자1>
sudo usermod -aG admgrp <운영자2>
```

### 디렉토리 구조

```
/opt/pki/                (root:root, 700)
├── ca/                   (root:root, 700)
│   ├── openssl.cnf       (root:root, 600)
│   ├── ca.key            (root:root, 400)
│   ├── ca.crt            (root:root, 444)
│   ├── db/                (root:root, 700)
│   └── newcerts/          (root:root, 700)
├── certs/                (root:awsgrp, 750)
│   └── client.crt
└── private/               (root:awsgrp, 750)
    └── client.key         (root:awsgrp, 640)
```

CA 개인키는 `root:root 400`으로 최상위 root만 접근 가능하게, 발급된 클라이언트 인증서와 개인키는 자격증명 그룹만 접근 가능하도록 나눠서 권한 설정

### sudo 권한 정리


CA 발급 계정에 대한 `su` 명령만 sudoers에 제한적으로 허용

```bash
# /etc/sudoers.d/keymgr
# openssl ca 명령만 허용 (config 경로 고정)
keymgr ALL=(root) NOPASSWD: /usr/bin/openssl ca -config /opt/pki/ca/openssl.cnf *
```

실제 업무상 필요 없던 sudo 그룹 소속 계정 4개를 이번에 제거
Rocky Linux 서버는 `/etc/sudoers`에 직접 박혀있던 개별 계정 항목도 함께 정리 대상으로 확인

```bash
# Ubuntu — sudo 그룹에서 제거
sudo gpasswd -d <계정> sudo
```

### CA 발급 계정 접근 제어 (PAM)

관리자 그룹 소속만 CA 발급 계정으로 `su` 전환이 가능하도록 PAM으로 제한

```
# /etc/security/access.conf
+:(admgrp):ALL
+:keymgr:ALL
-:keymgr:ALL
```

Ubuntu는 `/etc/pam.d/su`의 `@include common-account` 위에 `account required pam_access.so`를 추가하면 되는데, Rocky Linux는 기본 설정에 `account sufficient pam_succeed_if.so uid = 0 use_uid quiet` 라인이 있어 root 계정이 이 검사를 우회하는 이슈 확인
이 라인을 주석 처리해야 하는데, 부작용 검증이 더 필요해 이 부분은 보류 상태로 남음

### aws_signing_helper 설치 및 자격증명 연동

```bash
sudo mv /opt/aws_signing_helper /usr/local/bin/
sudo chmod 755 /usr/local/bin/aws_signing_helper
aws_signing_helper version
```

`~/.aws/config`에 `credential_process`로 인증서 기반 자격증명 발급 연결

```ini
[profile roles-anywhere]
credential_process = aws_signing_helper credential-process \
  --certificate /opt/pki/certs/client.crt \
  --private-key /opt/pki/private/client.key \
  --trust-anchor-arn arn:aws:rolesanywhere:ap-northeast-2:<AWS 계정>:trust-anchor/<id> \
  --profile-arn arn:aws:rolesanywhere:ap-northeast-2:<AWS 계정>:profile/<id> \
  --role-arn arn:aws:iam::<AWS 계정>:role/<role-name>
```

```bash
aws s3 ls s3://<bucket-name> --profile roles-anywhere
```

## 결과

- X.509 인증서 기반 AWS 임시 자격증명 발급 체계 구축 (Ubuntu 24.04, Rocky Linux 9.5)
- CA 발급 권한과 자격증명 실행 권한을 계정 단위로 분리
- 불필요한 sudo 권한 4건 정리, PAM 기반 CA 발급 계정 접근 제어 적용

## 회고

계정과 권한을 처음부터 역할 단위로 분리한 게 이후 확장에도 도움이 될 것 같음
다만 Rocky Linux의 PAM root 우회 이슈는 이번에 완전히 해결하지 못하고 보류로 남았는데, 자칫 계정 분리 설계 전체를 무력화할 수 있는 지점이라 우선순위를 높여 재검토 필요

남은 작업도 정리하면 다음과 같음

- **인증서 자동 갱신** — 30일 주기 자동 갱신 미구현
- **접근 감사 로그** — `/opt/pki` 접근 auditd 구성 미착수
- **Trust Anchor/Profile/Role 구성** — 이어서 정리 필요
- **sudoers 설정** — 자격증명 실행 계정 몫이 아직 남음

이 체계 위에서 실제로 처음 돌아간 워크로드는 [[04-CloudWatch 연동 및 데일리 리포트|Wazuh 이벤트를 CloudWatch로 밀어올리는 파이프라인]]

## 관련 문서
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
- [[04-CloudWatch 연동 및 데일리 리포트]]
- [[온프렘 보안 모니터링 스택 구축|프로젝트 전체 타임라인]]
