# 01-01. OSPF Multi-Area 설계

## 면접관: "고객사가 현재 본사 1곳, 지사 10곳을 운영 중인데, 전부 OSPF Single Area로 돌아가고 있어요. 앞으로 지사가 20곳까지 확장될 예정이고, 이미 라우터들 CPU 사용률이 높다는 이야기가 나오고 있습니다. 어떻게 재설계하시겠어요?"

현재 Single Area 0 구성에서 모든 라우터가 동일한 LSDB를 유지하고 있기 때문에, 지사가 늘어날수록 SPF 계산 부하와 LSDB 크기가 선형적으로 증가합니다. Multi-Area 설계로 전환해서 장애 도메인을 분리하는 것이 핵심입니다.

설계 방향은 다음과 같습니다:

- **Area 0 (Backbone):** 본사 코어 라우터 2대 구성, 모든 Area의 ABR이 연결되는 중심축
- **Area 10~40:** 지사를 지역별로 묶어서 4개 Area로 분할 (서울권/경기권/중부권/남부권)
- **ABR 역할:** 본사 코어 라우터가 ABR을 겸하거나, 각 지역 허브에 ABR 배치
- **Summarization:** ABR에서 Area 간 경로를 요약하여 LSDB 축소

지사당 서브넷이 3~5개라고 가정하면, 20개 지사 확장 시 100개 이상의 경로가 생기는데, Area별로 요약하면 Area 0에는 4개의 Summary 경로만 존재하게 됩니다.

> 토폴로지 이미지 추가 예정

### 핵심 설정

**ABR (본사 코어 라우터) 설정:**
```
router ospf 1
 router-id 10.0.0.1
 auto-cost reference-bandwidth 10000
 !
 area 10 range 10.10.0.0 255.255.0.0
 area 20 range 10.20.0.0 255.255.0.0
 area 30 range 10.30.0.0 255.255.0.0
 area 40 range 10.40.0.0 255.255.0.0
!
interface GigabitEthernet0/0
 description TO-CORE-BACKBONE
 ip address 10.0.0.1 255.255.255.252
 ip ospf 1 area 0
 ip ospf network point-to-point
!
interface GigabitEthernet0/1
 description TO-SEOUL-REGION
 ip address 10.10.0.1 255.255.255.252
 ip ospf 1 area 10
```

**지사 라우터 설정:**
```
router ospf 1
 router-id 10.10.1.1
 auto-cost reference-bandwidth 10000
!
interface GigabitEthernet0/0
 ip address 10.10.1.1 255.255.255.0
 ip ospf 1 area 10
!
interface GigabitEthernet0/1
 description LAN-SEGMENT
 ip address 10.10.10.1 255.255.255.0
 ip ospf 1 area 10
```

---

## 면접관: "Multi-Area로 나누면 LSA가 어떻게 달라지는지 설명해주세요. LSA Type별로 어떤 역할을 하죠?"

OSPF LSA Type은 Multi-Area 동작의 핵심입니다.

| LSA Type | 이름 | 생성자 | 범위 | 설명 |
|----------|------|--------|------|------|
| **Type 1** | Router LSA | 모든 라우터 | Area 내부 | 자신의 인터페이스, 링크 상태, 비용 정보를 광고 |
| **Type 2** | Network LSA | DR | Area 내부 | Multi-access 네트워크(Ethernet)에서 DR이 해당 세그먼트 참여 라우터 목록 광고 |
| **Type 3** | Summary LSA | ABR | Area 간 | ABR이 한 Area의 경로를 다른 Area로 전달. Distance Vector처럼 동작 |
| **Type 4** | ASBR Summary LSA | ABR | Area 간 | 외부 경로를 가진 ASBR의 위치(Router-ID)를 다른 Area에 알림 |
| **Type 5** | External LSA | ASBR | 전체 도메인 | 외부에서 재분배된 경로. E1(누적)/E2(고정) 두 가지 |
| **Type 7** | NSSA External LSA | ASBR (NSSA 내) | NSSA Area 내부 | NSSA Area 내 ASBR이 생성, ABR에서 Type 5로 변환 |

Multi-Area의 핵심 이점은 **Type 1, 2 LSA가 Area 경계를 넘지 못한다**는 것입니다. 즉, Area 10에서 링크 하나가 플랩해도 Area 20, 30, 40 라우터는 SPF 재계산을 하지 않습니다. ABR이 Type 3으로 변환해서 전달하기 때문에, 서브넷 레벨의 변화가 아닌 이상 다른 Area에 영향이 없습니다.

---

## 면접관: "좋습니다. 그런데 그렇게 Multi-Area를 설계해서 운영하고 있는데, 고객사에서 '라우팅 테이블이 여전히 크고, 특정 지사 라우터가 메모리 부족 경고를 띄운다'고 연락이 왔습니다. 어디부터 확인하시겠어요?"

단계별로 접근하겠습니다.

**1단계: 현재 라우팅 테이블과 LSDB 크기 확인**
```
show ip route summary
show ip route ospf
show ip ospf database database-summary
```

여기서 OSPF 경로 수가 비정상적으로 많은지 확인합니다. Multi-Area인데도 경로가 많다면, Summarization이 제대로 적용되지 않았을 가능성이 높습니다.

**2단계: ABR에서 Summarization 동작 확인**
```
show ip ospf border-routers
show ip ospf database summary
show running-config | section router ospf
```

`area X range` 명령이 빠져 있거나, 범위가 맞지 않아서 개별 Type 3 LSA가 전부 통과하고 있는지 확인합니다.

**3단계: 외부 경로(Type 5) 유입 여부 확인**
```
show ip ospf database external
show ip route ospf | include E1|E2
```

어딘가에서 default route가 아닌 full external route가 재분배되고 있다면, 모든 Area로 Type 5 LSA가 퍼지면서 테이블이 비대해집니다.

**4단계: 메모리 상태 확인**
```
show processes memory sorted
show ip ospf statistics
```

LSDB가 크면 SPF 계산 빈도도 확인해야 합니다. 잦은 SPF 재계산은 CPU/메모리 부하의 직접적 원인입니다.

가장 흔한 원인은 **ABR에서 Summarization 미적용** 또는 **외부 경로 무분별 재분배**입니다.

---

## 면접관: "Summarization을 적용한다고 했는데, 구체적으로 OSPF에서 경로 요약이 어떻게 동작하는지 설명해주세요. 일반 라우팅의 Summary와 뭐가 다른가요?"

OSPF의 Inter-Area Summarization은 ABR에서만 가능합니다. `area [area-id] range [address] [mask]` 명령을 사용하면, 해당 Area에서 넘어오는 Type 1/2 LSA 기반의 세부 경로들을 하나의 Type 3 Summary LSA로 축약해서 Area 0(또는 다른 Area)에 광고합니다.

일반 라우팅의 Summary Route와의 차이점:

- **일반 Summary (ip route summary):** 단순히 라우팅 테이블에 축약 경로를 만드는 것. Null0로 향하는 방지 경로 포함
- **OSPF Summarization:** LSA 자체를 줄이는 것. LSDB 크기가 줄어들어 다른 Area 라우터들의 SPF 계산 부하까지 줄어듦

중요한 점은, OSPF Summarization도 자동으로 **Null0 경로**를 ABR 자신에게 생성합니다. 이는 축약 범위 내에 실제 경로가 없는 서브넷으로의 트래픽이 루프되는 것을 방지합니다.

```
! ABR의 라우팅 테이블에서 확인
show ip route | include Null0
  O    10.10.0.0/16 is directly connected, Null0
```

또한 OSPF에는 External Route Summarization도 있는데, 이것은 ASBR에서 `summary-address` 명령으로 Type 5 LSA를 축약합니다.

```
router ospf 1
 summary-address 192.168.0.0 255.255.0.0
```

---

## 면접관: "고객사가 추가로 '일부 소규모 지사는 외부 경로가 필요 없으니 라우팅 테이블을 최대한 줄여달라'고 요청합니다. 어떻게 대응하시겠어요?"

Stub Area 계열을 적용하겠습니다. 소규모 지사의 특성에 따라 단계별로 선택할 수 있습니다.

**Stub Area:**
- Type 4, 5 LSA가 차단됨
- ABR이 자동으로 Default Route (Type 3 LSA, 0.0.0.0/0)를 해당 Area에 주입
- Area 내부 경로(Type 1, 2)와 Inter-Area 경로(Type 3)는 유지

```
! ABR과 Area 내 모든 라우터에 설정
router ospf 1
 area 10 stub
```

**Totally Stubby Area:**
- Type 3, 4, 5 LSA 모두 차단 (ABR의 Default Route Type 3만 허용)
- 지사 라우터 입장에서 라우팅 테이블이 극단적으로 줄어듦
- 자기 Area 내부 경로 + Default Route만 존재

```
! ABR에서만 no-summary 추가
router ospf 1
 area 10 stub no-summary
!
! Area 내부 라우터는 동일하게 stub만
router ospf 1
 area 10 stub
```

**선택 기준:**
- 소규모 지사에 WAN 이중화가 없고 단일 경로로 나가도 되는 경우 → **Totally Stubby**
- Inter-Area 간 최적 경로 선택이 필요한 경우 (ABR 2개 이상) → **Stub** (Type 3은 유지)

적용 전에 반드시 확인할 사항:
1. 해당 Area에 ASBR이 없는지 확인 (Stub Area에는 ASBR이 존재할 수 없음)
2. Area 내 모든 라우터에 stub 설정을 해야 함 (하나라도 빠지면 adjacency가 안 맺어짐)
3. Virtual Link가 해당 Area를 통과하지 않는지 확인 (Stub Area를 Transit으로 사용 불가)

---

## 면접관: "Stub Area에는 ASBR이 존재할 수 없다고 했는데, 만약 소규모 지사에서 다른 프로토콜 경로를 재분배해야 하는 상황이 생기면 어떻게 하죠?"

그 경우에는 **NSSA (Not-So-Stubby Area)**를 사용합니다. NSSA는 Stub Area의 장점(Type 5 차단으로 테이블 축소)을 유지하면서, Area 내부에서 외부 경로 재분배를 허용합니다.

동작 방식:
1. NSSA 내부의 ASBR이 외부 경로를 **Type 7 LSA**로 생성 (Type 5가 아님)
2. Type 7 LSA는 NSSA 내부에서만 전파
3. ABR이 NSSA 경계에서 Type 7 → Type 5로 변환하여 Area 0과 나머지 Area에 전파
4. 외부에서 들어오는 Type 5는 여전히 차단

```
! ABR
router ospf 1
 area 10 nssa default-information-originate
!
! NSSA 내부 라우터 (ASBR 역할)
router ospf 1
 area 10 nssa
 redistribute static subnets
```

주의사항:
- NSSA에서는 ABR이 자동으로 Default Route를 주입하지 않음 → `default-information-originate` 옵션 필요
- Totally NSSA도 가능: `area 10 nssa no-summary` → Type 3까지 차단 + Default Route 자동 주입

```
! Type 7 → Type 5 변환 확인
show ip ospf database nssa-external
show ip ospf database external
```

---

## 면접관: "마지막으로, 이 전체 Multi-Area 설계에서 auto-cost reference-bandwidth를 설정하신 이유가 뭔가요? 안 하면 어떻게 되죠?"

OSPF Cost 계산 공식은 `Reference Bandwidth / Interface Bandwidth`입니다. Cisco 기본값은 Reference Bandwidth가 **100 Mbps**입니다.

문제는 100 Mbps 기준이면:
- FastEthernet (100M): Cost = 1
- GigabitEthernet (1G): Cost = 1
- 10GigabitEthernet (10G): Cost = 1

즉, **100 Mbps 이상의 모든 링크가 동일한 Cost 1**이 됩니다. 이러면 OSPF가 GigE와 10GigE를 구분하지 못하고, 최적 경로 선택이 불가능합니다.

`auto-cost reference-bandwidth 10000` (10 Gbps 기준)으로 설정하면:
- FastEthernet (100M): Cost = 100
- GigabitEthernet (1G): Cost = 10
- 10GigabitEthernet (10G): Cost = 1

이렇게 해야 대역폭이 큰 링크를 선호하는 정상적인 최적 경로 계산이 됩니다.

**반드시 전체 OSPF 도메인의 모든 라우터에 동일한 값을 설정해야 합니다.** 일부만 바꾸면 Cost 불일치로 비대칭 라우팅이 발생합니다.

```
! 모든 라우터에 동일하게 적용
router ospf 1
 auto-cost reference-bandwidth 10000
!
! 확인
show ip ospf interface brief
show ip ospf interface GigabitEthernet0/0 | include Cost
```
