---
title: Wazuh FIM 노이즈 제거
date: 2026-06-29
tags: [wazuh, fim, syscheck, ignore, noise]
status: done
---
## 배경

Wazuh를 처음 올렸을 때 데일리 리포트에 FIM 이벤트가 너무 많았음
`/usr/bin/tar`, `/usr/bin/python3`, `/etc/rmt` 같은 항목들이 매일 수십 건씩 올라왔는데, 알고 보니 전부 패키지 업데이트 후 정상 변경된 파일들
실제 위협과 무관한 이벤트가 리포트를 가득 채우면 정말 중요한 alert를 놓치게 됨
FIM 설정을 환경에 맞게 다듬는 작업 필요

## 접근

FIM(File Integrity Monitoring)은 지정한 경로의 파일이 변경되면 alert를 발생시키는 기능
문제는 Wazuh 기본 설정이 `/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`, `/boot` 전체를 모니터링 대상으로 잡고 있다는 점
이 경로들은 패키지 매니저가 관리하는 시스템 바이너리 경로라 업데이트할 때마다 대량의 변경 발생

우리 팀 온프렘 서버는 개발 서버 위주라 패키지 업데이트가 잦음
바이너리 경로 전체를 ignore하고 `/etc` 설정 파일 경로만 모니터링하는 방향으로 범위 축소

운영 노드가 추가될 경우는 다름
운영 서버에서 `/usr/bin` 같은 경로에 악성 바이너리가 심기는 건 실제로 탐지해야 할 시나리오
그래서 이 설정은 개발 서버 전용으로 적용하고, 운영 노드는 에이전트 레벨에서 별도 설정을 가져가기로 결정

## 구현

### Manager syscheck 수정

```xml
<!-- 변경 전 -->
<directories>/etc,/usr/bin,/usr/sbin</directories>
<directories>/bin,/sbin,/boot</directories>

<!-- 변경 후 -->
<directories>/etc</directories>
```

추가된 ignore 항목
`/etc/rmt`는 tar 패키지 업데이트 시 자동 변경되고, `/etc/ld.so.cache`는 라이브러리 캐시라 패키지 설치 때마다 갱신
`/etc/ca-certificates`는 CA 인증서 업데이트 시 자동 변경

```xml
<ignore>/etc/rmt</ignore>
<ignore>/etc/ld.so.cache</ignore>
<ignore>/etc/ca-certificates</ignore>
<ignore type="sregex">.log$|.swp$|.tmp$|.cache$|.pid$</ignore>
```

### 에이전트 syscheck 수정 (개발 서버)

Manager 설정과 에이전트 설정은 독립적이라 에이전트 각각의 `ossec.conf`도 동일하게 수정
에이전트 설정에는 Docker 관련 ignore가 추가로 있어 그 부분은 유지

```xml
<ignore>/var/lib/containerd</ignore>
<ignore>/var/lib/docker/overlay2</ignore>
```

### snap 관련 ignore 추가 (2026-07-14)

이미지 처리 서버가 데스크탑 환경이라 Firefox, Thunderbird, GNOME 등 snap 패키지가 다수 설치
snap 업데이트 시 `/etc/systemd/system/snap-*` 경로에 mount unit 파일이 자동으로 생성/변경되면서 FIM alert가 반복 발생
실제 보안 위협과 무관한 노이즈로 판단해 제외

```xml
<ignore>/etc/systemd/system/snap-</ignore>
<ignore type="sregex">snap\.</ignore>
```

snap 바이너리 자체(`/usr/bin/snap`)는 제외 대상 아님
실행 파일 변조는 계속 탐지되어야 함

### CUPS 관련 ignore 추가 (2026-07-14)

`/etc/cups/subscriptions.conf`와 `.O` 백업 파일이 반복 감지
프린터 서비스가 자동으로 변경하는 구독 설정 파일이라 ignore에 추가

```xml
<ignore>/etc/cups/subscriptions.conf</ignore>
<ignore>/etc/cups/subscriptions.conf.O</ignore>
```

### FIM 이벤트 정상 여부 판단 기준

| 상황 | 판단 |
|---|---|
| FIM 변경 시각과 패키지 업데이트 시각 30분 이내 일치 | 정상 변경 |
| 업데이트 이력 없음 + `/etc` 설정 파일 변경 | 변경 원인 확인 필요 |
| 업데이트 이력 없음 + 시스템 바이너리 변경 | 즉시 조사 필요 |

```bash
# dpkg 업데이트 이력 확인
grep "upgrade\|install\|remove" /var/log/dpkg.log | tail -20
# Rocky Linux는 yum 이력 확인
yum history list | head -20
```

## 결과

수정 전에는 데일리 리포트에 FIM 이벤트가 하루 수십 건씩 발생
수정 후 다음 syscheck 스캔 주기(12시간)부터 `/usr/bin`, `/usr/sbin` 관련 노이즈 소멸
남아있는 FIM 이벤트는 `/etc` 경로 변경이 전부라 하나하나 확인 가능한 수준으로 개선

## 회고

범위를 `/etc`로 줄이는 결정이 효과적
개발 서버 특성상 바이너리 경로 모니터링의 실익이 크지 않다는 판단이 적중

snap, CUPS 관련 노이즈는 설치 직후가 아니라 며칠 운영해보고 나서야 파악
처음부터 서버 환경(데스크탑인지 서버인지)을 더 꼼꼼히 파악하고 적용했으면 좋았을 것
운영 노드가 추가되면 해당 에이전트는 `/usr/bin`, `/usr/sbin` 모니터링을 별도로 활성화해야 하고, Ansible로 에이전트 설정을 관리하게 되면 개발/운영 프로파일을 나눠서 적용하는 구조를 만들 예정

## 관련 문서
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
- [[01-에이전트 설치 및 구성]]
- [[03-커스텀 룰 관리]]
- [[04-CloudWatch 연동 및 데일리 리포트]]
- [[온프렘 보안 모니터링 스택 구축|프로젝트 전체 타임라인]]
