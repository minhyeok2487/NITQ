# 02-03. Single-Area OSPF

> CCNA 200-301 라우팅 기초 면접 시나리오
> 주제: OSPF 기본 개념, Single-Area 설정, 네이버 상태, DR/BDR 선출, 검증 및 트러블슈팅

---

### 면접관: "OSPF가 무엇인지 기본 개념을 설명해주세요. 다른 라우팅 프로토콜과 비교했을 때 OSPF의 특징은 무엇입니까?"

OSPF(Open Shortest Path First)는 Link-State 기반의 내부 게이트웨이 프로토콜(IGP)입니다. RFC 2328에 정의된 표준 프로토콜로, 특정 벤더에 종속되지 않아 멀티벤더 환경에서 사용할 수 있습니다.

OSPF의 핵심 특징을 설명하겠습니다:

**1. Link-State 알고리즘**
각 라우터가 자신의 링크 상태 정보(LSA)를 Area 내 모든 라우터에 전파합니다. 모든 라우터가 동일한 토폴로지 데이터베이스(LSDB)를 가지고, SPF(Dijkstra) 알고리즘으로 최적 경로를 계산합니다.

**2. Area 구조**
대규모 네트워크를 Area로 분할하여 확장성을 확보합니다. 모든 Area는 Area 0(Backbone Area)에 연결되어야 합니다. CCNA 범위에서는 Single-Area OSPF를 주로 다룹니다.

**3. 메트릭 = Cost**
OSPF는 인터페이스 대역폭을 기반으로 Cost를 계산합니다. 기본 공식은 `Reference Bandwidth / Interface Bandwidth`입니다.

| 프로토콜 | 유형 | AD 값 | 메트릭 | 표준/독점 |
|---------|------|-------|--------|---------|
| OSPF | Link-State | 110 | Cost (대역폭) | 표준 (RFC) |
| EIGRP | Advanced Distance Vector | 90 | 복합 메트릭 | Cisco 독점 (현재 RFC화) |
| RIP | Distance Vector | 120 | Hop Count | 표준 (RFC) |

**4. 빠른 수렴(Convergence)**
토폴로지 변경 시 LSA를 즉시 전파하고 SPF를 재계산하여 빠르게 수렴합니다. RIP처럼 주기적 업데이트에 의존하지 않습니다.

---

### 면접관: "Single-Area OSPF를 설정하는 두 가지 방법을 보여주세요. network 명령어 방식과 인터페이스 레벨 설정의 차이를 설명해주십시오."

두 가지 OSPF 설정 방법을 보여드리겠습니다.

**방법 1: network 명령어 방식 (전통적 방식)**

```
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 192.168.10.0 0.0.0.255 area 0
R1(config-router)# network 10.0.12.0 0.0.0.3 area 0
```

여기서 `ospf 1`의 1은 프로세스 ID로, 라우터 내부에서만 의미가 있습니다. 양쪽 라우터의 프로세스 ID가 달라도 네이버가 형성됩니다.

`0.0.0.255`는 와일드카드 마스크입니다. 서브넷 마스크의 반전으로, 0인 비트는 정확히 일치해야 하고 1인 비트는 무시합니다.

**방법 2: 인터페이스 레벨 설정 (권장)**

```
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# exit

R1(config)# interface GigabitEthernet 0/0
R1(config-if)# ip ospf 1 area 0

R1(config)# interface GigabitEthernet 0/1
R1(config-if)# ip ospf 1 area 0
```

**두 방식의 비교:**

| 항목 | network 명령어 | 인터페이스 레벨 |
|------|--------------|--------------|
| 설정 위치 | router ospf 하위 | 인터페이스 하위 |
| 와일드카드 마스크 | 필요 | 불필요 |
| 직관성 | 낮음 | 높음 (어느 인터페이스인지 명확) |
| CCNA 시험 | 두 방식 모두 출제 | 두 방식 모두 출제 |

인터페이스 레벨 방식이 더 명확하고 실수가 적어 실무에서 권장됩니다. 와일드카드 마스크를 잘못 입력하면 의도하지 않은 인터페이스가 OSPF에 포함될 수 있기 때문입니다.

---

### 면접관: "OSPF Router-ID는 무엇이며, 어떻게 결정됩니까? Router-ID가 중요한 이유는 무엇입니까?"

OSPF Router-ID는 OSPF 도메인 내에서 각 라우터를 고유하게 식별하는 32비트 값입니다. IP 주소 형식(A.B.C.D)으로 표현되지만, 실제 IP 주소가 아닌 식별자입니다.

**Router-ID 결정 우선순위:**

1. **수동 설정**: `router-id` 명령어로 직접 지정한 값 (최우선)
2. **Loopback 인터페이스**: 활성화된 Loopback 중 가장 높은 IP 주소
3. **물리 인터페이스**: 활성화된 물리 인터페이스 중 가장 높은 IP 주소

```
! 권장: 명시적으로 Router-ID 설정
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
```

Router-ID가 중요한 이유는 다음과 같습니다:

- **네이버 식별**: OSPF Hello 패킷에 Router-ID가 포함되어 네이버를 식별합니다.
- **DR/BDR 선출**: DR/BDR 선출 시 우선순위가 같으면 Router-ID가 높은 라우터가 선출됩니다.
- **LSA 식별**: 각 LSA의 출처를 Router-ID로 구분합니다.
- **중복 방지**: 같은 OSPF 도메인에서 Router-ID가 중복되면 라우팅 문제가 발생합니다.

Router-ID 변경 후에는 OSPF 프로세스를 재시작해야 적용됩니다:

```
R1# clear ip ospf process
Reset ALL OSPF processes? [no]: yes
```

---

### 면접관: "OSPF 네이버 관계가 형성되는 과정을 상태별로 설명해주세요. Down 상태부터 Full 상태까지의 과정이 어떻게 됩니까?"

OSPF 네이버 관계 형성은 다음 단계를 거칩니다:

**1. Down 상태**
초기 상태입니다. 상대방으로부터 아무런 Hello 패킷도 수신하지 못한 상태입니다.

**2. Init 상태**
상대방의 Hello 패킷을 수신했지만, 그 Hello 패킷의 Neighbor 필드에 자신의 Router-ID가 포함되지 않은 상태입니다. 즉, 상대방은 아직 나를 인식하지 못했습니다.

**3. 2-Way 상태**
상대방의 Hello 패킷에 자신의 Router-ID가 포함된 것을 확인한 상태입니다. 양방향 통신이 확인되었습니다. **DR/BDR 선출은 이 단계에서 이루어집니다.** Point-to-Point 링크에서는 DR/BDR 선출이 없으므로 바로 다음 단계로 진행합니다.

**4. ExStart 상태**
DBD(Database Description) 교환을 위한 마스터/슬레이브 관계를 설정합니다. Router-ID가 높은 쪽이 마스터가 됩니다.

**5. Exchange 상태**
DBD 패킷을 교환하여 상대방이 어떤 LSA를 가지고 있는지 요약 정보를 공유합니다.

**6. Loading 상태**
DBD 교환 결과, 자신에게 없는 LSA를 LSR(Link State Request)로 요청하고 LSU(Link State Update)로 수신합니다.

**7. Full 상태**
LSDB 동기화가 완료된 상태입니다. 정상적인 네이버 관계가 형성되었습니다.

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

실무에서 `show ip ospf neighbor` 출력에 **Full** 또는 **2-Way**(DROther 간)가 아닌 다른 상태가 지속적으로 나타나면 문제가 있는 것입니다.

---

### 면접관: "DR과 BDR이 무엇이며, 왜 필요합니까? 선출 과정은 어떻게 됩니까?"

DR(Designated Router)과 BDR(Backup Designated Router)은 멀티액세스 네트워크(이더넷 등)에서 OSPF가 효율적으로 동작하기 위해 선출하는 특별한 역할의 라우터입니다.

**DR/BDR이 필요한 이유:**
멀티액세스 네트워크에 라우터가 N대 있으면 Full 메시 네이버 관계는 N(N-1)/2개가 됩니다. 라우터가 10대이면 45개의 네이버 관계가 필요합니다. DR/BDR을 선출하면 모든 라우터(DROther)는 DR/BDR하고만 Full 관계를 맺으므로 네이버 관계 수가 크게 줄어듭니다.

**선출 규칙:**

1. **OSPF Priority가 가장 높은** 라우터가 DR이 됩니다 (기본값: 1, 범위: 0~255).
2. Priority가 동일하면 **Router-ID가 가장 높은** 라우터가 DR이 됩니다.
3. Priority가 0인 라우터는 DR/BDR 선출에 참여하지 않습니다.
4. **DR 선출은 비선점형(Non-preemptive)**: 더 높은 우선순위의 라우터가 나중에 추가되어도 기존 DR을 교체하지 않습니다.

```
! DR 강제 지정을 위한 Priority 설정
R1(config)# interface GigabitEthernet 0/0
R1(config-if)# ip ospf priority 255

! DR 선출에서 제외
R2(config)# interface GigabitEthernet 0/0
R2(config-if)# ip ospf priority 0
```

검증 명령어:

```
R1# show ip ospf neighbor
Neighbor ID   Pri   State          Dead Time   Address       Interface
2.2.2.2         1   FULL/BDR       00:00:35    10.0.0.2      Gi0/0
3.3.3.3         1   FULL/DROTHER   00:00:33    10.0.0.3      Gi0/0

R1# show ip ospf interface GigabitEthernet 0/0
...
State DR, Priority 255
Designated Router (ID) 1.1.1.1, Interface address 10.0.0.1
Backup Designated Router (ID) 2.2.2.2, Interface address 10.0.0.2
```

State 컬럼에서 `FULL/DR`, `FULL/BDR`, `FULL/DROTHER`로 각 라우터의 역할을 확인할 수 있습니다.

---

### 면접관: "OSPF 네이버가 형성되지 않는 일반적인 원인들을 5가지 이상 말씀해주시고, 각각의 트러블슈팅 방법을 설명해주세요."

OSPF 네이버가 형성되지 않는 주요 원인과 확인 방법은 다음과 같습니다:

**1. Area ID 불일치**
양쪽 인터페이스가 같은 Area에 속해야 합니다.
```
! 확인
R1# show ip ospf interface GigabitEthernet 0/0
! "Area 0" 부분 확인
```

**2. 서브넷 마스크 불일치**
같은 링크의 양쪽 인터페이스는 동일한 서브넷 마스크를 사용해야 합니다. 한쪽이 /24이고 다른쪽이 /30이면 네이버가 형성되지 않습니다.
```
! 확인
R1# show ip interface brief
R1# show ip ospf interface GigabitEthernet 0/0
```

**3. Hello/Dead Timer 불일치**
양쪽의 Hello Interval과 Dead Interval이 동일해야 합니다. 기본값은 이더넷에서 Hello 10초, Dead 40초입니다.
```
! 확인
R1# show ip ospf interface GigabitEthernet 0/0
! "Timer intervals configured, Hello 10, Dead 40" 확인
```

**4. 인증 설정 불일치**
한쪽에만 OSPF 인증이 설정되어 있거나, 패스워드가 다르면 네이버가 형성되지 않습니다.
```
! 확인
R1# show ip ospf interface GigabitEthernet 0/0
! Authentication 관련 항목 확인
```

**5. OSPF Network Type 불일치**
양쪽의 네트워크 타입이 달라 Hello/Dead Timer가 자동으로 다르게 설정될 수 있습니다.
```
! 확인
R1# show ip ospf interface GigabitEthernet 0/0
! "Network Type" 확인
```

**6. ACL 또는 방화벽에 의한 차단**
OSPF는 프로토콜 번호 89를 사용합니다. ACL에서 OSPF 트래픽을 차단하면 네이버가 형성되지 않습니다.

**7. 물리적 연결 문제**
가장 기본적이지만 가장 흔한 원인입니다. 인터페이스가 down이거나 케이블 문제가 있으면 당연히 네이버가 형성되지 않습니다.
```
R1# show ip interface brief
! Status/Protocol이 모두 up인지 확인
```

**8. Passive Interface 설정**
해당 인터페이스에 `passive-interface`가 설정되면 Hello 패킷을 보내지 않아 네이버가 형성되지 않습니다.
```
! 확인
R1# show ip ospf interface GigabitEthernet 0/0
! "No Hellos (Passive interface)" 메시지 확인

! 해결
R1(config)# router ospf 1
R1(config-router)# no passive-interface GigabitEthernet 0/0
```

트러블슈팅은 `show ip ospf neighbor`, `show ip ospf interface`, `show ip interface brief` 세 가지 명령어로 시작하는 것이 기본입니다.

---

### 면접관: "OSPF Cost는 어떻게 계산되며, 경로 선택에 어떤 영향을 미칩니까? Cost를 수동으로 변경해야 하는 경우는 언제입니까?"

OSPF Cost 계산 공식은 다음과 같습니다:

```
Cost = Reference Bandwidth / Interface Bandwidth
```

기본 Reference Bandwidth는 100 Mbps(10^8)입니다.

| 인터페이스 | 대역폭 | Cost |
|-----------|--------|------|
| FastEthernet | 100 Mbps | 100M / 100M = 1 |
| GigabitEthernet | 1 Gbps | 100M / 1000M = 1 (소수점 이하 버림) |
| 10GigabitEthernet | 10 Gbps | 100M / 10000M = 1 |
| Serial (T1) | 1.544 Mbps | 100M / 1.544M = 64 |

여기서 문제가 발생합니다. FastEthernet, GigabitEthernet, 10GigabitEthernet 모두 Cost가 1로 동일하게 계산됩니다. 기본 Reference Bandwidth가 100 Mbps이기 때문입니다.

이를 해결하려면 Reference Bandwidth를 변경해야 합니다:

```
! 모든 OSPF 라우터에서 동일하게 변경해야 함
R1(config)# router ospf 1
R1(config-router)# auto-cost reference-bandwidth 10000
! 10000 Mbps = 10 Gbps 기준

! 변경 후 Cost
! GigabitEthernet: 10000 / 1000 = 10
! 10GigabitEthernet: 10000 / 10000 = 1
! FastEthernet: 10000 / 100 = 100
```

또는 인터페이스 레벨에서 Cost를 직접 지정할 수도 있습니다:

```
R1(config)# interface GigabitEthernet 0/0
R1(config-if)# ip ospf cost 5
```

OSPF는 출발지에서 목적지까지의 경로상 모든 인터페이스 Cost를 합산하여 총 Cost가 가장 낮은 경로를 최적 경로로 선택합니다. 이것이 Shortest Path First의 의미입니다.

Cost를 수동으로 변경해야 하는 대표적인 경우:
- 고대역폭 인터페이스 간 Cost를 구분해야 할 때
- 특정 경로로 트래픽을 유도하고 싶을 때 (Traffic Engineering 기초)
- 대역폭이 같지만 품질이 다른 링크를 구분해야 할 때
