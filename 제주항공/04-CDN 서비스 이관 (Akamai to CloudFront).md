---
title: CDN 서비스 이관 (Akamai to CloudFront)
date: 2025-06-01
tags: [제주항공, cdn, cloudfront, akamai, lambda-edge, route53, athena]
status: draft
updated: 2026-11-16
---
## 배경

Akamai CDN 라이선스 비용 부담, AWS 아키텍처와 통합 필요성까지 겹쳐 CloudFront 전환 검토
- CDN(Content Delivery Network): 사용자 가까운 엣지 서버에 컨텐츠를 캐시해 응답하는 분산 전송망
CDN 비용은 전송량에서 주로 발생, 전송량 상당 부분은 이미지, JS, CSS 같은 정적 컨텐츠
Akamai가 AWS 오리진에서 컨텐츠를 가져갈 때마다 AWS 인터넷 송신 비용 별도 발생

## 접근

**이관 범위**: 정적 컨텐츠 도메인만 대상, 채널 도메인(홈페이지, 서치, PAY, 웹체크인)과 동적 요청은 Akamai 경유 유지
- 정적 도메인만 빼도 Akamai 전송량 크게 감소
- AWS 오리진에서 CloudFront로 가는 구간은 전송 비용 없음, 오리진 송신 비용 제거
- 동적 요청은 예약, 결제가 걸려 있어 영향 범위 큼 → 이관 대상에서 제외

**전환 방식**: DNS 일괄 변경 시 트래픽 급변, 장애 시 전체 영향 → Route53 가중치 기반 라우팅으로 단계 전환 채택
- 가중치 기반 라우팅: 같은 이름에 레코드 여러 개를 두고 비율대로 응답을 나눠 주는 Route53 라우팅 정책
**기능 연속성**: Akamai Edge 규칙(리다이렉션, 헤더 조작, 이미지 최적화)을 Lambda@Edge로 재구현
- Lambda@Edge: CloudFront 엣지에서 요청, 응답을 가로채 실행하는 Lambda

## 구현

### 사전 측정
- 이관 전 Akamai 정적 도메인 트래픽 측정: 평균 약 200 hits/s, 최대 273 hits/s (2025-02-10 오후 약 3시간 반 구간)
- 이관 후 CloudFront 도메인별 일일 비용 집계로 전후 비교 기준 확보

### 전환 절차
1. 정적 컨텐츠 도메인에 Akamai 엣지 주소와 CloudFront 주소를 Route53과 온프렘 내부 DNS(Windows)에 모두 등록
2. 전환 전 레코드 TTL 축소, 가중치 변경이 클라이언트에 빨리 반영되도록 준비
   - TTL(Time To Live): 클라이언트, 리졸버가 DNS 응답을 캐시하는 시간
3. Route53 가중치 Akamai 9 : CloudFront 1로 시작
4. 이상 없으면 CloudFront 쪽 1씩 증가, 최종 0 : 10
5. 롤백: 문제 발생 시 Akamai 가중치 복원

### Edge 규칙 재구현
- 리다이렉션, 헤더 조작, 이미지 최적화를 Lambda@Edge로 구현
- 이미지 최적화에 가장 많은 시간 소요, 2025-02부터 2025-04까지 v1에서 v4로 개선
- 최종 버전(v4) 동작
  - Accept 헤더 기반 AVIF, WebP, JPEG, PNG 포맷 선택, `Vary: Accept` 응답으로 캐시 분리
  - 2048px 리사이즈, 5000px 초과 이미지는 원본 반환
  - 애니메이션 WebP 150프레임 제한, 알파 채널 PNG는 WebP 변환
  - Lambda@Edge 응답 크기 제한(1MB) 초과 시 원본 반환

### 검증, 모니터링
- CloudFront 표준 로그 S3 적재 → Athena로 URL별, 국가별 집계
  - Athena: S3에 쌓인 파일을 SQL로 바로 조회하는 서버리스 쿼리 서비스
- URL별 집계(2025-02-24): 약 5,060만 요청, URL 6,608개 중 js, png, jpg, woff, css 등 정적 파일 99.95% → 이관 범위가 정적 컨텐츠로 한정됐음을 확인
- 나머지 0.05%는 `/.git/config`, `/server-status` 같은 경로를 찔러보는 스캐너 요청, 대부분 3xx, 4xx 응답
- 국가별 p95 응답 시간: 한국 약 13ms, 일본 약 21ms, 미국 약 25ms, 중국 약 28ms
- CloudWatch와 CloudFront 메트릭 통합 대시보드 구축, CDN 장애와 비정상 트래픽 급증 실시간 확인

## 결과

| 항목 | 결과 |
|---|---|
| 월 운영 비용 | 약 15% 절감 |
| 서비스 영향 | 무중단 전환 |
| Cache Hit Ratio | 기존 Akamai 수준 유지, URL별 집계 요청 가중 평균 약 89% |
| CloudFront 측 비용(2025-02-12부터 13일) | 총 $879, CloudFront $626, WAF $253, Lambda $0.02 |

- Cache Hit Ratio: 전체 요청 중 오리진까지 가지 않고 엣지 캐시에서 응답한 비율
- WAF(Web Application Firewall): HTTP 요청을 규칙으로 검사해 차단하는 L7 방화벽, CloudFront에 연결
Cache Hit Ratio 개선 없음, 기존 수준 그대로 이관
CloudFront 측 비용 중 WAF가 약 29% 차지

## 회고

**잘된 것**
가중치 단계와 롤백 경로를 먼저 정해 두고 전환 → 무중단 달성
범위를 정적 컨텐츠로 좁혀 예약, 결제 같은 동적 요청은 영향권에서 제외

**아쉬운 것**
15% 절감의 비교 기준(기간, 포함 항목)을 문서로 남기지 않아 지금 재현 불가
이미지 최적화 Lambda를 v4까지 반복 수정, 처음부터 응답 크기 제한과 포맷 협상을 설계에 넣었으면 반복 감소

**다음에 할 것**
WAF 비중이 커서 규칙 구성 대비 비용 재검토

## 관련 문서
- [[제주항공/00-index|제주항공 인덱스]]
