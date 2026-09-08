---
title: CI에서 Terraform Plan 결과 로깅하는 로직
date: 2026-08-27
tags: [terraform, ci-cd, bitbucket-pipelines]
status: done
---
## 배경

인프라 레포는 도메인별로 디렉토리가 나뉘어 있고 PR 하나에 여러 도메인이 걸리는 경우도 있음
PR을 올릴 때마다 어떤 리소스가 어떻게 바뀌는지 리뷰어가 파이프라인 로그만 보고 판단할 수 있어야 하는 상황
처음에는 terraform plan 출력을 그대로 찍었는데, init 로그와 provider 스키마 관련 잡음이 너무 많아서 실제로 바뀌는 리소스를 찾기 어려움
그래서 PR 파이프라인에서 변경된 디렉토리만 골라 plan을 돌리고, 결과 중 의미 있는 줄만 필터링해서 보여주는 로직으로 정리

## 접근

가장 먼저 시도한 건 PR에서 변경된 .tf 파일을 git diff로 찾아서 해당 디렉토리만 plan 대상으로 삼는 것
근데 모듈을 참조하는 디렉토리는 정작 자기 자신의 .tf가 안 바뀌었는데도 plan 결과가 바뀔 수 있어서, 모듈이 변경되면 그 모듈을 source로 참조하는 상위 디렉토리까지 같이 찾아주는 로직을 추가

## 구현

CI 구성은 Bitbucket Pipelines(`bitbucket-pipelines.yml`)이고, PR 파이프라인은 OIDC로 AWS 역할을 assume한 뒤 `scripts/tf-plan-pr.sh`를 실행한다.

- **변경 디렉토리 탐지** — `git diff --name-only ${BITBUCKET_PR_DESTINATION_COMMIT}...HEAD`로 변경된 `.tf` 탐지, `bootstrap/` 제외
- **Terraform 루트 판별** (`find_root`) — 변경 파일 상위 탐색, `provider.tf` 있는 첫 디렉토리를 루트로 판단
- **모듈 의존 디렉토리 탐지** (`find_dependents`) — 루트 못 찾으면(모듈 자체 변경) 해당 모듈을 `source`로 참조하는 파일까지 plan 대상 추가
- **환경변수 주입** — 대상 디렉토리별 `.env`를 S3(레포 구조 미러링)에서 로드, 없으면 스킵
- **plan 실행 및 필터링** — `terraform -chdir="$dir" plan -input=false -no-color`
  - 실패 시 로그 전체 출력
  - 정상 시 `grep -E '^\s*(#|[~+\-]|Plan:|No changes)'`로 변경 표시(`#`/`+`/`-`/`~`)와 `Plan:`/`No changes` 요약만 출력
- **실패 처리** — 대상 디렉토리 중 하나라도 plan 실패 시 전체 파이프라인 실패(`exit 1`)

커스텀 파이프라인용 단일 도메인 plan(`scripts/tf-plan.sh`)도 같은 grep 필터를 쓰지만, 여긴 도메인 하나만 대상이라 디렉토리 탐지 로직 없이 바로 `terraform plan`을 돌림

## 결과

- **로그 정제** — init 잡음 제거, 리소스별 변경 라인 + `Plan: N to add, M to change, K to destroy` 요약만 노출, 로그 스크롤만으로 변경 범위 파악 가능
- **도메인 구분** — PR 하나에 여러 도메인이 걸려도 `[Terraform Plan] $dir` 헤더로 섹션 구분
- **에러 시 원본 유지** — plan 실패 시 필터링 없이 에러 로그 전체 노출, 디버깅 정보 손실 없음
- **채널 단일화** — PR 코멘트 자동 게시, 웹훅 알림 기능 제거, 파이프라인 콘솔 로그로 정보 전달 경로 통합

## 회고

grep 기반 필터링은 구현이 단순하고 지금까지는 충분히 잘 작동 중
다만 정규식으로 plan 출력 포맷을 파싱하는 방식이라 Terraform 버전이 올라가면서 출력 포맷이 바뀌면 필터가 깨질 수 있다는 점은 리스크로 남음
알림/코멘트 자동화를 시도했다가 인프라 제약(SG 정책)으로 포기한 경험은, 새 통합을 추가하기 전에 네트워크 접근성부터 먼저 확인 필요

## 관련 문서

- [[Terraform 레거시 src 마이그레이션]]
- [[env S3 관리 도입]]
