---
title: CDN 서비스 이관 (Akamai to CloudFront)
date: 2025-06-01
tags: [제주항공, cdn, cloudfront, akamai, lambda-edge]
status: draft
updated: 2026-11-16
---
## 배경

기존 Akamai CDN 라이선스 비용 부담
AWS 아키텍처와의 통합 필요성도 함께 대두되며 CloudFront로의 전환 검토
**이관 범위**: 정적 컨텐츠 도메인(웹 컨텐츠, 이미지 등)만 대상, 채널 도메인(홈페이지, 서치, PAY, 웹체크인)은 Akamai 경유 유지
정적 컨텐츠가 CDN 전송량 대부분 차지 → 정적만 분리해도 Akamai 트래픽 감소
AWS 오리진 → CloudFront 구간 전송 비용 없음 → Akamai가 AWS 오리진에서 가져갈 때 붙던 인터넷 송신 비용 제거

## 접근

전면 전환 시 트래픽 급변에 따른 리스크 우려 → 가중치 기반 DNS 라우팅(Route53)으로 단계적 트래픽 전환 방식 채택
Akamai에서 사용하던 Edge 규칙(리다이렉션, 헤더 조작, 이미지 최적화 등)을 Lambda@Edge로 동일하게 재구현해 기능 연속성 확보

## 구현

- **CDN 마이그레이션 (Akamai → CloudFront)** — 정적 컨텐츠 도메인 대상, 기존 글로벌 트래픽의 안정성을 유지하며 무중단 전환 진행
- **DNS 이중 등록**: 정적 컨텐츠 도메인에 Akamai 엣지 주소와 CloudFront 주소를 온프렘 내부 DNS(Windows)와 Route53에 모두 등록, Route53 가중치로 CloudFront 쪽에 단계적으로 전부 이동
- **이관 전후 트래픽, 비용 측정**: 이관 전 Akamai static 트래픽(엣지 hit) 측정, 이관 후 CloudFront 도메인별 일일 비용 집계
- **전환 전략 수립** — 서비스 중단 없는 이관을 위해 가중치 기반 DNS 라우팅(Route53)을 활용한 단계적 트래픽 전환 수행
- **Lambda@Edge 활용** — Edge 규칙(리다이렉션, 헤더 조작, 이미지 최적화 등)을 Lambda@Edge로 구현
- **데이터 기반 응답 성능 및 캐시 효율화** — 로그 분석을 통해 국가별, 객체별로 최적화, CloudWatch 표준 로그를 S3에 저장하고 Athena로 쿼리해 국가별 Latency 및 Cache Hit Ratio 분석
- **운영 가시성 확보 및 비용 최적화** — CloudWatch와 CloudFront 메트릭을 연동한 통합 대시보드 구축해 CDN 장애 및 비정상 트래픽 급증 실시간 확인, CloudFront 이관을 통해 월 운영 비용의 약 15% 절감

## 결과

월 운영 비용 약 15% 절감
Cache Hit Ratio는 기존 Akamai 수준을 유지하며 전환 (신규 개선이라기보다 기존 성능 그대로 이관)

## 회고

무중단 전환 목표 달성

## 관련 문서
- [[00-index|제주항공 인덱스]]
