---
title: LG생활건강 E980·E870 VIOS 신규 client 서버 구축
date: 2019-08-01
tags: [ibm-kts, aix, vios, npiv]
status: draft
---
## 배경

시스템 고도화를 위한 데이터 마이그레이션 필요

## 접근

가상화 방식은 vSCSI 대신 NPIV(N_Port ID Virtualization)로 결정
NPIV는 client LPAR마다 고유 WWPN을 할당해 SAN 스토리지에서 직접 인식되는 구조
LUN 추가·변경 시 VIOS 재구성 없이 스토리지 단에서 바로 매핑 가능하고 client 측에서 네이티브 멀티패스 드라이버를 그대로 사용 가능
데이터 마이그레이션 과정에서 문제가 생기면 되돌릴 수 있도록, 신규 파티션 구성 전에 기존 VIOS mapping 정보부터 백업

## 구현

- 기존 VIOS 서버 mapping 정보 백업
- SAN 스위치 zoning 작업 — client LPAR WWPN 기준으로 신규 zone 구성
- NPIV 기반 신규 파티션 구성

활용 기술 — AIX, VIOS

## 회고

특별한 이슈 없이 완료


## 관련 문서
- [[00-index|IBM KTS 인덱스]]
