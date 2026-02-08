# 03-06. CDP와 LLDP

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 - 네트워크 디스커버리 프로토콜

---

### 면접관: "새로 이직한 회사에서 네트워크 문서가 전혀 없습니다. 장비실에 스위치와 라우터가 여러 대 있는데, 어떤 장비가 어떤 장비와 연결되어 있는지 파악해야 합니다. 어떻게 하시겠습니까?"

네트워크 토폴로지를 파악하는 가장 효율적인 방법은 **CDP(Cisco Discovery Protocol)** 또는 **LLDP(Link Layer Discovery Protocol)**를 활용하는 것입니다.

CDP는 Cisco 장비에서 기본적으로 활성화되어 있는 프로토콜로, 직접 연결된(directly connected) 이웃 장비의 정보를 자동으로 수집합니다.

장비 하나에 접속해서 CDP 정보를 확인하고, 거기서 발견된 이웃 장비에 다시 접속해서 CDP를 확인하는 방식으로, 단계적으로 전체 네트워크 토폴로지를 그려나갈 수 있습니다.

```
SW1# show cdp neighbors

Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater, P - Phone

Device ID    Local Intrfce   Holdtme    Capability  Platform  Port ID
SW2          Gig 0/1         172        S I         WS-C2960  Gig 0/1
SW3          Gig 0/2         168        S I         WS-C2960  Gig 0/2
R1           Fas 0/24        155        R S I       ISR4321   Gig 0/0/0
PHONE-101    Fas 0/1         140        H P         CP-7821   Port 1
```

이 출력 하나만으로도 다음 정보를 파악할 수 있습니다:

| 확인 가능 정보 | 예시 |
|--------------|------|
| 이웃 장비 이름 | SW2, SW3, R1 |
| 내 포트 | Gig 0/1, Gig 0/2, Fas 0/24 |
| 이웃 장비의 포트 | Gig 0/1, Gig 0/2, Gig 0/0/0 |
| 장비 종류 | S(Switch), R(Router), P(Phone) |
| 장비 모델 | WS-C2960, ISR4321 |

이 정보를 바탕으로 네트워크 다이어그램을 작성할 수 있습니다.

---

### 면접관: "CDP neighbors 말고 더 자세한 정보를 보고 싶으면 어떻게 합니까? CDP의 동작 원리도 함께 설명해 주세요."

더 자세한 정보는 `show cdp neighbors detail` 또는 `show cdp entry *` 명령어로 확인합니다.

```
SW1# show cdp neighbors detail

-------------------------
Device ID: SW2
Entry address(es):
  IP address: 192.168.1.2
Platform: cisco WS-C2960-24TT-L,  Capabilities: Switch IGMP
Interface: GigabitEthernet0/1,  Port ID (outgoing port): GigabitEthernet0/1
Holdtime: 145 sec

Version :
Cisco IOS Software, C2960 Software (C2960-LANBASEK9-M), Version 15.2(7)E2
Technical Support: http://www.cisco.com/techsupport

advertisement version: 2
VTP Management Domain: 'COMPANY'
Native VLAN: 99
Duplex: full
Management address(es):
  IP address: 192.168.1.2
```

detail 출력에서 추가로 알 수 있는 정보:

- **IP 주소**: 이웃 장비의 관리 IP (원격 접속에 활용)
- **IOS 버전**: 소프트웨어 버전 정보
- **VTP Domain**: VTP 관리 도메인
- **Native VLAN**: 트렁크의 Native VLAN 설정
- **Duplex**: 포트의 Duplex 모드

#### CDP 동작 원리

CDP는 **Layer 2 프로토콜**로, 다음과 같이 동작합니다:

1. CDP가 활성화된 모든 인터페이스에서 **CDP 프레임(advertisement)**을 주기적으로 전송합니다
2. 기본 전송 간격: **60초**마다
3. Hold Time: **180초** (이 시간 동안 CDP를 받지 못하면 이웃 정보 삭제)
4. CDP 프레임은 **멀티캐스트** (목적지 MAC: 0100.0CCC.CCCC)
5. Layer 2 프로토콜이므로 **라우터를 넘어가지 못합니다** (직접 연결된 이웃만 발견)

```
! CDP 전역 상태 확인
SW1# show cdp

Global CDP information:
    Sending CDP packets every 60 seconds
    Sending a holdtime value of 180 seconds
    Sending CDPv2 advertisements is enabled

! CDP 타이머 변경
SW1(config)# cdp timer 30        ! 30초마다 전송
SW1(config)# cdp holdtime 120    ! 120초 hold time
```

---

### 면접관: "LLDP는 CDP와 무엇이 다릅니까? 언제 LLDP를 사용합니까?"

**LLDP(Link Layer Discovery Protocol)**는 IEEE 802.1AB 표준 프로토콜로, CDP와 동일한 역할을 하지만 **벤더 중립적(vendor-neutral)**입니다.

#### CDP vs LLDP 비교

| 구분 | CDP | LLDP |
|------|-----|------|
| 표준 | Cisco 독점 | IEEE 802.1AB |
| 호환성 | Cisco 장비끼리만 | 모든 벤더 장비 |
| 기본 상태 | Cisco에서 기본 활성화 | 일부 장비에서 수동 활성화 필요 |
| 전송 간격 | 60초 | 30초 |
| Hold Time | 180초 | 120초 |
| Layer | Layer 2 | Layer 2 |
| 멀티캐스트 주소 | 0100.0CCC.CCCC | 0180.C200.000E |

LLDP를 사용해야 하는 경우:

1. **멀티벤더 환경**: Cisco, Juniper, Arista, HP 장비가 혼재된 네트워크
2. **IP 전화기 연결**: LLDP-MED(Media Endpoint Discovery)로 VoIP 장비와 통신
3. **표준 준수가 요구되는 환경**: 기업 보안 정책상 표준 프로토콜만 허용

```
! LLDP 전역 활성화
SW1(config)# lldp run

! LLDP 이웃 정보 확인
SW1# show lldp neighbors

Capability codes:
    (R) Router, (B) Bridge, (T) Telephone, (C) DOCSIS Cable Device
    (W) WLAN Access Point, (P) Repeater, (S) Station, (O) Other

Device ID       Local Intf     Hold-time  Capability  Port ID
JuniperSW1      Gi0/1          120        B,R         ge-0/0/1
AristaSW1       Gi0/2          120        B           Ethernet1

! LLDP 상세 정보
SW1# show lldp neighbors detail

! LLDP 전역 상태 확인
SW1# show lldp

Global LLDP Information:
    Status: ACTIVE
    LLDP advertisements are sent every 30 seconds
    LLDP hold time advertised is 120 seconds
    LLDP interface reinitialisation delay is 2 seconds
```

실무에서는 CDP와 LLDP를 **동시에 활성화**하는 경우도 많습니다. Cisco 장비끼리는 CDP로, 타 벤더 장비와는 LLDP로 정보를 교환합니다.

---

### 면접관: "CDP와 LLDP를 인터페이스 단위로 제어할 수 있습니까? 보안 관점에서 어떤 포트에서는 비활성화해야 합니까?"

네, 둘 다 전역(global) 또는 인터페이스 단위로 제어할 수 있습니다.

#### CDP 제어

```
! 전역 비활성화
SW1(config)# no cdp run

! 전역 활성화 (기본값)
SW1(config)# cdp run

! 특정 인터페이스에서만 비활성화
SW1(config)# interface GigabitEthernet 0/1
SW1(config-if)# no cdp enable

! 특정 인터페이스에서 활성화
SW1(config-if)# cdp enable
```

#### LLDP 제어

LLDP는 전송(transmit)과 수신(receive)을 **개별적으로** 제어할 수 있습니다:

```
! 전역 활성화/비활성화
SW1(config)# lldp run
SW1(config)# no lldp run

! 특정 인터페이스에서 전송만 비활성화
SW1(config-if)# no lldp transmit

! 특정 인터페이스에서 수신만 비활성화
SW1(config-if)# no lldp receive

! 전송과 수신 모두 비활성화
SW1(config-if)# no lldp transmit
SW1(config-if)# no lldp receive
```

#### 보안 관점에서 비활성화해야 하는 포트

CDP/LLDP는 장비의 IP 주소, IOS 버전, 플랫폼 정보 등 민감한 정보를 노출합니다. 따라서 다음 포트에서는 **반드시 비활성화**해야 합니다:

| 포트 유형 | CDP/LLDP | 이유 |
|----------|---------|------|
| 외부 연결 포트 (ISP, 파트너) | 비활성화 | 외부에 내부 장비 정보 노출 방지 |
| 일반 사용자 Access 포트 | 비활성화 권장 | 내부 사용자의 정보 수집 방지 |
| 스위치 간 Trunk 포트 | 활성화 유지 | 네트워크 관리에 필요 |
| 라우터 간 연결 포트 | 상황에 따라 | 내부 네트워크면 활성화, 외부면 비활성화 |

```
! 실무 적용 예시: 모든 Access 포트에서 CDP 비활성화
SW1(config)# interface range FastEthernet 0/1 - 22
SW1(config-if-range)# no cdp enable
SW1(config-if-range)# no lldp transmit
SW1(config-if-range)# no lldp receive
SW1(config-if-range)# exit

! Trunk 포트에서는 유지
SW1(config)# interface range GigabitEthernet 0/1 - 2
SW1(config-if-range)# cdp enable
SW1(config-if-range)# lldp transmit
SW1(config-if-range)# lldp receive
```

공격자가 CDP/LLDP 정보를 수집하면 네트워크 장비의 모델명, IOS 버전, IP 주소를 알 수 있고, 이를 바탕으로 알려진 취약점을 이용한 공격이 가능합니다.

---

### 면접관: "CDP를 이용해서 실제로 트러블슈팅을 하는 시나리오를 하나 설명해 주세요."

실무에서 흔한 시나리오를 하나 들겠습니다.

#### 시나리오: "5층 회의실 PC가 네트워크에 연결이 안 됩니다"

1. **장비실에서 해당 PC가 연결된 스위치 포트 확인**

사용자로부터 PC의 IP 주소(192.168.10.50)를 확인합니다. 코어 스위치에서 해당 IP의 MAC 주소를 찾고, MAC 주소 테이블로 어떤 스위치의 어떤 포트인지 추적합니다.

하지만 네트워크가 복잡하면 MAC 추적이 어려울 수 있습니다. 이때 CDP가 도움됩니다.

2. **스위치 간 연결 관계 확인**

```
! 코어 스위치에서 이웃 확인
CoreSW# show cdp neighbors
Device ID    Local Intrfce   Holdtme    Capability  Platform  Port ID
SW-5F        Gig 0/3         165        S I         WS-C2960  Gig 0/1

! 5층 스위치를 찾았으면, 해당 스위치에 접속
CoreSW# ssh -l admin 192.168.1.5
```

3. **5층 스위치에서 문제 포트 확인**

```
SW-5F# show cdp neighbors
! PC는 CDP를 지원하지 않으므로 나타나지 않음
! 하지만 IP Phone이 연결된 포트는 CDP에 표시됨

SW-5F# show interfaces status
Port      Name          Status       Vlan  Duplex  Speed  Type
Fa0/15    CONF-ROOM-5F  err-disabled 10    auto    auto   10/100BaseTX
```

4. **원인 파악 및 해결**

```
SW-5F# show interfaces FastEthernet 0/15 status err-disabled
Port      Name          Status       Reason               Err-disabled Vlans
Fa0/15    CONF-ROOM-5F  err-disabled psecure-violation    --

! Port Security 위반이 원인 → 다른 MAC 주소 장비를 연결했을 가능성
SW-5F(config)# interface FastEthernet 0/15
SW-5F(config-if)# shutdown
SW-5F(config-if)# no shutdown
```

이처럼 CDP를 활용하면 물리적으로 장비실에 가지 않고도 원격으로 어떤 스위치의 어떤 포트에 문제가 있는지 빠르게 추적할 수 있습니다.

---

### 면접관: "CDP와 LLDP에서 수집되는 정보를 정리하고, 이 정보가 네트워크 관리에 어떻게 활용되는지 말씀해 주세요."

CDP와 LLDP가 수집하는 주요 정보와 활용 방안을 정리하겠습니다:

#### 수집 정보 비교

| 정보 | CDP | LLDP |
|------|-----|------|
| 장비 이름 (Device ID) | O | O |
| 포트 ID (Local/Remote) | O | O |
| IP 주소 | O | O |
| 장비 모델 (Platform) | O | O (System Description) |
| 소프트웨어 버전 | O | O |
| Capabilities (Router/Switch 등) | O | O |
| Native VLAN | O | X (LLDP-MED로 가능) |
| Duplex 정보 | O | X |
| VTP Domain | O | X |
| VLAN 이름 | O (일부) | X |
| 전력 정보 (PoE) | O | O (LLDP-MED) |

#### 네트워크 관리 활용 방안

1. **네트워크 토폴로지 문서화**
   - CDP/LLDP 정보를 수집하여 자동으로 네트워크 다이어그램 생성
   - NMS(Network Management System) 도구들이 CDP/LLDP를 활용

2. **장비 인벤토리 관리**
   - 장비 모델, IOS 버전을 자동 수집하여 관리
   - 보안 패치가 필요한 장비 식별

3. **케이블 연결 검증**
   - 설계 문서와 실제 연결이 일치하는지 확인
   - 잘못된 포트 연결 빠르게 발견

4. **IP Phone 연결 관리**
   - CDP/LLDP-MED를 통해 IP Phone에 Voice VLAN 정보 자동 전달
   - PoE 전력 협상

```
! 네트워크 다이어그램 작성을 위한 정보 수집 스크립트 예시
! 각 스위치에서 실행
SW1# show cdp neighbors detail | include Device|IP address|Interface|Platform
SW1# show lldp neighbors detail | include System Name|Management Address|Interface
```

실무에서는 이런 정보 수집을 Python 스크립트나 Ansible로 자동화하여, 정기적으로 네트워크 토폴로지 변경 사항을 추적하는 경우도 많습니다.

---

### 면접관: "마지막으로, CDP와 LLDP 관련 show 명령어들을 정리해 주세요."

핵심 show 명령어를 정리하겠습니다:

#### CDP 관련 명령어

```
! CDP 전역 상태 (타이머, 버전 등)
SW1# show cdp

! 이웃 장비 요약 정보 (가장 많이 사용)
SW1# show cdp neighbors

! 이웃 장비 상세 정보 (IP, IOS 버전 등)
SW1# show cdp neighbors detail

! 특정 이웃 장비 정보
SW1# show cdp entry SW2

! 특정 인터페이스의 CDP 상태
SW1# show cdp interface GigabitEthernet 0/1

! CDP 트래픽 통계
SW1# show cdp traffic
```

#### LLDP 관련 명령어

```
! LLDP 전역 상태
SW1# show lldp

! 이웃 장비 요약 정보
SW1# show lldp neighbors

! 이웃 장비 상세 정보
SW1# show lldp neighbors detail

! 특정 이웃 장비 정보
SW1# show lldp entry JuniperSW1

! 특정 인터페이스의 LLDP 상태
SW1# show lldp interface GigabitEthernet 0/1

! LLDP 트래픽 통계
SW1# show lldp traffic
```

#### 빠른 참조 표

| 목적 | CDP 명령어 | LLDP 명령어 |
|------|----------|------------|
| 이웃 목록 | show cdp neighbors | show lldp neighbors |
| 이웃 상세 | show cdp neighbors detail | show lldp neighbors detail |
| 전역 상태 | show cdp | show lldp |
| 인터페이스 상태 | show cdp interface | show lldp interface |
| 활성화 | cdp run | lldp run |
| 비활성화 | no cdp run | no lldp run |
| 포트 비활성화 | no cdp enable | no lldp transmit / no lldp receive |

---

> **핵심 정리**: CDP(Cisco 독점)와 LLDP(IEEE 표준)는 직접 연결된 이웃 장비의 정보를 자동으로 수집하는 Layer 2 프로토콜입니다. 네트워크 토폴로지 파악, 트러블슈팅, 인벤토리 관리에 필수적이지만, 보안을 위해 외부 연결 포트와 사용자 Access 포트에서는 반드시 비활성화해야 합니다. 멀티벤더 환경에서는 LLDP를 사용합니다.
