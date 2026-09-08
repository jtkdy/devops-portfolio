---
title: KB그룹 IDC 이전 작업
date: 2019-06-01
tags: [ibm-kts, idc-migration, powersystem]
status: draft
---
## 배경

KB그룹 신규 데이터센터 설립에 따라 기존 인프라를 새 IDC로 이전

## 접근

MOD 4 신규 장비 자원 분배는 부서별 사용량 기준으로 산정

## 구현

- 이전 계획 및 전략 수립
- 신규 장비 MOD 4 구성을 위한 자원 분배
  - IBM SAS 디스크 격납장치 후면 모드 스위치를 Mode 4로 설정 → 내부 디스크 베이 4개 독립 그룹으로 분할
  - 외부 SAS 케이블(X 케이블, IBM 지정 SAS 케이블) 사용
  - 연결 절차
    1. 서버 후면 SAS 어댑터 포트 4개 확인
    2. X 케이블 커넥터를 격납장치 후면 지정 SAS 포트에 연결, 분기된 끝을 각 SAS 어댑터 포트에 연결
    3. 4개 독립 시스템(파티션)이 하나의 격납장치 내 디스크 구역을 나누어 제어
- 기존 OS 데이터 migration
- 스토리지 펌웨어 업데이트 담당 (10ea)

활용 기술 — IBM PowerSystem, IBM Storage

## 회고

초기 구축 프로젝트라 특별한 이슈 없이 완료

## 관련 문서
- [[00-index|IBM KTS 인덱스]]
