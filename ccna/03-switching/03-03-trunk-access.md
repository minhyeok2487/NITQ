# 03-03. Trunk와 Access 포트

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 - Trunk/Access 포트 구성

---

### 면접관: "스위치 두 대가 있고 각각에 VLAN 10, 20이 설정되어 있습니다. 두 스위치를 연결해서 같은 VLAN끼리 통신하게 하려면 어떻게 해야 합니까?"

두 스위치 간에 여러 VLAN 트래픽을 전달하려면 **Trunk 링크**를 구성해야 합니다.

먼저 Access 포트와 Trunk 포트의 차이를 설명드리겠습니다:

| 구분 | Access 포트 | Trunk 포트 |
|------|------------|-----------|
| VLAN 수 | 하나의 VLAN만 소속 | 여러 VLAN의 트래픽 전달 |
| 태깅 | 태그 없이 프레임 전송 | 802.1Q 태그를 붙여서 전송 |
| 연결 대상 | PC, 서버 등 종단 장비 | 스위치, 라우터 등 네트워크 장비 |
| 용도 | 종단 장비 연결 | 스위치 간 연결, 라우터 연결 |

만약 Trunk 없이 각 VLAN마다 별도의 물리적 링크를 연결한다면, VLAN이 10개면 케이블도 10개가 필요합니다. Trunk를 사용하면 **하나의 물리적 링크로 모든 VLAN 트래픽**을 전달할 수 있습니다.

---

### 면접관: "802.1Q 태깅이 정확히 어떻게 동작하는지 설명해 주세요. 프레임에 어떤 변화가 생깁니까?"

802.1Q는 IEEE 표준 트렁킹 프로토콜로, 이더넷 프레임에 **4바이트 태그**를 삽입합니다.

프레임 구조의 변화:

```
[일반 이더넷 프레임]
| Dst MAC | Src MAC | EtherType | Data | FCS |

[802.1Q 태그가 추가된 프레임]
| Dst MAC | Src MAC | 802.1Q Tag (4B) | EtherType | Data | FCS |
                     ↓
              | TPID (2B) | TCI (2B) |
              | 0x8100    | PRI(3bit) | CFI(1bit) | VLAN ID(12bit) |
```

핵심 필드 설명:

- **TPID (Tag Protocol Identifier)**: 0x8100 값으로, 이 프레임이 802.1Q 태그를 포함하고 있음을 표시합니다
- **PRI (Priority)**: 3비트, QoS용 CoS(Class of Service) 값 (0~7)
- **VLAN ID**: 12비트, VLAN 번호를 나타냄 (0~4095)

동작 과정:

1. PC가 Access 포트로 **태그 없는 일반 프레임**을 보냅니다
2. 스위치가 프레임을 수신하면, 해당 포트의 VLAN 정보를 내부적으로 인식합니다
3. 이 프레임을 Trunk 포트로 내보낼 때 **802.1Q 태그를 삽입**합니다
4. 상대 스위치가 Trunk 포트로 수신하면 태그를 확인하고, 해당 VLAN의 Access 포트로 보낼 때 **태그를 제거**합니다

이렇게 태그를 넣고 빼는 과정을 통해 하나의 물리적 링크에서 여러 VLAN을 구분할 수 있습니다.

---

### 면접관: "Native VLAN이 무엇이고, 왜 중요합니까? Native VLAN 불일치가 발생하면 어떤 문제가 생깁니까?"

**Native VLAN**은 802.1Q 트렁크에서 **태그를 붙이지 않고 전송하는 VLAN**입니다. 기본값은 VLAN 1입니다.

Trunk 포트로 나가는 프레임 중에서:
- Native VLAN에 속한 프레임: **태그 없이** 전송됩니다
- 그 외 VLAN의 프레임: **802.1Q 태그를 붙여서** 전송됩니다

Native VLAN이 존재하는 이유는 802.1Q를 지원하지 않는 구형 장비와의 호환성 때문입니다.

#### Native VLAN 불일치 문제

양쪽 스위치의 Native VLAN이 다르면 심각한 문제가 발생합니다:

```
[SW1] Trunk (Native VLAN = 1) -------- Trunk (Native VLAN = 10) [SW2]
```

이 경우:
- SW1이 VLAN 1 프레임을 태그 없이 전송
- SW2는 태그 없는 프레임을 Native VLAN 10으로 인식
- 결과: VLAN 1의 트래픽이 VLAN 10에 들어가는 **VLAN 누출(VLAN leaking)** 발생

Cisco 스위치는 CDP를 통해 이를 감지하고 경고 메시지를 출력합니다:

```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on
GigabitEthernet0/1 (1), with SW2 GigabitEthernet0/1 (10).
```

#### 보안 모범 사례

```
! Native VLAN을 사용하지 않는 VLAN으로 변경
SW1(config)# interface GigabitEthernet 0/1
SW1(config-if)# switchport trunk native vlan 999

! 양쪽 스위치 모두 동일하게 설정해야 함
SW2(config)# interface GigabitEthernet 0/1
SW2(config-if)# switchport trunk native vlan 999
```

Native VLAN을 VLAN 1이 아닌 사용하지 않는 VLAN(예: 999)으로 변경하면 VLAN Hopping 공격을 방지할 수 있습니다.

---

### 면접관: "Trunk를 실제로 설정하는 CLI 명령어를 보여주시고, DTP에 대해서도 설명해 주세요."

#### Trunk 포트 설정

```
! SW1 설정
SW1(config)# interface GigabitEthernet 0/1
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# exit

! SW2 설정 (동일하게)
SW2(config)# interface GigabitEthernet 0/1
SW2(config-if)# switchport trunk encapsulation dot1q
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk allowed vlan 10,20,30,99
SW2(config-if)# switchport trunk native vlan 99
SW2(config-if)# exit
```

참고로 최신 스위치에서 802.1Q만 지원하는 경우 `switchport trunk encapsulation dot1q` 명령어는 필요 없을 수 있습니다.

#### DTP (Dynamic Trunking Protocol)

DTP는 Cisco 독점 프로토콜로, 스위치 포트가 상대방과 자동으로 trunk 협상을 할 수 있게 합니다.

| 모드 | 설명 | DTP 전송 |
|------|------|---------|
| `switchport mode trunk` | 무조건 trunk | DTP 전송함 |
| `switchport mode access` | 무조건 access | DTP 전송하지 않음 |
| `switchport mode dynamic desirable` | 적극적으로 trunk 협상 시도 | DTP 전송함 |
| `switchport mode dynamic auto` | 상대가 요청하면 trunk 수락 (기본값) | DTP 전송함 |

DTP 협상 결과표:

| | trunk | desirable | auto | access |
|---|---|---|---|---|
| **trunk** | Trunk | Trunk | Trunk | 제한적 |
| **desirable** | Trunk | Trunk | Trunk | Access |
| **auto** | Trunk | Trunk | Access | Access |
| **access** | 제한적 | Access | Access | Access |

**보안상 DTP는 비활성화하는 것이 권장됩니다:**

```
! Trunk 포트에서 DTP 비활성화
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport nonegotiate

! Access 포트에서 DTP 비활성화
SW1(config-if)# switchport mode access
SW1(config-if)# switchport nonegotiate
```

`switchport nonegotiate` 명령으로 DTP 프레임 전송을 막아서 불필요한 trunk 협상을 방지합니다. 공격자가 DTP를 이용해 자신의 포트를 trunk로 만드는 것을 차단할 수 있습니다.

---

### 면접관: "Trunk에서 allowed VLAN을 설정하는 이유와 관련 명령어를 정리해 주세요."

기본적으로 Trunk 포트는 **모든 VLAN(1~4094)**의 트래픽을 전달합니다. 하지만 필요 없는 VLAN 트래픽까지 전달하면:

1. **대역폭 낭비**: 불필요한 브로드캐스트가 trunk를 통해 전달됨
2. **보안 위험**: 해당 스위치에 없는 VLAN 트래픽까지 노출
3. **STP 부하**: 불필요한 VLAN의 STP 계산까지 수행

따라서 실제 필요한 VLAN만 allowed 목록에 포함시킵니다:

```
! Trunk에서 허용할 VLAN 지정
SW1(config-if)# switchport trunk allowed vlan 10,20,30,99

! VLAN 추가
SW1(config-if)# switchport trunk allowed vlan add 40

! VLAN 제거
SW1(config-if)# switchport trunk allowed vlan remove 30

! 기본값으로 복원 (모든 VLAN 허용)
SW1(config-if)# switchport trunk allowed vlan all
```

주의할 점은, `switchport trunk allowed vlan 10,20`이라고 입력하면 기존 설정을 **덮어쓰기** 합니다. 기존 목록에 추가하려면 반드시 **add** 키워드를 사용해야 합니다. 이것을 모르면 기존에 허용되었던 VLAN이 갑자기 통신이 안 되는 장애가 발생할 수 있습니다.

---

### 면접관: "Trunk 설정이 제대로 되었는지 확인하는 명령어들을 보여주세요."

Trunk 검증에 사용하는 주요 명령어들입니다:

#### show interfaces trunk

```
SW1# show interfaces trunk

Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gi0/1       10,20,30,99

Port        Vlans allowed and active in management domain
Gi0/1       10,20,30,99

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/1       10,20,30,99
```

이 출력에서 확인할 사항:

1. **Status**: trunking 상태인지
2. **Native vlan**: 양쪽이 동일한지
3. **Vlans allowed**: 필요한 VLAN만 허용되었는지
4. **Vlans in spanning tree forwarding**: 실제로 트래픽이 전달되는 VLAN

#### show interfaces switchport (특정 포트)

```
SW1# show interfaces GigabitEthernet 0/1 switchport
Name: Gi0/1
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 99 (MANAGEMENT)
Administrative Native VLAN tagging: enabled
```

여기서 **Administrative Mode**와 **Operational Mode**가 모두 trunk인지 확인합니다. Administrative는 설정값이고, Operational은 실제 동작 상태입니다. 설정은 trunk인데 Operational이 다르다면 상대방과의 협상에 문제가 있는 것입니다.

---

### 면접관: "실무에서 Trunk 관련 트러블슈팅을 한다면, 어떤 점들을 순서대로 확인하시겠습니까?"

Trunk 트러블슈팅 체크리스트를 순서대로 정리하겠습니다:

#### 점검 순서

1. **물리적 연결 확인**: 포트 LED가 녹색인지, 케이블 문제는 없는지
2. **양쪽 모두 Trunk 모드인지 확인**

```
SW1# show interfaces Gi0/1 switchport | include Mode
Administrative Mode: trunk
Operational Mode: trunk
```

3. **Native VLAN 일치 여부 확인**

```
SW1# show interfaces trunk | include native
! 양쪽 스위치에서 동일한 Native VLAN인지 확인
```

4. **Allowed VLAN 확인**: 필요한 VLAN이 양쪽 모두에서 허용되었는지

```
SW1# show interfaces trunk | begin allowed
```

5. **VLAN이 실제로 존재하는지 확인**: Trunk에서 허용했더라도, 해당 VLAN이 스위치에 생성되어 있지 않으면 트래픽 전달이 안 됩니다

```
SW1# show vlan brief | include active
```

6. **STP에서 차단되고 있지 않은지 확인**

```
SW1# show spanning-tree vlan 10
```

| 증상 | 가능한 원인 | 확인 방법 |
|------|-----------|----------|
| 특정 VLAN만 통신 안 됨 | Allowed VLAN에서 누락 | show interfaces trunk |
| 모든 VLAN 통신 안 됨 | Trunk 자체가 안 됨 | show interfaces switchport |
| 간헐적 통신 장애 | Native VLAN 불일치 | CDP 경고 메시지 확인 |
| VLAN이 pruned 상태 | VTP pruning 또는 STP | show interfaces trunk 마지막 섹션 |

체계적으로 한 단계씩 확인하면 대부분의 Trunk 문제를 해결할 수 있습니다.

---

> **핵심 정리**: Access 포트는 하나의 VLAN, Trunk 포트는 여러 VLAN의 트래픽을 802.1Q 태그로 구분하여 전달합니다. Native VLAN 불일치, DTP 보안, allowed VLAN 관리가 핵심 포인트이며, 실무에서는 DTP를 비활성화하고 필요한 VLAN만 trunk에 허용하는 것이 모범 사례입니다.
