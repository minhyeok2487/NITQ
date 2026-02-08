# 02-04. Private VLAN

---

## 면접관: "고객사 데이터센터에 서버가 30대 있는데, 같은 서브넷에 있으면서도 서버 간에는 직접 통신을 못 하게 해달라고 합니다. 게이트웨이하고만 통신되면 된대요. 서브넷을 30개로 쪼개면 IP 낭비가 심하잖아요. 어떻게 하시겠어요?"

Private VLAN(PVLAN)을 사용하겠습니다. 하나의 Primary VLAN 안에 Secondary VLAN을 만들어서, 같은 서브넷을 유지하면서도 서버 간 L2 통신을 격리할 수 있습니다.

**설계:**

| 구분 | VLAN ID | 타입 | 용도 |
|------|---------|------|------|
| Primary VLAN | 100 | Primary | 서버팜 전체 (10.1.100.0/24) |
| Isolated VLAN | 101 | Isolated | 개별 서버 격리 (서버 간 통신 불가) |
| Community VLAN 1 | 102 | Community | 웹 서버 그룹 (그룹 내 통신 허용) |
| Community VLAN 2 | 103 | Community | DB 서버 그룹 (그룹 내 통신 허용) |

```
[게이트웨이 L3 SW]
       │ Promiscuous Port (VLAN 100)
       │
[서버팜 스위치]
  │        │        │        │
 Srv1     Srv2     Web1     DB1
 Iso(101) Iso(101) Com(102) Com(103)
```

> 토폴로지 이미지 추가 예정

### 포트 타입 설명

**Promiscuous Port (P-Port):**
- Primary VLAN에 소속
- 모든 Secondary VLAN의 포트와 통신 가능
- 게이트웨이, 방화벽, 모니터링 서버 등에 연결

**Isolated Port (I-Port):**
- Isolated VLAN에 소속
- 같은 Isolated VLAN 내의 다른 포트와도 통신 불가
- Promiscuous Port와만 통신 가능
- 서버팜에서 개별 서버 격리에 적합

**Community Port (C-Port):**
- Community VLAN에 소속
- 같은 Community 내의 포트끼리는 통신 가능
- 다른 Community 포트와는 통신 불가
- Promiscuous Port와 통신 가능
- 웹 서버 클러스터 등 그룹 내 통신이 필요한 경우

### 핵심 설정

**1단계: VTP Transparent 모드 설정 (PVLAN 필수 조건)**

```
vtp mode transparent
```

PVLAN은 VTP Server/Client 모드에서는 동작하지 않습니다. VTPv3에서는 가능하지만, Transparent가 안전합니다.

**2단계: VLAN 생성 및 타입 지정**

```
vlan 100
 name SERVER-FARM-PRIMARY
 private-vlan primary

vlan 101
 name ISOLATED-SERVERS
 private-vlan isolated

vlan 102
 name WEB-SERVERS-COMMUNITY
 private-vlan community

vlan 103
 name DB-SERVERS-COMMUNITY
 private-vlan community
```

**3단계: Primary-Secondary VLAN 연결 (Association)**

```
vlan 100
 private-vlan association 101,102,103
```

**4단계: 포트 설정**

```
! Promiscuous Port (게이트웨이 연결)
interface GigabitEthernet0/48
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 100 101,102,103
 no shutdown

! Isolated Port (개별 서버)
interface range GigabitEthernet0/1 - 10
 switchport mode private-vlan host
 switchport private-vlan host-association 100 101
 no shutdown

! Community Port - 웹 서버 그룹
interface range GigabitEthernet0/11 - 15
 switchport mode private-vlan host
 switchport private-vlan host-association 100 102
 no shutdown

! Community Port - DB 서버 그룹
interface range GigabitEthernet0/16 - 20
 switchport mode private-vlan host
 switchport private-vlan host-association 100 103
 no shutdown
```

**5단계: L3 스위치 SVI 설정 (Inter-VLAN Routing)**

```
interface Vlan100
 ip address 10.1.100.1 255.255.255.0
 private-vlan mapping 101,102,103
 no shutdown
```

SVI는 Primary VLAN에만 생성하고, Secondary VLAN을 mapping합니다. 이렇게 하면 모든 서버가 10.1.100.1을 게이트웨이로 사용하면서 같은 서브넷을 공유합니다.

---

## 면접관: "통신 매트릭스를 정리해주세요. 어디서 어디로 통신이 되고 안 되는지 헷갈리거든요."

**PVLAN 통신 매트릭스:**

| 출발 → 도착 | Promiscuous | Isolated A | Isolated B | Community 1 | Community 2 |
|-------------|:-----------:|:----------:|:----------:|:-----------:|:-----------:|
| **Promiscuous** | O | O | O | O | O |
| **Isolated A** | O | X | X | X | X |
| **Isolated B** | O | X | X | X | X |
| **Community 1** | O | X | X | O | X |
| **Community 2** | O | X | X | X | O |

핵심 규칙:
1. **Promiscuous는 모두와 통신 가능** (양방향)
2. **Isolated는 Promiscuous와만 통신 가능.** 같은 Isolated끼리도 불가
3. **Community는 같은 Community끼리 + Promiscuous와 통신 가능**
4. **다른 Community 간 통신 불가**

**프레임 포워딩 관점:**

Isolated 포트에서 보낸 프레임은 스위치 내부에서 Primary VLAN으로 변환되어 Promiscuous 포트로만 전달됩니다. 이 과정은 L2에서 이루어지므로 IP 서브넷이 같더라도 L2 포워딩 자체가 차단됩니다.

---

## 면접관: "그런데 고객사에서 'Isolated 포트에 연결된 서버에서 게이트웨이로 ping이 안 된다'고 합니다. PVLAN 설정을 했는데 뭐가 문제일까요?"

PVLAN 설정에서 자주 발생하는 문제들을 순서대로 확인하겠습니다.

**1단계: PVLAN Association 확인**

```
show vlan private-vlan

! 출력 예시:
! Primary  Secondary  Type       Ports
! -------  ---------  ---------  ------
! 100      101        isolated   Gi0/1, Gi0/2
! 100      102        community  Gi0/11, Gi0/12
! 100      103        community  Gi0/16, Gi0/17
! 100      -          -          Gi0/48 (promiscuous)
```

여기서 association이 빠져 있으면 통신이 안 됩니다.

**2단계: Promiscuous 포트 Mapping 확인**

```
show interfaces GigabitEthernet0/48 switchport

! 확인 포인트:
! Private-vlan mapping: 101,102,103 ← 모든 Secondary가 포함되어야 함
```

**가장 흔한 원인: Promiscuous 포트의 mapping에 Secondary VLAN이 누락된 경우**

```
! 수정
interface GigabitEthernet0/48
 switchport private-vlan mapping 100 add 101
```

**3단계: SVI Mapping 확인**

```
show interfaces Vlan100
show running-config interface Vlan100

! private-vlan mapping 101,102,103 이 있어야 함
```

SVI에 `private-vlan mapping`이 없으면, L3 라우팅이 Secondary VLAN의 트래픽을 처리하지 못합니다.

**4단계: VLAN 타입 확인**

```
show vlan private-vlan type

! 간혹 VLAN을 생성만 하고 private-vlan 타입 지정을 빠뜨린 경우
! 일반 VLAN으로 남아있으면 PVLAN 동작 안 함
```

**5단계: VTP 모드 확인**

```
show vtp status

! VTP Server/Client 모드이면 PVLAN 동작 안 함
! Transparent 또는 Off여야 함
```

**종합 확인 명령어:**

```
show vlan private-vlan
show interfaces private-vlan mapping
show vlan private-vlan type
```

---

## 면접관: "Protected Port라는 것도 있잖아요. PVLAN과 뭐가 다른가요?"

**Protected Port (Switchport Protected):**

가장 단순한 형태의 포트 격리입니다. 같은 스위치 내에서 Protected Port끼리 L2 트래픽을 차단합니다.

```
interface GigabitEthernet0/1
 switchport protected

interface GigabitEthernet0/2
 switchport protected
```

이 설정만으로 Gi0/1과 Gi0/2 사이의 유니캐스트, 멀티캐스트, 브로드캐스트 트래픽이 모두 차단됩니다.

**PVLAN vs Protected Port 비교:**

| 항목 | Private VLAN | Protected Port |
|------|-------------|---------------|
| **범위** | 여러 스위치에 걸쳐 동작 가능 | 단일 스위치 내에서만 동작 |
| **유연성** | Isolated + Community로 세분화 | 단순 격리만 가능 |
| **설정 복잡도** | 높음 (Primary, Secondary, Association, Mapping) | 매우 낮음 (한 줄) |
| **VLAN 소비** | Secondary VLAN 필요 | 추가 VLAN 불필요 |
| **Trunk 전파** | PVLAN Trunk로 다른 스위치까지 격리 유지 | Trunk를 넘어가면 격리 해제 |
| **L3 연동** | SVI mapping으로 라우팅 가능 | 별도 설정 불필요 |
| **적용 환경** | 데이터센터, 대규모 서버팜 | 소규모 환경, 임시 격리 |

**Protected Port의 한계:**

```
[Switch A]                    [Switch B]
Gi0/1(Protected) ─── Trunk ─── Gi0/1(Protected)
Gi0/2(Protected)               Gi0/2(Protected)
```

Switch A의 Gi0/1과 Gi0/2는 통신 불가. 하지만 Switch A의 Gi0/1에서 Trunk를 거쳐 Switch B의 Gi0/2로는 통신 가능합니다. Protected Port는 로컬 스위치 내에서만 동작하기 때문입니다.

PVLAN은 PVLAN Trunk를 통해 여러 스위치에 걸쳐 격리를 유지할 수 있습니다.

---

## 면접관: "PVLAN 환경에서 보안을 더 강화하려면 어떻게 하나요? Isolated라도 L3에서 우회 가능하지 않나요?"

맞습니다. PVLAN은 **L2 격리**이므로, 트래픽이 L3 게이트웨이를 거쳐 라우팅되면 Isolated 서버 간에도 통신이 가능해집니다.

**PVLAN Proxy Attack (L3 우회 공격):**

1. Isolated 서버 A가 패킷을 게이트웨이(10.1.100.1)로 전송
2. 게이트웨이가 라우팅 테이블을 참조하여 목적지 서버 B(같은 서브넷)로 전달
3. 서버 B에 도달 → L2 격리 우회 성공

**방어: PVLAN + ACL 조합**

SVI에 ACL을 적용하여 같은 서브넷 내에서의 라우팅을 차단합니다.

```
! 같은 서브넷 내 트래픽 차단 (L3 우회 방지)
ip access-list extended PVLAN-PROTECT
 deny   ip 10.1.100.0 0.0.0.255 10.1.100.0 0.0.0.255
 permit ip any any

interface Vlan100
 ip address 10.1.100.1 255.255.255.0
 private-vlan mapping 101,102,103
 ip access-group PVLAN-PROTECT in
```

이 ACL은 10.1.100.0/24 내부에서 10.1.100.0/24로 향하는 트래픽을 차단합니다. 서버들이 외부(다른 서브넷)로 나가는 트래픽은 허용됩니다.

**더 세밀한 제어가 필요한 경우 - VACL (VLAN Access Map):**

```
! Web 서버 그룹(102)에서 DB 서버 그룹(103)의 특정 포트만 허용
ip access-list extended WEB-TO-DB
 permit tcp 10.1.100.0 0.0.0.255 10.1.100.0 0.0.0.255 eq 3306
 permit tcp 10.1.100.0 0.0.0.255 10.1.100.0 0.0.0.255 eq 5432

vlan access-map PVLAN-SECURITY 10
 match ip address WEB-TO-DB
 action forward

vlan access-map PVLAN-SECURITY 20
 action drop

vlan filter PVLAN-SECURITY vlan-list 100
```

---

## 면접관: "PVLAN을 여러 스위치에 걸쳐 확장해야 하는 경우는 어떻게 하나요?"

PVLAN Trunk를 사용합니다. 일반 802.1Q Trunk로는 PVLAN 정보가 유실되므로, PVLAN 전용 Trunk 설정이 필요합니다.

**시나리오: 서버팜 스위치 2대에 걸친 PVLAN**

```
[L3 Gateway SW]
  │ Promiscuous
  │
[Server SW1] ═══ PVLAN Trunk ═══ [Server SW2]
  │                                  │
 Srv1(Iso)                        Srv3(Iso)
 Srv2(Com102)                     Srv4(Com102)
```

### PVLAN Trunk 설정 (Isolated VLAN 트렁킹)

스위치 간 Trunk 포트에서 Secondary VLAN 정보를 유지하려면 **Isolated Trunk** 또는 **Promiscuous Trunk**를 설정합니다.

**Server SW1:**

```
! 스위치 간 Trunk (PVLAN Trunk)
interface GigabitEthernet0/47
 switchport mode private-vlan trunk secondary
 switchport private-vlan trunk native vlan 99
 switchport private-vlan trunk allowed vlan 100,101,102,103
 switchport private-vlan association trunk 100 101
 no shutdown
```

**Server SW2:**

```
interface GigabitEthernet0/47
 switchport mode private-vlan trunk secondary
 switchport private-vlan trunk native vlan 99
 switchport private-vlan trunk allowed vlan 100,101,102,103
 switchport private-vlan association trunk 100 101
 no shutdown
```

**Promiscuous Trunk (게이트웨이 방향):**

```
! L3 Gateway로의 Trunk
interface GigabitEthernet0/48
 switchport mode private-vlan trunk promiscuous
 switchport private-vlan trunk native vlan 99
 switchport private-vlan trunk allowed vlan 100,101,102,103
 switchport private-vlan mapping trunk 100 101,102,103
 no shutdown
```

**확인:**

```
show interfaces GigabitEthernet0/47 switchport
show vlan private-vlan
show interfaces private-vlan mapping
```

**주의사항:**
- PVLAN Trunk는 플랫폼에 따라 지원 여부가 다릅니다. Catalyst 3560/3750은 제한적이고, Catalyst 4500/6500/9000 시리즈에서 완전 지원
- 설정이 복잡하므로 반드시 유지보수 시간에 작업하고, 롤백 계획을 수립해야 합니다

---

## 면접관: "마지막으로, 실무에서 PVLAN 대신 사용할 수 있는 대안은 뭐가 있을까요?"

상황에 따라 여러 대안이 있습니다.

**1. Micro-Segmentation (SDN 기반):**

VMware NSX, Cisco ACI 등에서 제공하는 마이크로 세그멘테이션이 PVLAN의 현대적 대안입니다. 워크로드 단위로 정책을 적용할 수 있어 훨씬 유연합니다.

**2. VRF (Virtual Routing and Forwarding):**

서버 그룹별로 완전히 다른 라우팅 테이블을 사용하여 격리합니다. L3에서의 격리이므로 PVLAN보다 강력하지만, 같은 서브넷을 공유할 수 없습니다.

**3. Port-based ACL (PACL):**

```
! 포트 단위로 ACL을 적용하여 특정 트래픽만 허용
interface GigabitEthernet0/1
 ip access-group SERVER1-ACL in
```

PVLAN처럼 L2를 차단하지는 못하지만, L3/L4 기준으로 세밀한 제어가 가능합니다.

**4. 802.1X + dACL (Dynamic ACL):**

인증 기반으로 포트에 동적 ACL을 적용합니다. RADIUS 서버에서 사용자/디바이스별 정책을 중앙 관리할 수 있습니다.

**현실적 선택 가이드:**

| 환경 | 권장 솔루션 |
|------|-----------|
| 전통적 데이터센터, Cisco 스위치 | PVLAN |
| 가상화 환경 (VMware, Hyper-V) | NSX/ACI 마이크로 세그멘테이션 |
| 소규모, 빠른 적용 필요 | Protected Port + ACL |
| 멀티벤더 환경 | ACL + 802.1X |
| 클라우드/하이브리드 | Security Group (AWS SG, Azure NSG) |

---
