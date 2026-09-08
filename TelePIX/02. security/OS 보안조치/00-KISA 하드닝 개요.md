---
title: KISA 하드닝 스크립트 개요
date: 2026-06-01
tags: [security, kisa, hardening, ubuntu, al2023, rocky]
status: done
---
## 배경

인증심사(CSAP, 클라우드 보안 인증) 대비를 위해 온프렘·EC2에서 운영 중인 Ubuntu 24.04, Amazon Linux 2023, Rocky Linux 9.5 세 가지 OS에 KISA(한국인터넷진흥원) 주요정보통신기반시설 기술적 취약점 분석·평가 가이드 기준 보안 조치가 필요
Unix 서버 점검 항목(U-01~U-67) 67개를 서버마다 수작업으로 확인하고 조치하는 건 현실적이지 않아, OS별 자동화 스크립트로 제작 결정

## 접근

체크와 조치를 분리하지 않고 하나의 스크립트에서 모드로 전환하는 방식을 채택
운영 서버에 바로 조치를 걸기 전에 먼저 현재 상태를 점검만 해보고 싶은 경우가 많아, 기본 동작과 함께 네 가지 모드를 지원하도록 설계

- **`--check-only`** — 점검만 수행
- **`--dry-run`** — 조치 내용을 출력만
- **`--fix-only`** — 무조건 조치
- **`--no-verify`** — 조치는 하되 검증은 생략

각 항목의 결과 상태도 네 단계로 구분

- **`PASS`** — 처음부터 양호
- **`FIXED + VERIFIED`** — 조치 후 검증까지 통과
- **`VERIFY_FAIL`** — 조치는 했지만 검증에 실패해 수동 확인 필요
- **`FAILED`** — 조치 자체가 실패했거나 check-only로 실행

이렇게 나눠두면 스크립트 실행 로그만 보고도 어디를 사람이 다시 봐야 하는지 바로 파악 가능

## 구현

OS별로 스크립트를 따로 관리

- Ubuntu 24.04용 스크립트 ([[jtkdy/TelePIX/02. security/OS 보안조치/scripts/Ubuntu 20, 24]])
- Amazon Linux 2023용 스크립트 ([[jtkdy/TelePIX/02. security/OS 보안조치/scripts/AmazonLinux 2023]])
- Rocky Linux 9용 스크립트 ([[jtkdy/TelePIX/02. security/OS 보안조치/scripts/Rocky 9]])

Ubuntu는 스크립트 항목 외에 공유메모리 보안 설정을 별도로 추가
`/etc/fstab`에 `tmpfs /run/shm tmpfs defaults,noexec,nosuid,nodev 0 0`을 추가하고 재부팅해야 적용

검증은 `--check-only` 옵션으로 실행, 점검 로그는 `/var/log/security-hardening/`에 CHECK 항목별 파일/디렉터리 경로까지 포함해서 기록
로그 파일명에 호스트네임을 포함해두면 서버가 여러 대일 때 구분 용이

스크립트 기반 점검과는 별개로 OpenSCAP으로 교차 검증도 진행

```bash
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_standard \
  --results /tmp/al2023_scan_results.xml \
  --report /tmp/al2023_scan_report.html \
  /usr/share/xml/scap/ssg/content/ssg-al2023-ds.xml
```

## 결과

- OS 3종(Ubuntu 24.04, AL2023, Rocky 9.5)에 U-01~U-67 전 항목 자동 점검/조치 스크립트 적용
- CHECK/FIX/VERIFY 3단계 + 4가지 실행 모드로 운영 상황에 맞게 유연하게 사용 가능
- OpenSCAP 교차 검증으로 스크립트 결과의 신뢰도 확보

## 회고

회사 사무실 네트워크 환경 특성상 일부 보안 조치는 적용 불가
예를 들어 `hosts.allow` 기반 IP 접근 제한은 서버 접근 경로가 인터넷 → AWS 보안그룹 → OS iptables/nftables → TCP Wrappers → sshd 순으로 여러 겹인데, 이걸 더 촘촘히 하려면 VPN이나 별도 서버 접근 제어 구축이 필요하고 그만큼 비용 발생
이 부분은 우선순위 조정이 필요한 상태로 남음

스크립트를 CHECK/FIX/VERIFY로 분리해둔 게 실제로 유용
조치를 걸기 전에 `--check-only`로 먼저 영향 범위를 파악하고, 문제가 될 만한 항목은 `--dry-run`으로 실제 변경 내용을 먼저 확인한 뒤에 적용하는 흐름이 자연스럽게 자리 잡음

## 관련 문서
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
