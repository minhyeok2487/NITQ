# 03-05. EtherChannel 기본 (LACP)

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 - EtherChannel과 LACP

---

### 면접관: "두 스위치 사이에 1Gbps 링크가 하나 있는데 대역폭이 부족합니다. 그렇다고 10Gbps 장비로 교체하기에는 예산이 없습니다. 어떻게 해결하시겠습니까?"

이 경우 **EtherChannel**을 사용하여 해결하겠습니다.

EtherChannel은 여러 개의 물리적 링크를 하나의 논리적 링크로 묶는 기술입니다. 예를 들어, 1Gbps 링크 4개를 묶으면 **논리적으로 4Gbps 대역폭**을 확보할 수 있습니다.

EtherChannel의 장점:

1. **대역폭 증가**: 물리적 링크의 대역폭을 합산
2. **이중화(Redundancy)**: 묶인 링크 중 하나가 다운되어도 나머지 링크로 통신 지속
3. **STP 효율**: STP는 EtherChannel을 하나의 링크로 인식하므로, 개별 링크가 차단되지 않음
4. **비용 절감**: 고가의 상위 장비로 교체하지 않고도 대역폭 확보 가능

만약 EtherChannel 없이 스위치 간에 여러 링크를 연결하면, STP가 루프로 판단하여 하나의 링크만 남기고 나머지를 **모두 차단**합니다. EtherChannel을 사용하면 STP가 묶인 링크들을 하나로 인식하여 모든 링크가 활성 상태로 유지됩니다.

---

### 면접관: "EtherChannel을 구성하는 프로토콜에는 어떤 것들이 있습니까? 각각의 차이점을 설명해 주세요."

EtherChannel을 구성하는 방법은 세 가지가 있습니다:

| 구분 | LACP | PAgP | Static (On) |
|------|------|------|-------------|
| 표준 | IEEE 802.3ad | Cisco 독점 | 프로토콜 없음 |
| 협상 | 양쪽이 협상하여 구성 | 양쪽이 협상하여 구성 | 협상 없이 강제 구성 |
| 호환성 | 멀티벤더 환경 가능 | Cisco 장비끼리만 | 양쪽 모두 On 설정 필요 |
| 모드 | Active / Passive | Desirable / Auto | On |
| 권장 여부 | 가장 권장 | Cisco 환경에서 가능 | 비권장 (트러블슈팅 어려움) |

**CCNA에서는 LACP가 가장 중요합니다.** IEEE 표준이기 때문에 벤더에 관계없이 사용할 수 있고, 실무에서도 가장 많이 사용됩니다.

#### LACP 모드 조합

| SW1 | SW2 | 결과 |
|-----|-----|------|
| Active | Active | EtherChannel 형성 |
| Active | Passive | EtherChannel 형성 |
| Passive | Passive | EtherChannel 미형성 |

- **Active**: 적극적으로 LACP 패킷을 전송하여 협상을 시도
- **Passive**: 상대방이 LACP 패킷을 보내면 응답 (자신이 먼저 시작하지 않음)

양쪽 모두 Passive이면 아무도 협상을 시작하지 않으므로 EtherChannel이 형성되지 않습니다. 따라서 **최소 한쪽은 Active**여야 합니다.

#### PAgP 모드 조합

| SW1 | SW2 | 결과 |
|-----|-----|------|
| Desirable | Desirable | EtherChannel 형성 |
| Desirable | Auto | EtherChannel 형성 |
| Auto | Auto | EtherChannel 미형성 |

PAgP의 Desirable은 LACP의 Active와, Auto는 Passive와 유사한 역할입니다.

---

### 면접관: "LACP EtherChannel을 실제로 설정하는 과정을 보여주세요. 주의할 점도 함께 설명해 주세요."

네, 4개의 Gigabit 포트로 LACP EtherChannel을 구성하겠습니다.

#### SW1 설정

```
SW1(config)# interface range GigabitEthernet 0/1 - 4

! 먼저 포트 설정을 동일하게 맞춤
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# switchport trunk allowed vlan 10,20,30
SW1(config-if-range)# switchport trunk native vlan 99

! EtherChannel 구성 (LACP Active 모드, Channel-group 1)
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit

! Port-Channel 인터페이스 설정 확인/추가
SW1(config)# interface Port-Channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# exit
```

#### SW2 설정

```
SW2(config)# interface range GigabitEthernet 0/1 - 4
SW2(config-if-range)# switchport mode trunk
SW2(config-if-range)# switchport trunk allowed vlan 10,20,30
SW2(config-if-range)# switchport trunk native vlan 99
SW2(config-if-range)# channel-group 1 mode active
SW2(config-if-range)# exit

SW2(config)# interface Port-Channel 1
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk allowed vlan 10,20,30
SW2(config-if)# switchport trunk native vlan 99
SW2(config-if)# exit
```

#### 설정 시 주의사항

EtherChannel에 묶이는 모든 포트는 다음 설정이 **반드시 동일**해야 합니다:

| 항목 | 설명 |
|------|------|
| Speed | 모든 포트 속도 동일 |
| Duplex | 모든 포트 duplex 동일 |
| Switchport Mode | 모두 access 또는 모두 trunk |
| VLAN 설정 | Access VLAN 또는 Trunk allowed VLAN 동일 |
| Native VLAN | Trunk인 경우 Native VLAN 동일 |
| STP 설정 | PortFast 등 STP 관련 설정 동일 |

하나라도 다르면 EtherChannel이 형성되지 않거나, 형성 후에도 **개별 포트가 suspended 상태**가 됩니다.

---

### 면접관: "EtherChannel의 로드 밸런싱은 어떻게 동작합니까? 4Gbps 대역폭이라고 했는데, 모든 트래픽이 균등하게 분산됩니까?"

좋은 질문입니다. EtherChannel의 로드 밸런싱은 **완벽하게 균등하지는 않습니다**.

스위치는 특정 기준(해싱 알고리즘)에 따라 각 플로우를 특정 물리적 링크에 할당합니다. 사용 가능한 로드 밸런싱 방법:

| 방법 | 설명 |
|------|------|
| src-mac | 출발지 MAC 주소 기반 |
| dst-mac | 목적지 MAC 주소 기반 |
| src-dst-mac | 출발지 + 목적지 MAC 조합 |
| src-ip | 출발지 IP 주소 기반 |
| dst-ip | 목적지 IP 주소 기반 |
| src-dst-ip | 출발지 + 목적지 IP 조합 |
| src-port | 출발지 TCP/UDP 포트 기반 |
| dst-port | 목적지 TCP/UDP 포트 기반 |

```
! 현재 로드 밸런싱 방법 확인
SW1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        src-dst-ip

! 로드 밸런싱 방법 변경
SW1(config)# port-channel load-balance src-dst-ip
```

**src-dst-ip**가 가장 일반적으로 사용됩니다. 출발지와 목적지 IP의 조합으로 해시값을 계산하여 링크를 선택하므로, 다양한 통신 쌍이 있을 때 비교적 고르게 분산됩니다.

다만, 같은 출발지-목적지 쌍의 트래픽은 항상 같은 링크를 사용합니다. 이는 프레임 순서를 보장하기 위한 것입니다. 따라서 하나의 큰 파일 전송은 하나의 링크만 사용하여 1Gbps를 넘지 못합니다.

---

### 면접관: "EtherChannel 구성 후 상태를 확인하는 방법을 보여주세요."

핵심 확인 명령어는 `show etherchannel summary`입니다:

```
SW1# show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer 3      S - Layer 2
        U - in use       f - failed to allocate aggregator

        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-------------------------------
1      Po1(SU)       LACP        Gi0/1(P)  Gi0/2(P)  Gi0/3(P)  Gi0/4(P)
```

출력 해석:

- **Po1(SU)**: Port-Channel 1이 Layer 2(S)이고 사용 중(U)
- **Gi0/1(P)**: 각 포트가 Port-Channel에 묶여 있음(P = bundled)
- **(D)**: 다운 상태이면 문제 있음
- **(s)**: suspended - 설정 불일치로 비활성화
- **(I)**: stand-alone - EtherChannel에 참여하지 못함

추가 확인 명령어:

```
! 상세 정보 확인
SW1# show etherchannel detail

! Port-Channel 인터페이스 상태
SW1# show interfaces Port-Channel 1

! LACP 상태 확인
SW1# show lacp neighbor
SW1# show lacp internal
```

```
SW1# show lacp neighbor
Flags:  S - Device is requesting Slow LACPDUs
        F - Device is requesting Fast LACPDUs
        A - Device is in Active mode
        P - Device is in Passive mode

Channel group 1 neighbors
Partner's information:
          Partner               Partner
Port      System ID             Port Number   Age    Flags
Gi0/1     32768,aabb.cc00.0200  0x1           12s    FA
Gi0/2     32768,aabb.cc00.0200  0x2           11s    FA
Gi0/3     32768,aabb.cc00.0200  0x3           12s    FA
Gi0/4     32768,aabb.cc00.0200  0x4           11s    FA
```

이를 통해 상대방 스위치의 LACP 정보와 동작 모드를 확인할 수 있습니다.

---

### 면접관: "EtherChannel 구성 시 자주 발생하는 실수나 장애 상황에 대해 말씀해 주세요."

EtherChannel 관련 흔한 실수들을 정리하겠습니다:

#### 실수 1: 포트 설정 불일치

가장 흔한 문제입니다. 묶을 포트 중 하나라도 설정이 다르면 EtherChannel이 실패합니다.

```
! 잘못된 예: Gi0/4만 다른 VLAN 설정
SW1(config)# interface GigabitEthernet 0/4
SW1(config-if)# switchport trunk allowed vlan 10,20,30,40  ! 다른 포트는 10,20,30만 허용

! 결과: Gi0/4가 suspended 상태가 됨
```

#### 실수 2: 프로토콜 불일치

```
! SW1은 LACP, SW2는 PAgP로 설정하면 절대 형성되지 않음
SW1(config-if)# channel-group 1 mode active    ! LACP
SW2(config-if)# channel-group 1 mode desirable  ! PAgP  ← 호환 불가
```

#### 실수 3: On 모드 한쪽만 설정

```
! 한쪽은 On, 다른 쪽은 LACP Active → 형성 안 됨
SW1(config-if)# channel-group 1 mode on
SW2(config-if)# channel-group 1 mode active  ← On은 LACP/PAgP와 호환 불가
```

On 모드는 반드시 **양쪽 모두 On**이어야 합니다.

#### 실수 4: 기존 설정이 있는 포트에 EtherChannel 적용

포트에 기존 설정이 남아 있으면 문제가 될 수 있습니다. 깨끗한 상태에서 시작하는 것이 좋습니다:

```
! 포트 설정 초기화 후 EtherChannel 구성
SW1(config)# default interface range GigabitEthernet 0/1 - 4
SW1(config)# interface range GigabitEthernet 0/1 - 4
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# channel-group 1 mode active
```

#### 트러블슈팅 체크리스트

| 확인 사항 | 명령어 |
|----------|--------|
| EtherChannel 상태 | `show etherchannel summary` |
| 포트별 상태 플래그 | P(정상), s(suspended), I(독립) |
| 포트 설정 일치 여부 | `show running-config interface Gi0/1` (각 포트별) |
| LACP 협상 상태 | `show lacp neighbor` |
| 물리적 링크 상태 | `show interfaces status` |
| Port-Channel 자체 상태 | `show interfaces Port-Channel 1` |

EtherChannel 문제가 발생하면, 먼저 `show etherchannel summary`로 전체 상태를 파악하고, suspended나 stand-alone 포트가 있으면 해당 포트의 설정을 다른 포트와 비교하여 불일치를 찾는 것이 핵심입니다.

---

### 면접관: "LACP에서 최대 몇 개의 포트를 묶을 수 있습니까?"

LACP에서는 최대 **16개의 물리적 포트**를 하나의 EtherChannel 그룹에 할당할 수 있습니다. 하지만 실제로 활성화되어 데이터를 전송하는 포트는 **최대 8개**입니다.

나머지 8개는 **Hot-Standby** 상태로 대기합니다. 활성 포트 중 하나가 다운되면 Hot-Standby 포트가 자동으로 활성화됩니다.

```
! show etherchannel summary에서 Hot-Standby 포트 표시
Group  Port-channel  Protocol    Ports
1      Po1(SU)       LACP        Gi0/1(P)  Gi0/2(P)  Gi0/3(P)  Gi0/4(P)
                                 Gi0/5(P)  Gi0/6(P)  Gi0/7(P)  Gi0/8(P)
                                 Gi0/9(H)  Gi0/10(H)
```

여기서 **(H)**는 Hot-Standby 상태를 의미합니다.

LACP는 포트 우선순위를 사용하여 어떤 포트가 활성이 되고 어떤 포트가 대기하는지 결정합니다:

```
! LACP 포트 우선순위 설정 (기본값 32768, 낮을수록 우선)
SW1(config-if)# lacp port-priority 100
```

반면, PAgP와 Static(On) 모드는 **최대 8개**까지만 지원하고 Hot-Standby 기능이 없습니다. 이것도 LACP가 더 권장되는 이유 중 하나입니다.

---

> **핵심 정리**: EtherChannel은 여러 물리적 링크를 하나의 논리적 링크로 묶어 대역폭 증가와 이중화를 동시에 제공합니다. LACP(IEEE 표준)이 가장 권장되며, 묶이는 모든 포트의 설정이 동일해야 합니다. 로드 밸런싱은 해싱 기반으로 동작하며, 트러블슈팅 시 `show etherchannel summary`가 핵심 명령어입니다.
