# 01-03. EIGRP 고급 설정과 마이그레이션

## 면접관: "고객사가 오래된 Cisco 환경이라 EIGRP Classic Mode로 운영 중입니다. 장비 교체 시점에 Named Mode로 마이그레이션하고 싶다고 하는데, 어떻게 접근하시겠어요?"

먼저 Classic Mode와 Named Mode의 차이를 이해하고, 마이그레이션 전략을 세워야 합니다.

**Classic vs Named Mode 주요 차이:**

| 항목 | Classic Mode | Named Mode |
|------|-------------|------------|
| 설정 구조 | 분산 (router eigrp + 인터페이스) | 통합 (address-family 아래 집중) |
| IPv6 지원 | 별도 프로세스 필요 | 동일 구조에서 IPv4/IPv6 통합 |
| VRF | 별도 설정 | address-family에서 통합 관리 |
| Wide Metric | 미지원 (32-bit) | 지원 (64-bit) → 고속 링크 구분 가능 |
| 기능 확장 | 제한적 | Stub leak-map 등 신기능 |

마이그레이션의 핵심은 **Named Mode와 Classic Mode가 호환된다**는 점입니다. 동일 AS Number를 사용하면 두 모드 간 Neighbor가 정상적으로 맺어집니다. 따라서 장비 교체 시 순차적으로 전환이 가능합니다.

> 토폴로지 이미지 추가 예정

### 핵심 설정

**기존 Classic Mode:**
```
router eigrp 100
 network 10.0.0.0 0.0.255.255
 no auto-summary
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
!
interface GigabitEthernet0/0
 ip address 10.1.1.1 255.255.255.252
 ip bandwidth-percent eigrp 100 50
 ip hello-interval eigrp 100 5
 ip hold-time eigrp 100 15
```

**Named Mode 전환 후:**
```
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  network 10.0.0.0 0.0.255.255
  af-interface default
   passive-interface
   hello-interval 5
   hold-time 15
  exit-af-interface
  !
  af-interface GigabitEthernet0/0
   no passive-interface
   bandwidth-percent 50
  exit-af-interface
  !
  af-interface GigabitEthernet0/1
   no passive-interface
  exit-af-interface
  !
  topology base
   no auto-summary
  exit-af-topology
 exit-address-family
```

**마이그레이션 순서:**
1. 새 장비(교체 대상)부터 Named Mode로 설정
2. 기존 장비와 Neighbor 정상 확인
3. 운영 안정화 후, 나머지 장비를 유지보수 시 순차 전환
4. 전체 전환 완료 후 Wide Metric 등 신기능 활성화

---

## 면접관: "EIGRP에서 경로를 선택할 때 DUAL 알고리즘이 동작한다고 하는데, 구체적으로 설명해주세요. Feasibility Condition이 뭔가요?"

DUAL (Diffusing Update Algorithm)은 EIGRP의 경로 선택 및 수렴 알고리즘입니다.

**핵심 용어:**
- **Successor:** 목적지까지의 최적 경로를 가진 Next-hop 라우터
- **Feasible Distance (FD):** 자신에서 목적지까지의 최소 총 비용
- **Reported Distance (RD) = Advertised Distance (AD):** Neighbor가 보고한, 그 Neighbor에서 목적지까지의 비용
- **Feasible Successor (FS):** 백업 경로. Successor가 다운되면 즉시 대체 가능

**Feasibility Condition (FC):**
> Neighbor의 RD < 현재 FD 이면, 그 Neighbor를 Feasible Successor로 선정

이 조건의 의미는 "백업 경로가 루프를 만들지 않음을 보장"하는 것입니다.

예시:
```
목적지: 10.10.0.0/16

경로 A (Successor):
  Next-hop: R2, RD=50, 내 비용=30 → FD = 80

경로 B:
  Next-hop: R3, RD=70, 내 비용=20 → 총=90
  FC 확인: RD(70) < FD(80)? → YES → Feasible Successor!

경로 C:
  Next-hop: R4, RD=85, 내 비용=10 → 총=95
  FC 확인: RD(85) < FD(80)? → NO → FS 아님
```

경로 B는 FS가 되어, R2가 다운되면 **Query를 보내지 않고 즉시** 라우팅 테이블에 올립니다. 이것이 EIGRP의 빠른 수렴 핵심입니다.

경로 C는 FS가 아니므로, 만약 Successor와 FS 모두 없어지면 DUAL이 **Active 상태**로 전환되어 Neighbor들에게 Query를 보냅니다.

```
! DUAL 상태 확인
show ip eigrp topology
show ip eigrp topology 10.10.0.0/16
show ip eigrp topology all-links   ! FC 미충족 경로 포함 전체 보기
```

---

## 면접관: "고객사에서 '본사에서 데이터센터로 가는 경로가 2개 있는데, 한쪽이 1G 다른 쪽이 500M입니다. 두 경로를 대역폭 비율대로 부하분산해달라'고 요청합니다. EIGRP에서 어떻게 하죠?"

이것은 **Unequal Cost Load Balancing**으로, EIGRP만이 기본 지원하는 기능입니다. `variance` 명령을 사용합니다.

**전제 조건:**
- 부하분산할 경로가 반드시 **Feasible Successor**여야 함 (FC 충족 필수)
- variance 값은 FD의 배수를 의미

**설계:**
```
경로 1 (1G 링크): FD = 2816   ← Successor
경로 2 (500M 링크): FD = 5120  ← RD가 FD(2816)보다 작으면 FS

variance 계산: 5120 / 2816 = 1.82 → variance 2 (올림)
```

### 핵심 설정
```
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  topology base
   variance 2
   traffic-share balanced
  exit-af-topology
```

Classic Mode에서는:
```
router eigrp 100
 variance 2
 traffic-share balanced
```

**동작 확인:**
```
show ip eigrp topology
  P 10.20.0.0/16, 1 successors, FD is 2816
    via 10.1.1.2 (2816/256), GigabitEthernet0/0     ← Successor
    via 10.1.2.2 (5120/512), GigabitEthernet0/1     ← Feasible Successor

show ip route 10.20.0.0
  Known via "eigrp 100", distance 90, metric 2816
    * 10.1.1.2, via GigabitEthernet0/0     ← 더 많은 트래픽
    * 10.1.2.2, via GigabitEthernet0/1     ← 비율대로 적은 트래픽
```

`traffic-share balanced`는 metric에 반비례하여 트래픽을 분배합니다. CEF가 활성화되어 있으면 per-destination 기반으로 분산됩니다.

**주의:** `traffic-share min across-interfaces`로 설정하면, 실제 트래픽은 Successor로만 보내되 두 경로 모두 라우팅 테이블에 설치합니다 (failover 목적).

---

## 면접관: "variance를 설정했는데, 고객사에서 '특정 지사에서 본사 연결이 간헐적으로 끊긴다, 콘솔에 SIA라는 메시지가 보인다'고 합니다. 이게 뭐고, 어떻게 해결하죠?"

**SIA (Stuck-In-Active)**는 EIGRP에서 가장 심각한 장애 상황 중 하나입니다.

**발생 과정:**
1. Successor와 FS가 모두 없어져서 경로가 **Active** 상태로 전환
2. 라우터가 모든 Neighbor에게 Query를 전송
3. Query를 받은 Neighbor도 자신의 FS가 없으면 다시 자신의 Neighbor에게 Query 전파 (Query Propagation)
4. **3분(기본 Active Timer) 내에 Reply를 받지 못하면 SIA 발생**
5. SIA 상태의 Neighbor와 Adjacency를 강제로 끊어버림

**SIA 발생 원인:**
- 대규모 네트워크에서 Query가 끝없이 전파 (Query Scope 미제한)
- Neighbor가 CPU 과부하로 Reply를 처리하지 못함
- 중간 경로의 패킷 유실로 Query/Reply 손실
- 단방향 링크 문제

**해결 방법:**

**1. Query Scope 제한 - EIGRP Stub 설정**
```
! 지사 라우터 (리프 노드)
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary
```
Stub 라우터는 Query를 받아도 자기 뒤에 대안 경로가 없다고 즉시 Reply합니다. 지사 라우터를 Stub으로 설정하면 Query 전파를 대폭 줄일 수 있습니다.

**2. Summary Route로 Query 경계 설정**
```
! 허브 라우터에서 각 지역으로 요약 경로 광고
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/1
   summary-address 10.10.0.0/16
  exit-af-interface
```
Summary Route가 있으면, 해당 범위의 세부 경로에 대한 Query가 Summary 경계에서 멈춥니다.

**3. Active Timer 조정 (임시 조치)**
```
! 기본 3분 → 필요시 조정
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  topology base
   timers active-time 5    ! 5분 (분 단위)
  exit-af-topology
```

**확인 명령어:**
```
show ip eigrp topology active
show ip eigrp topology | include Active
show logging | include SIA
show ip eigrp neighbors     ! SIA로 끊어진 neighbor 확인
```

---

## 면접관: "EIGRP Stub에서 connected, summary 외에 다른 옵션들은 뭐가 있죠? 그리고 Named Mode에서 추가된 leak-map은 뭔가요?"

**EIGRP Stub 옵션:**

| 옵션 | 광고하는 경로 |
|------|-------------|
| `connected` | 직접 연결된 네트워크 |
| `summary` | Summary Route |
| `static` | 재분배된 Static 경로 |
| `redistributed` | 모든 재분배 경로 |
| `receive-only` | 아무것도 광고 안 함 (수신만) |

기본값: `eigrp stub connected summary`

**leak-map (Named Mode 전용):**

Stub 라우터는 기본적으로 Query에 대해 "경로 없음"으로 즉시 Reply합니다. 하지만 특정 경로만 선택적으로 Query에 응답하고 싶을 때 leak-map을 사용합니다.

예를 들어, 지사 A가 Stub이지만 지사 B로 가는 백업 경로를 가지고 있고, 이 경로를 허브에 알려주고 싶은 경우:

```
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  eigrp stub connected summary leak-map ALLOW-BACKUP
  !
  topology base
  exit-af-topology
 exit-address-family
!
route-map ALLOW-BACKUP permit 10
 match ip address prefix-list BRANCH-B-NETS
!
ip prefix-list BRANCH-B-NETS permit 10.20.0.0/16
```

이렇게 하면 Stub의 이점(Query 제한)은 유지하면서, 특정 경로만 선택적으로 광고할 수 있습니다.

---

## 면접관: "고객사가 '이번 장비 교체 때 일부 지사를 OSPF로 바꾸고 싶다'고 합니다. EIGRP에서 OSPF로 점진적 마이그레이션은 어떻게 하시겠어요?"

전면 교체가 아닌 점진적 마이그레이션이므로, **양방향 재분배(Mutual Redistribution)**를 통한 전환 기간을 두어야 합니다.

**마이그레이션 전략:**

```
Phase 1: 경계 라우터(ABR 역할)에서 EIGRP ↔ OSPF 양방향 재분배 설정
Phase 2: 신규 지사부터 OSPF로 구성
Phase 3: 기존 지사를 순차적으로 OSPF로 전환
Phase 4: 모든 지사 OSPF 전환 완료 후 재분배 제거, EIGRP 종료
```

### 경계 라우터 핵심 설정
```
! EIGRP → OSPF 재분배
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
!
! OSPF → EIGRP 재분배
router eigrp CORPORATE
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 route-map OSPF-TO-EIGRP
  exit-af-topology
!
! 루프 방지를 위한 Route-map + Tag
route-map EIGRP-TO-OSPF permit 10
 match tag 0            ! Tag 없는(원본 EIGRP) 경로만 재분배
 set tag 110            ! OSPF로 갈 때 Tag 110 부여
!
route-map OSPF-TO-EIGRP permit 10
 match tag 0            ! Tag 없는(원본 OSPF) 경로만 재분배
 set tag 90             ! EIGRP로 갈 때 Tag 90 부여
```

**주의사항:**
- 재분배 시 Seed Metric 설정 필수 (EIGRP default metric, OSPF metric)
- 경계 라우터가 2대 이상이면 라우팅 루프 위험 → Tag 기반 필터링 필수
- AD 값 차이 (EIGRP Internal 90, OSPF 110, EIGRP External 170) 주의
- 마이그레이션 기간 중 모니터링 강화

이 내용은 `01-06-route-redistribution.md`에서 더 심층적으로 다룹니다.

---

## 면접관: "EIGRP의 Wide Metric에 대해 간단히 설명해주세요. Named Mode에서만 된다고 했는데, 왜 필요한 거죠?"

Classic EIGRP의 Composite Metric은 **32-bit** 값입니다. 계산 공식:

```
Metric = 256 * [(K1*BW) + (K2*BW)/(256-Load) + (K3*Delay)] * [K5/(Reliability+K4)]
기본 K값: K1=1, K2=0, K3=1, K4=0, K5=0
→ 실질적으로: Metric = 256 * (BW + Delay)
  BW = 10^7 / 최소대역폭(kbps)
  Delay = 인터페이스 Delay 합(10μs 단위)
```

문제는 **고속 링크 구분 불가**입니다:
- 10G: BW = 10^7 / 10,000,000 = 1
- 40G: BW = 10^7 / 40,000,000 = 0 (소수점 이하 절삭!)
- 100G: 마찬가지로 0

**Wide Metric (64-bit):**
Named Mode에서 지원하며, `rib-scale`을 사용해서 64-bit 내부 계산 값을 32-bit RIB에 맞게 축소합니다.

```
Wide BW = 10^7 * 65536 / BW(kbps)
Wide Delay = Delay(picoseconds) * 65536 / 10^6
Wide Metric = (K1*BW + K2*BW/(256-Load) + K3*Delay + K6*ExtAttr) * [K5/(K4+Reliability)]
```

K6는 Named Mode에서 추가된 새로운 K값으로, Extended Attributes(Jitter, Energy 등)를 반영할 수 있습니다.

```
! Wide Metric 확인
show eigrp address-family ipv4 topology
show eigrp address-family ipv4 neighbors detail
!
! Metric 계산 방식 확인
show eigrp protocols
```

핵심: Named Mode 환경에서도 Classic Mode Neighbor와 혼용 시, 자동으로 32-bit로 변환하여 호환성을 유지합니다. 따라서 점진적 마이그레이션이 가능합니다.
