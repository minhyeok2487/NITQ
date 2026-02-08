# 06-05. 이중화 설계 종합 시나리오

---

### 면접관: "고객사에서 신규 데이터센터 구축 RFP가 왔는데, 핵심 요구사항이 '모든 구간에서 단일 장애점이 없어야 한다'입니다. 전체 네트워크 이중화 설계를 어떻게 하시겠어요?"

전체 인프라를 계층별로 나누어 이중화 설계를 진행하겠습니다. SPOF(Single Point of Failure)가 없는 설계의 핵심은 **모든 경로에 최소 2개의 독립적인 통신 경로**를 확보하는 것입니다.

> 토폴로지 이미지 추가 예정

#### 전체 이중화 아키텍처

```
[ISP-A]         [ISP-B]              ← WAN 이중화 (Dual ISP + BGP)
   |               |
[BR-R1]─────────[BR-R2]             ← Border Router 이중화
   |               |
[ASA-A]─────────[ASA-B]             ← 방화벽 이중화 (ASA Failover)
   |     (Failover)  |
[DSW1]──────────[DSW2]              ← L3 Distribution 이중화 (HSRP)
  |  \          /  |
  |   ╲        ╱   |                ← Cross-link (L2 이중화)
  |    ╲      ╱    |
[ASW1] [ASW2] [ASW3] [ASW4]        ← Access 이중화 (Dual Uplink + STP)
  |      |      |      |
[Servers / End Users]
```

#### 계층별 이중화 설계

#### 1. WAN 이중화: Dual ISP + BGP
```
! BR-R1 - ISP-A 연결
router bgp 65001
 neighbor 203.0.113.1 remote-as 100
 neighbor 203.0.113.1 route-map ISP-A-IN in
 neighbor 203.0.113.1 route-map ISP-A-OUT out
 network 198.51.100.0 mask 255.255.255.0

! BR-R2 - ISP-B 연결
router bgp 65001
 neighbor 198.51.100.1 remote-as 200
 neighbor 198.51.100.1 route-map ISP-B-IN in
 neighbor 198.51.100.1 route-map ISP-B-OUT out
 network 198.51.100.0 mask 255.255.255.0

! ISP-A를 Primary로, ISP-B를 Backup으로 설정 (Local Preference)
route-map ISP-A-IN permit 10
 set local-preference 200

route-map ISP-B-IN permit 10
 set local-preference 100
```

#### 2. 방화벽 이중화: ASA Active/Standby Failover
```
! ASA-A (Primary)
failover
failover lan unit primary
failover lan interface FO GigabitEthernet0/3
failover link STATE GigabitEthernet0/4
failover interface ip FO 192.168.100.1 255.255.255.252 standby 192.168.100.2
failover interface ip STATE 192.168.200.1 255.255.255.252 standby 192.168.200.2
failover polltime unit 1 holdtime 3
failover replication http

interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 10.255.1.1 255.255.255.0 standby 10.255.1.2

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.255.2.1 255.255.255.0 standby 10.255.2.2
```

#### 3. L3 Distribution 이중화: HSRP + VLAN별 Active 분산
```
! DSW1 - VLAN 10 Active, VLAN 20 Standby
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt delay minimum 30
 standby 10 track 1 decrement 20

interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 90
 standby 20 preempt delay minimum 30

! DSW2 - VLAN 10 Standby, VLAN 20 Active
interface Vlan10
 ip address 10.10.10.3 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 100
 standby 10 preempt delay minimum 30

interface Vlan20
 ip address 10.10.20.3 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 110
 standby 20 preempt delay minimum 30
 standby 20 track 2 decrement 20
```

#### 4. L2 Access 이중화: STP + Root Bridge 일치
```
! DSW1 - VLAN 10 STP Root (HSRP Active와 일치)
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 8192

! DSW2 - VLAN 20 STP Root (HSRP Active와 일치)
spanning-tree vlan 10 priority 8192
spanning-tree vlan 20 priority 4096

! Access 스위치 - Dual Uplink
interface GigabitEthernet0/1
 description To_DSW1
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 spanning-tree portfast trunk

interface GigabitEthernet0/2
 description To_DSW2
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 spanning-tree portfast trunk
```

#### 5. 물리 링크 이중화: EtherChannel
```
! DSW1-DSW2 간 EtherChannel
interface Port-channel1
 description DSW1-DSW2_Interconnect
 switchport mode trunk

interface range GigabitEthernet0/23-24
 channel-group 1 mode active
 switchport mode trunk
```

---

### 면접관: "이 설계에서 SPOF가 완전히 없다고 확신할 수 있어요? 혹시 놓친 부분이 있을까요?"

좋은 질문입니다. 솔직히 말씀드리면, 장비 이중화만으로는 SPOF가 완전히 제거되지 않습니다. 실무에서 자주 놓치는 SPOF들이 있습니다.

#### 흔히 놓치는 SPOF

**1. Management Server (단일 SMS, 단일 NMS)**
- 방화벽 정책 관리 서버, 모니터링 서버가 1대이면 관리 평면(Management Plane)의 SPOF
- 해결: SMS 이중화, NMS HA 구성

**2. DNS / DHCP / AAA 서버**
- DNS 서버 1대가 죽으면 이름 해석 불가 -> 실질적 서비스 불가
- 해결: Primary/Secondary DNS, DHCP Failover

**3. 케이블/광 경로**
- 장비는 이중화했는데 **케이블이 같은 트레이(Cable Tray)를 지나가면** 물리 사고 시 양쪽 다 끊어짐
- 해결: 이중화 경로의 물리적 경로 분리

**4. 전원**
- 이중화 장비가 같은 UPS, 같은 PDU, 같은 전원 회선을 사용하면 전원 SPOF
- 해결: Dual Power Supply, A/B 전원 분리, 서로 다른 UPS 연결

**5. 같은 랙(Rack)**
- 이중화 장비를 같은 랙에 넣으면 랙 단위 장애(화재, 냉각 실패) 시 양쪽 다 영향
- 해결: Primary/Secondary 장비를 **물리적으로 다른 랙**에 배치

**6. 방화벽-스위치 간 연결 구간**
```
[ASA-A] ──── [DSW1]
              [DSW2]  ← ASA-A에서 DSW2로 가는 경로가 없으면 DSW1 장애 시 문제
```
- 해결: ASA의 Inside를 DSW1, DSW2 **양쪽 모두에 연결** (삼각형 구조)

**7. Spanning-Tree 관련 SPOF**
- STP Root Bridge가 1대이면, 그 장비 장애 시 STP 재수렴으로 30~50초 네트워크 불안정
- 해결: RSTP(Rapid STP) 적용 + Root/Secondary Root 명시적 지정

#### SPOF 식별 체크리스트

| 계층 | 확인 항목 | SPOF 여부 |
|------|----------|----------|
| WAN | ISP 회선 | Dual ISP ✓ |
| WAN | BGP 피어링 | 양쪽 라우터 ✓ |
| Security | 방화벽 | Failover ✓ |
| Security | Failover Link 자체 | **Dedicated EtherChannel 권장** |
| L3 | Distribution Switch | HSRP ✓ |
| L3 | 라우팅 프로토콜 | OSPF/EIGRP Area 설계 ✓ |
| L2 | Access Uplink | Dual Uplink ✓ |
| L2 | STP Root | Root/Secondary ✓ |
| Physical | 전원 | Dual PSU + A/B Feed 필요 |
| Physical | 케이블 경로 | 물리적 경로 분리 필요 |
| Service | DNS/DHCP/AAA | Primary/Secondary 필요 |
| Management | NMS/SMS | HA 또는 이중화 필요 |

---

### 면접관: "이중화 구성을 다 했는데, 방화벽 Failover 시 특정 서비스가 끊어진다고 합니다. 확인해보니 Asymmetric Routing 문제라고 하는데, 이게 뭐고 어떻게 해결하나요?"

**Asymmetric Routing(비대칭 라우팅)**은 요청 패킷과 응답 패킷이 **서로 다른 경로**를 통과하는 현상입니다. 일반 라우터에서는 문제가 안 되지만, **방화벽(Stateful Inspection 장비)에서는 심각한 문제**를 유발합니다.

#### 왜 문제가 되는가?

방화벽은 **Stateful**이므로 TCP 세션의 SYN -> SYN-ACK -> ACK 흐름을 추적합니다. 요청(SYN)이 ASA-A로 들어왔는데, 응답(SYN-ACK)이 ASA-B로 돌아오면 ASA-B는 해당 세션을 모르기 때문에 **패킷을 드랍**합니다.

#### 발생 시나리오 예시

```
[Client] --요청--> [BR-R1] ---> [ASA-A: Active] ---> [Server]
[Client] <--응답-- [BR-R2] <-X- [ASA-B: Standby, 세션 없음!] <--- [Server]
```

서버의 응답이 OSPF ECMP나 다른 라우팅 경로 때문에 ASA-B쪽으로 돌아가면, ASA-B의 Connection Table에 해당 세션이 없으므로 드랍됩니다.

#### 확인 방법

```
! ASA에서 드랍 확인
show asp drop

Flow is denied by configured rule (acl-drop)              0
No matching connection (no-matching-connection)       12345    <-- 이 수치가 증가!
TCP packet with invalid SYN                              67

! 패킷 트레이서로 비대칭 패킷 확인
packet-tracer input outside tcp 203.0.113.100 12345 10.0.1.10 80

! Syslog에서 확인
%ASA-6-106015: Deny TCP (no connection) from 10.0.1.10/80 to 203.0.113.100/12345
```

#### 해결 방법

**방법 1: 라우팅 설계로 해결 (근본적 해결, 최우선)**

방화벽 앞뒤의 라우팅을 조정해서 **같은 세션의 요청/응답이 같은 방화벽을 통과**하도록 합니다:

```
! BR-R1, BR-R2에서 내부 네트워크로의 경로를 Active 방화벽(ASA-A) 쪽으로만 향하게
! OSPF Cost 조정 또는 Static Route 사용
ip route 10.0.0.0 255.0.0.0 10.255.1.1    ! ASA-A의 Inside IP
```

DSW1, DSW2에서도 마찬가지로 외부 트래픽이 항상 Active ASA를 통과하도록 경로를 일치시킵니다.

**방법 2: ASA tcp-state-bypass (임시 우회)**
```
! 특정 트래픽에 대해 Stateful Inspection을 우회 (비권장, 임시)
access-list BYPASS_ACL extended permit tcp host 10.0.1.10 eq 80 any
class-map BYPASS_CLASS
 match access-list BYPASS_ACL
policy-map global_policy
 class BYPASS_CLASS
  set connection advanced-options tcp-state-bypass
```

이 방법은 **보안을 약화시키므로** 근본적인 라우팅 수정이 완료될 때까지의 임시 조치로만 사용해야 합니다.

**방법 3: ASA Cluster(Active/Active) + ECMP 지원**

최신 ASA(FTD 포함)나 Palo Alto는 Active/Active 클러스터에서 **Asymmetric 트래픽을 내부적으로 리다이렉트**하는 기능이 있어서, 비대칭 라우팅을 방화벽 레벨에서 처리할 수 있습니다. 하지만 구성 복잡도가 높습니다.

#### 비대칭 라우팅 방지 설계 원칙

1. **방화벽 앞뒤의 모든 경로가 Active 방화벽을 통과**하도록 라우팅 설계
2. OSPF/EIGRP Cost를 활용해 **Primary/Backup 경로를 명확히 구분**
3. ECMP(Equal-Cost Multi-Path)를 방화벽 구간에서는 **의도적으로 회피**
4. 방화벽 Failover 시 라우팅 수렴이 올바르게 동작하는지 반드시 테스트

---

### 면접관: "BFD라는 것도 이중화에서 쓰인다고 들었는데, 뭐예요? 이 설계에 어디에 적용하면 좋을까요?"

**BFD(Bidirectional Forwarding Detection)**는 두 장비 간의 링크 장애를 **매우 빠르게 감지**하기 위한 프로토콜입니다. RFC 5880에 정의되어 있습니다.

#### 왜 BFD가 필요한가?

OSPF의 기본 Dead Interval은 **40초**입니다. 링크가 끊어져도 OSPF가 이웃 관계를 해제하는 데 40초가 걸리면, 그동안 블랙홀이 발생합니다. 물리 인터페이스가 down이면 즉시 감지되지만, **L2 스위치를 경유하는 환경**에서는 중간 스위치 장애 시 양쪽 인터페이스가 UP인 상태에서 통신 불가가 발생합니다.

BFD는 **50ms~수백ms 주기의 Heartbeat**를 교환해서, L2 구간을 포함한 End-to-End 통신 장애를 거의 실시간으로 감지합니다.

#### BFD 동작 원리

```
[Router-A] ===BFD Hello(50ms)===> [Router-B]
[Router-A] <===BFD Hello(50ms)=== [Router-B]

! 3번 연속 미응답 시 (150ms) -> BFD Session Down
! -> OSPF/EIGRP/BGP/HSRP에 즉시 알림 -> 경로 재계산
```

#### 이 설계에서의 BFD 적용 포인트

**1. Border Router 간 (BR-R1 <-> BR-R2)**
```
! BR-R1
interface GigabitEthernet0/0
 ip address 10.255.0.1 255.255.255.252
 bfd interval 100 min_rx 100 multiplier 3

router ospf 1
 network 10.255.0.0 0.0.0.3 area 0
 bfd all-interfaces
```
- BFD Interval: 100ms, Multiplier: 3 -> **300ms**에 장애 감지

**2. DSW1 <-> DSW2 (Distribution 간 연결)**
```
! DSW1
interface Vlan999
 description Inter-Distribution Link
 ip address 10.255.99.1 255.255.255.252
 bfd interval 100 min_rx 100 multiplier 3

router ospf 1
 bfd all-interfaces
```

**3. HSRP + BFD 연동**
```
! DSW1
interface Vlan10
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt delay minimum 30
 standby bfd all-interfaces           ! BFD로 피어 장애 빠르게 감지
```

**4. BGP + BFD (Dual ISP 구간)**
```
! BR-R1
router bgp 65001
 neighbor 203.0.113.1 remote-as 100
 neighbor 203.0.113.1 fall-over bfd
```

#### BFD 적용 시 주의사항

1. **CPU 부하**: BFD Interval이 너무 짧으면 CPU 부하가 증가합니다. 50ms 이하는 하드웨어 BFD 지원 장비에서만 사용
2. **False Positive**: 네트워크가 일시적으로 혼잡할 때 BFD가 오탐할 수 있으므로, Multiplier를 3~5로 설정해서 여유를 줍니다
3. **양쪽 모두 설정**: BFD는 양쪽 장비 모두 설정해야 동작합니다
4. **ASA에서의 BFD**: ASA 9.x 이상에서 BFD 지원이 추가되었지만 제한적입니다. FTD에서는 더 완전한 지원

#### BFD 확인 명령어
```
show bfd neighbors                    ! BFD 세션 상태
show bfd neighbors detail             ! 상세 BFD 타이머 확인
debug bfd events                      ! BFD 이벤트 디버깅
```

출력 예시:
```
NeighAddr       LD/RD    RH/RS    State   Int
10.255.0.2      1/2      Up       Up      Gi0/0
10.10.10.3      3/4      Up       Up      Vl10
```

---

### 면접관: "이중화 구축을 다 완료하고, 고객사에 Failover Drill(장애 훈련) 테스트 계획서를 제출해야 합니다. 어떻게 작성하시겠어요?"

Failover Drill은 이중화가 실제로 동작하는지 **사전에 검증**하는 절차입니다. 구축 후 한 번만이 아니라, **정기적(분기/반기)**으로 수행해야 합니다. 종이 위의 이중화와 실제 동작하는 이중화는 다르기 때문입니다.

#### Failover Drill 테스트 계획서 구조

#### 1. 사전 준비
```
□ 유지보수 시간 확보 (고객 승인)
□ 현재 상태 기록 (Baseline)
    - show standby brief (HSRP)
    - show failover (ASA)
    - cphaprob stat (Checkpoint)
    - show spanning-tree summary
    - 각 구간 ping/traceroute 기록
□ Rollback 절차 문서화
□ 모니터링 도구 준비 (ICMP 지속 모니터링, Syslog 확인)
□ 관련 부서 사전 통보
```

#### 2. 테스트 시나리오 및 기대 결과

| # | 시나리오 | 수행 방법 | 기대 결과 | 허용 다운타임 | Rollback |
|---|---------|----------|----------|-------------|----------|
| T1 | **DSW1 HSRP Failover** | DSW1 VLAN10 `shutdown` | DSW2가 VLAN10 Active 승격 | < 3초 | DSW1 `no shutdown` |
| T2 | **DSW1 업링크 장애** | DSW1 업링크 `shutdown` | Tracking에 의해 DSW2 Active | < 5초 | DSW1 업링크 `no shutdown` |
| T3 | **ASA Failover** | ASA-A에서 `no failover active` | ASA-B Active 승격, 세션 유지 | < 1초 (Stateful) | `failover active` |
| T4 | **ASA 인터페이스 장애** | ASA-A Outside `shutdown` | Monitored Interface에 의해 Failover | < 3초 | `no shutdown` |
| T5 | **STP 장애** | DSW1-ASW1 링크 `shutdown` | RSTP 수렴, 대체 경로 활성화 | < 1초 (RSTP) | `no shutdown` |
| T6 | **BGP ISP 장애** | BR-R1 ISP-A 연결 `shutdown` | ISP-B로 자동 전환 | < BGP 수렴 시간 | `no shutdown` |
| T7 | **복합 장애** | DSW1 전원 OFF | HSRP + STP + 라우팅 전부 Failover | < 10초 | 전원 ON |
| T8 | **전체 복구** | 모든 장비 정상화 | Preempt에 의해 원래 상태 복귀 | 설계에 따라 | - |

#### 3. 각 시나리오별 상세 절차 (T3 ASA Failover 예시)

```
[사전 상태 기록]
ASA-A# show failover | include This host
  This host:  Primary - Active

[모니터링 시작]
외부 PC에서: ping -t 10.0.1.10 (내부 서버)
내부 PC에서: 웹 브라우저로 외부 사이트 접속 유지

[Failover 실행]
ASA-A# no failover active

[확인 항목]
1. ASA-B# show failover | include This host
   This host:  Secondary - Active     ← 승격 확인

2. Ping 끊김 횟수 기록
   Reply from 10.0.1.10: bytes=32 time=1ms
   Request timed out.                  ← 1회 이하 목표
   Reply from 10.0.1.10: bytes=32 time=2ms

3. 기존 웹 세션 유지 여부 확인 (Stateful Failover 검증)

4. show conn count 양쪽 비교

[Rollback]
ASA-A# failover active
! 원래 상태 복귀 확인
```

#### 4. 결과 기록 템플릿

| 시나리오 | 수행 시각 | 실제 다운타임 | 세션 유지 | 결과 | 비고 |
|---------|----------|-------------|---------|------|------|
| T1 | 02:00 | 1.5초 | N/A | **PASS** | - |
| T3 | 02:15 | 0.8초 | 유지됨 | **PASS** | - |
| T6 | 02:30 | 45초 | N/A | **주의** | BGP 수렴 시간 예상보다 김 |

#### 5. 테스트 후 필수 조치

```
□ 모든 장비가 원래 Active/Standby 상태로 복귀했는지 확인
□ HSRP: show standby brief
□ ASA: show failover state
□ STP: show spanning-tree root
□ BGP: show ip bgp summary
□ 비정상 항목 발견 시 즉시 원인 분석 및 조치
□ 테스트 결과 보고서 작성 및 고객 전달
□ 발견된 문제점에 대한 개선 계획 수립
```

---

### 면접관: "마지막으로, 이중화 설계에서 가장 중요하다고 생각하는 원칙이나 경험적 교훈이 있다면?"

#### 이중화 설계 5대 원칙

**1. "테스트하지 않은 이중화는 이중화가 아니다"**

가장 중요한 원칙입니다. 설계와 설정은 완벽한데, 실제 Failover를 한 번도 테스트하지 않은 환경이 너무 많습니다. 정기적인 Failover Drill을 **운영 절차에 포함**시켜야 합니다. 실제 장애 발생 시 "처음 겪는 Failover"가 되면 안 됩니다.

**2. "HSRP Active = STP Root = 트래픽 경로 일치"**

L2와 L3 이중화가 **불일치**하면 비효율 경로나 비대칭 라우팅이 발생합니다. 이중화 설계에서 가장 흔한 실수이고, 가장 간과하기 쉬운 부분입니다.

**3. "이중화의 적은 복잡성이다"**

이중화를 과도하게 하면 복잡성이 기하급수적으로 증가하고, 오히려 장애 대응이 어려워집니다. 이중화의 목표는 **"장애 시 서비스 연속성 보장"**이지, "절대 안 죽는 시스템"이 아닙니다. 비용 대비 효과를 고려한 적절한 수준의 이중화를 설계해야 합니다.

**4. "Failover 후의 성능도 설계에 포함"**

장비 2대가 50%씩 부하를 나누다가, 1대가 죽으면 남은 1대가 **100% 부하**를 감당해야 합니다. 이때 성능이 충분한지 반드시 확인해야 합니다. 장비 용량 설계 시 **Failover 상태에서의 최대 부하를 기준**으로 산정해야 합니다.

**5. "문서화는 이중화의 일부"**

이중화 구성의 상세 설계서, 현재 Active/Standby 상태, Failover 시 예상 동작, 수동 전환 절차 등을 **문서화**해두지 않으면 담당자 부재 시 장애 대응이 불가능합니다. 특히:

- 현재 Active/Standby 상태 현황
- 수동 전환 절차 (Step-by-step)
- Failover 시 영향받는 서비스 목록
- 각 장비별 역할 및 IP 할당표
- 긴급 연락처 (ISP, 벤더 TAC, 내부 담당자)

이 다섯 가지를 체크하면 대부분의 이중화 설계 및 운영에서 발생하는 문제를 예방할 수 있습니다.

#### 검증 명령어 종합 (Quick Reference)

```
! === HSRP ===
show standby brief
show standby vlan [id]
show track

! === ASA Failover ===
show failover
show failover state
show failover history
show monitor-interface

! === STP ===
show spanning-tree root
show spanning-tree summary
show spanning-tree vlan [id]

! === BGP ===
show ip bgp summary
show ip bgp neighbor [ip]
show ip route bgp

! === BFD ===
show bfd neighbors
show bfd neighbors detail

! === 일반 ===
show ip route
show ip interface brief
show etherchannel summary
show logging | include failover
```
