# 03-04. STP 기본 동작

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 - Spanning Tree Protocol

---

### 면접관: "스위치 네트워크에서 이중화를 위해 스위치 간에 여러 경로를 만들었습니다. 그런데 STP가 없으면 어떤 문제가 발생합니까?"

이중화 구성 시 STP가 없으면 **Layer 2 루프(Loop)**가 발생하며, 이는 세 가지 심각한 문제를 일으킵니다.

#### 1. 브로드캐스트 스톰 (Broadcast Storm)

브로드캐스트 프레임이 루프를 따라 무한히 순환합니다. 프레임이 돌 때마다 복제되어 기하급수적으로 증가하고, 결국 네트워크 대역폭을 모두 소비하여 정상 통신이 불가능해집니다.

Layer 3의 IP 패킷은 TTL(Time To Live)이 있어서 무한 루프를 방지하지만, **Layer 2 이더넷 프레임에는 TTL이 없습니다**. 따라서 프레임이 한 번 루프에 빠지면 스스로 멈출 수 있는 방법이 없습니다.

#### 2. MAC 주소 테이블 불안정 (MAC Flapping)

같은 MAC 주소가 여러 포트에서 번갈아 학습됩니다. 스위치의 MAC 주소 테이블이 계속 변경되어 정상적인 유니캐스트 전달도 실패합니다.

#### 3. 중복 프레임 수신

수신 장비가 동일한 프레임을 여러 번 받게 되어 상위 계층 프로토콜에 오류를 일으킬 수 있습니다.

**STP(Spanning Tree Protocol, IEEE 802.1D)**는 이러한 루프를 방지하기 위해, 이중화 경로 중 일부를 논리적으로 차단하여 **루프가 없는 트리 구조**를 만듭니다. 장애 발생 시 차단된 경로를 활성화하여 이중화 효과도 유지합니다.

---

### 면접관: "STP에서 Root Bridge가 어떻게 선출되는지 설명해 주세요. Bridge ID의 구성 요소는 무엇입니까?"

Root Bridge 선출은 **Bridge ID**를 기반으로 이루어집니다.

#### Bridge ID 구성

```
Bridge ID (8 bytes) = Bridge Priority (2 bytes) + MAC Address (6 bytes)
                      ↓
              Priority (4 bits) + Extended System ID (12 bits)
              기본값: 32768      VLAN 번호
```

실제 Bridge Priority는 다음과 같이 계산됩니다:
- 기본 Priority: 32768
- VLAN 1인 경우: 32768 + 1 = **32769**
- VLAN 10인 경우: 32768 + 10 = **32778**

#### Root Bridge 선출 과정

1. 모든 스위치가 자신이 Root Bridge라고 주장하며 **BPDU(Bridge Protocol Data Unit)**를 전송합니다
2. BPDU에는 자신의 Bridge ID가 포함되어 있습니다
3. **가장 낮은 Bridge ID**를 가진 스위치가 Root Bridge가 됩니다
4. Priority가 같으면 **MAC 주소가 가장 낮은** 스위치가 선출됩니다

```
! Root Bridge를 수동으로 지정하려면 Priority를 낮게 설정
SW1(config)# spanning-tree vlan 10 priority 4096

! 또는 매크로 명령어 사용
SW1(config)# spanning-tree vlan 10 root primary    ! Priority를 24576으로 설정
SW2(config)# spanning-tree vlan 10 root secondary  ! Priority를 28672로 설정
```

Priority 값은 **4096의 배수**로만 설정 가능합니다 (0, 4096, 8192, 12288, ..., 61440).

확인 명령어:

```
SW1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    4106
             Address     0011.2233.4455
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106  (priority 4096 sys-id-ext 10)
             Address     0011.2233.4455
```

실무에서는 Root Bridge를 반드시 **성능이 좋은 코어/디스트리뷰션 스위치**로 수동 지정해야 합니다. 자동 선출에 맡기면 MAC 주소가 가장 낮은 가장 오래된 스위치가 Root가 될 수 있습니다.

---

### 면접관: "Root Bridge가 선출된 후, 각 포트의 역할(Root Port, Designated Port, Blocked Port)은 어떻게 결정됩니까?"

Root Bridge 선출 후 나머지 스위치들은 각 포트의 역할을 결정합니다.

#### 포트 역할 결정 규칙

**1) Root Port (RP)**: Non-Root 스위치에서 Root Bridge까지 **최소 비용 경로**의 포트

각 스위치마다 **정확히 하나의 Root Port**가 있습니다.

STP 경로 비용:

| 링크 속도 | STP Cost (Revised) |
|----------|-------------------|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

Root Port 선출 시 동률(tie-breaking) 순서:
1. Root Bridge까지의 **최소 경로 비용**
2. 상대방의 **가장 낮은 Bridge ID** (Sender Bridge ID)
3. 상대방의 **가장 낮은 Port ID** (Port Priority + Port Number)

**2) Designated Port (DP)**: 각 세그먼트(링크)에서 Root Bridge 방향으로 BPDU를 전달하는 포트

각 링크마다 **정확히 하나의 Designated Port**가 있습니다. Root Bridge의 모든 포트는 Designated Port입니다.

**3) Non-Designated Port (Blocked Port)**: Root Port도 Designated Port도 아닌 포트. **차단 상태**로 데이터 프레임을 전달하지 않습니다.

#### 예시 토폴로지

```
        [SW1 - Root Bridge]
        Gi0/1(DP)    Gi0/2(DP)
          |              |
     Cost 4         Cost 4
          |              |
     Gi0/1(RP)     Gi0/1(RP)
        [SW2]          [SW3]
     Gi0/2(DP)     Gi0/2(Blocked)
          |              |
     Cost 4         Cost 4
          +----- SW2-SW3 링크 -----+
```

SW2-SW3 링크에서: SW2의 Bridge ID가 SW3보다 낮다면, SW2의 Gi0/2가 Designated Port가 되고, SW3의 Gi0/2가 Blocked Port가 됩니다.

---

### 면접관: "STP 포트 상태 변화 과정을 설명해 주세요. 포트가 Forwarding 상태가 되기까지 얼마나 걸립니까?"

STP(802.1D)의 포트 상태는 5가지이며, 다음 순서로 전환됩니다:

| 포트 상태 | BPDU 수신 | MAC 학습 | 데이터 전달 | 지속 시간 |
|----------|----------|---------|-----------|----------|
| **Disabled** | X | X | X | 관리자 shutdown |
| **Blocking** | O (수신만) | X | X | Max Age (20초) |
| **Listening** | O (송수신) | X | X | Forward Delay (15초) |
| **Learning** | O (송수신) | O | X | Forward Delay (15초) |
| **Forwarding** | O (송수신) | O | O | 정상 동작 |

포트가 활성화되어 Forwarding 상태가 되기까지의 수렴 시간:

```
Blocking → Listening → Learning → Forwarding
  20초    +   15초    +   15초   = 최대 50초
(Max Age)  (Fwd Delay) (Fwd Delay)
```

**최대 50초**가 걸릴 수 있습니다. 이 시간 동안 해당 포트를 통한 데이터 통신은 불가능합니다.

#### STP 타이머

| 타이머 | 기본값 | 설명 |
|--------|--------|------|
| Hello Time | 2초 | Root Bridge가 BPDU를 보내는 간격 |
| Max Age | 20초 | BPDU를 수신하지 못하면 토폴로지 변경으로 간주하는 시간 |
| Forward Delay | 15초 | Listening → Learning, Learning → Forwarding 전환 시간 |

이 타이머들은 Root Bridge에서 설정하며, 모든 스위치에 BPDU를 통해 전파됩니다.

```
! 타이머 확인
SW1# show spanning-tree vlan 10

! 타이머 변경 (Root Bridge에서만 의미 있음 - 권장하지 않음)
SW1(config)# spanning-tree vlan 10 hello-time 1
SW1(config)# spanning-tree vlan 10 max-age 14
SW1(config)# spanning-tree vlan 10 forward-time 10
```

50초의 수렴 시간은 현대 네트워크에서는 너무 느리기 때문에, 이를 개선한 것이 **RSTP(Rapid STP)**입니다.

---

### 면접관: "RSTP가 기존 STP와 비교해서 어떻게 개선되었는지 설명해 주세요."

**RSTP(Rapid Spanning Tree Protocol, IEEE 802.1w)**는 STP의 수렴 시간을 획기적으로 단축한 프로토콜입니다.

#### 주요 개선 사항

**1) 포트 상태 간소화**

| STP (802.1D) | RSTP (802.1w) | 설명 |
|-------------|--------------|------|
| Disabled | Discarding | 데이터 전달하지 않음 |
| Blocking | Discarding | 데이터 전달하지 않음 |
| Listening | Discarding | 데이터 전달하지 않음 |
| Learning | Learning | MAC 학습은 하지만 전달은 안 함 |
| Forwarding | Forwarding | 정상 전달 |

5개 상태가 3개로 줄었습니다.

**2) 새로운 포트 역할 추가**

| 역할 | 설명 |
|------|------|
| Root Port | STP와 동일 |
| Designated Port | STP와 동일 |
| **Alternate Port** | Root Port의 백업 (Blocking 상태의 다른 스위치로 가는 경로) |
| **Backup Port** | Designated Port의 백업 (같은 세그먼트의 또 다른 포트) |

**3) 빠른 수렴 메커니즘**

- **Proposal/Agreement**: 타이머 기반이 아닌 직접 협상 방식으로 포트 전환
- 수렴 시간이 **1~2초 이내**로 단축 (STP의 50초 대비)
- Edge Port: PC가 연결되는 Access 포트는 즉시 Forwarding으로 전환

```
! RSTP 활성화 (PVST+ → Rapid PVST+)
SW1(config)# spanning-tree mode rapid-pvst

! Edge Port 설정 (PortFast와 동일한 효과)
SW1(config-if)# spanning-tree portfast

! 확인
SW1# show spanning-tree summary
Switch is in rapid-pvst mode
```

CCNA 시험에서는 Cisco의 **Rapid PVST+**가 가장 많이 출제됩니다. 이는 RSTP를 VLAN별로 동작하게 만든 Cisco의 구현입니다.

---

### 면접관: "PortFast와 BPDU Guard에 대해 설명해 주세요. 왜 Access 포트에 설정합니까?"

#### PortFast

일반적으로 스위치 포트가 활성화되면 STP 과정을 거쳐 Forwarding까지 30~50초가 걸립니다. PC를 연결한 포트에서 이 대기 시간은 불필요합니다.

**PortFast**는 포트가 즉시 Forwarding 상태로 전환되게 합니다:

```
! 개별 포트에 설정
SW1(config)# interface FastEthernet 0/1
SW1(config-if)# spanning-tree portfast

! 모든 Access 포트에 전역 설정
SW1(config)# spanning-tree portfast default
```

**주의**: PortFast는 반드시 **종단 장비(PC, 서버)가 연결된 Access 포트에만** 설정해야 합니다. 스위치가 연결된 포트에 PortFast를 설정하면 루프가 발생할 수 있습니다.

#### BPDU Guard

PortFast가 설정된 포트에서 BPDU가 수신되면 (즉, 누군가 스위치를 연결하면), 해당 포트를 **즉시 err-disabled** 상태로 변경합니다:

```
! 개별 포트에 설정
SW1(config)# interface FastEthernet 0/1
SW1(config-if)# spanning-tree bpduguard enable

! 모든 PortFast 포트에 전역 설정
SW1(config)# spanning-tree portfast bpduguard default
```

BPDU Guard가 동작하면:

```
%SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Fa0/1 with BPDU Guard enabled.
Disabling port.
%PM-4-ERR_DISABLE: bpduguard error detected on Fa0/1, putting Fa0/1 in err-disable state
```

| 기능 | 목적 | 적용 위치 |
|------|------|----------|
| PortFast | 즉시 Forwarding 전환 (STP 대기 시간 제거) | 종단 장비 연결 Access 포트 |
| BPDU Guard | 비인가 스위치 연결 방지 | PortFast가 설정된 포트 |
| Root Guard | 비인가 Root Bridge 방지 | Designated Port |

이 세 가지는 STP 보안에서 가장 기본적이면서 중요한 기능들입니다.

---

### 면접관: "현재 네트워크의 STP 상태를 확인하는 명령어와 출력 해석 방법을 보여주세요."

STP 상태 확인의 핵심 명령어는 `show spanning-tree`입니다:

```
SW2# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    4106
             Address     aabb.cc00.0100
             Cost        4
             Port        1 (GigabitEthernet0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     aabb.cc00.0200
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------
Gi0/1               Root FWD 4         128.1    P2p
Gi0/2               Desg FWD 4         128.2    P2p
Fa0/1               Desg FWD 19        128.3    P2p Edge
```

출력 해석:

- **Root ID 섹션**: Root Bridge의 Priority와 MAC 주소. "This bridge is the root" 문구가 있으면 현재 스위치가 Root Bridge
- **Bridge ID 섹션**: 현재 스위치 자신의 정보
- **Cost**: 현재 스위치에서 Root Bridge까지의 총 경로 비용
- **Port**: Root Port 정보

인터페이스 테이블:
- **Role**: Root(Root Port), Desg(Designated Port), Altn(Alternate Port), Back(Backup Port)
- **Sts**: FWD(Forwarding), BLK(Blocking), LRN(Learning), LIS(Listening)
- **Type**: P2p(Point-to-Point), Shr(Shared), Edge(PortFast가 설정된 포트)

추가 유용한 명령어:

```
! 전체 VLAN의 STP 요약
SW2# show spanning-tree summary

! Root Bridge 정보만 확인
SW2# show spanning-tree root

! 특정 포트의 STP 상세 정보
SW2# show spanning-tree interface GigabitEthernet 0/1 detail
```

---

> **핵심 정리**: STP는 L2 루프를 방지하기 위해 이중화 경로의 일부를 차단합니다. Root Bridge 선출(낮은 Bridge ID), 포트 역할 결정(Root/Designated/Blocked), 포트 상태 전환 과정을 이해하는 것이 중요합니다. RSTP는 수렴 시간을 50초에서 1~2초로 단축했으며, PortFast와 BPDU Guard는 Access 포트의 필수 보안 설정입니다.
