---
title: .env S3 관리 도입
date: 2026-08-26
tags: [terraform, s3, cicd, pipeline]
status: done
---
## 배경

PR 파이프라인에서 `terraform plan`을 돌릴 때 루트 모듈별 `.env`가 필요한데 `.gitignore`로 git에서 제외돼 있어서 파이프라인이 변수값을 읽지 못하고 plan이 계속 실패
로컬에선 잘 되는데 파이프라인에서만 죽는 상황이라 원인 파악에 시간 소요

## 접근

`.env`를 git에 올리는 건 처음부터 배제
민감도가 낮은 값이라도 git 이력에 한번 들어가면 완전히 지우기 번거롭고 습관이 무너지면 나중에 진짜 시크릿이 들어갈 수 있어서 S3에 올리고 파이프라인 실행 전에 자동으로 내려받는 방식으로 결정

접근 권한은 파이프라인 전용 Role에만 `s3:GetObject`만 허용
쓰기 권한까지 주면 파이프라인이 `.env`를 덮어쓸 수 있는 구조가 되니까 최소한으로 제한

## 구현

S3 버킷 구조는 레포 디렉토리 구조를 그대로 미러링
루트 모듈 경로만 보면 어떤 `.env`인지 바로 알 수 있어서 관리 편의성 확보

```
s3-버킷/
├── infra/aws/.env
├── internal/groundctrl/.env
└── project/
    ├── mps/product/.env
    ├── coreservice/product/.env
    ├── coreservice/shared/.env
    └── sso/product/.env
```

보안 설정은 퍼블릭 액세스 차단, SSE-S3 암호화, 버전 관리 활성화로 구성
`.env` 안에는 AWS 계정 ID, ECR 이미지 URL 같은 민감도 낮은 값만 들어가고 실제 시크릿은 Secrets Manager에서 따로 관리하는 구조라 S3에 올려도 무방하다고 판단

## 결과

파이프라인에서 plan이 정상적으로 실행
`.env`는 git에 올라가지 않고 S3에서만 관리되니까 이력 관리도 정리됨

## 회고

레포 구조를 S3 경로에 그대로 미러링한 게 나중에 루트 모듈 추가될 때도 직관적으로 관리할 수 있어서 좋은 선택
다만 `.env` 값이 바뀔 때마다 S3 재업로드를 수동으로 해야 하는 부분은 나중에 자동화 고려 대상

## 관련 문서
