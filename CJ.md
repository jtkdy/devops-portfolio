IBM KTS (2016.06 ~ 2021.07)
Field 시스템 엔지니어 — 금융/공공 등 고객사 대상 엔터프라이즈 인프라(IBM PowerSystem/AIX) 구성·변경작업 지원, 월 정기점검(장애 예방 및 발생 시 기술지원), 대규모 IDC 이전 프로젝트 지원
주요 고객사: 한국사이버결제, 한국거래소, 한국투자증권, 메리츠화재, 디비금융투자, LG, SK에너지, KTDS, 현대해상, 롯데, 네이버, 순천향병원, 교보생명, 대신증권, 삼성카드, KB그룹, 우리FIS 등
매달 30건 이상 고객사 인프라 장비 장애 접수 해결 및 기술지원

[KB금융그룹 IDC 이전 프로젝트] (2019.03 ~ 2019.06, 참여율 20%)
- 신규 데이터센터 이전에 따른 이전 계획 수립
- MOD 4 구성 신규 장비 자원 분배 (부서별 사용량 기준)
- 기존 OS 데이터 마이그레이션, 스토리지 펌웨어 업데이트 (10대)
- 활용기술: IBM PowerSystem, IBM Storage

[대신저축은행 IDC 이전 프로젝트] (2019.04 ~ 2019.09, 참여율 20%)
- 장비 노후화에 따른 서버 교체 및 IDC 이전
- 다운타임 최소화 기준 절체 계획 수립, 사전 실사 및 스펙 매칭
- 서버 절체·이전, 케이블 포설, 기능 검증
- 활용기술: IBM PowerSystem, AIX

[한국사이버결제 신규 DB서버 구축] (2019.05 ~ 2019.06, 참여율 70%)
- 신규 Oracle 서버(S914) OS 환경 구축 및 RAC 구성
- Oracle RAC 워크로드 기준 OS 파라미터 튜닝 (VMM, Block Size, ulimit, Network 등)
- 활용기술: AIX, Oracle RAC

[LG생활건강 E980·E870 VIOS 신규 Client 서버 구축] (2019.08, 참여율 70%)
- 시스템 고도화를 위한 데이터 마이그레이션
- 가상화 방식으로 NPIV 채택, SAN zoning 및 신규 파티션 구성
- 활용기술: AIX, VIOS

[교보생명 라이프플래닛 차세대 서버 구축] (2020.03 ~ 2020.07, 참여율 70%)
- 차세대 데이터센터 이전 목적 서버 구축 (E950/S914)
- alt disk 방식 OS 마이그레이션, 중요 서버 HACMP 선별 구성
- 활용기술: IBM PowerSystem, IBM Storage

AK아이에스 - 제주항공 인프라 운영 및 DevOps (2021.07 ~ 2025.06)
하이브리드 인프라(AWS + 온프레미스 Nutanix/vSphere) 운영, Public Cloud 구축, 미들웨어/백업/보안 솔루션 운영

[담당업무]
- Public Cloud(AWS) 기반 홈페이지 시스템 구축 및 운영 (EKS, IaC, Ingress, CloudFront 등)
- CI/CD(Jenkins) 마스터/에이전트 운영 및 부하 기반 스케일링, ECR push·Pod 배포까지의 멀티스테이지 파이프라인 설계
- 서버(Linux/Windows/AIX)·스토리지·백업·가상화(vSphere/Nutanix) 운영
- MS AD, VDI, 서버접근제어(SAC) 등 보안 솔루션 운영
- 미들웨어(Webtob/Jeus/LENA/IBM MQ) 및 DB 이중화(MCCS) 운영
- 이벤트(동접 10만~50만) 대비 Apache JMeter 부하테스트 수행 및 모니터링·장애 대응
- ISMS/ITGC 인증심사 대응

[차세대 PSS 프로젝트] (2021.07 ~ 2021.12)
- SITA 사업 철수에 따른 Legacy 예약시스템 → EKS 전환 (중간 합류)
- 무중단 배포 체계 구축, NLB/ALB 기반 트래픽 분산
- AWS MQ로 캐시 갱신/예약정보 동기화 이원화
- ELK 스택 기반 애플리케이션 로그 온프레미스 중앙집중화, Kibana로 통합 관제
- AWS 계정 분리 및 IAM 정책 최적화로 권한 관리 체계 확립
- CI/CD 파이프라인 구축 (LENA WEB/WAS 구성 및 APM TunA 설정)
- 활용기술: AWS EKS, EC2, Ingress Controller, IAM, LENA, TunA, Networker

[운송통합시스템 구축] (2023.10 ~ 2024.03)
- Nutanix 장비 구축 및 펌웨어 업데이트, 네트워크/VM 구성
- LENA WEB/WAS 구성 및 OS 보안조치
- APM(TunA) Transaction Trace 기반 트랜잭션 로그 분석으로 병목 구간·Memory Leak 포인트 파악 및 사전 제거
- 활용기술: Nutanix, RedHat 8.9, LENA, TunA

[EKS 관리형 노드그룹 최적화] (2024.12 ~ 2025.04)
- Kubecost/Prometheus 분석 기반 노드그룹 19개 → 5개 통합
- Blue-Green 배포 전략으로 서비스 중단 없이 최적화 수행
- CloudWatch 메트릭 기반 Auto Scaling 정책 재설계로 트래픽 급증 대응 체계 구축
- Multi-AZ 기반 글로벌 트래픽 분산 아키텍처 설계
- Terraform/CloudFormation IaC 전환, 미사용 리소스(SG/LB/EBS) 정리
- 월 약 2,500만원 비용 절감
- 활용기술: EKS, CloudFormation, IAM, AutoScalingGroup, Terraform

[CDN 이관 (Akamai → CloudFront)] (2025.05 ~ 2025.06)
- Akamai 라이선스 비용 부담으로 CloudFront 전환, Route53 가중치 기반 단계적 트래픽 이관
- Akamai Edge 규칙(리다이렉션, 헤더 조작, 이미지 최적화)을 Lambda@Edge로 재구현
- 활용기술: CloudFront, Lambda@Edge, Route53

[대량 트래픽 DB 커넥션 고갈 대응]
- 이벤트 트래픽 폭증 시 특정 R/O 인스턴스에 CPU/커넥션 집중, DB 커넥션 고갈로 서비스 중단 반복
- RDS Proxy 도입으로 인스턴스 간 세션 분배 정상화, 이후 트래픽 급증에도 중단 없이 서비스 유지
- 활용기술: AWS RDS, RDS Proxy

[사내 챗봇 시스템 구축] (2025.05 ~ 2025.06)
- AWS Bedrock/Knowledge Base(S3) 기반 RAG 생성형 AI 챗봇 인프라 구축
- 문서 청킹 사이즈 1,000 설정, Titan 모델 벡터화, Claude 3 Sonnet/Haiku 교차 활용
- TunA 모니터링 구축, VPC 엔드포인트로 네트워크 격리
- 활용기술: AWS Bedrock, S3, EC2

[NetWorker 기반 백업 체계 구축] (2021.07 ~ 2025.06, 지속운영)
- EKS 전환 이후 미비했던 백업 체계 정비, 온프렘 통합로그 서버 구축 및 ELK 설정, Kibana로 로그 통합 관제
- NetBackup → NetWorker 전환
- 시스템 중요도별 백업 보관기간/주기 정책 수립 (감사/컴플라이언스 대응)
- 약 4년간 약 500TB 규모 백업 운영
- 활용기술: NetWorker, ELK

TelePIX
사내 개발자 포탈 구축 (2026.05 ~ 2026.09)
- JSON Store 기반 DB 없는 아키텍처, FastAPI/React 기반 설계
- 기능: 배포 승인·서버 접근제어·보안 모니터링·서비스 카탈로그·장애 관리 통합 포탈 단독 구축
- Bitbucket PR Approve 상태 서버 재검증 기반 배포 승인 게이트 구현, 상태 머신(pending→approved/rejected/executing/failed) 설계로 무리뷰 배포 차단
- 사내 공용 Keycloak 라이브러리 부분 도입한 SSO 마이그레이션
- 신규 서버 등록 절차 자동화,서버 접근 제어 등
관련기술: FastAPI, React, Keycloak(OIDC), Bitbucket API, AWS SSM, Ansible, Docker

------------------------------------------------------------------------------------------------

보안 스택 구축 및 인시던트 대응 (2026.02 ~ 2026.09)
- SSH 패스워드 인증 노출 크립토재킹 사고 원인 규명, 이를 계기로 MikroTik-Suricata-Wazuh 연동 온프렘 네트워크 IDS 구축
- KISA 기반 Linux 서버 보안 취약점 조치 자동화, OpenSCAP 교차 검증
- IAM Roles Anywhere(PKI) 기반 온프렘-AWS 무인증키 접근 체계 설계
- Wazuh 호스트 보안 모니터링 도입, FIM 노이즈 제거·CloudWatch/Slack 연동 자동화
- Self-Hosted Prowler(CSPM) 구축, 최초 스캔 1,838건 FAIL 확인 후 포탈 연동
- GuardDuty severity 9.0 IAM 액세스키 탈취 인시던트 대응, 로깅 체계 재점검
관련기술: Wazuh, Suricata, Prowler, AWS IAM Roles Anywhere, OpenSCAP, GuardDuty

------------------------------------------------------------------------------------------------

AWS/Terraform 인프라 관리 및 비용 최적화 (2026.02 ~ 2026.09)
- 레거시 Terraform 리소스 실사용 검증 후 이관
- 서비스별 비용 태깅 전략 수립, 루트 모듈 12개 전수 감사·정책 적용
- 콘솔 수동 생성 ALB/타겟그룹/리스너룰 Terraform import로 코드 편입
- 미사용 Redis 클러스터·태그 없는 DocumentDB 2개 교차검증 후 안전 폐기
관련기술: Terraform, AWS

------------------------------------------------------------------------------------------------

Grafana 기반 통합 모니터링 구축 (2026.03 ~ 2026.09)
- IAM 인증 기반 Grafana Cloud 통합 모니터링 구축
- CUR 2.0 + Athena 기반 서비스별/리소스별 비용 분해
- ECS 서비스 로그 Loki 실시간 수집
- SLO 기반 Alert 설계, OpenAI 연동 조치 가이드 자동 응답
- Lambda+EventBridge 기반 데일리/먼슬리 리포트 자동화 (비용 현황·CloudTrail 인프라 변경 이력·GuardDuty/Wazuh 보안 이벤트 매일 아침 Slack 요약 전송)
관련기술: Grafana, Loki, Athena, CUR, Lambda, EventBridge, OpenAI API
