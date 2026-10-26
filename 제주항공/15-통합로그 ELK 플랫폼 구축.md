---
title: 통합로그 ELK 플랫폼 구축
date: 2026-10-22
updated: 2026-10-22
tags: [제주항공, elk, elasticsearch, logstash, filebeat, kibana, logging]
status: draft
---
## 배경

- [[01-차세대 PSS 프로젝트 EKS 전환|PSS EKS 전환]]으로 항공 예약 시스템 로그가 EKS와 온프레미스에 흩어짐
- 기존 로그 확인 방식은 서버마다 SSH 접속 후 tail/grep
- 장애 시 여러 서버 로그를 사람이 직접 시간대 맞춰 대조 필요

## 접근

항공 예약 시스템 JVM 애플리케이션 로그 분석 전용으로 설계
저장소는 Elasticsearch(로그를 인덱싱해 검색·집계하는 분산 검색 엔진)
일 300-400GB 규모 로그량 감당 위해 Hot-Warm-Cold 구조 채택
Hot-Warm-Cold: 최근 로그는 고성능 노드(Hot), 시간이 지난 로그는 저비용 노드(Warm, Cold)로 옮겨 보관하는 Elasticsearch 데이터 계층 방식
Hot 노드 6대, Warm·Cold 노드 2대씩 배치
수집·저장 계층은 물리 서버, 조회 계층은 가상화 VM으로 분리
Kibana: Elasticsearch 데이터를 검색·시각화하는 웹 UI
담당자 1명과 공동, 2021.12 무렵 구축

## 구현

### 구성

| 역할 | 대수 | 환경 |
|---|---|---|
| Logstash + Elasticsearch master | 2 | 물리 서버 |
| Elasticsearch Hot | 6 | 물리 서버 |
| Elasticsearch Warm | 2 | 물리 서버 |
| Elasticsearch Cold | 2 | 물리 서버 |
| Kibana | 2 | 가상화 VM |

- 물리 서버 12대, VM 2대
- master 노드: 클러스터 상태 관리·샤드 배치를 맡는 Elasticsearch 노드, 별도 서버 없이 Logstash 서버 2대에 프로세스 함께 배치

### 수집 경로

1. 애플리케이션 노드의 Filebeat(경량 로그 수집 에이전트)가 로그 파일 수집
2. Logstash(로그 파싱·변환 서버)에서 파싱 후 Elasticsearch Hot 노드에 적재
3. 기간 경과에 따라 Warm, Cold 노드로 이동
4. Kibana 2대에서 조회

### 계층 이동 기준

| 구간 | 기준 |
|---|---|
| Hot → Warm | 3일 경과 |
| Warm → Cold | 7일 경과 |
| Cold → 삭제 | 30일 경과 |

당시 설정 원문 미보존, 기억 기준 근사값

### 백업

- Elasticsearch 스냅샷(인덱스 단위 백업본) 생성 후 NetWorker(온프레미스 백업 솔루션)로 백업
- 백업본 보관 기간은 NetWorker 정책 범위(97일-1년) 내 적용, 계층별 정확한 값 미확인
- 클러스터 내 보관(최대 30일)보다 긴 기간은 백업본으로 커버
- 통합로그 스냅샷 백업 스토리지 약 85TB, 중복제거율 0.55 (출처: [[08-NetWorker 기반 백업 체계 구축]])

## 결과

| 항목 | Before | After |
|---|---|---|
| 로그 확인 | 서버별 SSH 접속 후 tail/grep | Kibana 단일 화면 조회 |
| 로그 위치 | EKS, 온프레미스 서버별 분산 | 통합로그 클러스터 1곳 |

- 항공 예약 시스템 로그 일 300-400GB 단일 플랫폼으로 수집
- 외부 API 장애 발생 시 영향 범위 조회에 활용, 원인 분석·조치는 개발 담당

## 회고

- 잘된 것: 서버별 SSH 조회에서 단일 조회로 전환, 장애 시 영향 범위를 Kibana에서 바로 조회 가능
- 잘된 것: 클러스터 내 보관(30일)과 백업 보관(최대 1년)을 분리해 클러스터 디스크 부담 없이 장기 보관 가능
- 아쉬운 것: 구축 당시 master 후보는 Logstash 서버 2대뿐, 1대 장애 시 과반(quorum) 미충족으로 master 선출 불가 가능성
- 아쉬운 것: 구축 당시 장애를 Kibana에서 사후 확인하는 방식에 머묾, 에러 급증 알림까지는 미구성
- 다음에 할 것: master 후보 3대 이상 홀수 구성, 외부 API timeout·에러 급증 alert, 개발팀과 거래 ID 로그 형식 표준화로 요청 단위 추적

## 관련 문서
- [[00-index|제주항공 인덱스]]
- [[01-차세대 PSS 프로젝트 EKS 전환]]
- [[08-NetWorker 기반 백업 체계 구축]]
