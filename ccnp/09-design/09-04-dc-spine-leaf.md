# 09-04. DC 네트워크 설계 (Spine-Leaf)

---

### 면접관: "고객사가 클라우드 서비스를 제공하기 위해 데이터센터를 신축합니다. 서버 랙 200개 규모이고, VM 간 통신이 대부분이에요. 전통적인 3-Tier DC 구조 대신 다른 아키텍처를 제안해주세요."

**Spine-Leaf (Clos Fabric)** 아키텍처를 제안하겠습니다.

#### 기존 3-Tier DC의 한계

```
전통 3-Tier DC:
[Core] ─── [Aggregation] ─── [Access(ToR)]
```

- STP(Spanning Tree) 기반 → 이중화 경로 중 하나는 블로킹(Active-Standby)
- **대역폭 낭비**: 50%의 링크가 STP에 의해 비활성
- 서버 간 통신(East-West)이 Aggregation/Core까지 올라갔다 내려와야 함 → **Hairpinning**
- 랙 추가 시 Aggregation 계층 포트 부족 → 확장에 한계

#### Spine-Leaf 설계

```
        [Spine-1]   [Spine-2]   [Spine-3]   [Spine-4]
          /|\          /|\         /|\          /|\
         / | \        / | \       / | \        / | \
        /  |  \      /  |  \     /  |  \      /  |  \
  [Leaf-1] [Leaf-2] [Leaf-3] [Leaf-4] ... [Leaf-N]
     |        |        |        |              |
  [Rack1]  [Rack2]  [Rack3]  [Rack4]  ...  [Rack-N]
```

**핵심 원칙**:
- 모든 Leaf는 **모든 Spine에 연결** (Full Mesh)
- Leaf 간 직접 연결 없음, Spine 간 직접 연결 없음
- 어떤 서버 간 통신이든 **최대 2홉** (Leaf → Spine → Leaf)
- STP 사용하지 않음 → **모든 링크가 Active** (ECMP)

#### 장비 선정 및 수량

| 역할 | 장비 예시 | 수량 | 포트 |
|------|-----------|------|------|
| Spine | Nexus 9364C (64x 100G QSFP28) | 4대 | 각 Spine에서 모든 Leaf로 100G 연결 |
| Leaf (ToR) | Nexus 93180YC-FX (48x 10/25G + 6x 100G) | 50대 (랙 4개당 1대) | 다운링크 25G(서버), 업링크 100G(Spine) |
| Border Leaf | Nexus 9364C | 2대 | 외부(WAN/Internet) 연결 전용 |

#### 설계 근거

- **200 랙**: Leaf 스위치 약 50대 (ToR 모델, 랙 4개당 Leaf 1대)
- **Spine 4대**: 각 Spine은 64포트 100G → 50개 Leaf 연결 후 여유 포트 14개 (확장용)
- **ECMP**: 서버 A→B 통신 시 4개 Spine을 통해 4개의 Equal-Cost 경로 → 대역폭 4배 활용

> 토폴로지 이미지 추가 예정

---

### 면접관: "기존 3-Tier DC 대비 Spine-Leaf의 장점을 좀 더 구체적으로 설명해주세요. 특히 성능 측면에서요."

#### 1. 일관된 지연시간 (Predictable Latency)

**3-Tier**: 같은 Aggregation 아래 서버 → 2홉, 다른 Aggregation → 4~6홉
- 서버 위치에 따라 지연시간이 **비균일(non-uniform)**

**Spine-Leaf**: 어디든 최대 **2홉** (같은 Leaf 내부 제외)
- 지연시간이 **균일(uniform)**, 약 2~5us (마이크로초)

#### 2. STP 없는 Active-Active 링크 (ECMP)

**3-Tier**: STP가 루프 방지를 위해 절반의 링크를 차단
- 총 대역폭의 **50%만 사용 가능**

**Spine-Leaf**: L3 라우팅(BGP/OSPF) 기반 → ECMP로 **모든 링크 Active**
- 총 대역폭의 **100% 사용 가능**

```
예시: Leaf에서 Spine으로 100G x 4 업링크
3-Tier (STP): 100G만 Active → 100Gbps
Spine-Leaf (ECMP): 100G x 4 Active → 400Gbps
```

#### 3. 수평 확장 (Horizontal Scale-out)

**3-Tier**: 랙 추가 → Aggregation 포트 부족 → 상위 계층 장비 교체 필요

**Spine-Leaf**:
- 랙 추가 → **Leaf 추가 후 모든 Spine에 연결**하면 끝
- Spine 포트가 부족하면 → **Spine 추가**하면 끝 (기존 Leaf에 업링크 추가)

#### 4. East-West 트래픽 최적화

현대 DC 트래픽의 **80% 이상이 East-West** (서버 간 통신)입니다.

```
트래픽 패턴 변화:
과거: Client → Server (North-South 80%, East-West 20%)
현재: VM ↔ VM, Container ↔ Container (East-West 80%, North-South 20%)
```

3-Tier에서 East-West 트래픽은 Core까지 올라갔다 내려옵니다. Spine-Leaf에서는 Spine 하나만 경유하므로 **지연시간 절반 이하, 대역폭 활용도 극대화**됩니다.

---

### 면접관: "Spine-Leaf에서 VXLAN과 BGP EVPN을 사용한다고 했는데, 왜 필요하고 어떻게 동작하나요?"

#### 문제: L2 도메인 확장

Spine-Leaf는 **L3 라우팅 기반**(모든 Leaf↔Spine이 L3)입니다. 그런데 VM Live Migration, 클러스터링 등의 이유로 **서로 다른 Leaf에 있는 서버가 같은 L2 서브넷에 있어야** 하는 경우가 있습니다.

전통적으로 이를 위해 Leaf 간 VLAN을 확장(L2 stretch)했지만, 이러면 STP 문제가 다시 발생합니다.

#### 해결: VXLAN (Virtual Extensible LAN)

**VXLAN은 L2 프레임을 L3 패킷으로 캡슐화(encapsulation)**하는 오버레이 기술입니다.

```
[Original L2 Frame]
  → VXLAN 캡슐화 →
[Outer IP Header | Outer UDP Header | VXLAN Header | Original L2 Frame]
```

- **VNI (VXLAN Network Identifier)**: 24비트 → 약 1,600만 개의 논리 네트워크 (VLAN의 4,096개 한계 극복)
- **VTEP (VXLAN Tunnel Endpoint)**: 각 Leaf 스위치가 VTEP 역할, 캡슐화/디캡슐화 수행
- **Underlay**: Spine-Leaf의 L3 네트워크 (BGP 또는 OSPF)
- **Overlay**: VXLAN으로 생성된 논리적 L2 네트워크

#### BGP EVPN (Ethernet VPN)

VXLAN의 **Control Plane** 역할을 합니다. "어떤 MAC/IP가 어떤 VTEP 뒤에 있는지" 정보를 분배합니다.

초기 VXLAN은 **Flood-and-Learn** (BUM 트래픽 멀티캐스트) 방식이었는데, 이는 비효율적입니다. BGP EVPN은 이를 대체하여 **MAC/IP 학습을 BGP로 제어**합니다.

```
! Leaf-1 VXLAN + BGP EVPN 설정 예시 (NX-OS)
feature bgp
feature nv overlay
feature vn-segment-vlan-based
nv overlay evpn

! VXLAN VTEP 설정
interface nve1
 no shutdown
 host-reachability protocol bgp
 source-interface loopback0
 member vni 10100
  ingress-replication protocol bgp

! VLAN과 VNI 매핑
vlan 100
 vn-segment 10100

! BGP EVPN 설정
router bgp 65001
 address-family l2vpn evpn
 neighbor 10.255.0.1
  remote-as 65000
  address-family l2vpn evpn
   send-community both

! EVPN 인스턴스
evpn
 vni 10100 l2
  rd auto
  route-target import auto
  route-target export auto
```

#### 동작 흐름 요약

1. Server-A (Leaf-1, VLAN 100) → Server-B (Leaf-3, VLAN 100) 통신
2. Leaf-1은 BGP EVPN을 통해 Server-B의 MAC이 Leaf-3 뒤에 있음을 알고 있음
3. Leaf-1(VTEP)이 L2 프레임을 VXLAN 캡슐화 (VNI 10100, Dst IP: Leaf-3 VTEP)
4. Underlay L3 네트워크를 통해 Spine 경유하여 Leaf-3에 도달
5. Leaf-3(VTEP)이 VXLAN 디캡슐화 → Server-B에 원본 L2 프레임 전달

**결과**: 서로 다른 Leaf에 있는 서버가 **마치 같은 L2 스위치에 연결된 것처럼** 통신, 하지만 **Underlay는 완전한 L3 라우팅**(STP 없음, ECMP 활용)

---

### 면접관: "Spine 스위치 1대가 장애가 나면 어떤 영향이 있나요?"

#### 장애 시나리오: Spine-2 Down

```
        [Spine-1]   [Spine-2 X]   [Spine-3]   [Spine-4]
          /|\                        /|\          /|\
         / | \                      / | \        / | \
  [Leaf-1] [Leaf-2] [Leaf-3] [Leaf-4] ... [Leaf-N]
```

#### Failover 동작

1. **BGP 세션 감지**: 각 Leaf의 BGP 세션이 Spine-2와의 세션 Down 감지
   - BGP Hold Timer 기본 180초 → BFD(Bidirectional Forwarding Detection) 사용 시 **50ms~1초 이내** 감지

```
! BFD 설정으로 빠른 장애 감지
router bgp 65001
 neighbor 10.255.0.2
  bfd
!
interface Ethernet1/2
 bfd interval 250 min_rx 250 multiplier 3
```

2. **ECMP 경로 업데이트**: Spine-2를 경유하는 경로가 라우팅 테이블에서 제거
   - 기존 4개 ECMP 경로 → **3개로 줄어듦**

3. **트래픽 재분배**: Spine-2로 가던 트래픽이 나머지 3개 Spine으로 자동 분산

#### 영향 분석

| 항목 | 영향 |
|------|------|
| 서비스 중단 | **없음** (BFD 사용 시 1초 이내 Failover) |
| 대역폭 | **25% 감소** (4 Spine → 3 Spine) |
| 지연시간 | 변화 없음 (여전히 2홉) |
| 오버서브스크립션 | 증가 (Spine 1대 부족분) |

#### 설계 시 고려사항

- **Spine N+1 설계**: 정상 상태에서 Spine 1대가 죽어도 트래픽을 감당할 수 있도록 여유 대역폭 확보
- 예: 필요 대역폭이 300Gbps면, Spine 3대(300G)가 아닌 **4대(400G)**로 설계
- Spine 장애 시 300/400 = 75% 사용률 → Spine 1대 Down 시 300/300 = 100% → 위험
- **올바른 설계**: Spine을 5대로 → 1대 Down 시에도 300/400 = 75% 유지

#### Leaf 장애 vs Spine 장애

| 장애 위치 | 영향 범위 |
|-----------|-----------|
| Leaf 1대 장애 | **해당 Leaf에 연결된 랙(서버)만 영향** |
| Spine 1대 장애 | **전체 Fabric의 대역폭 감소, 서비스 중단은 없음** |
| Spine 전부 장애 | **전체 Fabric 다운** (발생 확률 극히 낮음) |

---

### 면접관: "East-West 트래픽과 North-South 트래픽에 대해 좀 더 설명해주세요. 각각 어떻게 처리되나요?"

#### 트래픽 패턴 정의

```
                    ┌──── Internet/WAN ────┐
                    │                      │
                    ▼  North-South         ▲
              [Border Leaf]          [Border Leaf]
                    │                      │
        ┌───────── Spine Fabric ──────────┐
        │                                  │
    [Leaf-1]                          [Leaf-3]
       │          East-West              │
    [Server-A] ◄──────────────────► [Server-B]
```

#### East-West (서버 간 / 수평)

- DC 내부 서버 ↔ 서버 통신
- 예: 웹서버 → 앱서버, 앱서버 → DB서버, VM 간 통신
- **경로**: Leaf → Spine → Leaf (2홉)
- 현대 DC 트래픽의 **70~80%**

#### VXLAN 환경에서 East-West 처리

**같은 VNI (L2 통신)**:
```
Server-A (Leaf-1, VNI 10100) → Server-B (Leaf-3, VNI 10100)
→ Leaf-1 VTEP 캡슐화 → Spine 경유 → Leaf-3 VTEP 디캡슐화 → Server-B
```

**다른 VNI (L3 통신, Inter-VXLAN Routing)**:
```
Server-A (VNI 10100) → Server-C (VNI 10200)
→ Leaf-1에서 Distributed Anycast Gateway로 L3 라우팅
→ VXLAN 캡슐화(VNI 10200) → Spine 경유 → Leaf-3 디캡슐화
```

```
! Distributed Anycast Gateway 설정
! 모든 Leaf에서 동일한 SVI IP + 동일한 MAC
fabric forwarding anycast-gateway-mac 0000.1111.2222

interface Vlan100
 vrf member TENANT-A
 ip address 10.100.1.1/24
 fabric forwarding mode anycast-gateway

interface Vlan200
 vrf member TENANT-A
 ip address 10.100.2.1/24
 fabric forwarding mode anycast-gateway
```

**Distributed Anycast Gateway**: 모든 Leaf가 동일한 게이트웨이 IP/MAC을 가짐. 서버가 어느 Leaf에 있든 **로컬에서 L3 라우팅** 가능. Hair-pinning이 발생하지 않음.

#### North-South (외부 ↔ DC / 수직)

- 외부(Internet, WAN) ↔ DC 내부 서버 통신
- 예: 사용자가 인터넷에서 DC 웹서버 접속
- **경로**: Internet → Border Leaf → Spine → Leaf → Server
- DC 트래픽의 **20~30%**

```
! Border Leaf에서 외부 라우팅
router bgp 65000
 address-family ipv4 unicast
 neighbor 203.0.113.1
  remote-as 64999
  description ISP-PEERING
  address-family ipv4 unicast
```

---

### 면접관: "좋습니다. 고객사가 사업이 잘 되어서 서버 랙을 100개 더 추가해야 한다고 합니다. 현재 200랙에서 300랙으로요. Spine-Leaf에서 어떻게 확장하나요?"

#### 확장 분석

**현재 상태**:
- Spine 4대 (각 64포트 100G)
- Leaf 50대 (랙 4개당 1대)
- 각 Leaf에서 Spine으로 100G x 4 업링크

**확장 목표**: 랙 100개 추가 → Leaf 25대 추가 → 총 Leaf 75대

#### 문제: Spine 포트 부족

- Spine 1대당 64포트 → 현재 50포트 사용 → 여유 14포트
- Leaf 25대 추가 → 14포트 부족 → **Spine 포트가 모자람**

#### 해결 방안

#### Option 1: Spine 스위치 추가 (Scale-out)

```
기존: Spine 4대 x 64포트 = 256포트 (50 Leaf 사용)
확장: Spine 6대 x 64포트 = 384포트 (75 Leaf 사용)
```

- Spine 2대 추가 구매
- 기존 50개 Leaf에도 새 Spine 2대로의 업링크 추가 (Leaf당 업링크 4→6)
- **장점**: ECMP 경로 증가 (4→6), 대역폭 50% 향상
- **단점**: Leaf의 업링크 포트가 충분한지 확인 필요 (93180YC-FX는 6x 100G 업링크)

```
! 기존 Leaf에 새 Spine 업링크 추가
interface Ethernet1/53
 description TO-SPINE-5
 no switchport
 mtu 9216
 ip address 10.0.5.1/31
 no shutdown
!
interface Ethernet1/54
 description TO-SPINE-6
 no switchport
 mtu 9216
 ip address 10.0.6.1/31
 no shutdown
!
router bgp 65001
 neighbor 10.0.5.0 remote-as 65000
 neighbor 10.0.6.0 remote-as 65000
```

#### Option 2: 상위 포트 밀도 Spine으로 교체 (Scale-up)

```
기존: Spine 4대 x 64포트 100G
교체: Spine 4대 x 128포트 100G (Nexus 9508 등 모듈형)
```

- **단점**: 기존 Spine 교체 필요 → 서비스 영향, 비용 높음
- Spine-Leaf의 장점인 Scale-out과 상반되므로 **비권장**

#### Option 3: Super-Spine (5-Stage Clos)

매우 대규모(Leaf 100대+)일 때, **Spine 위에 Super-Spine 계층**을 추가합니다.

```
                [Super-Spine-1] [Super-Spine-2]
                   /      \         /      \
          [Spine-A1][Spine-A2] [Spine-B1][Spine-B2]
            / | \     / | \     / | \     / | \
        [Leaf Pod-A]       [Leaf Pod-B]
```

- Leaf를 Pod 단위로 그룹화
- Pod 내부: Leaf ↔ Spine (기존 구조)
- Pod 간: Spine ↔ Super-Spine
- **사용 시나리오**: 수천 대 서버, 하이퍼스케일 DC

#### 이 프로젝트 권장

300랙(Leaf 75대) 규모면 **Option 1 (Spine 추가)**이 가장 적합합니다.
- Spine 6대 구성: 각 64포트 → 75 Leaf 연결 후 여유 약 50% (향후 추가 확장 가능)
- 비용 합리적, 기존 인프라 변경 최소

---

### 면접관: "ECMP 설계에 대해 좀 더 설명해주세요. 해시 불균형 문제는 어떻게 해결하나요?"

#### ECMP (Equal-Cost Multi-Path) 개요

동일 목적지로의 경로가 여러 개일 때, **비용(cost)이 같은 경로들에 트래픽을 분산**하는 기술입니다.

```
Leaf-1에서 Leaf-3으로의 경로:
Path 1: Leaf-1 → Spine-1 → Leaf-3 (cost 2)
Path 2: Leaf-1 → Spine-2 → Leaf-3 (cost 2)
Path 3: Leaf-1 → Spine-3 → Leaf-3 (cost 2)
Path 4: Leaf-1 → Spine-4 → Leaf-3 (cost 2)
→ 4개 경로 모두 cost 동일 → ECMP로 분산
```

#### BGP ECMP 설정

```
! BGP에서 ECMP 활성화 (NX-OS)
router bgp 65001
 address-family ipv4 unicast
  maximum-paths 16        ! 최대 16개 ECMP 경로
 address-family l2vpn evpn
  maximum-paths 16
```

#### 해시 알고리즘

ECMP는 **흐름(flow) 단위**로 경로를 결정합니다. 패킷 단위가 아닌 이유는 **패킷 순서 보장(in-order delivery)** 때문입니다.

```
해시 입력값 (5-Tuple):
- Source IP
- Destination IP
- Source Port
- Destination Port
- Protocol

Hash(5-Tuple) mod N = 경로 번호 (N = ECMP 경로 수)
```

#### 해시 불균형 (Hash Polarization) 문제

#### 원인

1. **동일 해시 알고리즘 + 동일 입력**: Leaf와 Spine이 같은 해시 알고리즘을 사용하면, Leaf에서 Spine-1로 간 트래픽이 Spine-1에서도 같은 Leaf로 몰림
2. **Elephant Flow**: 하나의 대용량 흐름(예: 백업, 복제)이 특정 경로에 집중
3. **비대칭 트래픽 패턴**: 특정 서버 쌍 간 트래픽이 지배적

#### 해결 방안

**1. 해시 시드 다양화 (Hash Seed Randomization)**

```
! 각 스위치에서 다른 해시 시드 사용 (NX-OS)
hardware access-list tcam region racl 512
ip load-sharing address source-destination port source-destination
!
! 일부 플랫폼에서 해시 시드 변경
hardware ecmp hash-seed 12345
```

**2. Inner Header 기반 해시 (VXLAN 환경)**

VXLAN 캡슐화 시 Outer Header가 동일해질 수 있으므로, **Inner Header(원본 패킷)의 5-Tuple**로 해시합니다.

```
! VXLAN Inner Header 해시 활성화
hardware ecmp hash-offset 0
```

또한 VXLAN의 Outer UDP Source Port를 **Inner 5-Tuple의 해시값으로 설정**하여 Spine에서도 고르게 분산되도록 합니다. 이것이 VXLAN RFC에서 권장하는 **엔트로피(entropy) 기법**입니다.

**3. Flowlet Switching**

- 같은 플로우 내에서도 패킷 간 간격(gap)이 일정 이상이면 **다른 경로로 전환**
- Elephant Flow를 여러 경로로 분산
- 패킷 순서가 바뀌지 않도록 gap threshold를 RTT보다 크게 설정

```
! Flowlet Switching 설정 (일부 플랫폼)
hardware profile latency-monitor threshold 200
```

**4. 모니터링 및 확인**

```
! ECMP 경로 확인
show ip route 10.100.1.0/24
  10.100.1.0/24, ubest/mbest: 4/0
    *via 10.0.1.0, Eth1/49, [20/0], 1d02h, bgp-65001, external, tag 65000
    *via 10.0.2.0, Eth1/50, [20/0], 1d02h, bgp-65001, external, tag 65000
    *via 10.0.3.0, Eth1/51, [20/0], 1d02h, bgp-65001, external, tag 65000
    *via 10.0.4.0, Eth1/52, [20/0], 1d02h, bgp-65001, external, tag 65000

! 인터페이스별 트래픽 분포 확인
show interface Eth1/49 | include rate
show interface Eth1/50 | include rate
show interface Eth1/51 | include rate
show interface Eth1/52 | include rate
```

4개 업링크의 트래픽 분포가 **25% +/- 5%** 이내면 정상, 특정 링크에 40% 이상 집중되면 해시 불균형을 의심하고 위 방안을 적용합니다.
