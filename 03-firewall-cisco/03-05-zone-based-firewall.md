# 03-05. Cisco Zone-Based Firewall

---

## 면접관: "소규모 지사에 방화벽이 필요한데, 별도 방화벽 장비를 구매할 예산이 없습니다. 이미 Cisco ISR 라우터가 있는데, 이걸 방화벽처럼 쓸 수 있는 방법이 있을까요?"

네, Cisco ISR 라우터에 내장된 **ZBF (Zone-Based Firewall)** 기능을 활용하면 별도 방화벽 없이도 Stateful Inspection 기반의 방화벽을 구현할 수 있습니다.

ZBF는 기존의 CBAC(Context-Based Access Control)을 대체하는 방식으로, **영역(Zone)을 정의하고 Zone 간의 트래픽 정책을 수립**하는 구조입니다.

기본 설계는 다음과 같습니다.

- **INSIDE Zone**: 내부 LAN (192.168.10.0/24) - GigabitEthernet0/0
- **OUTSIDE Zone**: 인터넷 연결 (WAN) - GigabitEthernet0/1
- **DMZ Zone**: 서버 영역 (172.16.1.0/24) - GigabitEthernet0/2 (있는 경우)

> 토폴로지 이미지 추가 예정

### 핵심 설정

```
! 1. Zone 정의
zone security INSIDE
zone security OUTSIDE
zone security DMZ

! 2. 인터페이스에 Zone 할당
interface GigabitEthernet0/0
 description LAN - Internal Network
 ip address 192.168.10.1 255.255.255.0
 zone-member security INSIDE
 no shutdown

interface GigabitEthernet0/1
 description WAN - Internet
 ip address dhcp
 zone-member security OUTSIDE
 no shutdown

interface GigabitEthernet0/2
 description DMZ - Server Network
 ip address 172.16.1.1 255.255.255.0
 zone-member security DMZ
 no shutdown
```

ZBF의 기본 규칙은 ASA와 유사합니다.
- **같은 Zone 내 트래픽**: 기본 **허용** (Zone 내부는 자유 통신)
- **다른 Zone 간 트래픽**: 기본 **차단** (명시적 정책 필요)
- **Zone에 속하지 않은 인터페이스**: 다른 Zone과 통신 불가

---

## 면접관: "Zone-pair라는 개념을 설명해주세요. Zone만 정의하면 되는 게 아닌가요?"

Zone만 정의하면 Zone 간 트래픽이 전부 차단됩니다. Zone 간에 트래픽을 허용하려면 **Zone-pair**를 정의하고 정책을 연결해야 합니다.

Zone-pair는 **출발 Zone → 도착 Zone** 방향을 지정하는 것입니다. 양방향 통신이 필요하면 Zone-pair를 2개 만들어야 할 수도 있지만, **Stateful Inspection을 적용하면 리턴 트래픽은 자동 허용**되므로 보통 한 방향만 설정합니다.

### Zone-pair 설계

```
+-------------------+
|    INSIDE Zone     |
|  (192.168.10.0/24) |
+--------+----------+
         |
         | zone-pair INSIDE→OUTSIDE (inspect: 허용)
         | zone-pair INSIDE→DMZ (inspect: 허용)
         ▼
+--------+----------+          +------------------+
|   OUTSIDE Zone     | ------→ |    DMZ Zone       |
|   (Internet)       |         | (172.16.1.0/24)   |
+-------------------+          +------------------+
  zone-pair OUTSIDE→DMZ (inspect: HTTP/HTTPS만 허용)
```

```
! Zone-pair 정의 (방향성 있음)
zone-pair security ZP_IN_TO_OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM_IN_TO_OUT

zone-pair security ZP_IN_TO_DMZ source INSIDE destination DMZ
 service-policy type inspect PM_IN_TO_DMZ

zone-pair security ZP_OUT_TO_DMZ source OUTSIDE destination DMZ
 service-policy type inspect PM_OUT_TO_DMZ
```

주의할 점:
- `zone-pair security ZP_IN_TO_OUT source INSIDE destination OUTSIDE`는 **INSIDE → OUTSIDE 방향**만 정의
- 리턴 트래픽(OUTSIDE → INSIDE)은 `inspect` action을 사용하면 Stateful하게 자동 허용
- OUTSIDE에서 INSIDE로 **새로운 연결을 시작**하려면 별도 zone-pair(`ZP_OUT_TO_IN`)가 필요

---

## 면접관: "그러면 실제로 Zone-pair에 연결할 정책은 어떻게 만드나요? 전체 설정을 보여주세요."

ZBF의 정책은 IOS의 **C3PL (Cisco Common Classification Policy Language)** 구조를 사용합니다. MPF와 유사하게 3단계입니다.

### C3PL 3요소

```
+-------------------+     +-------------------+     +-------------------+
| 1. Class Map      | --> | 2. Policy Map     | --> | 3. Zone-pair에    |
|    (type inspect)  |     |    (type inspect)  |     |    service-policy |
| "어떤 트래픽?"     |     | "어떤 Action?"     |     |    적용           |
+-------------------+     +-------------------+     +-------------------+
```

Action 유형:
- **inspect**: Stateful Inspection 적용 (세션 추적, 리턴 트래픽 자동 허용)
- **pass**: Stateless 통과 (리턴 트래픽 별도 정책 필요)
- **drop**: 차단 (기본값, 로깅 가능)

### 전체 ZBF 설정 예시

```
!=== 1단계: Class Map 정의 ===

! INSIDE → OUTSIDE: 일반 인터넷 트래픽
class-map type inspect match-any CM_INTERNET
 match protocol http
 match protocol https
 match protocol dns
 match protocol icmp
 match protocol ftp
 match protocol smtp
 match protocol pop3
 match protocol imap
 match protocol ntp

! OUTSIDE → DMZ: 웹서버 접근만 허용
class-map type inspect match-all CM_WEB_SERVER
 match protocol http
 match protocol https
 match access-group name ACL_TO_DMZ_WEB

! ACL for DMZ web server
ip access-list extended ACL_TO_DMZ_WEB
 permit tcp any host 172.16.1.10

! INSIDE → DMZ: 내부에서 DMZ 관리
class-map type inspect match-any CM_INSIDE_TO_DMZ
 match protocol http
 match protocol https
 match protocol ssh
 match protocol icmp

!=== 2단계: Policy Map 정의 ===

policy-map type inspect PM_IN_TO_OUT
 class type inspect CM_INTERNET
  inspect
 class class-default
  drop log

policy-map type inspect PM_OUT_TO_DMZ
 class type inspect CM_WEB_SERVER
  inspect
 class class-default
  drop log

policy-map type inspect PM_IN_TO_DMZ
 class type inspect CM_INSIDE_TO_DMZ
  inspect
 class class-default
  drop log

!=== 3단계: Zone-pair에 Policy 적용 ===

zone-pair security ZP_IN_TO_OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM_IN_TO_OUT

zone-pair security ZP_OUT_TO_DMZ source OUTSIDE destination DMZ
 service-policy type inspect PM_OUT_TO_DMZ

zone-pair security ZP_IN_TO_DMZ source INSIDE destination DMZ
 service-policy type inspect PM_IN_TO_DMZ
```

```
! NAT 설정 (ZBF와 별개로 필요)
ip nat inside source list ACL_NAT interface GigabitEthernet0/1 overload

ip access-list extended ACL_NAT
 permit ip 192.168.10.0 0.0.0.255 any

interface GigabitEthernet0/0
 ip nat inside
interface GigabitEthernet0/1
 ip nat outside
```

---

## 면접관: "ZBF를 설정했더니 라우터 자체에 SSH 접속이 안 됩니다. 원인이 뭔가요?"

이것은 **Self Zone** 관련 문제입니다. ZBF에서 매우 흔한 실수 중 하나입니다.

### Self Zone이란?

라우터 **자체로 향하는 트래픽**(또는 라우터 자체가 생성하는 트래픽)은 **Self Zone**이라는 특수한 Zone을 통해 처리됩니다.

```
+-----------+       Self Zone        +-----------+
|  INSIDE   | ----→ (라우터 자체) ---→ |  OUTSIDE  |
|  Zone     | ←---- (SSH, Telnet,    |  Zone     |
|           |        SNMP, NTP 등)   |           |
+-----------+                        +-----------+
```

**Self Zone의 기본 동작**:
- ZBF를 활성화하기 **전**: 라우터로의 모든 관리 트래픽 허용
- ZBF를 활성화한 **후**: Self Zone과 다른 Zone 간에 zone-pair가 없으면 기본적으로 **허용** (단, 일부 IOS 버전에서는 다르게 동작할 수 있음)

그런데 **Self Zone에 대한 zone-pair를 명시적으로 설정하면**, 해당 zone-pair에 매칭되지 않는 트래픽은 **차단**됩니다. 이것이 SSH 접속 불가의 원인일 수 있습니다.

### 해결 방법: Self Zone에 대한 정책 추가

```
! INSIDE에서 라우터 자체로의 관리 트래픽 허용
class-map type inspect match-any CM_MGMT_TO_SELF
 match protocol ssh
 match protocol telnet
 match protocol snmp
 match protocol icmp
 match protocol ntp
 match protocol https

! 라우터 자체에서 나가는 트래픽 허용 (DNS, NTP, Syslog 등)
class-map type inspect match-any CM_SELF_TO_OUTSIDE
 match protocol dns
 match protocol ntp
 match protocol syslog
 match protocol icmp

! 라우팅 프로토콜 (OSPF, EIGRP, BGP)
class-map type inspect match-any CM_ROUTING
 match protocol eigrp
 match protocol ospf

! Policy Map
policy-map type inspect PM_TO_SELF
 class type inspect CM_MGMT_TO_SELF
  inspect
 class type inspect CM_ROUTING
  pass
 class class-default
  drop log

policy-map type inspect PM_SELF_TO_OUT
 class type inspect CM_SELF_TO_OUTSIDE
  inspect
 class class-default
  drop log

! Zone-pair (self는 예약어)
zone-pair security ZP_IN_TO_SELF source INSIDE destination self
 service-policy type inspect PM_TO_SELF

zone-pair security ZP_SELF_TO_OUT source self destination OUTSIDE
 service-policy type inspect PM_SELF_TO_OUT
```

**주의사항**:
- `self`는 ZBF에서 예약된 Zone 이름이며, 별도로 `zone security self`를 정의할 필요 없음
- 라우팅 프로토콜(OSPF, EIGRP)은 `inspect`가 아닌 **`pass`**를 사용해야 함. 이들은 TCP/UDP 기반이 아닌 경우가 많아(OSPF는 IP protocol 89) Stateful Inspection이 적절하지 않을 수 있음
- DHCP relay를 사용한다면 Self Zone 정책에 DHCP도 포함해야 함

```
! 트러블슈팅: Self Zone 트래픽 확인
show policy-map type inspect zone-pair ZP_IN_TO_SELF sessions
show zone-pair security
```

---

## 면접관: "C3PL이라고 하셨는데, class-map에서 match-any와 match-all의 차이를 설명해주세요. 그리고 class-default는 뭔가요?"

### match-any vs match-all

```
! match-any: OR 조건 (하나라도 매칭되면 해당 class에 포함)
class-map type inspect match-any CM_WEB
 match protocol http       ← HTTP이거나
 match protocol https      ← HTTPS이면 매칭

! match-all: AND 조건 (모든 조건이 동시에 매칭되어야 함)
class-map type inspect match-all CM_SPECIFIC_WEB
 match protocol http            ← HTTP이면서 동시에
 match access-group name ACL1   ← ACL1에도 매칭되어야 함
```

**실무 활용 예시**:

```
! 여러 프로토콜 중 하나라도 해당되면 허용 → match-any
class-map type inspect match-any CM_ALLOWED_PROTOCOLS
 match protocol http
 match protocol https
 match protocol dns
 match protocol ftp

! 특정 출발지에서 특정 프로토콜만 허용 → match-all
class-map type inspect match-all CM_ADMIN_SSH
 match protocol ssh
 match access-group name ACL_ADMIN_SUBNET

ip access-list extended ACL_ADMIN_SUBNET
 permit ip 192.168.10.0 0.0.0.255 any
```

### class-default

```
policy-map type inspect PM_EXAMPLE
 class type inspect CM_ALLOWED
  inspect
 class class-default    ← 위의 어떤 class에도 매칭되지 않은 트래픽
  drop log              ← 차단하고 로그 남김
```

`class-default`는 Policy Map 내에서 **어떤 class에도 매칭되지 않은 나머지 모든 트래픽**을 처리합니다. 이것은 ASA ACL의 implicit deny와 유사한 역할입니다. `drop log`를 설정하면 차단된 트래픽을 로그로 확인할 수 있어 트러블슈팅에 유용합니다.

---

## 면접관: "ZBF와 ASA를 비교하면 어떤 장단점이 있나요? 실무에서 어떤 기준으로 선택하시겠어요?"

### ZBF vs ASA 비교

| 구분 | ZBF (Zone-Based Firewall) | ASA (Adaptive Security Appliance) |
|---|---|---|
| **플랫폼** | IOS/IOS-XE 라우터 (ISR, CSR) | 전용 방화벽 어플라이언스 |
| **주 목적** | 라우팅 + 부가 보안 | **전용 방화벽** |
| **성능** | 라우팅과 방화벽이 CPU 공유 | 방화벽 전용 최적화 하드웨어 |
| **최대 세션** | 모델별 제한 (수만~수십만) | 모델별 (5만~수백만) |
| **HA (이중화)** | HSRP 기반 (제한적) | Active/Standby, Active/Active |
| **VPN** | 강력 (다양한 VPN 지원) | 강력 (특히 RA-VPN) |
| **IPS 연동** | 제한적 | FirePOWER 모듈/FTD 전환 가능 |
| **관리** | CLI (C3PL 문법) | CLI + ASDM (GUI) |
| **설정 복잡도** | 높음 (C3PL 3단계 필수) | 상대적으로 단순 |
| **비용** | 추가 비용 없음 (라우터 내장) | 별도 장비 구매 필요 |
| **라이선스** | Security License 필요 (일부) | 별도 라이선스 체계 |

### 선택 기준

**ZBF를 선택하는 경우**:
- 소규모 지사/원격 사무소
- 별도 방화벽 예산이 없는 경우
- 이미 ISR 라우터가 설치되어 있는 경우
- 기본적인 Stateful Inspection만 필요한 경우
- WAN 라우터와 방화벽을 한 장비에서 해결하고 싶은 경우

**ASA/FTD를 선택하는 경우**:
- 본사/데이터센터 등 트래픽이 많은 환경
- 고급 보안 기능 필요 (IPS, AMP, URL Filtering)
- 고가용성(HA)이 필수인 경우
- 전문 보안 장비가 필요한 컴플라이언스 요구사항
- 대규모 ACL/NAT 정책 관리

```
! ZBF 상태 모니터링 명령어 정리
show zone security
show zone-pair security
show policy-map type inspect zone-pair
show policy-map type inspect zone-pair sessions
show class-map type inspect
show parameter-map type inspect
```

---

## 면접관: "마지막으로, ZBF를 운영 중인 지사 라우터에서 특정 Zone 간 트래픽이 차단되는데 원인을 모르겠다면, 어떻게 트러블슈팅하시겠어요?"

ZBF 트러블슈팅은 체계적인 단계를 밟아야 합니다.

### 1단계: Zone 할당 확인

```
! 인터페이스별 Zone 할당 상태 확인
show zone security

! 출력 예시:
! zone INSIDE
!   Member Interfaces:
!     GigabitEthernet0/0
!
! zone OUTSIDE
!   Member Interfaces:
!     GigabitEthernet0/1

! Zone에 할당되지 않은 인터페이스가 있는지 확인
show ip interface brief
```

인터페이스가 어떤 Zone에도 속하지 않으면 다른 Zone과 통신이 불가합니다.

### 2단계: Zone-pair 존재 여부 확인

```
show zone-pair security

! 출력 예시:
! Zone-pair name ZP_IN_TO_OUT
!   Source-Zone INSIDE  Destination-Zone OUTSIDE
!   service-policy PM_IN_TO_OUT
```

해당 방향의 zone-pair가 정의되어 있는지 확인합니다. zone-pair가 없으면 해당 방향의 모든 트래픽이 차단됩니다.

### 3단계: Policy Map 매칭 확인

```
! Zone-pair별 정책 상태와 히트 카운터 확인
show policy-map type inspect zone-pair ZP_IN_TO_OUT

! 출력 예시:
! Zone-pair: ZP_IN_TO_OUT
!   Service-policy inspect: PM_IN_TO_OUT
!     Class-map: CM_INTERNET (match-any)
!       Match: protocol http
!         152 packets, 23456 bytes     ← 매칭되고 있음
!       Match: protocol https
!         89 packets, 12345 bytes
!       Inspect
!         Session creations since subsystem startup: 241
!         Current session counts: 15
!
!     Class-map: class-default
!       Match: any
!       Drop
!         45 packets, 3200 bytes       ← 차단된 트래픽!
```

`class-default`에서 drop되는 패킷이 있다면, 해당 프로토콜이 class-map에 포함되지 않은 것입니다.

### 4단계: 디버깅 (주의: 트래픽 많은 환경에서는 CPU 부하)

```
! 방화벽 정책 처리 디버그
debug policy-firewall events
debug policy-firewall detail

! 특정 출발지에 대해서만 디버그 (CPU 부하 최소화)
debug policy-firewall events host 192.168.10.100

! 디버그 끄기
no debug all
```

### 5단계: 세션 상태 확인

```
! 활성 inspect 세션 확인
show policy-map type inspect zone-pair sessions

! 출력 예시:
! Established Sessions
!  Session 8A3F2B10 (192.168.10.100:54321) => (8.8.8.8:80)
!   http SIS_OPEN/SIS_OPEN
```

### 종합 트러블슈팅 체크리스트

```
1. show zone security                          → Zone에 인터페이스가 할당되었는가?
2. show zone-pair security                      → 해당 방향의 zone-pair가 존재하는가?
3. show policy-map type inspect zone-pair [name] → 정책이 올바른 트래픽을 매칭하는가?
4. show policy-map type inspect zone-pair [name] sessions → 세션이 생성되는가?
5. class-default에서 drop이 발생하는가?          → class-map에 프로토콜 추가 필요
6. Self Zone 문제인가?                          → 관리 트래픽용 zone-pair 확인
7. NAT 설정이 올바른가?                         → ZBF와 NAT는 독립적으로 동작
```

실무 팁: ZBF 트러블슈팅에서 가장 많이 빠지는 함정은 **Self Zone 정책 누락**과 **zone-pair 방향 실수**입니다. 항상 출발지→목적지 방향을 명확히 확인하고, 라우터 자체로 향하는 관리 트래픽도 빠짐없이 고려해야 합니다.
