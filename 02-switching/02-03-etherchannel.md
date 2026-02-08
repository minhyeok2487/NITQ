# 02-03. EtherChannel (LACP/PAgP)

---

## 면접관: "고객사에서 서버팜과 코어 스위치 사이의 1G 링크가 포화 상태인데, 10G 업그레이드 예산이 없대요. 어떻게 대역폭을 늘릴 수 있을까요?"

EtherChannel(Link Aggregation)을 제안하겠습니다. 여러 개의 물리 링크를 하나의 논리 링크로 묶어서 대역폭을 확장하고, 동시에 링크 이중화도 확보할 수 있습니다.

예를 들어 1G 포트 4개를 EtherChannel로 묶으면 논리적으로 4G 대역폭을 사용할 수 있습니다. 실제로는 로드 밸런싱 알고리즘에 따라 개별 플로우가 특정 링크에 할당되므로 이론적 최대치와는 차이가 있지만, 전반적인 throughput은 크게 향상됩니다.

**설계:**

```
                    Po1 (4x 1G = 논리적 4G)
[Server Farm SW] ================================ [Core SW]
                  Gi0/1 ─┐                ┌─ Gi1/0/1
                  Gi0/2 ─┤  Port-Channel  ├─ Gi1/0/2
                  Gi0/3 ─┤                ├─ Gi1/0/3
                  Gi0/4 ─┘                └─ Gi1/0/4
```

> 토폴로지 이미지 추가 예정

### 핵심 설정

**LACP를 사용한 L2 EtherChannel:**

```
! Server Farm Switch
interface range GigabitEthernet0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
 no shutdown

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30

! Core Switch
interface range GigabitEthernet1/0/1 - 4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 channel-group 1 mode active
 no shutdown

interface Port-channel1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
```

**설정 시 반드시 지켜야 할 규칙:**

EtherChannel 멤버 포트는 다음 속성이 모두 동일해야 합니다:
- Speed / Duplex
- STP 설정 (Cost, Priority)
- Switchport Mode (Access/Trunk)
- Allowed VLAN (Trunk인 경우)
- Native VLAN (Trunk인 경우)

하나라도 다르면 채널이 맺어지지 않거나, 맺어졌다가 개별 포트가 **suspended** 상태가 됩니다.

---

## 면접관: "LACP와 PAgP의 차이를 설명해주세요. 실무에서 어떤 걸 쓰나요?"

두 프로토콜 모두 EtherChannel 협상을 자동으로 수행하지만 근본적 차이가 있습니다.

**LACP (Link Aggregation Control Protocol) - IEEE 802.3ad:**

- **표준 프로토콜.** 모든 벤더 장비 간 호환 가능
- 최대 16개 포트 가능 (8개 Active + 8개 Standby)
- Standby 링크가 있어 Active 링크 장애 시 자동 교체
- LACP 프레임을 30초(Normal) 또는 1초(Fast) 간격으로 교환

**PAgP (Port Aggregation Protocol) - Cisco 전용:**

- Cisco 장비끼리만 사용 가능
- 최대 8개 포트 (Standby 없음)
- 기능적으로 LACP보다 제한적

**모드 비교:**

| | LACP Active | LACP Passive | PAgP Desirable | PAgP Auto | Static (On) |
|---|---|---|---|---|---|
| **LACP Active** | O | O | X | X | X |
| **LACP Passive** | O | X | X | X | X |
| **PAgP Desirable** | X | X | O | O | X |
| **PAgP Auto** | X | X | O | X | X |
| **Static (On)** | X | X | X | X | O |

**모드 설명:**
- **Active / Desirable:** 적극적으로 협상 시도
- **Passive / Auto:** 상대가 먼저 요청하면 수락 (수동적)
- **On:** 프로토콜 없이 강제 묶음. 양쪽 모두 `on`이어야 하고, 협상이 없으므로 불일치 감지 불가

**실무 권장:**

```
! 항상 LACP Active - Active 사용 (권장)
channel-group 1 mode active

! LACP Fast 타이머 (장애 감지 시간 단축: 30초 → 3초)
interface range GigabitEthernet0/1 - 4
 lacp rate fast
```

실무에서는 **LACP Active-Active**를 표준으로 사용합니다. 이유:
1. 멀티벤더 환경 호환성
2. Standby 링크 기능
3. 표준 프로토콜이므로 기술 문서가 풍부
4. `on` 모드는 절대 사용하지 않음 (한쪽 설정 오류 시 루프 발생 위험)

---

## 면접관: "그런데 고객사에서 '4개 링크를 묶었는데 EtherChannel이 안 맺어진다'고 합니다. 어디부터 보시겠어요?"

EtherChannel이 맺어지지 않는 원인은 대부분 **양쪽 설정 불일치**입니다. 체계적으로 확인하겠습니다.

**1단계: EtherChannel 상태 확인**

```
show etherchannel summary

! 출력 예시:
! Group  Port-channel  Protocol    Ports
! ------+-------------+-----------+------
! 1      Po1(SD)       LACP      Gi0/1(I) Gi0/2(I) Gi0/3(s) Gi0/4(s)
```

**플래그 해석:**

| 플래그 | 의미 |
|--------|------|
| **D** | Down |
| **S** | Layer2 |
| **R** | Layer3 |
| **U** | Up, in use (정상) |
| **P** | Bundled in Port-channel (정상) |
| **I** | Stand-alone (Independent) - 채널에 안 묶임 |
| **s** | Suspended |
| **H** | Hot-standby (LACP) |
| **w** | Waiting |

`(I)` = 채널에 합류하지 못한 상태. `(s)` = Suspended 상태. 둘 다 설정 문제 지표.

**2단계: 양쪽 설정 비교**

```
! 양쪽 스위치 모두에서 실행
show etherchannel port-channel
show etherchannel detail

! 멤버 포트의 속성 비교
show interfaces GigabitEthernet0/1 switchport
show interfaces GigabitEthernet0/2 switchport
show interfaces GigabitEthernet0/3 switchport
show interfaces GigabitEthernet0/4 switchport
```

**3단계: 흔한 원인 체크리스트**

```
! 1. 프로토콜 모드 불일치 확인
show etherchannel summary
! 한쪽이 LACP, 다른 쪽이 PAgP이면 절대 안 맺힘
! 한쪽이 On, 다른 쪽이 Active여도 안 맺힘

! 2. Speed/Duplex 불일치
show interfaces GigabitEthernet0/1 status
show interfaces GigabitEthernet0/2 status

! 3. Trunk/Access 모드 불일치
show interfaces trunk

! 4. Allowed VLAN 불일치
show interfaces trunk | include allowed

! 5. Native VLAN 불일치
show interfaces trunk | include native

! 6. 포트가 err-disabled 상태인지
show interfaces status err-disabled
```

**4단계: LACP 협상 상태 확인**

```
show lacp neighbor
show lacp counters
show lacp internal

! LACP neighbor가 보이지 않으면 L1/L2 연결 문제
! counters에서 Marker나 LACPDUs가 증가하지 않으면 프레임이 도달하지 않는 것
```

**가장 흔한 실무 실수 Top 3:**

1. **Port-channel 인터페이스에만 설정하고 멤버 포트에는 안 한 경우 (또는 그 반대):** 멤버 포트에 `channel-group`을 설정하면 Port-channel 인터페이스가 자동 생성됨. 하지만 Trunk 설정 등은 순서에 따라 불일치 발생 가능
2. **한쪽은 `mode active`, 다른 쪽도 `mode active`로 해야 하는데 `mode on`으로 한 경우**
3. **STP Cost나 Priority가 멤버 포트 간 다른 경우**

---

## 면접관: "EtherChannel 로드 밸런싱이 불균형하다는 불만이 있어요. 4개 링크인데 트래픽이 1개 링크에만 몰린대요. 왜 그런 거고, 어떻게 해결하나요?"

EtherChannel의 로드 밸런싱은 **플로우 기반**이지 **패킷 기반**이 아닙니다. 이것이 불균형의 근본 원인입니다.

**동작 원리:**

스위치는 프레임의 특정 필드(MAC, IP, Port 등)를 해싱하여 멤버 링크를 선택합니다. 같은 소스-목적지 쌍의 트래픽은 항상 같은 링크로 전송되어 **프레임 순서를 보장**합니다.

패킷 기반으로 분산하면 순서가 뒤바뀌어 TCP 재전송이 발생하고 오히려 성능이 저하됩니다.

**로드 밸런싱 알고리즘:**

```
show etherchannel load-balance

! 기본값 (플랫폼마다 다름):
! src-mac       : Source MAC 기반
! dst-mac       : Destination MAC 기반
! src-dst-mac   : Source + Destination MAC 기반
! src-ip        : Source IP 기반
! dst-ip        : Destination IP 기반
! src-dst-ip    : Source + Destination IP 기반 (L3 환경 권장)
! src-port      : Source TCP/UDP Port 기반
! dst-port      : Destination TCP/UDP Port 기반
! src-dst-port  : Source + Destination Port 기반
! src-dst-mixed-ip-port : IP + Port 혼합 (최적 분산, 지원 플랫폼 한정)
```

**불균형 시나리오 예시:**

서버팜에 서버가 2대뿐이고 `src-mac`으로 해싱하면, 2개의 MAC만 존재하므로 최대 2개 링크만 사용됩니다. 나머지 2개는 유휴 상태.

**해결:**

```
! L3 환경: src-dst-ip 권장
port-channel load-balance src-dst-ip

! 최신 플랫폼에서 L4까지 포함하면 최적 분산
port-channel load-balance src-dst-mixed-ip-port
```

**특정 트래픽이 어느 링크로 가는지 확인:**

```
! 특정 Source/Destination으로 어느 멤버 포트를 사용하는지 시뮬레이션
test etherchannel load-balance interface Port-channel1 ip 10.1.10.100 10.1.20.200

! 출력: Would select Gi0/3 of Po1
```

**로드 밸런싱 최적화 가이드:**

| 환경 | 권장 알고리즘 | 이유 |
|------|-------------|------|
| L2 전용 (Access-Core) | src-dst-mac | L2 환경에서 사용 가능한 유일한 기준 |
| L3 환경 (서버팜) | src-dst-ip | IP 주소가 MAC보다 다양 |
| 웹 서버팜 (다수 클라이언트 → 소수 서버) | src-dst-mixed-ip-port | Port 번호까지 포함하여 최대한 분산 |
| 단일 서버 대량 트래픽 | 근본적 한계 | EtherChannel로 해결 불가. 10G/25G 업그레이드 필요 |

---

## 면접관: "고객사가 라우팅되는 구간에도 대역폭을 묶고 싶대요. L3 EtherChannel은 어떻게 구성하나요?"

L3 EtherChannel은 Port-channel 인터페이스에 직접 IP 주소를 할당하여 라우팅 인터페이스로 사용합니다.

**시나리오:** 코어 라우터와 방화벽 사이에 L3 EtherChannel 구성

```
[Core Router] ═══ Po1 (L3, 4x1G) ═══ [Firewall]
  10.1.1.1/30                           10.1.1.2/30
```

### 핵심 설정

**Core Router:**

```
interface range GigabitEthernet0/1 - 4
 no switchport
 no ip address
 channel-group 1 mode active
 no shutdown

interface Port-channel1
 no switchport
 ip address 10.1.1.1 255.255.255.252
 no shutdown
```

**Firewall (Cisco 라우터인 경우):**

```
interface range GigabitEthernet0/1 - 4
 no switchport
 no ip address
 channel-group 1 mode active
 no shutdown

interface Port-channel1
 no switchport
 ip address 10.1.1.2 255.255.255.252
 no shutdown
```

**L2 vs L3 EtherChannel 비교:**

| 항목 | L2 EtherChannel | L3 EtherChannel |
|------|-----------------|-----------------|
| 멤버 포트 모드 | switchport (L2) | no switchport (L3) |
| Port-channel 설정 | Trunk/Access | IP 주소 |
| STP | 적용됨 | 적용 안 됨 |
| 용도 | 스위치 간 VLAN 트렁킹 | 라우터 간 라우팅 링크 |
| 라우팅 프로토콜 | 불가 | OSPF, EIGRP 등 가능 |

**L3 EtherChannel + OSPF 예시:**

```
interface Port-channel1
 ip address 10.1.1.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0
 no shutdown

router ospf 1
 router-id 1.1.1.1
```

L3 EtherChannel에서 OSPF를 사용할 때, 인터페이스의 Cost는 멤버 링크의 합산 대역폭 기준으로 자동 계산됩니다. 4x1G이면 Reference Bandwidth가 100인 경우 Cost가 1이 되지만, `auto-cost reference-bandwidth 10000`으로 조정하는 것이 좋습니다.

---

## 면접관: "EtherChannel 멤버 링크 중 하나가 장애 나면 어떻게 동작하나요? 그리고 LACP의 System Priority와 Port Priority는 뭔가요?"

### 멤버 링크 장애 시 동작

**LACP의 경우:**

1. 장애 링크 감지 (LACP Fast: 3초, Normal: 90초)
2. 해당 링크를 Port-channel에서 제거
3. 로드 밸런싱 해시가 재계산되어 트래픽 재분배
4. Standby 링크가 있으면 자동으로 Active로 승격

```
! 장애 전
! Po1: Gi0/1(P) Gi0/2(P) Gi0/3(P) Gi0/4(P)  ← 4개 Active

! 장애 후 (Gi0/2 Down)
! Po1: Gi0/1(P) Gi0/3(P) Gi0/4(P)  ← 3개 Active, 대역폭 3G로 감소

! Standby가 있었다면
! Po1: Gi0/1(P) Gi0/3(P) Gi0/4(P) Gi0/5(P)  ← Standby가 승격
```

**최소 링크 설정 (안정성 확보):**

```
interface Port-channel1
 port-channel min-links 2
```

멤버 링크가 2개 미만이 되면 Port-channel 전체를 Down 시킵니다. 이렇게 하면 대역폭이 너무 낮아진 상태에서 서비스가 지속되는 것을 방지하고, FHRP나 라우팅 프로토콜이 대체 경로로 전환하도록 유도합니다.

### LACP System Priority와 Port Priority

**System Priority (시스템 우선순위):**

LACP는 양쪽 스위치 중 하나를 **LACP Master**로 선출합니다. Master가 실제 어떤 포트를 Active/Standby로 할지 결정합니다.

```
! System Priority 설정 (낮을수록 우선)
lacp system-priority 100

! 확인
show lacp sys-id
```

기본값은 32768. 더 낮은 값을 가진 쪽이 Master.

**Port Priority (포트 우선순위):**

Master 측에서 어떤 포트를 Active로 선택할지 결정합니다. 8개 이상 포트가 있을 때 Active 8개, Standby 나머지를 결정하는 기준입니다.

```
! 포트 우선순위 설정 (낮을수록 우선 Active)
interface GigabitEthernet0/1
 lacp port-priority 100

! 확인
show lacp internal
```

**실무 예시 - 16포트 중 8개 Active, 8개 Standby:**

```
! 반드시 Active여야 하는 포트 (예: 서로 다른 라인카드에 분산)
interface GigabitEthernet1/0/1
 lacp port-priority 100
interface GigabitEthernet2/0/1
 lacp port-priority 100
! ... (8개)

! Standby 포트
interface GigabitEthernet1/0/5
 lacp port-priority 200
! ... (8개)
```

이렇게 하면 라인카드 장애 시에도 Standby에서 다른 라인카드의 포트가 승격되어 단일 라인카드 장애에 대한 복원력을 확보할 수 있습니다.

---

## 면접관: "마지막으로, EtherChannel과 관련된 STP 동작은 어떻게 되나요?"

STP는 EtherChannel을 **하나의 논리 포트**로 인식합니다. 이것이 매우 중요한 포인트입니다.

**STP 관점에서의 EtherChannel:**

1. **Cost 계산:** 멤버 링크의 합산 대역폭으로 STP Cost 결정
   - 1G x 4 = 4G → STP Cost: 보통 2 (표준 기준)
   - 개별 1G 링크의 Cost 4보다 낮으므로 EtherChannel 경로가 선호됨

2. **BPDU 처리:** BPDU는 Port-channel 인터페이스를 통해 하나만 전송/수신
   - 멤버 링크 개별로 BPDU가 나가는 것이 아님

3. **포트 역할:** Port-channel 전체가 하나의 Root Port 또는 Designated Port

**EtherChannel이 맺어지지 않았을 때의 STP 위험:**

양쪽에서 EtherChannel 설정이 불일치하면, 개별 링크가 독립적으로 동작합니다. 이때 STP가 일부를 Blocking해야 하지만, `channel-group mode on`으로 설정된 경우 STP 수렴 전에 루프가 발생할 수 있습니다.

이것이 **LACP를 반드시 사용해야 하는 또 다른 이유**입니다. LACP가 실패하면 개별 포트가 독립적으로 동작하므로 STP가 루프를 방지할 수 있습니다. `on` 모드에서는 이 보호가 없습니다.

```
! EtherChannel 관련 STP 확인
show spanning-tree interface Port-channel1
show spanning-tree interface Port-channel1 detail
```

---
