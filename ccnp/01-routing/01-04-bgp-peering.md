# 01-04. BGP 기본 구성과 Peering

### 면접관: "고객사가 현재 ISP 하나만 쓰고 있는데, 인터넷 회선 이중화를 하고 싶다고 합니다. ISP 2곳과 eBGP를 맺는 Dual-homed 구성을 설계해주세요."

Dual-homed eBGP 설계는 인터넷 이중화의 가장 기본적인 구성입니다.

**설계 방향:**
- 고객사 Edge Router 1대에 ISP-A, ISP-B 각각 eBGP Peering
- 기본적으로 ISP-A를 Primary, ISP-B를 Secondary로 운영
- 장애 시 자동 Failover
- Full Route 수신 vs Default Route만 수신할지는 고객사 라우터 성능에 따라 결정

**Full Route vs Default Route 결정 기준:**
- 라우터 메모리 4GB 이상, Internet Full Table(약 93만 경로) 수용 가능 → Full Route 수신으로 최적 경로 선택
- 메모리 부족하거나 단순 이중화만 필요 → Default Route만 수신

이 고객사는 중소규모로 가정하고, **Default Route + 부분 경로(Partial Route)**를 수신하는 구성으로 설계하겠습니다.

> 토폴로지 이미지 추가 예정

#### 핵심 설정

```
router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 !
 ! ISP-A (Primary) - AS 64500
 neighbor 203.0.113.1 remote-as 64500
 neighbor 203.0.113.1 description ISP-A-PRIMARY
 neighbor 203.0.113.1 prefix-list FROM-ISP-A in
 neighbor 203.0.113.1 prefix-list TO-ISP out
 neighbor 203.0.113.1 route-map ISP-A-IN in
 !
 ! ISP-B (Secondary) - AS 64600
 neighbor 198.51.100.1 remote-as 64600
 neighbor 198.51.100.1 description ISP-B-SECONDARY
 neighbor 198.51.100.1 prefix-list FROM-ISP-B in
 neighbor 198.51.100.1 prefix-list TO-ISP out
 neighbor 198.51.100.1 route-map ISP-B-IN in
 !
 ! 고객사 공인 IP 대역 광고
 network 100.64.10.0 mask 255.255.255.0
!
! ISP-A를 Primary로 선호 (Local Preference 높게)
route-map ISP-A-IN permit 10
 set local-preference 200
!
route-map ISP-B-IN permit 10
 set local-preference 100
!
! 보안: Bogon/Private 대역 필터링
ip prefix-list FROM-ISP-A deny 10.0.0.0/8 le 32
ip prefix-list FROM-ISP-A deny 172.16.0.0/12 le 32
ip prefix-list FROM-ISP-A deny 192.168.0.0/16 le 32
ip prefix-list FROM-ISP-A deny 0.0.0.0/0 ge 25
ip prefix-list FROM-ISP-A permit 0.0.0.0/0 le 24
!
ip prefix-list FROM-ISP-B deny 10.0.0.0/8 le 32
ip prefix-list FROM-ISP-B deny 172.16.0.0/12 le 32
ip prefix-list FROM-ISP-B deny 192.168.0.0/16 le 32
ip prefix-list FROM-ISP-B deny 0.0.0.0/0 ge 25
ip prefix-list FROM-ISP-B permit 0.0.0.0/0 le 24
!
! 고객사 대역만 광고
ip prefix-list TO-ISP permit 100.64.10.0/24
```

---

### 면접관: "BGP Peer가 맺어지는 과정을 설명해주세요. State Machine이 어떻게 되죠?"

BGP는 TCP 포트 179를 사용하는 유일한 라우팅 프로토콜이며, Peer 수립 과정이 명확한 State Machine으로 정의됩니다.

**BGP Finite State Machine (FSM):**

```
1. Idle
   └→ BGP 프로세스 시작, TCP 연결 시도

2. Connect
   └→ TCP 3-way handshake 진행 중
   ├→ 성공 시: Open 메시지 전송 → OpenSent
   └→ 실패 시: Active로 이동 (재시도)

3. Active
   └→ TCP 연결 재시도 중
   ├→ 성공 시: Open 메시지 전송 → OpenSent
   └→ 반복 실패 시: Idle로 리셋

4. OpenSent
   └→ Open 메시지를 보내고 상대방의 Open을 기다림
   ├→ 상대 Open 수신 + 파라미터 일치: Keepalive 전송 → OpenConfirm
   └→ 파라미터 불일치: Notification → Idle

5. OpenConfirm
   └→ 상대방의 Keepalive를 기다림
   ├→ Keepalive 수신: Established!
   └→ Notification 수신: Idle

6. Established ★
   └→ 정상 Peering 완료, Update 메시지 교환 시작
```

**Open 메시지에 포함되는 정보:**
- BGP Version (4)
- My AS Number
- Hold Time (기본 180초, 양쪽 중 작은 값 사용)
- BGP Router-ID
- Optional Parameters (Capabilities: MP-BGP, Route Refresh 등)

**주요 메시지 유형:**
| 메시지 | 용도 |
|--------|------|
| Open | 파라미터 협상 |
| Update | 경로 광고/철회 |
| Keepalive | 생존 확인 (기본 60초) |
| Notification | 에러 통보 후 세션 종료 |
| Route-Refresh | 경로 재요청 (soft reset) |

---

### 면접관: "좋습니다. 설정을 다 했는데, 고객사에서 'ISP-A와 BGP가 안 맺어진다'고 합니다. 어디부터 확인하시겠어요?"

BGP Peering 장애는 체계적으로 접근해야 합니다. Layer별로 올라가면서 확인합니다.

**1단계: Layer 1/2 - 물리 연결 확인**
```
show interface GigabitEthernet0/0
show ip interface brief | include GigabitEthernet0/0
```
인터페이스 UP/UP 확인. ISP 회선이 물리적으로 연결되었는지.

**2단계: Layer 3 - IP 연결 확인**
```
ping 203.0.113.1 source 203.0.113.2
```
eBGP는 directly connected가 기본이므로, IP 통신이 되어야 합니다.

**3단계: TCP 179 연결 확인**
```
show tcp brief | include .179
telnet 203.0.113.1 179
```
ACL이나 방화벽에서 TCP 179가 차단되어 있지 않은지 확인. 이것이 매우 흔한 원인입니다.

**4단계: BGP 설정 확인**
```
show bgp neighbors 203.0.113.1
show bgp summary
show running-config | section router bgp
```

**가장 흔한 원인들:**

| 증상 | 원인 | 해결 |
|------|------|------|
| Idle | remote-as 불일치, neighbor IP 오류 | 설정 확인 |
| Active | TCP 연결 실패 (ACL, 방화벽, 라우팅) | L3 연결 확인 |
| OpenSent | Open 파라미터 불일치 | AS, Router-ID 확인 |
| Idle (admin) | neighbor shutdown 상태 | `no neighbor X shutdown` |

**5단계: 디버깅 (운영 환경 주의)**
```
debug ip bgp 203.0.113.1 events
debug ip bgp 203.0.113.1 updates
!
! 디버깅 후 반드시
undebug all
```

가장 빈번한 실무 원인: **ISP 측에서 고객 AS Number나 Peer IP를 잘못 설정**, 또는 **중간 방화벽에서 TCP 179 미허용**.

---

### 면접관: "eBGP는 이해했습니다. 그런데 고객사 내부에 라우터가 여러 대 있고, 내부에서도 BGP 경로를 공유해야 한다면 iBGP를 써야 하잖아요. iBGP Full-mesh 문제와 해결책을 설명해주세요."

**iBGP의 핵심 규칙:**
> iBGP로 받은 경로는 다른 iBGP Peer에게 재광고하지 않는다 (Split Horizon Rule)

이 규칙은 루프 방지를 위한 것이지만, 결과적으로 **모든 iBGP 라우터가 서로 직접 Peering을 맺어야** 합니다 (Full-mesh).

라우터가 n대일 때 필요한 iBGP 세션 수: **n(n-1)/2**
- 5대: 10 세션
- 10대: 45 세션
- 50대: 1,225 세션

대규모 네트워크에서는 관리가 불가능합니다.

**해결책 1: Route Reflector (RR)**

RR은 iBGP의 Split Horizon 규칙을 완화합니다:
- **RR Client → RR:** 경로 수신 후, 다른 Client 및 non-Client iBGP Peer에게 Reflect
- **RR → Client:** non-Client이나 다른 Client에서 받은 경로를 Client에게 전달
- **Cluster 구성:** RR + Client들을 하나의 Cluster로 묶음

```
! Route Reflector 설정 (본사 코어 라우터)
router bgp 65001
 bgp cluster-id 1.1.1.1
 !
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source Loopback0
 neighbor 10.0.0.2 route-reflector-client
 !
 neighbor 10.0.0.3 remote-as 65001
 neighbor 10.0.0.3 update-source Loopback0
 neighbor 10.0.0.3 route-reflector-client
 !
 neighbor 10.0.0.4 remote-as 65001
 neighbor 10.0.0.4 update-source Loopback0
 neighbor 10.0.0.4 route-reflector-client
```

**RR 이중화:**
RR이 SPOF가 되면 안 되므로, 동일 Cluster 내 RR 2대를 구성합니다.

```
! RR-1
router bgp 65001
 bgp cluster-id 1.0.0.1
 neighbor 10.0.0.3 route-reflector-client
!
! RR-2 (동일 cluster-id)
router bgp 65001
 bgp cluster-id 1.0.0.1
 neighbor 10.0.0.3 route-reflector-client
```

동일 `cluster-id`를 사용하면 RR 간 루프를 방지합니다 (Cluster-List에 자신의 cluster-id가 있으면 Drop).

**해결책 2: BGP Confederation**

AS를 내부적으로 Sub-AS로 분할. 각 Sub-AS 내부는 iBGP Full-mesh, Sub-AS 간은 eBGP처럼 동작하지만 외부에는 하나의 AS로 보입니다. 대규모 ISP에서 주로 사용하며, 일반 기업에서는 Route Reflector가 더 일반적입니다.

---

### 면접관: "iBGP에서 next-hop-self가 왜 필요한지 설명해주세요. 안 하면 어떻게 되죠?"

이것은 iBGP 운영에서 가장 흔한 실수입니다.

**문제 상황:**
1. Edge Router(R1)가 ISP로부터 eBGP로 경로를 수신: `100.200.0.0/16, next-hop=203.0.113.1`
2. R1이 iBGP Peer(R2)에게 이 경로를 광고할 때, **Next-hop을 변경하지 않음** (iBGP 기본 동작)
3. R2가 받는 경로: `100.200.0.0/16, next-hop=203.0.113.1`
4. R2 입장에서 203.0.113.1은 ISP의 IP → **IGP 라우팅 테이블에 없음 → Next-hop Unreachable → 경로 사용 불가!**

```
! R2에서 확인하면
show bgp ipv4 unicast
  *  100.200.0.0/16   203.0.113.1     ← r 마크 없음 = RIB에 미설치
                                        Next-hop이 도달 불가능하므로 best 아님
```

**해결: next-hop-self**
```
! Edge Router (R1)
router bgp 65001
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source Loopback0
 neighbor 10.0.0.2 next-hop-self
```

이렇게 설정하면 R1이 iBGP Peer에게 경로를 광고할 때 Next-hop을 자신의 주소(Loopback0)로 변경합니다. R2는 IGP를 통해 R1의 Loopback에 도달할 수 있으므로 경로를 정상 사용합니다.

**확인:**
```
! next-hop-self 적용 후 R2에서
show bgp ipv4 unicast 100.200.0.0/16
  BGP routing table entry for 100.200.0.0/16
  Paths: (1 available, best #1)
    64500
      10.0.0.1 (metric 10) from 10.0.0.1   ← Next-hop이 R1의 Loopback
      Origin IGP, localpref 100, valid, internal, best
```

**Route Reflector 환경에서의 next-hop-self:**
RR에도 `next-hop-self`를 적용할지 여부는 설계에 따라 다릅니다. 일반적으로:
- Edge Router에서 `next-hop-self` 적용 (가장 일반적)
- RR에서는 Next-hop을 그대로 전달 (Client가 Edge Router의 Loopback에 도달 가능하면 됨)

---

### 면접관: "iBGP Peering을 Loopback으로 맺는 이유는 뭐죠? Physical Interface로 맺으면 안 되나요?"

**Loopback으로 iBGP Peering을 맺는 이유:**

1. **고가용성:** Physical Interface가 다운되면 Peering이 즉시 끊어짐. Loopback은 라우터가 살아있는 한 항상 UP. 여러 물리 경로가 있으면 IGP가 자동으로 우회하여 Loopback 간 연결 유지

2. **멀티패스:** R1↔R2 사이에 물리 링크가 2개 있으면, Physical Interface로 Peering하면 1개 링크에 종속. Loopback이면 IGP가 양쪽 링크를 모두 활용

3. **일관성:** 어떤 물리 인터페이스로 패킷이 들어오든, BGP 세션의 Source/Destination은 항상 동일한 Loopback IP

**설정 시 필수사항:**
```
! update-source 지정 필수 (기본은 outgoing interface가 source)
router bgp 65001
 neighbor 10.0.0.2 remote-as 65001
 neighbor 10.0.0.2 update-source Loopback0
!
! Loopback 간 IGP 도달성 확보 (OSPF/EIGRP에 Loopback 포함)
router ospf 1
 network 10.0.0.1 0.0.0.0 area 0
```

**eBGP의 경우는 다릅니다:**
eBGP는 기본적으로 TTL=1이므로 directly connected여야 합니다. Loopback으로 맺으려면:

```
! eBGP Multihop (Loopback으로 맺을 때)
router bgp 65001
 neighbor 1.1.1.1 remote-as 64500
 neighbor 1.1.1.1 ebgp-multihop 2
 neighbor 1.1.1.1 update-source Loopback0
!
! 상대방 Loopback에 대한 Static Route 필요
ip route 1.1.1.1 255.255.255.255 203.0.113.1
```

하지만 eBGP Multihop은 보안상 권장되지 않으며, 일반적으로 eBGP는 Physical Interface(directly connected)로 맺는 것이 표준입니다.

---

### 면접관: "마지막으로, 이 Dual-homed 구성에서 고객사 공인 IP를 양쪽 ISP에 광고하잖아요. 만약 고객사 라우터가 다운되면 어떻게 됩니까? 블랙홀 발생하지 않나요?"

맞습니다. 이것은 Dual-homed Single Router 구성의 근본적인 **SPOF(Single Point of Failure)** 문제입니다.

**블랙홀 시나리오:**
1. 고객사가 100.64.10.0/24를 양쪽 ISP에 광고 중
2. Edge Router 다운
3. ISP-A와 ISP-B는 BGP Hold Timer(180초) 만료 전까지 해당 경로를 계속 보유
4. 인터넷에서 100.64.10.0/24로의 트래픽이 ISP로 전달되지만, 다음 홉이 없어서 Drop

**해결 방법:**

**1. Dual-homed Dual Router (권장)**
Edge Router 2대, 각각 ISP 하나씩 연결. HSRP/VRRP + iBGP로 내부 이중화.

**2. BGP Timer 튜닝**
Hold Timer를 줄여서 빠른 경로 철회:
```
router bgp 65001
 neighbor 203.0.113.1 timers 10 30    ! Keepalive 10초, Hold 30초
```

**3. BFD (Bidirectional Forwarding Detection)**
BGP보다 훨씬 빠른 장애 감지 (밀리초 단위):
```
router bgp 65001
 neighbor 203.0.113.1 fall-over bfd
!
interface GigabitEthernet0/0
 bfd interval 300 min_rx 300 multiplier 3
```
BFD는 300ms x 3 = 900ms 만에 장애를 감지하고 BGP 세션을 즉시 끊어, ISP가 빠르게 경로를 철회합니다.

**4. ISP에 Maximum Prefix 설정 요청**
ISP 측에서 고객 AS로부터 비정상적으로 많은 경로가 오면 세션을 끊도록:
```
! ISP 측 설정 예시
neighbor 203.0.113.2 maximum-prefix 10 80 warning-only
```

실무에서는 Dual Router + BFD 조합이 가장 안정적인 설계입니다.
