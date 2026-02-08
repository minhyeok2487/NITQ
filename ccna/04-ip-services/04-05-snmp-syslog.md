# 04-05. SNMP와 Syslog

> CCNA 200-301 레벨 네트워크 엔지니어 면접 시나리오
> 주제: SNMPv2c vs v3, 커뮤니티 스트링, SNMP Get/Set/Trap, MIB, Syslog 심각도 레벨, 로깅 설정, 모니터링 구축

---

### 면접관: "네트워크에 라우터 20대, 스위치 50대가 있습니다. 이 장비들의 상태를 효율적으로 모니터링하려면 어떤 기술을 사용하시겠습니까?"

네트워크 모니터링에는 크게 **SNMP**와 **Syslog**, 두 가지 핵심 기술을 함께 사용합니다.

**SNMP(Simple Network Management Protocol)**는 네트워크 장비의 상태 정보를 수집하고 관리하는 프로토콜입니다. CPU 사용률, 메모리, 인터페이스 트래픽, 장비 가동 시간 등의 **성능 데이터를 주기적으로 수집(폴링)**하거나, 특정 이벤트 발생 시 **실시간 알림(Trap)**을 받을 수 있습니다.

**Syslog**는 네트워크 장비에서 발생하는 **이벤트 로그 메시지를 중앙 서버로 전송**하는 프로토콜입니다. 인터페이스 상태 변경, 설정 변경, 오류 등의 이벤트를 기록합니다.

두 기술의 차이를 정리하면 다음과 같습니다.

| 구분 | SNMP | Syslog |
|------|------|--------|
| 목적 | 성능 데이터 수집 및 장비 관리 | 이벤트 로그 기록 및 알림 |
| 동작 방식 | 폴링(주기적 수집) + Trap(이벤트 알림) | 이벤트 발생 시 로그 전송 |
| 프로토콜 | UDP 161 (Agent), UDP 162 (Trap) | UDP 514 |
| 데이터 | 수치 데이터 (CPU%, 트래픽량 등) | 텍스트 메시지 (이벤트 설명) |
| 활용 | 그래프, 대시보드, 임계값 알림 | 장애 원인 분석, 감사 추적 |

실무에서는 SNMP로 성능 추이를 모니터링하고, Syslog로 이벤트를 기록하여 두 가지를 상호 보완적으로 사용합니다.

---

### 면접관: "SNMP의 동작 방식을 좀 더 자세히 설명해주세요. Get, Set, Trap은 각각 무엇입니까?"

SNMP는 **Manager(NMS)**와 **Agent(네트워크 장비)** 사이에서 동작하며, 세 가지 주요 동작이 있습니다.

```
[NMS - Manager]                    [Network Device - Agent]
       │                                    │
       │─── SNMP Get Request ──────────────►│  "CPU 사용률 알려줘"
       │◄── SNMP Get Response ──────────────│  "현재 45%야"
       │                                    │
       │─── SNMP Set Request ──────────────►│  "인터페이스를 Shutdown 해"
       │◄── SNMP Set Response ──────────────│  "완료했어"
       │                                    │
       │◄── SNMP Trap ─────────────────────│  "링크가 다운됐어!" (비동기 알림)
       │                                    │
```

| 동작 | 방향 | 설명 |
|------|------|------|
| **Get** | Manager → Agent | 특정 정보를 요청 (읽기) |
| **GetNext** | Manager → Agent | MIB 트리의 다음 항목 요청 |
| **GetBulk** | Manager → Agent | 여러 항목을 한 번에 요청 (v2c/v3) |
| **Set** | Manager → Agent | 장비 설정 변경 (쓰기) |
| **Trap** | Agent → Manager | 이벤트 발생 시 비동기 알림 |
| **Inform** | Agent → Manager | Trap + 수신 확인 (v2c/v3) |

**Trap**은 Agent가 스스로 판단하여 보내는 비동기 알림이므로, Manager가 요청하지 않아도 인터페이스 다운, 장비 재부팅 등의 중요 이벤트를 즉시 알려줍니다. Trap은 UDP로 전송하므로 수신 확인이 없지만, **Inform**은 Manager의 확인 응답(Acknowledgment)을 요구하여 신뢰성이 더 높습니다.

---

### 면접관: "MIB은 무엇이고, 커뮤니티 스트링은 무엇입니까?"

**MIB(Management Information Base)**는 SNMP로 관리할 수 있는 객체들의 데이터베이스 구조입니다. 트리 형태로 구성되어 있으며, 각 객체는 **OID(Object Identifier)**라는 고유 번호로 식별됩니다.

```
MIB 트리 구조 (간략화):
iso(1)
  └── org(3)
       └── dod(6)
            └── internet(1)
                 └── mgmt(2)
                      └── mib-2(1)
                           ├── system(1)     → sysUpTime, sysName 등
                           ├── interfaces(2) → ifInOctets, ifOperStatus 등
                           └── ip(4)         → ipForwarding 등
```

예를 들어, 장비의 가동 시간을 나타내는 **sysUpTime**의 OID는 `1.3.6.1.2.1.1.3`입니다. NMS에서 이 OID를 Get 요청하면 장비가 부팅 이후 경과 시간을 반환합니다.

**커뮤니티 스트링(Community String)**은 SNMPv1과 v2c에서 사용하는 인증 메커니즘으로, 일종의 비밀번호입니다.

| 커뮤니티 스트링 | 권한 | 용도 |
|----------------|------|------|
| **RO (Read-Only)** | 읽기 전용 | Get 요청만 허용 (모니터링) |
| **RW (Read-Write)** | 읽기/쓰기 | Get + Set 요청 허용 (관리) |

기본값인 `public`(RO)과 `private`(RW)는 보안상 반드시 변경해야 합니다.

---

### 면접관: "SNMPv2c와 SNMPv3의 차이를 설명하고, 왜 v3를 권장하는지 말씀해주세요."

| 구분 | SNMPv2c | SNMPv3 |
|------|---------|--------|
| 인증 | 커뮤니티 스트링 (평문 전송) | 사용자 기반 인증 (USM) |
| 암호화 | 없음 | AES/DES 암호화 지원 |
| 접근 제어 | 제한적 | VACM (View-based Access Control) |
| 보안 수준 | 낮음 | 높음 |
| 설정 복잡도 | 간단 | 상대적으로 복잡 |

SNMPv3에는 세 가지 보안 수준이 있습니다.

| 보안 수준 | 인증 | 암호화 | 설명 |
|-----------|------|--------|------|
| **noAuthNoPriv** | 없음 | 없음 | 사용자 이름만으로 접근 |
| **authNoPriv** | MD5/SHA | 없음 | 인증은 하되 데이터는 평문 |
| **authPriv** | MD5/SHA | AES/DES | 인증 + 데이터 암호화 (권장) |

#### SNMPv2c 설정 예시

```
Router(config)# snmp-server community MyR0only RO
Router(config)# snmp-server community MyRWstr1ng RW
Router(config)# snmp-server host 10.0.0.50 version 2c MyR0only
Router(config)# snmp-server enable traps
```

#### SNMPv3 설정 예시

```
! 그룹 생성
Router(config)# snmp-server group MONITOR v3 priv

! 사용자 생성 (인증: SHA, 암호화: AES128)
Router(config)# snmp-server user admin1 MONITOR v3 auth sha AuthPass123 priv aes 128 PrivPass456

! Trap 수신 호스트 설정
Router(config)# snmp-server host 10.0.0.50 version 3 priv admin1
Router(config)# snmp-server enable traps
```

v2c는 커뮤니티 스트링이 네트워크에서 평문으로 전송되므로, 패킷 캡처 시 쉽게 노출됩니다. 보안이 중요한 환경에서는 반드시 **SNMPv3 authPriv** 수준을 사용해야 합니다.

---

### 면접관: "이제 Syslog에 대해 이야기해봅시다. Syslog 심각도 레벨을 설명해주세요."

Syslog는 0부터 7까지 8개의 심각도(Severity) 레벨을 정의합니다. 숫자가 낮을수록 심각합니다.

| 레벨 | 키워드 | 설명 | 예시 |
|------|--------|------|------|
| 0 | **Emergency** | 시스템 사용 불가 | 시스템 크래시 |
| 1 | **Alert** | 즉각 조치 필요 | 프로세스 장애 |
| 2 | **Critical** | 심각한 상태 | 하드웨어 오류 |
| 3 | **Error** | 오류 발생 | 인터페이스 다운 |
| 4 | **Warning** | 경고 상태 | 메모리 부족 임박 |
| 5 | **Notification** | 정상이지만 주목할 이벤트 | 인터페이스 Up/Down |
| 6 | **Informational** | 정보 메시지 | 설정 변경 완료 |
| 7 | **Debugging** | 디버그 메시지 | 상세 동작 정보 |

암기 요령으로 **"Every Awesome Cisco Engineer Will Need Ice-cream Daily"**를 사용하기도 합니다 (Emergency, Alert, Critical, Error, Warning, Notification, Informational, Debugging).

Syslog 메시지의 형식은 다음과 같습니다.

```
*Feb  9 09:30:45.123: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to down
                       ├──────┤ ├┤ ├────┤
                       Facility  │  Mnemonic
                                 │
                            Severity (3 = Error)
```

---

### 면접관: "Cisco 장비에서 Syslog를 설정하는 방법을 보여주세요. 로그를 어디로 보낼 수 있습니까?"

Cisco 장비에서 로그를 보낼 수 있는 대상은 여러 가지입니다.

#### 핵심 설정

```
! 1. 콘솔 로깅 (콘솔 포트에서 직접 확인)
Router(config)# logging console 6
→ Informational(6) 이하 메시지를 콘솔에 표시

! 2. 버퍼 로깅 (장비 메모리에 저장)
Router(config)# logging buffered 16384 6
→ 16KB 버퍼에 Informational(6) 이하 메시지 저장

! 3. 외부 Syslog 서버로 전송
Router(config)# logging host 10.0.0.100
Router(config)# logging trap 6
→ Informational(6) 이하 메시지를 Syslog 서버로 전송

! 4. 터미널 라인 로깅 (VTY 세션에서 확인)
Router(config)# logging monitor 6
Router# terminal monitor    ← VTY 세션에서 실행 필요

! 5. 타임스탬프 추가 (매우 중요!)
Router(config)# service timestamps log datetime msec localtime

! 6. 로그 소스 인터페이스 설정
Router(config)# logging source-interface Loopback0
```

#### 검증 명령어

```
Router# show logging
Syslog logging: enabled (0 messages dropped, 0 messages rate-limited,
                0 flushes, 0 overruns, xml disabled, filtering disabled)

Console logging: level informational, 125 messages logged, xml disabled,
                 filtering disabled
Monitor logging: level informational, 0 messages logged, xml disabled,
                 filtering disabled
Buffer logging:  level informational, 125 messages logged, xml disabled,
                 filtering disabled
Logging Exception size (4096 bytes)
Count and timestamp logging messages: disabled
Trap logging: level informational, 125 message lines logged
    Logging to 10.0.0.100 (udp port 514, audit disabled,
        link up), 125 message lines logged,
        0 message lines rate-limited, 0 message lines dropped-by-MD,
        xml disabled, sequence number disabled
        filtering disabled
```

`service timestamps log datetime msec localtime` 설정은 매우 중요합니다. 이 설정이 없으면 로그에 정확한 타임스탬프가 포함되지 않아 장애 분석이 어렵습니다.

---

### 면접관: "실무에서 SNMP와 Syslog를 활용한 모니터링 체계를 어떻게 구축하시겠습니까?"

실무 모니터링 체계 구축 시 다음과 같은 구조를 권장합니다.

```
[네트워크 장비들]
  Router/Switch
      │
      ├── SNMP (UDP 161/162) ──► [NMS 서버] ──► 대시보드/그래프
      │                          (Zabbix, PRTG, LibreNMS 등)
      │
      └── Syslog (UDP 514) ───► [Syslog 서버] ──► 로그 분석/검색
                                 (Graylog, Splunk, rsyslog 등)
```

#### 단계별 구축 방법

**1단계: 기본 설정 (모든 장비 공통)**

```
! SNMP 설정
Router(config)# snmp-server community M0n1tor! RO
Router(config)# snmp-server host 10.0.0.50 version 2c M0n1tor!
Router(config)# snmp-server enable traps snmp linkdown linkup
Router(config)# snmp-server enable traps config

! Syslog 설정
Router(config)# logging host 10.0.0.100
Router(config)# logging trap 5
Router(config)# logging buffered 32768 6
Router(config)# service timestamps log datetime msec localtime
```

**2단계: NMS에서 SNMP 폴링 주기 설정**
- 인터페이스 트래픽: 5분 간격
- CPU/메모리: 5분 간격
- 장비 Uptime: 15분 간격

**3단계: 알림 임계값 설정**
- CPU 사용률 > 80%: Warning 알림
- 인터페이스 Utilization > 90%: Critical 알림
- 링크 Down: 즉시 Trap 알림

**4단계: Syslog 필터링 및 알림**
- Severity 0-3 (Emergency~Error): 즉시 메일/SMS 알림
- Severity 4-5 (Warning~Notification): 대시보드 표시
- Severity 6-7 (Informational~Debug): 로그 저장만

이러한 체계를 구축하면 장애 발생 시 신속한 대응이 가능하고, 성능 추이 분석을 통해 사전에 문제를 예방할 수 있습니다.

---

### 면접관: "SNMP 보안 모범 사례를 간단히 정리해주세요."

CCNA 수준에서 알아야 할 SNMP 보안 모범 사례는 다음과 같습니다.

1. **기본 커뮤니티 스트링 변경**: `public`과 `private`은 절대 사용하지 않습니다.
2. **SNMPv3 사용**: 가능하면 v2c 대신 v3 authPriv를 사용합니다.
3. **ACL로 SNMP 접근 제한**: NMS 서버의 IP만 SNMP 접근을 허용합니다.

```
! ACL로 SNMP 접근을 NMS 서버(10.0.0.50)로만 제한
Router(config)# access-list 20 permit 10.0.0.50
Router(config)# snmp-server community M0n1tor! RO 20
```

4. **RW 커뮤니티 최소화**: 읽기 전용(RO)만 설정하고, Set이 필요한 경우에만 RW를 제한적으로 사용합니다.
5. **불필요한 SNMP 비활성화**: 사용하지 않는 장비에서는 SNMP를 끕니다.

```
! SNMP 완전 비활성화
Router(config)# no snmp-server
```

보안과 운영 편의성의 균형이 중요하며, 최소한 ACL로 접근을 제한하고 기본 커뮤니티 스트링을 변경하는 것은 반드시 적용해야 합니다.
