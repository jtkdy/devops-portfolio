---
title: 온프렘 서버 Swap 사용률 급등 대응
date: 2026-06-04
tags: [linux, swap, memory, onprem, incident]
status: draft
---
## 배경

개발 인프라 서버 Swap 사용률이 80%를 넘어 지속되는 걸 모니터링에서 감지
RAM이 11GB나 남아있는데 Swap을 5.5GB씩 쓰고 있는 게 이상해 원인 확인, `swappiness`(커널이 메모리 대신 Swap을 얼마나 적극적으로 사용할지 결정하는 값)가 기본값 60으로 설정된 상태로 확인

문제는 단순히 설정값이 아닌 Jenkins, Nexus, Bitbucket Runner 같은 CI/CD 인프라와 uvicorn, Next.js 같은 서비스 런타임이 전부 올라가 있었고, JVM 힙 제한도 없이 동작 중
과거 메모리 부족 시점에 Swap으로 밀려난 페이지들이 회수되지 않고 계속 누적된 구조

## 접근

Swap 점유 상위 프로세스를 먼저 뽑아 어디서 얼마나 먹고 있는지 파악
Jenkins가 1.5GB, Nexus가 800MB, Open WebUI가 586MB 순, 셋 다 JVM 힙 제한이 없거나 과다하게 설정된 상태

조치 순서는 리스크가 낮은 것부터 배치

- **swappiness 변경** — 재시작 없이 즉시 적용, 신규 Swap 점유 억제 효과 → 최우선
- **Jenkins 힙 제한** — 재시작 한 번으로 즉시 회수 가능 → 두 번째
- **Nexus 힙 축소** — 파이프라인 중지 타이밍이 필요 → 마지막

리스크와 즉시 효과를 같이 따져서 순서를 결정

## 구현

### swappiness 60 → 10 변경

```bash
sysctl vm.swappiness=10
echo 'vm.swappiness=10' | tee /etc/sysctl.d/99-swap.conf
sysctl -p /etc/sysctl.d/99-swap.conf
```

### Jenkins JVM 힙 제한 추가

```bash
docker stop jenkins_server && docker rm jenkins_server
docker run -d --name jenkins_server \
  -p xxxx:8080/tcp \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -e JAVA_OPTS="-Xms512m -Xmx1536m -XX:+UseG1GC" \
  jenkins/jenkins:latest
```

`jenkins_home` 볼륨을 그대로 유지해서 데이터 손실 없이 재시작, 즉시 1.6GB 회수

### Nexus JVM 힙 축소 (진행 예정)

파이프라인 중지 후 진행 예정
작업 전 데이터를 먼저 백업하고, `nexus.vmoptions`를 볼륨 마운트로 연결하는 방식으로 처리

```bash
# 백업
mkdir -p /home/backup
docker run --rm -v nexus_data:/nexus-data -v /home/backup:/backup alpine \
  tar czf /backup/nexus_data_$(date +%Y%m%d).tar.gz -C / nexus-data

# vmoptions 생성 후 재시작
docker stop nexus_server && docker rm nexus_server
docker run -d --name nexus_server \
  -p xxxx:8081/tcp \
  -v nexus_data:/nexus-data \
  -v /home/nexus.vmoptions:/opt/sonatype/nexus/bin/nexus.vmoptions \
  sonatype/nexus3:latest
```

### 전체 조치 완료 후 기존 Swap 강제 회수

Nexus 작업 완료 후 새벽 부하 최저 시점에 진행 예정

```bash
swapoff -a && swapon -a
```

## 결과

| 항목 | 조치 전 | 조치 후 |
|---|---|---|
| Swap 사용 | 5.4GB | 3.8GB |
| Swap 사용률 | 80% | 56% |

Jenkins 재시작만으로 1.6GB 즉시 회수
Nexus 힙 축소가 완료되면 추가로 800MB 회수 예상

## 회고

이번 건을 보면서 `docker run` 방식으로 JVM 서비스를 그냥 올려두면 힙 제한을 명시하지 않는 이상 메모리를 무한정 먹을 수 있다는 걸 재확인
Nexus가 `-Xmx2703m`을 자동 계산값으로 잡은 것도 의도한 설정이 아니라 그냥 방치된 상태

중장기적으로는 CI/CD 인프라와 서비스 런타임을 서버 단위로 분리하는 게 맞는 방향이라는 걸 알면서도, 당장 손대기 어려운 작업이라 계속 지연
`docker run` 방식 컨테이너들을 compose로 전환하고 JVM 옵션을 코드로 관리하는 것도 함께 가져가야 할 것 같음

## 관련 문서
- [[jtkdy/TelePIX/03. incidents/00-index|03. incidents 인덱스]]
- [[03-온프렘 서버 모니터링 구축|온프렘 서버 모니터링]] (Swap 임계치 Alert로 이 사고를 탐지한 체계)
