---
title: 02. security 인덱스
date: 2026-08-27
tags: [index, moc, security]
status: active
---
OS 하드닝부터 네트워크 IDS, 온프렘 AWS 인증, 호스트 보안 모니터링, CSPM까지 보안 스택 전반을 주제별로 정리
대략적인 구축 순서대로 나열

## 1. OS 보안조치 (KISA 하드닝)

기준: KISA 주요정보통신기반시설 기술적 취약점 분석·평가 방법 상세가이드, Unix 서버 U-01~U-67 전 항목 자동화. (2026-06-01)

| 문서 | 내용 |
|---|---|
| [[00-KISA 하드닝 개요]] | CHECK/FIX/VERIFY 모드 설계, OpenSCAP 교차 검증 |
| [[jtkdy/TelePIX/02. security/OS 보안조치/scripts/Ubuntu 20, 24]] | Ubuntu 24.04용 스크립트 |
| [[jtkdy/TelePIX/02. security/OS 보안조치/scripts/AmazonLinux 2023]] | Amazon Linux 2023용 스크립트 |
| [[jtkdy/TelePIX/02. security/OS 보안조치/scripts/Rocky 9]] | Rocky Linux 9용 스크립트 |

## 2. 네트워크 IDS (Suricata)

작년 크립토마이닝 25일 미탐지 사고를 계기로 온프렘에 네트워크 레벨 탐지를 구축. (2026-06-16)

| 문서 | 내용 |
|---|---|
| [[01-MikroTik-Suricata-Wazuh 온프렘 IDS 구축]] | MikroTik TZSP 미러링 → tzsptap → Suricata → Wazuh 연동 |
| [[02-WireGuard 터널로 EC2-온프렘 SSH 접근 구성]] | 22번 포트 노출 없이 기존 WireGuard 인프라로 EC2→온프렘 SSH 경로 구성 |

## 3. 온프렘 IAM Roles Anywhere (PKI)

액세스키 대신 X.509 인증서 기반 임시 자격증명으로 온프렘 → AWS 접근 체계 구축. (2026-06-24)

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [[01-온프렘 PKI 기반 IAM Roles Anywhere 구축]] | CA 발급/자격증명 실행 계정 분리, PAM 접근 제어, sudo 권한 정리 (설계) |
| 2 | [[02-220 서버 구성 참고]] | 실제 배포 구성 참고 문서 — 인증서 경로/만료일, serve 모드 + Docker 프록시 (수시 갱신) |

## 4. Wazuh (호스트 보안 모니터링)

에이전트 설치부터 노이즈 제거, 커스텀 룰, CloudWatch/Slack 연동까지. (2026-06-25 ~ 07-22)

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [[01-에이전트 설치 및 구성]] | 서버 3대 에이전트 설치, syscheck 범위 축소, buffer 튜닝 |
| 2 | [[02-FIM 노이즈 제거]] | 패키지 업데이트발 FIM 노이즈 제거 (snap/CUPS 포함) |
| 3 | [[03-커스텀 룰 관리]] | Docker veth 오탐 억제, CVE false positive 관리 |
| 4 | [[04-CloudWatch 연동 및 데일리 리포트]] | Wazuh 이벤트 → CloudWatch → Lambda → Slack 데일리 리포트 |

## 5. Prowler (CSPM)

AWS 인프라 설정 오류를 탐지하는 CSPM 도구를 Self-Hosted로 구축하고 사내 개발자 포탈에 연동. (2026-08-24 ~ 25)

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [[01-Prowler Self-Hosted 구축]] | EC2 분리 구축, SSM 포트포워딩 접근, 최초 스캔 1,838건 FAIL 확인 |
| 2 | [[02-Prowler 보안 대시보드 연동]] | 사내 개발자 포탈 보안 탭에 Prowler API 연동, 캐시/권한 체크 추가 |

## 관련 문서
- [[jtkdy/TelePIX/00-index|jtkdy 전체 인덱스]]
- [[jtkdy/TelePIX/03. incidents/00-index|03. incidents 인덱스]]
