# 02-02. STP/RSTP/MST

---

## 면접관: "고객사에서 코어 스위치를 이중화했는데, 연결하자마자 네트워크 전체가 마비됐어요. 모든 PC에서 인터넷이 안 되고, 스위치 CPU가 100%를 찍고 있습니다. 뭐가 문제인 것 같고, 어떻게 해결하시겠어요?"

**브로드캐스트 스톰**이 발생한 상황입니다. 코어 스위치를 이중화하면서 물리적 루프가 생겼고, STP(Spanning Tree Protocol)가 정상 동작하지 않아 브로드캐스트 프레임이 무한히 순환하고 있는 것입니다.

**즉시 조치:**

```
! 1. 이중화로 추가한 링크 중 하나를 물리적으로 분리 (장애 지속 방지)
! 2. 스위치 CPU 사용률 확인
show processes cpu sorted | exclude 0.00
show spanning-tree summary

! 3. STP가 동작 중인지 확인
show spanning-tree vlan 1
```

**근본 원인 분석:**

루프가 발생하는 일반적인 원인들:
1. 신규 스위치에서 STP가 비활성화되어 있었음 (`no spanning-tree vlan X`)
2. BPDU 필터가 Trunk 포트에 잘못 적용되어 BPDU가 차단됨
3. 양쪽 스위치의 STP 모드가 달라서 (PVST+ vs MST) BPDU를 인식하지 못함

**복구 후 정상 설계:**

> 토폴로지 이미지 추가 예정

```
! 코어 스위치 1 (Root Bridge로 지정)
spanning-tree mode rapid-pvst
spanning-tree vlan 1-4094 root primary
! 또는 명시적으로 priority 지정
spanning-tree vlan 1-4094 priority 0

! 코어 스위치 2 (Secondary Root)
spanning-tree mode rapid-pvst
spanning-tree vlan 1-4094 root secondary
! 또는 명시적으로 priority 지정
spanning-tree vlan 1-4094 priority 4096
```

---

## 면접관: "Root Bridge가 어떻게 선출되는지 과정을 설명해주세요. BID가 뭔지도요."

STP에서 Root Bridge는 **BID(Bridge ID)가 가장 낮은** 스위치가 선출됩니다.

**BID 구조 (8바이트):**

```
| Bridge Priority (2바이트) | MAC Address (6바이트) |
```

전통적 STP에서는 위와 같지만, **PVST+에서는 Extended System ID**를 사용합니다:

```
| Priority (4비트) | Extended System ID = VLAN ID (12비트) | MAC Address (6바이트) |
```

- Priority: 4비트이므로 4096 단위로만 설정 가능 (0, 4096, 8192, ..., 61440)
- 기본값: 32768
- PVST+에서 실제 BID = Priority + VLAN ID (예: 32768 + 10 = 32778)

**Root Bridge 선출 과정:**

1. 모든 스위치가 자신을 Root로 간주하고 BPDU를 전송
2. 수신한 BPDU의 Root BID와 자신의 BID를 비교
3. 더 낮은 BID를 가진 스위치를 Root로 인정
4. Root가 아닌 스위치는 자신의 BPDU 생성을 중단하고 Root의 BPDU를 릴레이
5. 최종적으로 가장 낮은 BID를 가진 스위치 하나가 Root Bridge로 수렴

**포트 역할 결정 (Root 선출 후):**

| 포트 역할 | 위치 | 상태 | 설명 |
|-----------|------|------|------|
| **Root Port** | Non-Root 스위치 | Forwarding | Root까지 최단 경로의 포트 |
| **Designated Port** | 각 세그먼트 | Forwarding | 해당 세그먼트에서 Root로의 최단 경로를 제공하는 포트 |
| **Alternate Port** | Non-Root 스위치 | Blocking(STP) / Discarding(RSTP) | Root Port의 백업 |
| **Backup Port** | 같은 스위치 | Blocking | Designated Port의 백업 (허브 환경) |

**Root Port 선출 기준 (순서대로):**

1. Root까지의 최소 Path Cost
2. 상대방 BID가 가장 낮은 포트
3. 상대방 Port ID가 가장 낮은 포트
4. 자신의 Port ID가 가장 낮은 포트

```
! 확인 명령어
show spanning-tree vlan 10
show spanning-tree root
show spanning-tree bridge
```

---

## 면접관: "그런데 고객사에서 '특정 VLAN에서만 간헐적으로 통신이 끊긴다'는 장애 신고가 들어왔어요. 확인해보니 Blocking이어야 할 포트가 Forwarding 상태입니다. 원인이 뭘까요?"

여러 가능성이 있지만, 가장 의심되는 것은 **Root Bridge가 의도하지 않은 스위치로 변경된 것**입니다.

**진단 순서:**

```
! 1. 현재 Root Bridge 확인
show spanning-tree vlan [VLAN-ID]

! Root Bridge의 BID, Priority, MAC 확인
! 의도한 코어 스위치가 Root인지 확인
```

```
! 2. 모든 스위치에서 Root 정보 비교
show spanning-tree root

! 출력 예시:
! Vlan  Root ID    Cost  Time  Age  Dly  Root Port
! V10   4106 aabb.cc00.0100  4  2  20  15  Gi0/1    ← 정상
! V20   32788 aabb.cc00.9900  19  2  20  15  Gi0/2  ← 비정상! Priority가 기본값
```

**가능한 원인들:**

1. **신규 스위치가 낮은 Priority를 가지고 투입됨:** 새 스위치의 MAC이 더 낮아서 Root 탈취
2. **기존 Root Bridge가 재부팅되면서 Priority 설정이 초기화됨:** startup-config에 저장 안 된 경우
3. **Rogue 스위치 연결:** 누군가 개인 스위치를 네트워크에 연결

**해결:**

```
! 정상 Root Bridge에서 Priority 재확인/재설정
spanning-tree vlan 20 priority 0

! 현재 잘못된 Root를 확인 후 해당 스위치에서 Priority 올림
spanning-tree vlan 20 priority 61440
```

**재발 방지 - Root Guard 적용:**

Root Bridge가 연결되면 안 되는 포트(Access 포트, 하위 스위치 방향)에 Root Guard를 설정합니다.

```
! Access 스위치의 다운링크 포트에 적용
interface range GigabitEthernet0/1 - 24
 spanning-tree guard root
```

Root Guard가 적용된 포트로 Superior BPDU(현재 Root보다 낮은 BID)가 수신되면, 해당 포트를 **Root-inconsistent** 상태로 전환하여 Blocking합니다. Superior BPDU 수신이 중단되면 자동 복구됩니다.

---

## 면접관: "STP와 RSTP의 차이를 수렴 시간 관점에서 비교해주세요. 왜 RSTP가 더 빠른가요?"

**STP (IEEE 802.1D)의 수렴:**

STP는 포트 상태 전환에 타이머 기반의 순차적 절차를 따릅니다.

```
Blocking → Listening → Learning → Forwarding
           (15초)      (15초)
```

- **Blocking:** BPDU만 수신. 데이터 전달 안 함
- **Listening:** BPDU 송수신. MAC 학습 안 함. Forward Delay (15초)
- **Learning:** BPDU 송수신. MAC 학습 시작. Forward Delay (15초)
- **Forwarding:** 정상 데이터 전달

총 수렴 시간: **Max Age (20초) + 2 x Forward Delay (15초) = 50초**

장애 발생 시 Root의 BPDU가 Max Age 동안 수신되지 않아야 토폴로지 변경이 시작되므로 최악의 경우 50초간 통신 단절이 발생합니다.

**RSTP (IEEE 802.1w)의 수렴:**

RSTP는 **Proposal/Agreement 메커니즘**으로 포트 역할을 능동적으로 협상합니다.

```
Discarding → Learning → Forwarding
          (즉시~수초)
```

**RSTP가 빠른 핵심 이유:**

1. **Proposal/Agreement:** 인접 스위치 간 직접 협상. 타이머 대기 불필요
   - Root Port가 결정되면 해당 스위치가 다른 포트로 Proposal 전송
   - 상대가 자신의 포트를 Discarding으로 전환 후 Agreement 응답
   - 즉시 Forwarding 전환. 전체 과정이 수초 내 완료

2. **Alternate Port의 즉시 전환:** Root Port 장애 시 Alternate Port가 타이머 없이 즉시 Root Port로 승격

3. **Edge Port:** PortFast와 동일 개념. 즉시 Forwarding

4. **토폴로지 변경 처리:** STP는 TCN BPDU를 Root까지 전달해야 하지만, RSTP는 변경을 감지한 스위치가 직접 TC 비트가 설정된 BPDU를 전파

**RSTP 포트 상태 비교:**

| STP | RSTP | 비고 |
|-----|------|------|
| Disabled | Discarding | 통합 |
| Blocking | Discarding | 통합 |
| Listening | Discarding | 통합 |
| Learning | Learning | 동일 |
| Forwarding | Forwarding | 동일 |

```
! RSTP 활성화 (Cisco에서는 Rapid-PVST+)
spanning-tree mode rapid-pvst
```

---

## 면접관: "BPDU Guard와 BPDU Filter의 차이를 설명하고, 각각 어디에 적용하는지 말씀해주세요. Root Guard와는 어떻게 다른가요?"

세 가지 모두 STP 보호 메커니즘이지만, 목적과 동작이 다릅니다.

### BPDU Guard

**목적:** Access 포트(엔드 디바이스만 연결되어야 하는 포트)에 스위치가 연결되는 것을 방지

**동작:** BPDU를 수신하면 포트를 즉시 **err-disabled** 상태로 전환

```
! 인터페이스별 적용
interface GigabitEthernet0/1
 spanning-tree bpduguard enable

! PortFast가 설정된 모든 포트에 전역 적용 (권장)
spanning-tree portfast bpduguard default
```

**복구:**

```
! 수동 복구
interface GigabitEthernet0/1
 shutdown
 no shutdown

! 자동 복구 설정 (권장)
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

### BPDU Filter

**목적:** BPDU 송수신 자체를 차단

**동작:** 인터페이스 레벨과 전역 레벨에서 동작이 다릅니다.

```
! 인터페이스 레벨: BPDU 완전 무시 (위험!)
interface GigabitEthernet0/1
 spanning-tree bpdufilter enable
! → BPDU를 보내지도, 받아도 무시. STP가 없는 것과 동일
! → 루프 방지 불가. 매우 위험하므로 특수한 경우에만 사용

! 전역 레벨: PortFast 포트에만 적용 (상대적으로 안전)
spanning-tree portfast bpdufilter default
! → PortFast 포트에서 BPDU를 11개까지 보내다가 중단
! → BPDU를 수신하면 PortFast 해제되고 정상 STP 동작으로 전환
```

### Root Guard

**목적:** 특정 포트를 통해 Root Bridge가 변경되는 것을 방지

**동작:** Superior BPDU 수신 시 포트를 **Root-inconsistent** (Blocking과 동일) 상태로 전환. Superior BPDU가 중단되면 자동 복구

```
interface GigabitEthernet0/24
 spanning-tree guard root
```

### 비교 요약

| 기능 | 트리거 | 동작 | 복구 | 적용 위치 |
|------|--------|------|------|----------|
| **BPDU Guard** | 모든 BPDU 수신 | err-disabled | 수동/자동 | Access 포트 (PC, 프린터 등) |
| **BPDU Filter** | N/A (BPDU 차단) | BPDU 무시 | N/A | 특수 환경만 |
| **Root Guard** | Superior BPDU 수신 | Root-inconsistent | 자동 | 하위 스위치 방향 포트 |

추가로 **Loop Guard**도 있습니다:

```
! 단방향 링크 장애로 BPDU 수신이 중단될 때
! Blocking → Forwarding 전환을 방지
spanning-tree loopguard default
```

Loop Guard는 Root Port나 Alternate Port에서 BPDU 수신이 중단되면 **Loop-inconsistent** 상태로 전환합니다. UDLD(Unidirectional Link Detection)와 함께 사용하면 효과적입니다.

---

## 면접관: "고객사가 VLAN이 50개가 넘는데, PVST+로는 VLAN마다 STP 인스턴스가 돌아서 스위치 리소스가 부족하대요. MST로 전환하면 어떻게 되나요? 설계해주세요."

MST(Multiple Spanning Tree, IEEE 802.1s)는 여러 VLAN을 하나의 STP 인스턴스에 매핑하여 리소스를 절약하면서도 경로 분산이 가능한 프로토콜입니다.

**PVST+ vs MST 리소스 비교:**

- PVST+: VLAN 50개 = STP 인스턴스 50개 (CPU, 메모리 50배 사용)
- MST: VLAN 50개를 2~3개 인스턴스에 매핑 가능

**MST 설계 예시:**

코어 스위치 2대(DSW1, DSW2)에서 경로를 분산합니다.

| MST Instance | VLAN | Root Bridge | 용도 |
|-------------|------|-------------|------|
| Instance 0 (IST) | 나머지 전부 | DSW1 | 기본 인스턴스 |
| Instance 1 | 10, 11, 12, ... 29 | DSW1 | 홀수 그룹 |
| Instance 2 | 30, 31, 32, ... 49 | DSW2 | 짝수 그룹 |

> 토폴로지 이미지 추가 예정

DSW1이 Instance 1의 Root, DSW2가 Instance 2의 Root가 되어 트래픽이 분산됩니다.

### 핵심 설정

**주의: MST 설정은 Region 내 모든 스위치에서 동일해야 합니다.**

MST Region 일치 조건 3가지:
1. Region Name
2. Revision Number
3. VLAN-to-Instance 매핑

하나라도 다르면 해당 스위치는 다른 Region으로 인식되어 IST(Instance 0)를 통해서만 통신합니다.

**DSW1 (Instance 1 Root):**

```
spanning-tree mode mst

spanning-tree mst configuration
 name CUSTOMER-NET
 revision 1
 instance 1 vlan 10-29
 instance 2 vlan 30-49
 exit

spanning-tree mst 0 priority 4096
spanning-tree mst 1 priority 0
spanning-tree mst 2 priority 4096
```

**DSW2 (Instance 2 Root):**

```
spanning-tree mode mst

spanning-tree mst configuration
 name CUSTOMER-NET
 revision 1
 instance 1 vlan 10-29
 instance 2 vlan 30-49
 exit

spanning-tree mst 0 priority 8192
spanning-tree mst 1 priority 4096
spanning-tree mst 2 priority 0
```

**Access Switch (모든 스위치 동일):**

```
spanning-tree mode mst

spanning-tree mst configuration
 name CUSTOMER-NET
 revision 1
 instance 1 vlan 10-29
 instance 2 vlan 30-49
 exit
```

**검증 명령어:**

```
show spanning-tree mst configuration
show spanning-tree mst 0
show spanning-tree mst 1
show spanning-tree mst 2
show spanning-tree mst configuration digest
```

**MST 전환 시 주의사항:**

1. **모든 스위치를 동시에 전환:** PVST+와 MST가 혼재하면 PVST Simulation이 동작하여 예기치 않은 포트 Blocking 발생 가능
2. **IST(Instance 0)의 역할 이해:** 매핑되지 않은 모든 VLAN은 자동으로 IST에 포함
3. **최대 인스턴스 수:** 플랫폼마다 다르지만 보통 최대 16개 (0~15)
4. **점검 시간 확보:** STP 모드 변경은 순간적으로 모든 포트가 재수렴하므로 반드시 유지보수 시간에 작업

---

## 면접관: "STP 관련 장애를 사전에 모니터링할 수 있는 방법이 있을까요?"

네, 여러 가지 사전 모니터링 방법이 있습니다.

**1. STP 토폴로지 변경 카운터 모니터링:**

```
show spanning-tree detail | include ieee|from|occur

! 토폴로지 변경 횟수와 마지막 변경 시각 확인
show spanning-tree vlan 10 detail
! Number of topology changes 15 last change occurred 00:05:32 ago
! from GigabitEthernet0/3
```

토폴로지 변경이 빈번하면 불안정한 링크나 flapping 포트가 있다는 신호입니다.

**2. SNMP Trap 설정:**

```
snmp-server enable traps bridge newroot topologychange
snmp-server host 10.1.99.10 version 2c COMMUNITY bridge
```

**3. EEM(Embedded Event Manager) 스크립트:**

```
event manager applet STP_ROOT_CHANGE
 event syslog pattern "SPANTREE-2-ROOTCHANGE"
 action 1.0 syslog msg "WARNING: STP Root Bridge changed!"
 action 2.0 cli command "enable"
 action 3.0 cli command "show spanning-tree root"
 action 4.0 mail server "10.1.99.20" to "admin@company.com" from "switch@company.com" subject "STP Root Change Alert" body "$_cli_result"
```

**4. 주기적 점검 체크리스트:**

```
! Root Bridge가 의도한 스위치인지 확인
show spanning-tree root

! 각 VLAN의 포트 역할/상태 확인
show spanning-tree vlan 10 brief

! err-disabled 포트 확인
show interfaces status err-disabled

! Inconsistent 포트 확인
show spanning-tree inconsistentports
```

정기적으로 이 명령어들의 출력을 비교하면 STP 관련 장애를 사전에 예방할 수 있습니다.

---
