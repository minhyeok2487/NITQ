# 05-04. DMVPN (Dynamic Multipoint VPN)

---

## 면접관: "고객사가 현재 지사 10개인데, 내년에 50개로 확장 예정입니다. 그리고 지사 간 직접 통신도 필요하다고 하는데, 기존 Point-to-Point IPSec으로는 한계가 있을 것 같습니다. 어떻게 설계하시겠어요?"

지사가 50개로 확장되면 Point-to-Point VPN은 관리가 불가능합니다. 지사 50개를 Full-Mesh로 연결하면 **n(n-1)/2 = 1,225개 터널**을 수동으로 관리해야 합니다. 또한 지사를 추가할 때마다 기존 모든 장비의 설정을 변경해야 합니다.

이런 요구사항에 **DMVPN(Dynamic Multipoint VPN)**이 정확히 적합합니다.

> 토폴로지 이미지 추가 예정

### DMVPN 개요

DMVPN은 **mGRE(Multipoint GRE)** + **NHRP(Next Hop Resolution Protocol)** + **IPSec**을 결합한 Cisco 솔루션입니다.

- **Hub(본사)**: mGRE 인터페이스로 모든 Spoke와 단일 터널로 연결
- **Spoke(지사)**: Hub를 향한 GRE 터널 1개만 설정. Spoke 간 통신은 **동적으로 직접 터널**이 생성됨
- **NHRP**: Spoke의 실제 WAN IP를 Hub에 등록하고, Spoke 간 통신 시 상대방 IP를 조회하는 프로토콜

### DMVPN Phase 3 설계 (최종 권장)

| 항목 | Hub (본사) | Spoke (지사) |
|------|-----------|-------------|
| 장비 | ISR 4451 | ISR 4221 |
| WAN IP | 203.0.113.1 | DHCP (유동 가능) |
| Tunnel IP | 172.16.0.1/24 | 172.16.0.x/24 |
| NHRP 역할 | NHS (NHRP Server) | NHC (NHRP Client) |
| 라우팅 | EIGRP / BGP | EIGRP / BGP |

### 핵심 설정 - Hub

```
! ── mGRE Tunnel 인터페이스 ──
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp network-id 1
 ip nhrp authentication DMVPN!Key
 ip nhrp map multicast dynamic
 ip nhrp redirect
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile IPSEC-PROF-DMVPN
 ip mtu 1400
 ip tcp adjust-mss 1360

! ── NHRP Redirect: Phase 3 핵심 명령어 ──
! Hub가 Spoke-to-Spoke 트래픽을 감지하면
! NHRP Redirect를 Spoke에게 보내서 직접 터널을 만들도록 유도

! ── EIGRP ──
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 10.1.0.0 0.0.255.255
 no auto-summary

! ── IKEv2 + IPSec Profile ──
crypto ikev2 proposal PROP-DMVPN
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-DMVPN
 proposal PROP-DMVPN

crypto ikev2 keyring KR-DMVPN
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key DMVPN-PSK!

crypto ikev2 profile PROF-DMVPN
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-DMVPN

crypto ipsec transform-set TSET-DMVPN esp-aes 256 esp-sha256-hmac
 mode transport

crypto ipsec profile IPSEC-PROF-DMVPN
 set transform-set TSET-DMVPN
 set ikev2-profile PROF-DMVPN
```

### 핵심 설정 - Spoke

```
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp network-id 1
 ip nhrp authentication DMVPN!Key
 ip nhrp nhs 172.16.0.1 nbma 203.0.113.1 multicast
 ip nhrp shortcut
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile IPSEC-PROF-DMVPN
 ip mtu 1400
 ip tcp adjust-mss 1360

! ── NHRP Shortcut: Phase 3 핵심 명령어 ──
! NHRP Redirect를 받으면 직접 터널을 만들어서
! Hub를 거치지 않고 Spoke-to-Spoke 직접 통신

router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 10.2.0.0 0.0.255.255
 no auto-summary

! IKEv2/IPSec 설정은 Hub와 동일 (peer ANY)
```

---

## 면접관: "DMVPN Phase 1, 2, 3의 차이점을 설명해주세요. 왜 Phase 3를 선택하셨나요?"

### Phase별 비교

| 항목 | Phase 1 | Phase 2 | Phase 3 |
|------|---------|---------|---------|
| Spoke-to-Spoke | **Hub 경유** (항상) | **직접 터널** | **직접 터널** |
| Spoke 터널 유형 | p2p GRE | **mGRE** | **mGRE** |
| Spoke 간 터널 생성 | 불가 | NHRP Resolution | **NHRP Redirect + Shortcut** |
| 라우팅 Next-hop | Hub IP | **Spoke 실제 Tunnel IP** | Hub IP (CEF Shortcut 오버라이드) |
| Hub 설정 변경 (Spoke 추가 시) | 필요 | 불필요 | 불필요 |
| 확장성 | 낮음 | 중간 | **높음** |
| Summarization | 불가 | **불가** | **가능** |

### Phase 1: Hub-and-Spoke Only

```
! Spoke: Point-to-Point GRE
interface Tunnel0
 tunnel mode gre ip             ← p2p GRE
 tunnel destination 203.0.113.1  ← Hub IP 고정

! 모든 Spoke 트래픽이 반드시 Hub를 경유
! Spoke 간 직접 통신 불가
```

### Phase 2: Spoke-to-Spoke 가능, 제약 있음

```
! Spoke: mGRE로 변경
interface Tunnel0
 tunnel mode gre multipoint     ← mGRE
 ! tunnel destination 없음

! Spoke 간 직접 터널 생성 가능
! 하지만 라우팅에서 Next-hop이 실제 Spoke IP여야 함
! → Hub에서 next-hop을 변경하면 안 됨 (no ip next-hop-self)
! → 라우트 Summarization 불가 (Spoke IP 정보가 사라지므로)
```

### Phase 3: 최적 (현재 권장)

```
! Hub: ip nhrp redirect
! Spoke: ip nhrp shortcut

! 동작 방식:
! 1. Spoke A → Hub → Spoke B (처음에는 Hub 경유)
! 2. Hub가 "나를 거치지 말고 직접 가라"는 NHRP Redirect 전송
! 3. Spoke A가 Spoke B에게 NHRP Resolution Request 전송
! 4. Spoke B가 자신의 NBMA(실제 WAN) IP로 응답
! 5. Spoke A ↔ Spoke B 직접 터널 생성 (CEF Shortcut)
! 6. 이후 트래픽은 Hub를 거치지 않음

! 장점: 라우팅 Next-hop과 무관하게 동작
! → Hub에서 Route Summarization 가능!
! → 대규모 환경에서 라우팅 테이블 최적화 가능
```

**Phase 3를 선택한 이유**: 지사 50개 환경에서 Hub가 Summarization을 할 수 없으면 모든 Spoke가 50개의 개별 라우트를 갖게 됩니다. Phase 3는 Summarization이 가능하면서도 Spoke-to-Spoke 직접 통신을 지원하는 유일한 Phase입니다.

---

## 면접관: "구성을 완료했는데, Spoke A에서 Spoke B로 통신 시 직접 터널이 안 맺어지고 계속 Hub를 경유합니다. 어디를 확인해야 하나요?"

### 진단 순서

```
! 1단계: NHRP 캐시 확인 (Spoke A에서)
show ip nhrp
show ip nhrp shortcut

! Spoke B의 엔트리가 없거나 "incomplete"이면 NHRP Resolution 실패
```

```
! 2단계: Hub에서 NHRP Redirect가 나가는지 확인
show ip nhrp redirect

debug nhrp
! "Sending NHRP Redirect" 메시지가 보여야 함
```

```
! 3단계: 핵심 명령어 누락 확인
! Hub 측:
show running-config interface Tunnel0 | include nhrp redirect
! "ip nhrp redirect" 가 있어야 함

! Spoke 측:
show running-config interface Tunnel0 | include nhrp shortcut
! "ip nhrp shortcut" 가 있어야 함
```

### 주요 장애 원인 및 해결

| # | 원인 | 확인 방법 | 해결 |
|---|------|-----------|------|
| 1 | Hub에 `ip nhrp redirect` 누락 | `show run int Tunnel0` | 명령어 추가 |
| 2 | Spoke에 `ip nhrp shortcut` 누락 | `show run int Tunnel0` | 명령어 추가 |
| 3 | Spoke가 p2p GRE 모드 | `show int Tunnel0` | `tunnel mode gre multipoint` 변경 |
| 4 | IPSec SA 생성 실패 (Spoke간) | `debug crypto ikev2` | PSK, Proposal 확인 |
| 5 | ISP에서 GRE(Protocol 47) 차단 | Spoke 간 GRE Ping 테스트 | ISP 확인 또는 DMVPN over HTTPS 고려 |
| 6 | NHRP Authentication 불일치 | `debug nhrp` | 양쪽 `ip nhrp authentication` 일치 확인 |
| 7 | Tunnel Key 불일치 | `show int Tunnel0` | `tunnel key` 값 통일 |
| 8 | NAT 뒤의 Spoke | NBMA 주소 확인 | NAT-T 활성화 + NHRP registration 확인 |

### Spoke-to-Spoke 터널 수립 과정 확인

```
! Spoke A에서 Spoke B로 ping 후:
show ip nhrp
! Type이 "dynamic" → 직접 터널 수립 성공
! Type이 없거나 "incomplete" → 실패

show ip nhrp nhs detail
! NHS(Hub) 상태가 "RE" (Registered, Expected replies) 인지 확인

show dmvpn
! Spoke 간 연결 상태, Attrb(Attribute)에 "S"(Spoke)가 보이는지 확인

show dmvpn detail
! NBMA 주소, Tunnel 주소, State 확인
```

---

## 면접관: "NHRP Resolution과 NHRP Redirect의 차이를 좀 더 자세히 설명해주세요."

### NHRP 프로토콜 핵심 메시지

| 메시지 | 방향 | 용도 | 사용되는 Phase |
|--------|------|------|----------------|
| **Registration Request** | Spoke → Hub | Spoke가 자신의 Tunnel IP ↔ NBMA IP 매핑을 Hub에 등록 | 모든 Phase |
| **Registration Reply** | Hub → Spoke | 등록 확인 응답 | 모든 Phase |
| **Resolution Request** | Spoke → Hub (또는 Spoke) | 특정 Tunnel IP에 대한 실제 NBMA IP 조회 | Phase 2, 3 |
| **Resolution Reply** | Hub (또는 Spoke) → Spoke | 조회 결과 응답 | Phase 2, 3 |
| **Redirect** | Hub → Spoke | "나를 거치지 말고 직접 가라" 알림 | **Phase 3만** |
| **Traffic Indication** | Spoke → Spoke | Redirect 받은 후 상대방에게 직접 Resolution 요청의 트리거 | **Phase 3만** |

### Phase 2의 NHRP Resolution 방식

```
1. Spoke A가 Spoke B(10.2.0.0/16)로 패킷 전송
2. 라우팅 테이블에서 Next-hop이 172.16.0.12(Spoke B의 Tunnel IP)
3. Spoke A → Hub: "172.16.0.12의 NBMA(실제 WAN IP)가 뭐야?"
   (NHRP Resolution Request)
4. Hub → Spoke A: "198.51.100.12야" (Resolution Reply)
5. Spoke A → Spoke B 직접 mGRE 터널 생성

! 핵심: Next-hop이 반드시 Spoke B의 Tunnel IP여야 함
! → Hub에서 Summarization하면 Next-hop이 Hub IP가 되어 Resolution 불가
```

### Phase 3의 NHRP Redirect 방식

```
1. Spoke A가 Spoke B(10.2.0.0/16)로 패킷 전송
2. 라우팅 테이블에서 Next-hop이 172.16.0.1(Hub IP) ← Summarized Route
3. 패킷이 Hub에 도착
4. Hub가 "이 패킷은 Spoke B로 가야 하는데, Spoke A가 직접 보내면 되겠네"
   → Hub → Spoke A: NHRP Redirect 전송
   → Hub는 패킷도 Spoke B로 전달 (첫 패킷은 Hub 경유)
5. Spoke A가 Redirect를 받고 → Spoke B에게 직접 Resolution Request
6. Spoke B → Spoke A: Resolution Reply (NBMA IP 알려줌)
7. Spoke A의 CEF 테이블에 Shortcut 엔트리 생성
8. 이후 Spoke A → Spoke B 직접 통신 (Hub 미경유)

! 핵심: 라우팅 Next-hop과 무관하게 NHRP Redirect가 동작
! → Hub에서 Summarization 가능!
```

### CEF Shortcut의 의미

```
! Spoke A에서 확인
show ip cef 10.2.0.0/16
! Shortcut 적용 전: via 172.16.0.1 (Hub)
! Shortcut 적용 후: via 172.16.0.12, Tunnel0 (Spoke B 직접)

show ip nhrp shortcut
! NHRP Shortcut 엔트리 목록 확인
```

---

## 면접관: "DMVPN 위에서 돌릴 라우팅 프로토콜로 EIGRP를 선택하셨는데, BGP를 쓰는 경우도 있나요? 어떤 기준으로 선택하나요?"

### DMVPN 라우팅 프로토콜 비교

| 항목 | EIGRP | OSPF | BGP |
|------|-------|------|-----|
| Spoke 수 | ~100개 | ~50개 | **수백~수천 개** |
| Hub 부하 | 낮음 | 높음 (DR 선출 이슈) | 중간 |
| Summarization | Hub에서 쉽게 가능 | Area 경계에서만 | Hub에서 쉽게 가능 |
| Split Horizon | **주의 필요** | 문제 없음 | iBGP 규칙 주의 |
| 수렴 속도 | 빠름 | 보통 | 느림 (tuning 필요) |
| Multi-vendor | Cisco Only | 표준 | **표준** |
| Phase 3 호환 | 우수 | 주의 필요 | **우수** |
| 대규모 권장 | X | X | **O** |

### EIGRP 사용 시 주의사항

```
! Split Horizon 문제:
! Hub의 Tunnel0은 mGRE (multi-access) 인터페이스
! EIGRP Split Horizon이 기본 활성화 → Hub가 Spoke A에서 받은 경로를 Spoke B에게 전파 안 함

! 해결: Split Horizon 비활성화
interface Tunnel0
 no ip split-horizon eigrp 100

! Next-hop 설정 (Phase 2 필수, Phase 3에서는 선택적)
 no ip next-hop-self eigrp 100
```

### BGP가 적합한 경우

지사가 **100개 이상**이거나, **Multi-vendor 환경**, **복잡한 정책 라우팅**이 필요할 때 BGP를 사용합니다.

```
! ── Hub BGP 설정 ──
router bgp 65000
 bgp log-neighbor-changes
 neighbor DMVPN-SPOKES peer-group
 neighbor DMVPN-SPOKES remote-as 65000
 neighbor DMVPN-SPOKES route-reflector-client
 neighbor DMVPN-SPOKES next-hop-self

 ! Spoke들을 peer-group으로 등록
 neighbor 172.16.0.11 peer-group DMVPN-SPOKES
 neighbor 172.16.0.12 peer-group DMVPN-SPOKES
 ! ... (동적 추가: listen range 활용)

 ! Dynamic Neighbor (Spoke 자동 등록 - 대규모 환경 필수)
 bgp listen range 172.16.0.0/24 peer-group DMVPN-SPOKES

 address-family ipv4
  neighbor DMVPN-SPOKES activate
  neighbor DMVPN-SPOKES route-reflector-client

  ! Hub에서 Summarization
  aggregate-address 10.0.0.0 255.0.0.0 summary-only

! ── Spoke BGP 설정 ──
router bgp 65000
 neighbor 172.16.0.1 remote-as 65000
 address-family ipv4
  neighbor 172.16.0.1 activate
  network 10.2.0.0 mask 255.255.0.0
```

### 선택 기준 정리

| 조건 | 권장 프로토콜 |
|------|--------------|
| Spoke 50개 미만, Cisco only | **EIGRP** |
| Spoke 50~100개, 빠른 수렴 필요 | **EIGRP** |
| Spoke 100개 이상 | **BGP** |
| Multi-vendor 환경 | **BGP** 또는 **OSPF** |
| 복잡한 라우팅 정책 | **BGP** |
| 간단한 Hub-Spoke, 소규모 | **OSPF** 가능 (주의사항 있음) |

---

## 면접관: "마지막으로, Dual Hub 이중화는 어떻게 구성하나요? Hub가 단일 장애점이 되면 안 될 텐데요."

맞습니다. Hub가 SPOF(Single Point of Failure)가 되면 모든 Spoke가 통신 불가능해지므로, **Dual Hub** 구성은 실무에서 필수입니다.

### Dual Hub 구성 방식

```
                   [Hub 1: 203.0.113.1]
                  /    (Primary NHS)    \
[Spoke A] ──────<                        >────── [Spoke B]
                  \    (Backup NHS)     /
                   [Hub 2: 203.0.113.2]
```

### Spoke 설정 (Dual NHS)

```
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
 ip nhrp network-id 1
 ip nhrp authentication DMVPN!Key

 ! Primary NHS
 ip nhrp nhs 172.16.0.1 nbma 203.0.113.1 multicast
 ! Backup NHS
 ip nhrp nhs 172.16.0.2 nbma 203.0.113.2 multicast

 ! NHS Fallback (Primary 복구 시 자동 전환)
 ip nhrp nhs fallback 30

 ip nhrp shortcut
 tunnel source GigabitEthernet0/0/0
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile IPSEC-PROF-DMVPN
```

### 라우팅 이중화 (EIGRP)

```
! Hub 1 (Primary): 더 좋은 metric 광고
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
 delay 100

! Hub 2 (Backup): 더 나쁜 metric 광고
interface Tunnel0
 ip summary-address eigrp 100 10.0.0.0 255.0.0.0
 delay 200

! → Spoke는 평상시 Hub 1로 라우팅
! → Hub 1 장애 시 Hub 2로 자동 Failover
```

### 라우팅 이중화 (BGP)

```
! Hub 1: Higher Local Preference
router bgp 65000
 address-family ipv4
  neighbor DMVPN-SPOKES route-map SET-LP-HIGH out

route-map SET-LP-HIGH permit 10
 set local-preference 200

! Hub 2: Lower Local Preference
route-map SET-LP-LOW permit 10
 set local-preference 100
```

### 장애 감지 및 Failover 확인

```
! Spoke에서 NHS 상태 확인
show ip nhrp nhs detail
! Expected Replies: Hub 1 (RE), Hub 2 (RE)
! Hub 1 장애 시: Hub 1 (E - Expected만, Reply 없음)

show dmvpn
! Hub 1의 State가 "NHRP" → "IKE" → 사라짐 → Hub 2로 전환

! EIGRP Neighbor 확인
show ip eigrp neighbors
! Hub 1이 사라지고 Hub 2만 남는지 확인

! 전체 DMVPN 상태 요약
show dmvpn detail
```

### Dual Hub 설계 고려사항

| 항목 | 설명 |
|------|------|
| Hub 간 통신 | Hub 1 ↔ Hub 2 간 별도 터널 또는 LAN 연결 필요 |
| Stateful Failover | IPSec SA는 Stateful Failover 안 됨 → 터널 재수립 필요 |
| Failover 시간 | NHRP Hold Timer + 라우팅 수렴 시간 (약 30~60초) |
| DNS 기반 | Hub IP를 FQDN으로 설정하면 DNS 기반 Failover 가능 |
| 동일 NHRP Network-ID | 두 Hub가 동일한 NHRP Network에 속해야 함 |
