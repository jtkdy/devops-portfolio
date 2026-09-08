---
title: 03. incidents 인덱스
date: 2026-08-27
tags: [index, moc, incident]
status: active
---
장애/인시던트/트러블슈팅 기록
시간 순으로 정리

| 순서  | 문서                                           | 날짜         | 상태            | 내용                                                           |
| --- | -------------------------------------------- | ---------- | ------------- | ------------------------------------------------------------ |
| 1   | [[홈페이지 임시서버 크립토재킹 사고]]                       | 2026-04-13 | done          | SSH 패스워드 인증 노출로 침투, Monero 크립토재킹. OS 하드닝이 이미 완료된 상태에서도 뚫린 사례 |
| 2   | [[Lambda 스크래퍼 타임아웃 무한 재시도]]                  | 2026-06-03 | investigating | 배치 트랜잭션 타임아웃으로 15분마다 같은 데이터 재처리 반복                           |
| 3   | [[온프렘 서버 Swap 사용률 급등 대응]]                    | 2026-06-04 | draft         | swappiness 기본값 + 리소스 과밀 배치로 인한 Swap 누적                       |
| 4   | [[SDK 1.x EOS 알림 조사]]                        | 2026-06-30 | investigating | AWS Health EOS 알림 조사 중 CloudTrail Data Event 로깅 공백 발견        |
| 5   | [[Docker 컨테이너 서버 aws-roles-anywhere 서비스 중복]] | 2026-07-08 | resolved      | 동일 역할 systemd 서비스 중복으로 포트 충돌, 2개월간 미인지                       |
| 6   | [[satchat-gpu CloudFormation 이중소유 정리 사고]]    | 2026-07-23 | done          | Terraform 편입 작업 중 CREATE_FAILED 스택 정리하다 서비스 삭제·재생성·자동 롤백 발생  |
| 7   | [[IAM 액세스키 탈취 인시던트 대응]]                      | 2026-08-12 | done          | GuardDuty severity 9.0 탐지, 액세스키 탈취 대응 및 로깅 체계 재점검            |

## 관련 문서
- [[jtkdy/TelePIX/00-index|jtkdy 전체 인덱스]]
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
