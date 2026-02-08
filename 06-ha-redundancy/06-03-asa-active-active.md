# 06-03. Cisco ASA Active/Active Failover

---

## 면접관: "고객사에서 ASA 방화벽을 2대로 Active/Standby Failover를 구성해 놨는데, '1대는 놀고 있는 거 아니냐, 2대 다 트래픽 처리하게 해달라'고 요구합니다. 어떻게 하시겠어요?"

네, 이건 ASA **Active/Active Failover**로 해결할 수 있습니다. 다만 Active/Standby처럼 단순하지 않고, **Multi-Context Mode**가 전제 조건입니다.

Active/Active의 핵심 개념은 이렇습니다:
1. ASA를 **Multi-Context Mode**로 전환해서 여러 개의 가상 방화벽(Context)을 생성
2. 각 Context를 **Failover Group**에 할당
3. Failover Group 1은 ASA-A가 Active, Failover Group 2는 ASA-B가 Active
4. 결과적으로 **양쪽 모두 Active 역할**을 수행

> 토폴로지 이미지 추가 예정

### 설계 예시

| 구성 요소 | ASA-A (Primary) | ASA-B (Secondary) |
|----------|-----------------|-------------------|
| Failover Group 1 | **Active** | Standby |
| Failover Group 2 | Standby | **Active** |
| Context-OFFICE | FG1 소속 -> ASA-A에서 처리 | FG1 소속 -> ASA-B에서 대기 |
| Context-DMZ | FG2 소속 -> ASA-A에서 대기 | FG2 소속 -> ASA-B에서 처리 |

### 핵심 설정

**1단계: Multi-Context Mode 전환 (System Context에서)**
```
! 주의: 이 명령은 리부팅 필요하고, 기존 Single-Mode 설정이 초기화됨
mode multiple
```

**2단계: System Context에서 Failover 설정**
```
! ASA-A (Primary)
failover
failover lan unit primary
failover lan interface FAILOVER GigabitEthernet0/3
failover link STATE GigabitEthernet0/4
failover interface ip FAILOVER 192.168.100.1 255.255.255.252 standby 192.168.100.2
failover interface ip STATE 192.168.200.1 255.255.255.252 standby 192.168.200.2
failover key SECRET_KEY

! Failover Group 설정
failover group 1
 primary
 preempt 300
failover group 2
 secondary
 preempt 300
```

**3단계: Context 생성 및 Failover Group 할당**
```
! Admin Context (필수)
admin-context admin
context admin
 allocate-interface GigabitEthernet0/5
 config-url disk0:/admin.cfg

! Office Context -> Failover Group 1
context OFFICE
 allocate-interface GigabitEthernet0/0 outside
 allocate-interface GigabitEthernet0/1 inside
 config-url disk0:/office.cfg
 join-failover-group 1

! DMZ Context -> Failover Group 2
context DMZ
 allocate-interface GigabitEthernet0/6 dmz-outside
 allocate-interface GigabitEthernet0/7 dmz-inside
 config-url disk0:/dmz.cfg
 join-failover-group 2
```

**4단계: 각 Context에서 개별 방화벽 설정**
```
changeto context OFFICE
!
interface outside
 ip address 203.0.113.1 255.255.255.0
!
interface inside
 ip address 10.10.0.1 255.255.255.0
!
access-list OUTSIDE_IN extended permit tcp any host 203.0.113.10 eq 443
access-group OUTSIDE_IN in interface outside
```

이렇게 하면 ASA-A는 OFFICE 트래픽을, ASA-B는 DMZ 트래픽을 처리하면서, 어느 한쪽이 죽어도 나머지가 모두 인수받는 구조가 됩니다.

---

## 면접관: "Multi-Context Mode가 정확히 뭐예요? Single Mode랑 뭐가 다른 거죠?"

Multi-Context Mode는 하나의 **물리 ASA를 여러 개의 독립적인 가상 방화벽으로 분할**하는 기능입니다. 서버 가상화의 VM과 유사한 개념입니다.

### Single Mode vs Multi-Context Mode

| 항목 | Single Mode | Multi-Context Mode |
|------|------------|-------------------|
| **방화벽 수** | 1개 | 여러 개 (Context) |
| **설정 파일** | 하나의 running-config | System Config + 각 Context별 Config |
| **라우팅** | 단일 라우팅 테이블 | Context별 독립 라우팅 테이블 |
| **관리** | 전체 관리자 1명 | System Admin + Context별 Admin 가능 |
| **인터페이스** | 직접 사용 | `allocate-interface`로 Context에 할당 |
| **라이선스** | 기본 | Context 수에 따라 추가 라이선스 |
| **동적 라우팅** | 지원 | 지원 (최신 버전) |
| **VPN** | 지원 | **제한적** (일부 Context에서 불가) |

### Context 계층 구조

```
System Context (최상위)
├── admin-context (관리용 Context, 필수)
├── Context-OFFICE (사용자 정의)
├── Context-DMZ (사용자 정의)
└── Context-GUEST (사용자 정의)
```

- **System Context**: 물리 인터페이스 관리, Failover 설정, Context 생성/삭제 등 하드웨어 레벨 관리. 실제 트래픽 처리는 안 함
- **Admin Context**: 장비 관리 접근(SSH, ASDM 등)을 위한 특수 Context. Failover Group에 할당 불가
- **User Context**: 실제 트래픽을 처리하는 가상 방화벽

중요한 점은 Multi-Context 전환 시 **기존 Single Mode 설정이 모두 사라집니다.** 반드시 기존 설정을 백업하고, 전환 후 각 Context에 맞게 재설정해야 합니다. 이것 때문에 **신규 구축 시에 Active/Active를 결정하는 것이 이상적**이고, 운영 중인 Active/Standby를 Active/Active로 변경하는 것은 사실상 재구축에 가깝습니다.

---

## 면접관: "Failover Group이 정확히 어떤 개념인지 좀 더 설명해 주세요."

Failover Group은 **Context들을 묶어서 Failover 단위를 정의**하는 개념입니다.

Active/Standby에서는 장비 전체가 하나의 Failover 단위였습니다. Active/Active에서는 **Failover Group 단위로 Active/Standby가 결정**됩니다.

### 동작 방식

```
ASA-A                              ASA-B
┌─────────────────────┐           ┌─────────────────────┐
│ Failover Group 1    │           │ Failover Group 1    │
│ [ACTIVE]            │           │ [STANDBY]           │
│ - Context-OFFICE    │◄────────►│ - Context-OFFICE    │
│ - Context-GUEST     │  동기화   │ - Context-GUEST     │
├─────────────────────┤           ├─────────────────────┤
│ Failover Group 2    │           │ Failover Group 2    │
│ [STANDBY]           │           │ [ACTIVE]            │
│ - Context-DMZ       │◄────────►│ - Context-DMZ       │
│ - Context-SERVER    │  동기화   │ - Context-SERVER    │
└─────────────────────┘           └─────────────────────┘
```

### 핵심 규칙

1. **Failover Group은 최대 2개**만 생성 가능 (장비가 2대이므로)
2. 각 Context는 **반드시 하나의 Failover Group**에 소속
3. **Admin Context는 Failover Group에 할당 불가** (항상 양쪽에서 접근 가능)
4. 각 Failover Group은 독립적으로 Failover 발생 가능

### Failover 시나리오

**시나리오 1: ASA-A 완전 장애**
```
Before:                              After:
ASA-A: FG1=Active, FG2=Standby     ASA-A: DOWN
ASA-B: FG1=Standby, FG2=Active     ASA-B: FG1=Active, FG2=Active (모두 처리)
```

**시나리오 2: ASA-A의 Inside 인터페이스만 장애**
```
FG1만 Failover 발생 (Inside가 FG1에 할당된 Context에 속한 경우)
FG2는 영향 없음
```

이것이 Active/Active의 장점입니다. **일부 장애 시 해당 Failover Group만 전환**되고 나머지는 영향을 받지 않습니다.

### Failover Group 설정 상세
```
failover group 1
 primary            ! ASA-A(Primary)가 이 그룹의 Active를 선호
 preempt 300        ! 복구 시 300초 후 Active 탈환
 polltime unit 1 holdtime 3
 polltime interface 3 holdtime 10
 interface-policy 1

failover group 2
 secondary           ! ASA-B(Secondary)가 이 그룹의 Active를 선호
 preempt 300
 polltime unit 1 holdtime 3
 polltime interface 3 holdtime 10
 interface-policy 1
```

---

## 면접관: "구축 후에 Context 간 통신이 안 된다고 연락이 왔어요. OFFICE Context의 사용자가 DMZ Context에 있는 웹서버에 접속이 안 된대요. 어디부터 확인하시겠어요?"

이건 Multi-Context 환경에서 **자주 발생하는 문제**입니다. 기본적으로 각 Context는 **완전히 독립된 가상 방화벽**이므로, 같은 물리 ASA 안에 있어도 직접 통신이 안 됩니다.

### 원인 분석

Context 간 트래픽은 **ASA 내부에서 직접 전달되지 않습니다.** 반드시 **외부 스위치/라우터를 통해 나갔다 들어와야** 합니다.

```
[OFFICE Context] --outside--> [외부 L3 스위치] --dmz-inside--> [DMZ Context]
```

이것을 **"Hairpinning" 또는 "U-turn"** 트래픽이라고 합니다.

### 확인 순서

**1단계: Context 간 인터페이스 연결 확인**
```
! System Context에서
show context detail
```
각 Context에 할당된 인터페이스가 물리적으로 올바르게 연결되어 있는지 확인합니다.

**2단계: 외부 라우팅 확인**
```
changeto context OFFICE
show route

changeto context DMZ
show route
```
OFFICE에서 DMZ 네트워크로의 경로가 외부 라우터를 통해 존재하는지 확인합니다.

**3단계: ACL/NAT 확인**
```
changeto context DMZ
show access-list
show nat
```
DMZ Context에서 OFFICE 대역에서 오는 트래픽을 허용하는지 확인합니다.

### 해결 방법

**방법 1: 외부 L3 스위치에서 라우팅 (권장)**
```
! 외부 L3 스위치
interface Vlan100
 description To_OFFICE_Context_Outside
 ip address 203.0.113.254 255.255.255.0

interface Vlan200
 description To_DMZ_Context_Outside
 ip address 172.16.100.254 255.255.255.0

ip route 10.10.0.0 255.255.0.0 203.0.113.1    ! OFFICE 내부 -> OFFICE Context
ip route 172.16.1.0 255.255.255.0 172.16.100.1 ! DMZ 내부 -> DMZ Context
```

**방법 2: Shared Interface 사용**
같은 물리 인터페이스를 여러 Context에서 공유할 수 있습니다:
```
! System Context에서
interface GigabitEthernet0/0
 no shutdown

context OFFICE
 allocate-interface GigabitEthernet0/0 shared-outside
context DMZ
 allocate-interface GigabitEthernet0/0 shared-outside
```

하지만 Shared Interface는 MAC 주소 충돌 방지를 위해 **가상 MAC 설정이 필요**합니다:
```
mac-address auto
```

실무에서는 **방법 1(외부 라우터/스위치를 통한 U-turn)**이 더 일반적이고 트러블슈팅이 쉽습니다.

---

## 면접관: "Admin Context는 정확히 어떤 역할이에요? 왜 Failover Group에 할당이 안 되는 거죠?"

Admin Context는 **ASA 장비 자체의 관리 접근을 위한 특수 Context**입니다.

### Admin Context의 역할

1. **장비 관리 접근점**: SSH, ASDM, SNMP, Syslog 등 관리 트래픽의 진입점
2. **System Context와의 인터페이스**: System Context는 직접 네트워크 인터페이스를 가질 수 없으므로, Admin Context를 통해 장비에 접근
3. **장애 시에도 접근 필요**: 어떤 Failover Group이 전환되든 장비 관리는 계속 가능해야 함

### Failover Group에 할당 불가인 이유

Admin Context가 특정 Failover Group에 속하면, 그 그룹이 Standby인 ASA에서는 Admin Context도 Standby가 됩니다. 그러면 **해당 ASA의 관리 접근이 불가능**해집니다.

```
! 잘못된 예: Admin Context를 FG1에 할당
ASA-A: FG1=Active  -> Admin Context Active  -> SSH 가능
ASA-B: FG1=Standby -> Admin Context Standby -> SSH 불가!  (문제!)
```

Admin Context는 Failover Group에 속하지 않으므로 **양쪽 ASA 모두에서 항상 Active 상태**입니다. 각 ASA에 독립적인 관리 IP가 할당되어 있어서, 어느 쪽이든 접속해서 관리할 수 있습니다.

### Admin Context 설정 예시
```
! System Context에서
admin-context admin
context admin
 allocate-interface Management0/0
 config-url disk0:/admin.cfg

! Admin Context 설정
changeto context admin
interface Management0/0
 nameif management
 ip address 10.99.1.1 255.255.255.0

http server enable
http 10.99.0.0 255.255.0.0 management
ssh 10.99.0.0 255.255.0.0 management
```

---

## 면접관: "고객사가 Context마다 사용할 수 있는 리소스를 제한하고 싶다고 합니다. 한 Context가 리소스를 다 먹어서 다른 Context에 영향 주는 걸 방지하고 싶대요."

이건 **Resource Class**를 사용해서 해결합니다. Multi-Context에서 각 Context가 사용할 수 있는 리소스(Connection 수, Rate 등)에 상한을 설정하는 기능입니다.

### Resource Class 설정

```
! System Context에서 Resource Class 정의
class OFFICE_CLASS
 limit-resource Conns 50000          ! 최대 동시 연결 50,000
 limit-resource Rate Conns 1000      ! 초당 연결 1,000
 limit-resource Hosts 5000           ! 최대 호스트 5,000
 limit-resource Mac-addresses 2048   ! 최대 MAC 주소 2,048
 limit-resource Routes 500           ! 최대 라우팅 엔트리 500
 limit-resource Xlates 20000         ! 최대 NAT 변환 20,000
 limit-resource Inspects 50000       ! 최대 Inspect 엔트리 50,000
 limit-resource ASDM 3               ! 최대 ASDM 세션 3

class DMZ_CLASS
 limit-resource Conns 100000         ! DMZ는 서버 트래픽이 많으므로 더 많이 할당
 limit-resource Rate Conns 5000
 limit-resource Hosts 10000
 limit-resource Routes 200
 limit-resource Xlates 50000

! Context에 Resource Class 할당
context OFFICE
 member OFFICE_CLASS
 allocate-interface GigabitEthernet0/0 outside
 allocate-interface GigabitEthernet0/1 inside
 config-url disk0:/office.cfg
 join-failover-group 1

context DMZ
 member DMZ_CLASS
 allocate-interface GigabitEthernet0/6 dmz-outside
 allocate-interface GigabitEthernet0/7 dmz-inside
 config-url disk0:/dmz.cfg
 join-failover-group 2
```

### 리소스 사용량 모니터링

```
! System Context에서 전체 리소스 현황
show resource usage all

! 특정 Context의 리소스 사용량
show resource usage context OFFICE

Resource                   Current       Peak        Limit
Conns                      12345         35000       50000
Hosts                      2100          3500        5000
Xlates                     8900          15000       20000
Routes                     45            120         500
Rate Conns [per second]    450           890         1000
```

### Resource Class 미설정 시 기본 동작

Resource Class를 할당하지 않은 Context는 **default class**가 적용되며, 대부분의 리소스가 무제한(`unlimited`)입니다. 이 상태에서는 하나의 Context가 **ASA 전체 리소스를 독점**할 수 있어서, 다른 Context의 성능에 직접적인 영향을 줍니다.

따라서 Multi-Context 환경에서는 **반드시 Resource Class를 설정**하는 것이 Best Practice입니다. 특히 서로 다른 부서나 서비스를 Context로 분리한 경우, 한쪽의 DDoS 공격이나 트래픽 폭증이 다른 쪽에 영향을 주지 않도록 격리하는 것이 중요합니다.

---

## 면접관: "Active/Active 구성의 장단점과, Active/Standby와 비교해서 언제 어떤 걸 선택해야 하는지 정리해 주세요."

### Active/Active vs Active/Standby 비교

| 항목 | Active/Standby | Active/Active |
|------|---------------|---------------|
| **리소스 활용** | Standby는 유휴 | 양쪽 모두 트래픽 처리 |
| **필요 조건** | Single / Multi 모두 가능 | **Multi-Context 필수** |
| **복잡도** | 낮음 | 높음 |
| **트러블슈팅** | 비교적 쉬움 | 복잡함 (어느 Context, 어느 FG 문제인지 파악 필요) |
| **VPN 지원** | 완전 지원 | **제한적** |
| **Failover 단위** | 장비 전체 | Failover Group 단위 |
| **부하 분산** | 불가 (1대 유휴) | 가능 (FG 단위로 분산) |
| **라이선스** | 기본 | Context 라이선스 추가 필요 |

### 선택 기준

**Active/Standby를 선택하는 경우:**
- VPN(Site-to-Site, AnyConnect)을 많이 사용하는 환경
- 단순한 구성을 원할 때
- 운영 인력의 Multi-Context 경험이 부족할 때
- 예산이 제한적일 때 (추가 라이선스 불필요)

**Active/Active를 선택하는 경우:**
- 고가의 방화벽 2대의 투자 효율을 극대화하고 싶을 때
- 서로 다른 보안 정책이 필요한 여러 존(부서, 서비스)이 있을 때
- 높은 트래픽 처리량이 필요할 때
- 운영팀이 Multi-Context 운영 경험이 충분할 때

실무에서의 현실적인 조언을 드리면, **대부분의 중소규모 고객사에서는 Active/Standby가 적합**합니다. Active/Active는 구성 복잡도와 운영 난이도가 높아서, 충분한 경험과 문서화 없이 도입하면 오히려 장애 대응 시간이 길어질 수 있습니다. "2대 다 쓰고 싶다"는 요구에는 투자 효율보다 **안정성과 운영 용이성**이 더 중요하다는 점을 고객에게 설명하는 것도 엔지니어의 역할입니다.
