---
title: 온프렘 네트워크 IDS 구축기 — MikroTik + Suricata + Wazuh
date: 2026-06-16
tags: [suricata, ids, mikrotik, tzsp, wazuh, onprem, network-security]
status: done
---
## 배경

작년에 크립토마이닝 악성코드가 25일간 미탐지된 사고 발생
나중에 분석해보니 온프렘에 네트워크 레벨 탐지 수단이 전혀 없었던 게 원인
AWS는 GuardDuty가 VPC Flow Logs와 CloudTrail 기반으로 탐지를 커버하고 있었지만, 온프렘은 완전한 탐지 공백 상태
서버가 외부 C2 서버와 25일 동안 통신하는 동안 아무도 몰랐음

이번 구축의 목표는 단순
온프렘 서버/PC가 인터넷과 주고받는 트래픽을 실시간으로 들여다보고, 알려진 악성 패턴이 감지되면 즉시 알림을 받는 것

## 접근

처음에는 EC2에 IDS를 올리는 방안을 검토
하지만 온프렘 모니터링이 인터넷 연결성에 의존하는 구조가 되면 VPN 터널이 끊기는 순간 IDS 기능 전체가 상실
온프렘 보안 시스템이 클라우드 가용성에 종속되는 건 말이 안 된다고 판단
AWS는 GuardDuty가 이미 베이스라인을 커버하고 있으니, 온프렘 쪽에 투자하는 ROI가 훨씬 높다는 결론

네트워크 탭 위치는 MikroTik RB4011 라우터로 결정
온프렘 전체 인터넷 트래픽이 이 장비를 통과하기 때문
MikroTik은 TZSP(Tazmen Sniffer Protocol) 방식으로 트래픽을 미러링해서 다른 서버로 스트리밍하는 기능을 기본으로 지원
이 미러 트래픽을 보안 모니터링 서버에서 받아 Suricata로 분석하는 구조

미러링 대상을 처음에는 LAN 트렁크까지 포함하려 했는데, VLAN 내부 트래픽이 거의 1Gbps 포화 상태(VLAN20 TX 931Mbps, VLAN10 RX 926Mbps)라 서버 NIC 한계를 초과할 것으로 판단해 1단계에서는 제외
WAN 구간(WAN1, WAN2)만 미러링해도 인터넷 방향 위협은 전부 커버

TZSP 디캡슐화 도구는 tzsptap을 선택
tzsp2pcap + tcpreplay 조합도 검토했지만 Docker 기반 변환 단계가 추가되어 지연이 생기고, Suricata와 af-packet으로 직접 연동이 안 되는 게 단점
tzsptap은 TZSP 헤더를 벗겨내고 원본 이더넷 프레임을 TAP 인터페이스로 바로 밀어넣는 구조라 오버헤드 최소화

## 구현

### 전체 아키텍처

```
MikroTik RB4011 (WAN1 + WAN2)
│ TZSP 캡슐화 (UDP 37008) / 미러 트래픽 스트리밍
▼
보안 모니터링 서버 (10.1.x.x)
│ tzsptap: TZSP 헤더 제거 → 원본 이더넷 프레임 복원
▼
tap0 (가상 TAP 인터페이스, tzsptap 자체 생성)
│ Suricata 7.0.3 af-packet 모드 (IDS 패시브)
▼
룰 매칭 / EVE JSON 로그 생성
├── /var/log/suricata/fast.log
└── /var/log/suricata/eve.json
│ Wazuh Agent
▼
Wazuh Manager (Docker, 동일 서버) → Wazuh Dashboard
```

### MikroTik TZSP 설정

기존에 남아있던 트러블슈팅용 필터(특정 MAC, TCP/443만)를 전부 초기화한 뒤, WAN1·WAN2를 대상으로 보안 모니터링 서버로 스트리밍하도록 설정

```bash
/tool sniffer set \
  filter-interface=ether1,ether3 \
  filter-mac-address="" \
  filter-ip-protocol="" \
  filter-port="" \
  streaming-enabled=yes \
  streaming-server=10.1.x.x:37008
/tool sniffer start
```

장시간 운영 시 MikroTik sniffer의 기본 메모리 한도(10240KiB)에 도달하면 캡처가 중단될 수 있어, 1시간마다 자동으로 재시작하는 스케줄러를 추가해 대응

```bash
/system scheduler add name=sniffer-restart interval=1h \
  on-event="/tool sniffer stop; /tool sniffer start"
```

### tzsptap 설치

```bash
sudo mkdir -p /opt/tzsptap
cd /opt/tzsptap
sudo git clone https://github.com/0xc0decafe/tzsptap.git .
sudo make
sudo cp tzsptap /usr/local/bin/
```

처음에 `tzsp0` 인터페이스를 수동으로 만들어두고 tzsptap이 거기에 붙을 거라고 생각했는데, 실제로는 tzsptap이 소스 코드(`tun_alloc()`) 기준으로 `tap%d` 패턴으로 인터페이스를 자체 생성
외부에서 만든 인터페이스는 무시하고 `tap0`을 직접 만들어서 사용

```ini
[Unit]
Description=TZSP to TAP interface bridge for Suricata IDS
After=network.target
Wants=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/tzsptap -l 10.1.x.x -p 37008
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### Suricata 설치 및 설정

PPA 없이 Ubuntu 24.04 apt로 바로 설치

`/etc/suricata/suricata.yaml`에서 변경한 핵심 항목은 아래와 같음
HOME_NET은 회사 실제 내부망으로만 좁혀 오탐 감소, af-packet 인터페이스는 tzsptap이 생성하는 `tap0`으로 조정
checksum-validation을 끈 게 중요한데, TZSP 미러 트래픽은 NIC 체크섬 오프로드로 인해 체크섬이 0으로 찍혀 켜두면 구조적으로 오탐 발생

| 항목 | 기본값 | 변경값 |
|---|---|---|
| HOME_NET | 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12 | 회사 내부망 대역 |
| af-packet interface | eth0 | tap0 |
| checksum-validation | yes | no |
| unix socket | - | /run/suricata/suricata-command.socket |

### 룰셋 구성

ET Open 룰셋으로 66,633개를 로드했고 그중 50,386개가 활성화

TZSP 미러링 환경에서 `SURICATA` 접두사 프로토콜 이벤트 룰들은 구조적으로 오탐 발생
미러 트래픽이라 ① 체크섬이 0으로 찍히고, ② 세션 중간부터 패킷이 유입돼 TCP 상태 추적이 불가하며, ③ 패킷 방향 정보가 없기 때문
이 룰들을 전부 비활성화하고 ET Open 위협 탐지 룰에 집중

```
re:SURICATA STREAM        # TCP 세션 재조립 노이즈
re:SURICATA Applayer      # 방향 정보 부재로 인한 구조적 오탐
re:SURICATA HTTP          # 구조적 오탐
re:SURICATA UDPv4 invalid checksum
re:SURICATA UDPv6 invalid checksum
re:SURICATA TCPv4 invalid checksum
re:SURICATA TCPv6 invalid checksum
```

룰셋은 매일 새벽 3시 자동 업데이트되도록 cron에 등록

### 운영 자동화 및 Wazuh 연동

5분마다 tzsptap과 Suricata 프로세스를 확인하고, 비정상이면 자동 재시작 후 syslog에 기록하는 헬스체크 스크립트를 cron에 등록
Suricata EVE JSON 로그(`/var/log/suricata/eve.json`)는 [[01-에이전트 설치 및 구성|Wazuh Agent]]가 수집해서 Manager로 전달, Wazuh Dashboard에서 Suricata alert를 포함한 전체 보안 이벤트를 한 화면에서 확인 가능

## 결과

- MikroTik WAN1·WAN2 양방향으로 온프렘 전체 서버/PC의 인터넷 트래픽을 실시간으로 확인 가능해짐
- VLAN 4개(AI/R&D, 서버망, Office, 보안장비) 모두 WAN 구간이 커버됨
- **아웃바운드 탐지 체계 확보** — C2 비콘, 크립토마이닝 풀 연결 등 탐지

다만 탐지 못 하는 영역도 명확

- VLAN 내부 서버 간 lateral movement는 WAN 구간 미러링으로는 확인 불가
- HTTPS(443) 트래픽은 페이로드가 암호화돼 있어 TLS SNI와 JA3 핑거프린트 기반으로만 탐지 가능
- WireGuard VPN 내부 트래픽도 암호화되어 페이로드 탐지 불가

## 회고

tzsptap 선택이 잘 맞아떨어짐
af-packet 직접 연동으로 지연 없이 Suricata에 트래픽을 밀어넣을 수 있었고, Docker 없이 단순한 구조로 운영 부담이 낮음
SURICATA 계열 룰을 과감하게 비활성화한 것도 잘한 선택 — 초기에 켜두면 오탐이 너무 많아 실제 위협 탐지가 묻혀버림

LAN 트렁크 미러링을 1단계에서 제외한 게 아쉬운 점
내부 lateral movement 탐지가 빠진 상태라 Wazuh Agent 호스트 기반 탐지로 보완하고 있지만 완전하지 않음
TZSP 방식 자체의 한계(세션 중간 진입, 체크섬 오탐)도 구조적으로 해결이 안 되는 부분이라 운영하면서 계속 튜닝 필요
Office VLAN 미러링 추가와 SCA CIS Benchmark 조치, 오탐 패턴 추가 튜닝도 다음 작업으로 남음

## 관련 문서
- [[jtkdy/TelePIX/02. security/00-index|02. security 인덱스]]
- [[01-에이전트 설치 및 구성]]
- [[04-CloudWatch 연동 및 데일리 리포트]]
- [[온프렘 보안 모니터링 스택 구축|프로젝트 전체 타임라인]]
