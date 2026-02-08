# 01-06. Route Redistribution

## 면접관: "고객사가 다른 회사를 인수합병했는데, 기존 본사는 OSPF, 인수한 쪽은 EIGRP를 사용합니다. 두 네트워크를 연결해야 하는데, 어떻게 설계하시겠어요?"

인수합병 시나리오에서 라우팅 프로토콜이 다른 경우, **Route Redistribution(경로 재분배)**이 필수입니다. 장기적으로는 하나의 프로토콜로 통합하는 것이 목표지만, 과도기에는 양방향 재분배가 필요합니다.

**설계 방향:**
- 경계 라우터(ASBR) 2대에서 OSPF ↔ EIGRP 양방향 재분배
- 경계 라우터 2대인 이유: 이중화. 1대만 하면 SPOF
- 그러나 경계 라우터 2대 이상에서 양방향 재분배를 하면 **라우팅 루프** 위험 → 루프 방지 메커니즘 필수
- Route-map + Tag를 이용한 루프 방지 설계

**네트워크 구조:**
```
[OSPF Domain]                    [EIGRP Domain]
  본사 Core ─── ASBR-1 ─── 인수 회사 Core
               ASBR-2
```

> 토폴로지 이미지 추가 예정

### 핵심 설정

**ASBR-1 설정:**
```
! EIGRP → OSPF 재분배
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
!
! OSPF → EIGRP 재분배
router eigrp 100
 redistribute ospf 1 route-map OSPF-TO-EIGRP
 default-metric 10000 100 255 1 1500
!
! 루프 방지 Route-map
route-map EIGRP-TO-OSPF deny 10
 match tag 110
!
route-map EIGRP-TO-OSPF permit 20
 set tag 90
 set metric 100
 set metric-type type-1
!
route-map OSPF-TO-EIGRP deny 10
 match tag 90
!
route-map OSPF-TO-EIGRP permit 20
 set tag 110
```

**ASBR-2 설정 (동일 로직):**
```
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
!
router eigrp 100
 redistribute ospf 1 route-map OSPF-TO-EIGRP
 default-metric 10000 100 255 1 1500
!
route-map EIGRP-TO-OSPF deny 10
 match tag 110
!
route-map EIGRP-TO-OSPF permit 20
 set tag 90
 set metric 100
 set metric-type type-1
!
route-map OSPF-TO-EIGRP deny 10
 match tag 90
!
route-map OSPF-TO-EIGRP permit 20
 set tag 110
```

---

## 면접관: "Seed Metric이라는 걸 설정했는데, 이게 뭐죠? 왜 필요하고, 각 프로토콜별로 어떻게 다른가요?"

**Seed Metric**은 재분배할 때 목적지 프로토콜에서 사용할 초기 Metric 값입니다. 재분배되는 경로는 원래 프로토콜의 metric 체계와 목적지 프로토콜의 metric 체계가 완전히 다르기 때문에, 변환할 수 없습니다. 그래서 새로운 metric을 부여해야 합니다.

**프로토콜별 기본 Seed Metric 동작:**

| 재분배 방향 | Seed Metric 미설정 시 | 결과 |
|------------|---------------------|------|
| 어떤 프로토콜 → **OSPF** | metric 20 (Type-2) | 경로 설치됨 (기본값 있음) |
| 어떤 프로토콜 → **EIGRP** | metric 0 (infinity) | **경로 설치 안 됨!** |
| 어떤 프로토콜 → **BGP** | 원래 프로토콜의 MED 유지 | 경로 설치됨 |
| 어떤 프로토콜 → **RIP** | metric 0 (infinity) | **경로 설치 안 됨!** |

**EIGRP Seed Metric이 반드시 필요한 이유:**
EIGRP의 Composite Metric은 Bandwidth, Delay, Reliability, Load, MTU 5개 요소로 구성됩니다. 이 5가지를 모두 지정해야 유효한 metric이 됩니다.

```
! default-metric 명령의 5개 파라미터
router eigrp 100
 default-metric <bandwidth> <delay> <reliability> <load> <mtu>
 ! 예: default-metric 10000 100 255 1 1500
 !     BW=10000 kbps, Delay=100 (10us 단위), Rel=255(최대), Load=1(최소), MTU=1500
```

**OSPF Seed Metric:**
```
! OSPF는 기본 metric 20이지만, 명시적 설정 권장
router ospf 1
 redistribute eigrp 100 metric 100 subnets metric-type type-1
```

- **Type-1 (E1):** Seed Metric + 내부 OSPF Cost 누적 → 경로까지의 실제 거리 반영
- **Type-2 (E2):** Seed Metric 고정, 내부 Cost 무시 → 기본값, 단순하지만 차선 경로 구분 불가

실무에서는 재분배 경로가 많고 ASBR이 여러 대일 때 **Type-1**을 사용해야 최적 경로를 정확히 선택합니다.

---

## 면접관: "양방향 재분배를 했더니 고객사에서 '특정 네트워크로 가는 경로가 이상하다, 핑은 되는데 traceroute 하면 비효율적인 경로를 타고 있다'고 합니다. 라우팅 루프가 의심되는데, 어디부터 확인하시겠어요?"

양방향 재분배에서 가장 흔한 문제가 **라우팅 루프** 또는 **서브옵티멀 라우팅**입니다.

**문제 발생 원리:**
```
1. OSPF 경로 10.1.0.0/16이 ASBR-1에서 EIGRP로 재분배됨 (Tag 110)
2. EIGRP 도메인을 거쳐 ASBR-2에 도달
3. ASBR-2에서 이 경로를 다시 OSPF로 재분배하면 → 루프!
   (OSPF 원래 경로와 재분배된 경로가 공존)
4. AD 차이로 잘못된 경로를 선택할 수 있음
   OSPF Internal: AD 110
   EIGRP External: AD 170
   OSPF External: AD 110 (E1/E2)
```

**확인 순서:**

**1단계: 문제 경로의 출처 확인**
```
show ip route 10.1.0.0
! 경로가 OSPF Internal(O)인지, OSPF External(O E1/E2)인지, EIGRP External(D EX)인지 확인
```

**2단계: 양쪽 ASBR에서 해당 경로 확인**
```
! ASBR-1
show ip route 10.1.0.0
show ip ospf database external | include 10.1.0
show ip eigrp topology 10.1.0.0/16
!
! ASBR-2
show ip route 10.1.0.0
show ip ospf database external | include 10.1.0
show ip eigrp topology 10.1.0.0/16
```

**3단계: Tag 확인 (루프 방지 동작 여부)**
```
show ip route 10.1.0.0
  ! Tag 컬럼 확인
show ip eigrp topology 10.1.0.0/16
  ! Tag 확인
```

**4단계: Route-map 매칭 확인**
```
show route-map EIGRP-TO-OSPF
show route-map OSPF-TO-EIGRP
! Hit count가 0이면 매칭이 안 되고 있는 것
```

**가장 흔한 원인:**
1. **Tag 기반 필터가 미적용:** Route-map에 deny 규칙이 없거나, tag가 전달되지 않음
2. **AD(Administrative Distance) 문제:** OSPF External(110)과 EIGRP Internal(90) 차이로 재분배된 경로가 원본보다 선호됨
3. **Metric 문제:** 재분배된 경로의 metric이 원본보다 좋아서 잘못 선택

---

## 면접관: "Tag 기반 루프 방지의 원리를 좀 더 자세히 설명해주세요. 왜 Tag를 쓰는 건가요?"

Tag는 경로에 붙는 **마커(표시)**입니다. 재분배 시 원래 출처를 식별하는 데 사용합니다.

**루프 방지 로직:**

```
[OSPF Domain]          ASBR-1          [EIGRP Domain]
                       ASBR-2

순방향 (정상):
  OSPF 경로 10.1.0.0 → ASBR-1에서 EIGRP로 재분배 → Tag 110 부여
                                                      ↓
  EIGRP 도메인에서 Tag 110을 가진 채로 전파
                                                      ↓
역방향 (루프 차단):
  이 경로가 ASBR-2에 도착 → ASBR-2가 EIGRP→OSPF 재분배 시도
  Route-map: "match tag 110" → deny! → 재분배 차단 ✓
```

Tag 없이는 ASBR-2가 이 경로의 원래 출처를 알 수 없습니다. ASBR-2 입장에서는 "EIGRP에서 온 경로"일 뿐이고, 이것이 원래 OSPF에서 왔다가 재분배된 것인지 순수 EIGRP 경로인지 구분이 안 됩니다.

**Tag 값 설계 컨벤션:**

| Tag 값 | 의미 |
|--------|------|
| 90 | 원래 EIGRP에서 온 경로 (EIGRP AD 값과 맞춤) |
| 110 | 원래 OSPF에서 온 경로 (OSPF AD 값과 맞춤) |
| 200 | Static Route에서 온 경로 |

이렇게 프로토콜의 AD 값과 Tag를 맞추면 직관적으로 관리할 수 있습니다.

**주의:** Tag는 일부 프로토콜에서 전파 범위가 다릅니다:
- EIGRP: Tag를 다른 EIGRP 라우터에게 전파 (내부 경로/외부 경로 모두)
- OSPF: External LSA(Type 5, 7)에만 Tag 포함. Internal 경로에는 Tag 없음
- BGP: Community나 Extended Community에 Tag 정보 포함 가능

---

## 면접관: "Tag 외에 AD(Administrative Distance)를 조정해서 루프를 방지하는 방법도 있다고 들었는데, 설명해주세요."

맞습니다. AD 조정은 Tag와 함께 사용하거나, 단독으로 사용할 수 있는 루프 방지 기법입니다.

**문제 시나리오:**

ASBR-2에서 10.1.0.0/16 경로를 두 가지 경로로 알고 있을 때:
- OSPF Internal (O): AD 110 → 이것이 원본
- EIGRP External (D EX): AD 170 → ASBR-1이 재분배한 것

이 경우 OSPF(110)가 EIGRP External(170)보다 AD가 낮으므로 원본을 선택합니다. 문제없음.

하지만 반대 방향:
- EIGRP Internal (D): AD 90 → 이것이 원본
- OSPF External (O E2): AD 110 → ASBR-1이 재분배한 것

EIGRP(90)가 OSPF External(110)보다 AD가 낮으므로 역시 원본을 선택. 이것도 문제없음.

**AD 문제가 발생하는 경우:**
ASBR 자체에서 Connected + OSPF + EIGRP를 모두 돌리는 경우, 또는 서브넷 레벨에서 미묘한 차이가 있을 때:

```
! OSPF External AD를 EIGRP External보다 높게 조정
router ospf 1
 distance ospf external 180    ! OSPF External의 AD를 180으로 (EIGRP External 170보다 높게)
!
! 또는 특정 경로에 대해서만 AD 조정
router eigrp 100
 distance eigrp 90 170         ! Internal 90, External 170 (기본값)
```

**AD 조정 전략:**

| 경로 유형 | 기본 AD | 조정 목적 |
|-----------|---------|----------|
| EIGRP Internal | 90 | 유지 (원본 EIGRP 경로 선호) |
| OSPF Internal | 110 | 유지 (원본 OSPF 경로 선호) |
| EIGRP External | 170 | 재분배된 EIGRP 경로 |
| OSPF External | 110 | **180으로 올려서** 재분배 경로 비선호 |

```
! 확인
show ip route 10.1.0.0
  ! AD 값과 metric 확인
show ip protocols
  ! 현재 AD 설정 확인
```

**권장사항:** AD 조정은 **최후의 수단**으로 사용하고, 1차적으로는 Tag 기반 Route-map 필터링을 사용하세요. AD 조정은 전역적으로 영향을 미치므로 예상치 못한 부작용이 생길 수 있습니다.

---

## 면접관: "고객사가 '재분배 과정에서 불필요한 경로까지 다 넘어가고 있다, 필요한 경로만 선택적으로 재분배해달라'고 합니다. 어떻게 하시겠어요?"

**선택적 재분배**는 Route-map + Prefix-list/ACL 조합으로 구현합니다.

```
! OSPF에서 EIGRP로: 10.1.0.0/16, 10.2.0.0/16만 재분배
ip prefix-list OSPF-TO-EIGRP-ALLOWED seq 10 permit 10.1.0.0/16
ip prefix-list OSPF-TO-EIGRP-ALLOWED seq 20 permit 10.2.0.0/16
!
route-map OSPF-TO-EIGRP deny 10
 match tag 90
!
route-map OSPF-TO-EIGRP permit 20
 match ip address prefix-list OSPF-TO-EIGRP-ALLOWED
 set tag 110
!
! EIGRP에서 OSPF로: 172.16.0.0/16, 172.17.0.0/16만 재분배
ip prefix-list EIGRP-TO-OSPF-ALLOWED seq 10 permit 172.16.0.0/16
ip prefix-list EIGRP-TO-OSPF-ALLOWED seq 20 permit 172.17.0.0/16
!
route-map EIGRP-TO-OSPF deny 10
 match tag 110
!
route-map EIGRP-TO-OSPF permit 20
 match ip address prefix-list EIGRP-TO-OSPF-ALLOWED
 set tag 90
 set metric 100
 set metric-type type-1
```

**추가 고려사항:**

1. **distribute-list와 route-map 차이:**
   - `distribute-list`: 수신/송신 경로 필터링 (프로토콜 내부)
   - `route-map in redistribute`: 재분배 시 필터링 + attribute 설정 가능
   - 재분배에서는 route-map이 더 유연하고 권장됨

2. **경로 요약 후 재분배:**
```
! OSPF 쪽에서 재분배된 EIGRP 경로를 요약
router ospf 1
 summary-address 172.16.0.0 255.254.0.0    ! 172.16.0.0/15로 요약
```

3. **재분배 모니터링:**
```
show ip route ospf | include E1|E2
show ip route eigrp | include EX
show ip ospf database external
show ip eigrp topology | include Ext
!
! 재분배되는 경로 수 추적
show ip protocols | section Redistrib
```

---

## 면접관: "장기적으로 하나의 프로토콜로 통합하려면 어떤 순서로 진행하시겠어요?"

**통합 마이그레이션 4단계 전략:**

**Phase 1: 현상 유지 + 양방향 재분배 안정화 (1~2개월)**
- Tag 기반 루프 방지 완벽 검증
- 재분배 경로 모니터링 자동화
- 문서화: 현재 토폴로지, IP 체계, 경로 수

**Phase 2: 목표 프로토콜 선정 및 설계 (2주)**
- OSPF vs EIGRP 선택 (Cisco Only면 EIGRP, Multi-vendor면 OSPF)
- IP 체계 재설계 (Area 구조 또는 EIGRP Stub 설계)
- 마이그레이션 일정 수립

**Phase 3: Ships-in-the-Night 병행 구간 (2~4개월)**
- 인수된 쪽 네트워크에 목표 프로토콜을 **병행 구동**
- 양쪽 프로토콜이 동시에 돌아가는 상태
- AD 조정으로 기존 프로토콜 우선 유지
```
! 예: EIGRP를 OSPF로 전환할 때, OSPF AD를 높여서 EIGRP 경로 우선
router ospf 1
 distance 115    ! OSPF AD를 기본 110에서 115로 올림 (EIGRP 110보다 높게)
```
- 구간별로 OSPF 경로 확인 후, AD를 기본으로 복원하여 OSPF로 전환

**Phase 4: 기존 프로토콜 제거 (2~4주)**
- 모든 구간 OSPF 전환 확인
- EIGRP 프로세스 제거
- 재분배 설정 제거
- 최종 검증 및 문서화

```
! 마이그레이션 중 모니터링
show ip route summary          ! 프로토콜별 경로 수
show ip protocols              ! 재분배 상태
show ip ospf neighbor          ! OSPF neighbor 상태
show ip eigrp neighbor         ! EIGRP neighbor 상태
```

**핵심 원칙:** "한 번에 한 구간만 전환하고, 반드시 검증 후 다음 구간으로 진행." 전체를 한꺼번에 바꾸면 롤백이 불가능합니다.
