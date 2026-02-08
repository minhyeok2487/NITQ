# 09-01. Enterprise 3-Tier 아키텍처

---

### 면접관: "중견기업(직원 800명)이 신사옥으로 이전합니다. 15층 건물이고, 각 층마다 40~60명이 근무해요. 유선/무선 모두 지원해야 하고, 향후 IoT 디바이스 확장도 고려해달라는 RFP가 왔어요. 네트워크 아키텍처를 어떻게 설계하시겠습니까?"

전형적인 Enterprise Campus 네트워크이므로 **Cisco 3-Tier (Access - Distribution - Core)** 아키텍처를 기반으로 설계하겠습니다.

#### 계층별 설계

| 계층 | 역할 | 장비 예시 | 수량 |
|------|------|-----------|------|
| **Core** | 고속 백본, 라우팅 집약, 서버팜/WAN 연결 | Catalyst 9500 (또는 Nexus 9300) | 2대 (이중화) |
| **Distribution** | VLAN 간 라우팅, 정책(ACL/QoS), 경계 집약 | Catalyst 9400 | 4대 (2쌍, 층별 그룹) |
| **Access** | 엔드포인트 연결, PoE(AP/IP Phone), 포트 보안 | Catalyst 9200/9300 | 15~20대 (층당 1~2대) |

#### 설계 구조

- **Core 2대**를 VSS 또는 StackWise Virtual로 논리적 1대 구성
- **Distribution은 2쌍(4대)**으로 구성: 1~8층 담당 1쌍, 9~15층 담당 1쌍
- 각 Distribution 쌍도 StackWise Virtual 또는 vPC로 이중화
- **Access → Distribution**: 듀얼 업링크 (10G SFP+)
- **Distribution → Core**: 40G 또는 100G 업링크
- Core에서 서버팜, WAN Edge, 인터넷 Edge로 연결

#### 설계 근거

800명 규모 + 15층 건물이면 Access 스위치가 15대 이상 필요합니다. Distribution 없이 Core에 직접 연결하면 Core의 포트 밀도가 부족하고, 정책 적용 지점이 모호해집니다. 3-Tier로 분리해야 **계층별 역할이 명확**하고 **장애 격리(fault domain)**가 가능합니다.

> 토폴로지 이미지 추가 예정

---

### 면접관: "각 계층의 역할을 좀 더 구체적으로 설명해주세요. 예를 들어 Distribution에서 라우팅을 한다고 했는데, Core에서 하면 안 되나요?"

각 계층의 설계 원칙은 역할 분리와 확장성입니다.

#### Access Layer

- **역할**: 엔드포인트(PC, IP Phone, AP, 프린터)에 물리적 포트 제공
- VLAN 할당, 802.1X 인증, PoE 공급, Storm Control, DHCP Snooping
- **L2 스위칭만 수행** (라우팅은 하지 않음)
- STP(Spanning Tree)의 Edge Port로 동작

```
! Access 스위치 포트 설정 예시
interface GigabitEthernet1/0/1
 description USER-PC
 switchport mode access
 switchport access vlan 100
 switchport port-security maximum 3
 spanning-tree portfast
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 15
```

#### Distribution Layer

- **역할**: Access에서 올라오는 VLAN을 **집약(aggregation)**하고, **Inter-VLAN 라우팅** 수행
- ACL, QoS 정책 적용 지점 (Policy Enforcement Point)
- STP Root Bridge 역할
- 라우팅 프로토콜(OSPF/EIGRP) 경계

```
! Distribution SVI 및 OSPF 설정 예시
interface Vlan100
 description OFFICE-USERS-FLOOR1-8
 ip address 10.1.100.1 255.255.255.0
 ip helper-address 10.1.200.10
 ip access-group OFFICE-USERS-IN in
!
router ospf 1
 router-id 10.255.0.1
 network 10.1.0.0 0.0.255.255 area 1
 network 10.255.0.0 0.0.0.255 area 0
```

#### Core Layer

- **역할**: Distribution 간, 그리고 서버팜/WAN으로의 **고속 전달(forwarding)**
- 정책 적용 최소화, **패킷을 빨리 전달하는 것이 핵심**
- OSPF Area 0 (Backbone Area) 역할
- 모든 Distribution이 Core를 경유하여 통신

**"Core에서 라우팅하면 안 되느냐"는 질문에 대해**: Core에서도 당연히 라우팅은 합니다. 하지만 **VLAN 간 라우팅(Inter-VLAN)**은 Distribution에서 수행합니다. Core는 Distribution 간의 경로, 서버팜/WAN으로의 경로만 처리합니다. 이렇게 해야 Core의 부하를 줄이고, 정책 적용 지점을 Distribution으로 명확히 할 수 있습니다.

---

### 면접관: "지금 설계에서 Distribution 스위치 한 대가 장애가 나면 어떤 영향이 있죠? 해당 층 사용자들이 전부 끊기나요?"

아닙니다. Distribution을 **쌍(pair)**으로 구성했기 때문에 한 대가 죽어도 서비스는 유지됩니다.

#### Failover 시나리오

**전제**: Distribution A, B가 StackWise Virtual 또는 개별 이중화 구성

#### Case 1: StackWise Virtual 구성 시

- Distribution A-B가 논리적 1대로 동작
- A가 죽으면 B가 Active로 승격 (SSO: Stateful Switchover)
- Access 스위치의 MEC(Multi-chassis EtherChannel) 링크 중 A쪽 링크만 down
- **영향**: 전환 시간 약 1~3초, 사용자는 거의 체감 불가

#### Case 2: 개별 이중화 (HSRP/VRRP) 구성 시

```
! Distribution-A (Active)
interface Vlan100
 ip address 10.1.100.2 255.255.255.0
 standby 100 ip 10.1.100.1
 standby 100 priority 110
 standby 100 preempt
 standby 100 track GigabitEthernet1/0/1 decrement 20

! Distribution-B (Standby)
interface Vlan100
 ip address 10.1.100.3 255.255.255.0
 standby 100 ip 10.1.100.1
 standby 100 priority 100
 standby 100 preempt
```

- HSRP Active인 A가 죽으면 B가 Active로 승격
- **HSRP Failover 시간**: 기본 Hold Timer 10초, Tuning 시 약 1~3초
- Access 스위치의 업링크 중 A쪽 링크 down → STP Reconvergence 발생 가능
- **영향**: HSRP 전환 + STP 수렴으로 약 3~10초 단절 가능

#### 영향 범위

- 해당 Distribution 쌍에 연결된 층(예: 1~8층)만 영향
- 9~15층은 별도 Distribution 쌍이므로 영향 없음 → **장애 도메인 격리**

---

### 면접관: "그런데 800명 규모면 굳이 3-Tier로 할 필요가 있나요? Collapsed Core, 즉 2-Tier로 해도 되지 않습니까?"

좋은 지적입니다. 실제로 **Collapsed Core(2-Tier)**를 쓸 수 있는 경우가 있고, 이 프로젝트에서도 검토해볼 수 있습니다.

#### Collapsed Core (2-Tier) 개념

- Distribution과 Core를 **하나의 계층으로 합침**
- Access → Distribution/Core(합체) 2계층 구조
- Distribution 스위치가 Inter-VLAN 라우팅 + 고속 전달을 모두 수행

#### 언제 Collapsed Core를 쓰는가?

| 기준 | 3-Tier | Collapsed Core (2-Tier) |
|------|--------|-------------------------|
| 사용자 수 | 1,000명 이상 | 500명 이하 |
| Access 스위치 수 | 20대 이상 | 10대 이하 |
| 건물 수 | 다수 건물/캠퍼스 | 단일 건물 |
| 서버팜/WAN 규모 | 대규모 | 소규모 |
| 향후 확장 | 많음 | 제한적 |

#### 이 프로젝트의 판단

800명 + 15층 + IoT 확장 → **경계선에 있는 규모**입니다.

- 현재만 보면 Collapsed Core도 가능합니다.
- 하지만 RFP에 "IoT 확장 고려"가 있고, 향후 사원 증가 가능성, 무선 AP 밀도 증가 등을 감안하면 **3-Tier를 추천**합니다.
- 비용 절감이 최우선이라면 Collapsed Core로 시작하되, Core 분리가 가능한 장비(Catalyst 9500)를 Distribution으로 배치하여 **향후 3-Tier 전환이 용이**하도록 설계합니다.

---

### 면접관: "오버서브스크립션(Oversubscription) 비율은 어떻게 계산하고, 각 계층별로 얼마가 적절한가요?"

#### 오버서브스크립션 비율 정의

모든 다운링크 포트가 동시에 최대 속도로 트래픽을 보낼 때, 업링크가 이를 감당할 수 있는 비율입니다.

```
Oversubscription Ratio = 총 다운링크 대역폭 : 총 업링크 대역폭
```

#### 계산 예시 (이 설계 기준)

**Access Layer**:
- 다운링크: 48포트 x 1Gbps = 48Gbps
- 업링크: 2포트 x 10Gbps = 20Gbps
- 비율: 48:20 = **2.4:1**

```
실제 동시 사용률을 감안하면 (보통 5~20%), 2.4:1은 충분한 수준
```

**Distribution Layer**:
- 다운링크(Access에서 올라오는): 8대 x 20Gbps = 160Gbps
- 업링크(Core로): 2포트 x 40Gbps = 80Gbps
- 비율: 160:80 = **2:1**

**Core Layer**:
- 오버서브스크립션 최소화, 가능하면 **1:1 (Non-blocking)**

#### 계층별 권장 비율

| 계층 | 권장 비율 | 이유 |
|------|-----------|------|
| Access | 20:1 이하 | 엔드포인트가 동시에 full rate 사용하지 않음 |
| Distribution | 4:1 이하 | 집약 트래픽, 정책 처리 병목 방지 |
| Core | 2:1 이하 (이상적: 1:1) | 모든 트래픽이 통과, 병목 불가 |

핵심은 **상위 계층으로 갈수록 오버서브스크립션을 낮춰야** 한다는 것입니다. Access에서는 통계적 다중화(statistical multiplexing)에 의존할 수 있지만, Core에서 병목이 생기면 전체 네트워크가 영향을 받습니다.

---

### 면접관: "설계 완료 후 고객사에서 '각 층에 무선 AP를 10대씩 추가해달라'고 합니다. 어떤 영향이 있고, 설계를 어떻게 변경해야 하나요?"

#### 영향 분석

- AP 추가: 15층 x 10대 = **150대 AP**
- 각 AP는 PoE 필요 (802.11ax AP 기준: PoE+ 30W 또는 UPoE 60W)
- 각 AP당 최대 트래픽: 약 1~2Gbps (Wi-Fi 6 기준)

#### 1. Access Layer 영향

**PoE 전력 예산(Power Budget) 확인이 최우선**입니다.

```
! PoE 현황 확인
show power inline

! 예시 출력
Available: 740.0(w)  Used: 320.0(w)  Remaining: 420.0(w)
```

- 층당 AP 10대 x 30W = 300W 추가 필요
- 기존 IP Phone 등 PoE 디바이스 + AP PoE → Access 스위치의 전력 예산 초과 가능
- **대응**: PoE 예산 부족 시 Access 스위치 추가 배치, 또는 UPoE 지원 스위치로 교체

**포트 수도 확인**: 기존 48포트 중 여유 포트가 있는지, 없으면 스위치 스택 추가

#### 2. Distribution Layer 영향

- Access 스위치가 추가되면 Distribution 다운링크 포트 소모 증가
- AP 트래픽이 추가되면서 오버서브스크립션 비율 재계산 필요
- 무선 사용자 VLAN 추가 → Distribution에서 SVI 추가

```
! 무선 사용자 VLAN 추가
vlan 200
 name WIRELESS-USERS
!
interface Vlan200
 ip address 10.1.200.1 255.255.255.0
 ip helper-address 10.1.250.10
```

#### 3. 무선 컨트롤러(WLC) 배치

150대 AP를 관리하려면 **WLC(Wireless LAN Controller)**가 필요합니다.

- **Cisco Catalyst 9800** 시리즈 (또는 가상 어플라이언스 9800-CL)
- WLC 2대를 HA(Active-Standby)로 구성
- **배치 위치**: Core 또는 서버팜에 연결 (Central Switching 모드 시)
- 대안: FlexConnect 모드로 AP에서 로컬 스위칭 → Core 부하 감소

#### 4. 설계 변경 요약

1. Access 스위치 PoE 예산/포트 수 재산정 → 필요 시 스위치 추가
2. 무선 전용 VLAN 설계 (SSID별 VLAN 매핑)
3. WLC HA 구성 및 Core 연결
4. Distribution 업링크 대역폭 재검토 (오버서브스크립션 비율 재계산)
5. DHCP 스코프 확장 (무선 사용자 + AP 관리용)

---

### 면접관: "마지막으로, 이 3-Tier 설계에서 STP(Spanning Tree)는 어떻게 구성하시겠어요? RSTP? MST?"

#### STP 설계 전략

이 규모에서는 **Rapid-PVST+** 를 기본으로 하되, VLAN 수가 많아지면 **MST(Multiple Spanning Tree)** 전환을 고려합니다.

#### Rapid-PVST+ 구성

```
! 전체 스위치 공통
spanning-tree mode rapid-pvst

! Distribution-A: 홀수 VLAN Root
spanning-tree vlan 1,3,5,100,300 priority 4096
spanning-tree vlan 2,4,6,200,400 priority 8192

! Distribution-B: 짝수 VLAN Root
spanning-tree vlan 2,4,6,200,400 priority 4096
spanning-tree vlan 1,3,5,100,300 priority 8192
```

- Distribution 쌍에서 **VLAN별 Root를 분산** → 트래픽 로드밸런싱
- Access 스위치 포트: `spanning-tree portfast` + `bpduguard enable`
- Access 업링크: `spanning-tree guard root` (Access가 Root가 되는 것 방지)

#### MST 전환 기준

| 기준 | Rapid-PVST+ | MST |
|------|-------------|-----|
| VLAN 수 | ~100 | 100+ |
| STP 인스턴스 부하 | 감당 가능 | VLAN 수만큼 인스턴스 → CPU 부하 |
| 설계 복잡도 | 낮음 | Region/Instance 매핑 필요 |

VLAN이 100개를 넘어가면 Rapid-PVST+는 VLAN마다 STP 인스턴스가 돌아서 CPU 부하가 커집니다. 이때 MST로 전환하여 여러 VLAN을 하나의 인스턴스에 매핑합니다.

```
! MST 구성 예시
spanning-tree mode mst
spanning-tree mst configuration
 name CORP-NETWORK
 revision 1
 instance 1 vlan 1-100
 instance 2 vlan 101-200
!
spanning-tree mst 1 priority 4096
spanning-tree mst 2 priority 8192
```

다만 MST는 모든 스위치에서 **Region Name, Revision, VLAN-Instance 매핑이 동일**해야 하므로 운영 관리가 까다롭습니다. 이 프로젝트에서는 초기 VLAN 수가 50개 이하로 예상되므로 **Rapid-PVST+로 시작**하고, IoT 확장으로 VLAN이 증가하면 MST 전환을 계획합니다.
