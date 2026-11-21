---
title: CI/CD 파이프라인 운영 (GitLab, Nexus, Jenkins)
date: 2026-11-21
updated: 2026-11-21
tags: [제주항공, cicd, jenkins, gitlab, nexus, ecr, eks]
status: draft
---
## 배경

- [[01-차세대 PSS 프로젝트 EKS 전환|PSS EKS 전환]] 중반 합류 시점에 Jenkins 기반 빌드, 배포 파이프라인은 이미 운영 중
- 이후 2025.06까지 파이프라인 잡 실행, 스크립트 수정, 배포 장애 대응 담당 (구조 설계는 미참여, 넘겨받아 운영)
- 폐쇄망 환경이라 소스, 라이브러리, 빌드가 각각 사내 서버로 분리된 구조
- 업무 영역(채널, 대응개발, ESB)마다 GitLab, Jenkins, Nexus가 따로 있어 처음엔 어느 서버가 무슨 역할인지부터 정리 필요

## 접근

전환 당시 CI/CD 매뉴얼(2021.11)과 운영 중 남은 매니페스트, 인수인계 메모를 기준으로 흐름 재구성
매뉴얼은 개발자 IDE 설정, GitLab 사용법까지 포함한 문서라 그중 인프라가 책임지는 구간만 추림

인프라 관점 책임 구간
- 빌드 서버(Jenkins)에서 소스 수신부터 이미지 생성, ECR push까지
- 매니페스트 생성과 EKS 반영
- 배포 자격증명, 서버 자원(디스크), 장애 대응

## 구현

### 구성 요소와 역할

| 도구 | 역할 | 비고 |
|---|---|---|
| GitLab | 소스 저장소, 브랜치, Merge Request 리뷰 | 통합 GitLab + ESB 전용 GitLab |
| Nexus | Maven 라이브러리 저장소, 외부 라이브러리 반입 관문, 사내 공통 라이브러리 공유 | 채널, 대응개발, ESB 3대, Nexus 3.x |
| Jenkins | 빌드, 배포 실행 엔진 | 통합 Jenkins + ESB 전용 Jenkins, 온프렘 VM |
| ECR | 컨테이너 이미지 저장소 | 개발, 운영 경로 분리 |
| EKS | 배포 대상 | WEB, WAS를 LENA(WEB, WAS 미들웨어, [[02-미들웨어 플랫폼 전환 (JEUS to LENA)]]) 기반 컨테이너로 운영 |

- Nexus: Maven 빌드가 `pom.xml`의 라이브러리를 인터넷 대신 사내 Nexus에서 받음, 외부 저장소 등록은 보안 부서 협의 필수
- 개발자 IDE도 로컬 Maven 저장소 설정을 같은 Nexus로 지정 → 로컬 빌드와 Jenkins 빌드가 같은 라이브러리 사용
- 매뉴얼 기준 GitLab 프로젝트 레포 40개 이상 (채널, 대응개발 업무)
- 빌드, 라이브러리 서버는 4코어, 8GB VM, Jenkins만 이미지 저장용 디스크 500GB 별도

### 동작 흐름 (EKS 업무)

1. 개발자 브랜치 push → GitLab Merge Request 리뷰 후 병합 (Developer는 master push 불가, Maintainer만 병합)
2. Jenkins가 GitLab에서 소스 pull (개발 `develop`, 운영 `master`)
3. Maven 빌드(`clean install` 또는 `clean package`) → WAR 생성, 라이브러리는 Nexus에서 수신
4. Execute Shell 스크립트 5단계
   1. WAR를 Docker 빌드 디렉토리로 복사 (이전 파일 삭제 후)
   2. ECR 로그인 (`aws ecr get-login-password`)
   3. Dockerfile 생성: `FROM <LENA WAS 베이스 이미지>` + `ADD <war>`
   4. `docker build`, `docker push`, 공통 base 매니페스트를 `envsubst`(환경변수 값을 템플릿 자리에 채우는 리눅스 유틸)로 치환해 서비스별 yaml 생성
   5. (개발만) 생성 yaml을 매니페스트 레포에 push
5. EKS 반영
   - 운영: Jenkins가 만든 매니페스트로 `kubectl apply` (운영 기간 기억 기준, [[03-EKS 관리형 노드그룹 최적화]]와 동일), 2021.11 매뉴얼엔 Bastion 경유로 표기, kubectl 실행 호스트 세부는 기록 없음
   - 개발: 매뉴얼상 매니페스트 레포 push 후 ArgoCD(Git의 매니페스트를 클러스터에 자동 반영하는 GitOps 도구) sync 경로 존재, 직접 다룬 기억 없음

```text
개발자 ─push/MR─▶ GitLab ─pull─▶ Jenkins ─Maven─▶ WAR ─▶ Docker build ─push─▶ ECR
                                   ▲  │
                     라이브러리 ───┘  └─ envsubst 매니페스트 ─▶ EKS 반영
                     (Nexus)
```

### 이미지, 매니페스트

- 베이스 이미지: LENA WAS에 TunA(APM, 트랜잭션 추적 도구) 에이전트, 유틸을 얹어 ECR에 올린 공통 이미지 → 서비스 이미지는 WAR만 추가
- 태그: `<업무>-was-날짜시간` (서비스명 `<업무>-was` + 빌드 시각)
- 매니페스트: 공통 템플릿에 이미지 태그, 노드 라벨, 서비스명 치환
- 템플릿 배포 설정
  - 롤링: `maxUnavailable: 0`(교체 중 기존 파드 감소 없음), `maxSurge: 10%`(새 파드 추가 비율)
  - 종료 순서 (preStop: 컨테이너 종료 직전 실행 훅)
    1. 30초 대기
    2. 헬스체크 파일(`ready.html`) 삭제 → ALB 헬스체크 실패로 대상에서 먼저 제외
    3. 120초 대기, 처리 중 요청 마무리
    4. 전체 유예 `terminationGracePeriodSeconds: 240` 초과 시 강제 종료
  - `revisionHistoryLimit: 1`(보관 이전 버전 수)
  - Deployment에 replicas 없이 HPA(CPU 기준 파드 수 자동 조절)로 운영
- 잡 형태: 2021.11 매뉴얼 기준 Jenkinsfile이 아닌 UI 구성 잡 + Execute Shell 스크립트
- 잡 구성: 서버의 스크립트 디렉토리(개발 `D_`, 운영 `P_` 접두사)를 기존 잡에서 복사 후 서비스명, 경로 수정

### 운영 절차

- LENA 패치
  1. 베이스 이미지 교체 후 ECR push
  2. 스크립트의 베이스 이미지 지정 수정
  3. Jenkins 빌드, 배포
  4. 서비스 확인
- 온프렘 VM 업무(LENA on Nutanix)
  1. Maven 빌드
  2. 기존 WAR 백업(2일 보관) 후 새 WAR 원격 전달
  3. LENA stop, start
- 접근 권한: Jenkins 접속은 PM/PL만, GitLab, Jenkins 로그인은 AD 계정
- 이벤트 대비 사전 증설
  - HPA(min 20, max 60, CPU 40%)가 걸려 있었지만 실제 자동 확장은 드묾
  - 짧은 시간 몰리는 이벤트 트래픽은 자동 확장을 기다릴 수 없어 이벤트 전 HPA min 상향으로 파드 사전 증설
  - 증설 규모는 JMeter 부하 테스트 결과로 결정 (이벤트별 min 값, 테스트 수치는 기록 없음)
  - Deployment에 replicas를 두지 않는 이유: 두면 배포마다 파드 수가 그 값으로 되돌아감
- 배포 단위 분리 (2025.02): 업무별 namespace로 Ingress 분리, `group.name`(여러 Ingress가 ALB 하나를 공유하게 묶는 설정)으로 ALB 수는 유지
- 업무별 ECR Repository, 배포 정책 구성은 구성정의서로 정리 ([[07-PSS 시스템 아키텍처]])

### 장애 대응

| 증상 | 원인 | 조치 |
|---|---|---|
| 이미지 생성 실패 | Jenkins 서버 디스크 풀 (누적 이미지) | 미사용 Docker 이미지 정리 |
| 빌드 중 ECR 단계 실패 | AWS 통신 장애 | 재시도 |
| 스크립트 단계 실패 | 잡 복사 후 경로 미수정 | 스크립트 경로 확인, 수정 |
| 빌드 오류 | 대량 신규 라이브러리 다운로드 | 매뉴얼 이슈 사례로만 기록, 조치 세부 미확인 |
| 병합 실패 | Merge Conflict | 개발 담당자와 소스 선택 또는 revert |

## 결과

| 항목 | 내용 |
|---|---|
| 운영 범위 | GitLab 레포 40개 이상 업무의 빌드, 배포 파이프라인 |
| 배포 방식 | 전 서비스가 같은 템플릿 롤링 설정 사용, 요청 유실 방지 목적 종료 순서 적용 |
| 라이브러리 | Nexus 3대 경유, 외부 저장소 추가는 보안 부서 협의 절차 거침 |
| 자격증명 | EKS 애드온은 IRSA(서비스어카운트에 IAM Role 연결, 임시 자격증명)로 전환, 온프렘 Jenkins는 배포용 IAM User 장기 키 유지 ([[10-서버 접근제어 체계 정비]]) |

## 회고

잘된 것
- 템플릿 하나에 롤링, preStop을 넣어 두니 서비스별 배포 방식 편차 없음
- Nexus를 관문으로 둔 덕에 폐쇄망에서도 빌드가 외부 저장소 상태에 흔들리지 않음

아쉬운 것
- 온프렘 Jenkins라 IRSA 대상이 아니어서 ECR push, EKS 반영 모두 장기 키 의존
- `revisionHistoryLimit: 1` + 베이스 이미지 ECR 덮어쓰기 → 롤백할 이전 버전이 거의 없음
- 날짜시간 태그라 이미지와 커밋 연결이 바로 안 보임
- 잡을 디렉토리 복사로 만들어 서비스마다 스크립트가 조금씩 달라지는 구조
- Jenkins 서버 root 권한으로 빌드, push

다음에 할 것
- Jenkins는 빌드, 태그 커밋까지만 하고 반영은 클러스터 안 GitOps 도구로 넘겨 클러스터 쓰기 권한 회수
- 장기 키 대신 임시 자격증명 (IAM Roles Anywhere: 온프렘 서버가 인증서로 IAM Role 임시 자격증명 발급)
- 커밋 해시 포함 이미지 태그, 베이스 이미지는 버전 태그로 유지
- 잡 스크립트를 공유 라이브러리나 Jenkinsfile 템플릿으로 통일

## 관련 문서
- [[00-index|제주항공 인덱스]]
- [[01-차세대 PSS 프로젝트 EKS 전환]]
- [[02-미들웨어 플랫폼 전환 (JEUS to LENA)]]
- [[03-EKS 관리형 노드그룹 최적화]]
- [[07-PSS 시스템 아키텍처]]
- [[10-서버 접근제어 체계 정비]]
