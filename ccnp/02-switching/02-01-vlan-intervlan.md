# 02-01. VLAN 설계와 Inter-VLAN Routing

---

### 면접관: "고객사에서 본사 사무실을 새로 꾸미는데, 영업부/개발부/경영지원부 3개 부서가 같은 층에 있어요. 보안 때문에 부서별 네트워크를 분리해달라고 합니다. 스위치 2대, L3 스위치 1대 있고요. 어떻게 설계하시겠어요?"

VLAN을 이용해서 부서별 브로드캐스트 도메인을 분리하고, L3 스위치에서 SVI(Switch Virtual Interface)를 통해 Inter-VLAN Routing을 구성하겠습니다.

**설계 포인트:**

| 구분 | VLAN ID | 서브넷 | 게이트웨이 | 용도 |
|------|---------|--------|-----------|------|
| 영업부 | VLAN 10 | 10.1.10.0/24 | 10.1.10.1 | 영업 업무용 |
| 개발부 | VLAN 20 | 10.1.20.0/24 | 10.1.20.1 | 개발 업무용 |
| 경영지원부 | VLAN 30 | 10.1.30.0/24 | 10.1.30.1 | 경영지원 업무용 |
| Management | VLAN 99 | 10.1.99.0/24 | 10.1.99.1 | 스위치 관리용 |

Access 스위치 2대(SW1, SW2)에 각 부서 포트를 Access 포트로 할당하고, L3 스위치(DSW1)까지의 Uplink는 Trunk로 구성합니다. L3 스위치에 SVI를 만들어서 Inter-VLAN Routing을 처리합니다.

> 토폴로지 이미지 추가 예정

#### 핵심 설정

**Access Switch (SW1, SW2) 공통:**

```
! VLAN 생성
vlan 10
 name SALES
vlan 20
 name DEVELOPMENT
vlan 30
 name MANAGEMENT_SUPPORT
vlan 99
 name MGMT

! 영업부 포트 (예: Gi0/1~0/8)
interface range GigabitEthernet0/1 - 8
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! 개발부 포트 (예: Gi0/9~0/16)
interface range GigabitEthernet0/9 - 16
 switchport mode access
 switchport access vlan 20
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! 경영지원부 포트 (예: Gi0/17~0/24)
interface range GigabitEthernet0/17 - 24
 switchport mode access
 switchport access vlan 30
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! Uplink to DSW1 (Trunk)
interface GigabitEthernet0/48
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 switchport nonegotiate
 no shutdown
```

**L3 Switch (DSW1):**

```
! IP Routing 활성화
ip routing

! Trunk 포트 (SW1 방향)
interface GigabitEthernet1/0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 switchport nonegotiate
 no shutdown

! Trunk 포트 (SW2 방향)
interface GigabitEthernet1/0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,99
 switchport trunk native vlan 99
 switchport nonegotiate
 no shutdown

! SVI 설정
interface Vlan10
 ip address 10.1.10.1 255.255.255.0
 no shutdown

interface Vlan20
 ip address 10.1.20.1 255.255.255.0
 no shutdown

interface Vlan30
 ip address 10.1.30.1 255.255.255.0
 no shutdown

interface Vlan99
 ip address 10.1.99.1 255.255.255.0
 no shutdown
```

---

### 면접관: "Access 포트와 Trunk 포트의 차이를 프레임 처리 관점에서 설명해주세요."

**Access 포트**는 하나의 VLAN에만 소속되며, 프레임이 스위치로 들어올 때 해당 VLAN 태그를 내부적으로 부여하고 나갈 때는 태그를 제거합니다. 즉, 엔드 디바이스는 VLAN 태그의 존재를 인식하지 못합니다.

**Trunk 포트**는 여러 VLAN의 트래픽을 동시에 전달합니다. IEEE 802.1Q 표준에 따라 이더넷 프레임의 Source MAC과 EtherType 사이에 4바이트의 태그를 삽입합니다.

**802.1Q 태그 구조 (4바이트):**

```
| TPID (16bit) | PCP (3bit) | DEI (1bit) | VID (12bit) |
| 0x8100       | 우선순위    | Drop 여부   | VLAN ID     |
```

- **TPID (Tag Protocol Identifier):** 0x8100으로 고정. 이 프레임이 태그된 프레임임을 나타냄
- **PCP (Priority Code Point):** CoS 값 (0~7). QoS에서 사용
- **DEI (Drop Eligible Indicator):** 혼잡 시 드롭 대상 여부
- **VID (VLAN Identifier):** 12비트이므로 0~4095까지 가능. 실제 사용 가능한 범위는 1~4094

주의할 점은 Native VLAN의 트래픽은 Trunk에서 태그 없이 전송된다는 것입니다. 양쪽 Trunk 포트의 Native VLAN이 일치하지 않으면 VLAN Hopping이나 트래픽 혼선이 발생할 수 있습니다.

---

### 면접관: "그런데 고객사에서 '개발부에서 영업부 공유 서버로 접근이 안 된다'고 연락이 왔어요. 어디부터 보시겠어요?"

체계적으로 레이어별 하향식(Top-Down)으로 접근하겠습니다.

**1단계: 문제 범위 파악**

```
! 개발부 PC에서 기본 연결 확인
ping 10.1.10.100          ← 영업부 서버 IP
ping 10.1.20.1            ← 자기 게이트웨이
ping 10.1.10.1            ← 상대 VLAN 게이트웨이
```

게이트웨이까지 ping이 되면 Inter-VLAN Routing 자체는 동작하는 것이므로 서버 쪽 문제. 게이트웨이가 안 되면 L2 혹은 L3 설정 문제입니다.

**2단계: L3 스위치에서 SVI 상태 확인**

```
show ip interface brief | include Vlan
show interface vlan 10
show interface vlan 20
```

SVI가 `up/up`인지 확인합니다. `up/down`이면 해당 VLAN에 활성화된 포트가 하나도 없다는 의미입니다.

**3단계: Trunk 상태 확인**

```
show interfaces trunk
show interfaces GigabitEthernet1/0/1 switchport
```

여기서 `Allowed VLAN` 목록에 VLAN 10, 20이 모두 포함되어 있는지 확인합니다. `switchport trunk allowed vlan` 설정에서 특정 VLAN을 빠뜨린 경우가 매우 흔합니다.

**4단계: VLAN 존재 여부 확인**

```
show vlan brief
```

Access 스위치에서 VLAN 자체가 생성되어 있지 않으면 해당 포트는 비활성화됩니다. VTP를 사용하지 않는 환경에서 스위치마다 수동으로 VLAN을 만들어야 하는데, 한쪽 스위치에서 누락하는 경우가 있습니다.

**5단계: L3 라우팅 확인**

```
show ip route
show ip route 10.1.10.0
show ip route 10.1.20.0
```

`ip routing`이 활성화되어 있는지, Connected 경로가 정상적으로 보이는지 확인합니다.

**6단계: ACL 확인**

```
show ip access-lists
show running-config | section interface Vlan
```

SVI에 ACL이 적용되어 부서 간 통신을 차단하고 있을 수 있습니다.

실무에서 가장 흔한 원인은 (1) Trunk Allowed VLAN 누락, (2) SVI에 `no shutdown` 누락, (3) `ip routing` 미활성화 이 세 가지입니다.

---

### 면접관: "Native VLAN이 보안 문제가 될 수 있다고 했는데, 구체적으로 어떤 공격이 가능하고, 어떻게 방어하나요?"

**VLAN Hopping Attack** 두 가지 방식이 있습니다.

#### 1. Switch Spoofing

공격자가 자신의 PC를 Trunk 포트로 위장하여 DTP(Dynamic Trunking Protocol) 협상을 시도합니다. Trunk가 맺어지면 모든 VLAN 트래픽에 접근할 수 있습니다.

**방어:**

```
! 모든 Access 포트에서 DTP 비활성화
interface range GigabitEthernet0/1 - 24
 switchport mode access
 switchport nonegotiate

! 사용하지 않는 포트는 shutdown + 미사용 VLAN 할당
interface range GigabitEthernet0/25 - 48
 switchport mode access
 switchport access vlan 999
 shutdown
```

#### 2. Double Tagging (이중 태깅)

공격자가 Native VLAN과 같은 VLAN에 있을 때, 프레임에 802.1Q 태그를 이중으로 달아서 보냅니다.

동작 원리:
1. 공격자가 `[외부태그: Native VLAN][내부태그: 타겟 VLAN][페이로드]` 프레임 전송
2. 첫 번째 스위치가 외부 태그를 제거 (Native VLAN이므로 태그 없이 Trunk로 전달)
3. 두 번째 스위치가 내부 태그를 보고 타겟 VLAN으로 전달

이 공격은 단방향이라 응답을 받을 수 없지만, DoS 등에 활용 가능합니다.

**방어:**

```
! Native VLAN을 사용하지 않는 VLAN으로 변경
interface GigabitEthernet0/48
 switchport trunk native vlan 999

! Native VLAN에도 태그를 강제 (전역 설정)
vlan dot1q tag native

! Native VLAN에 어떤 호스트도 할당하지 않음
vlan 999
 name NATIVE_UNUSED
```

---

### 면접관: "DTP를 비활성화한다고 하셨는데, DTP의 동작 모드별 차이를 설명해주세요. 그리고 왜 실무에서 반드시 꺼야 하나요?"

DTP(Dynamic Trunking Protocol)는 Cisco 전용 프로토콜로, 인접 스위치 간 Trunk 협상을 자동으로 수행합니다.

**DTP 모드 매트릭스:**

| | **Dynamic Auto** | **Dynamic Desirable** | **Trunk** | **Access** |
|---|---|---|---|---|
| **Dynamic Auto** | Access | Trunk | Trunk | Access |
| **Dynamic Desirable** | Trunk | Trunk | Trunk | Access |
| **Trunk** | Trunk | Trunk | Trunk | 제한적 Trunk |
| **Access** | Access | Access | 제한적 Trunk | Access |

- **Dynamic Auto:** 상대방이 먼저 Trunk를 요청하면 수락. 수동적 모드
- **Dynamic Desirable:** 적극적으로 Trunk 협상 시도. 상대가 Auto여도 Trunk 성립
- **Trunk:** 무조건 Trunk. DTP 프레임은 여전히 전송
- **Access:** 무조건 Access

**실무에서 DTP를 꺼야 하는 이유:**

1. **보안:** 앞서 말한 Switch Spoofing 공격의 근본 원인
2. **예측 가능성:** 포트가 의도하지 않게 Trunk로 전환되는 것을 방지
3. **불필요한 트래픽:** DTP 프레임이 30초마다 전송되어 대역폭 낭비

```
! Access 포트: 명시적으로 access 설정 + nonegotiate
switchport mode access
switchport nonegotiate

! Trunk 포트: 명시적으로 trunk 설정 + nonegotiate
switchport mode trunk
switchport nonegotiate
```

`switchport nonegotiate`는 DTP 프레임 전송 자체를 중단합니다. `switchport mode access`만으로는 DTP 프레임은 계속 나갑니다.

---

### 면접관: "고객사가 VTP를 쓰고 있는데, 새 스위치를 추가했더니 기존 VLAN이 전부 날아갔다고 해요. 무슨 상황인지 설명하고, VTP 모드별 차이와 권장 설정을 알려주세요."

전형적인 **VTP Revision Number 문제**입니다.

**사고 시나리오:**

1. 랩이나 다른 환경에서 사용하던 스위치의 VTP Revision Number가 100이라고 가정
2. 기존 운영 네트워크의 VTP Revision Number는 50
3. 새 스위치를 네트워크에 연결하면, VTP는 Revision Number가 높은 쪽의 VLAN DB를 정본으로 간주
4. 새 스위치의 VLAN DB(랩 환경)가 전체 네트워크로 전파
5. 기존 운영 VLAN이 모두 삭제되거나 덮어씌워짐

**VTP 모드별 동작:**

| 모드 | VLAN 생성/삭제 | VTP 광고 전송 | VTP 광고 수신/적용 | Revision 증가 |
|------|---------------|-------------|-------------------|-------------|
| **Server** | 가능 | O | O | O |
| **Client** | 불가 | O | O | O |
| **Transparent** | 가능 (로컬만) | 전달만 (Relay) | 수신하되 무시 | X |
| **Off** (VTPv3) | 가능 (로컬만) | X | X | X |

**실무 권장 설정:**

대부분의 CCNP급 엔지니어들은 VTP를 **Transparent 모드**로 설정하거나, VTPv3를 지원하면 **Off 모드**를 권장합니다. VLAN 관리는 Ansible, Terraform 등 자동화 도구로 하는 것이 안전합니다.

```
! 방법 1: Transparent 모드 (권장)
vtp mode transparent

! 방법 2: VTPv3 Off 모드 (최선)
vtp version 3
vtp mode off

! 새 스위치 투입 전 Revision Number 초기화
! (domain name을 변경했다가 되돌리면 Revision이 0으로 리셋됨)
vtp domain TEMP_RESET
vtp domain PRODUCTION
```

**Revision Number 확인:**

```
show vtp status
```

VTP를 Server 모드로 운영해야 하는 경우에는 반드시 VTPv3의 **Primary Server** 기능을 사용하여 의도하지 않은 VLAN DB 전파를 방지해야 합니다.

---

### 면접관: "마지막으로, Router-on-a-Stick 방식과 SVI 방식의 Inter-VLAN Routing 차이를 비교해주시고, 언제 어떤 방식을 선택하는지 말씀해주세요."

두 방식 모두 Inter-VLAN Routing을 수행하지만 구조적으로 큰 차이가 있습니다.

#### Router-on-a-Stick (ROAS)

라우터의 물리 인터페이스 하나에 서브인터페이스를 만들어 각 VLAN을 처리합니다.

```
! Router 설정
interface GigabitEthernet0/0
 no shutdown

interface GigabitEthernet0/0.10
 encapsulation dot1q 10
 ip address 10.1.10.1 255.255.255.0

interface GigabitEthernet0/0.20
 encapsulation dot1q 20
 ip address 10.1.20.1 255.255.255.0

interface GigabitEthernet0/0.30
 encapsulation dot1q 30
 ip address 10.1.30.1 255.255.255.0
```

#### SVI (Multilayer Switch)

L3 스위치에서 VLAN 인터페이스를 직접 만들어 라우팅합니다. 앞서 설정한 방식입니다.

#### 비교

| 항목 | Router-on-a-Stick | SVI (L3 스위치) |
|------|-------------------|----------------|
| **처리 속도** | 소프트웨어 라우팅. 느림 | 하드웨어(ASIC/TCAM) 라우팅. 빠름 |
| **대역폭** | 물리 링크 1개 공유 (병목) | 백플레인 사용. 병목 없음 |
| **비용** | 라우터 + L2 스위치 (저비용) | L3 스위치 (고비용) |
| **확장성** | VLAN/트래픽 증가 시 한계 | 수백 개 VLAN도 처리 가능 |
| **장애 지점** | 라우터-스위치 간 링크가 SPOF | 스위치 내부 처리라 SPOF 감소 |
| **적용 환경** | 소규모 사무실, 학습 환경 | 엔터프라이즈 환경 표준 |

**선택 기준:**

- **소규모 사무실 (PC 50대 이하, VLAN 3~5개):** ROAS도 충분. 비용 절약
- **중규모 이상 사무실:** 반드시 SVI 방식. 성능과 확장성 필수
- **데이터센터:** SVI + VRF를 조합하여 테넌트 분리

실무에서는 거의 모든 환경에서 L3 스위치의 SVI 방식을 사용합니다. ROAS는 CCNA/CCNP 시험과 소규모 지사 정도에서만 볼 수 있습니다.

---
