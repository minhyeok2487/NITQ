# 08-01. OSPF Neighbor 장애

---

### 면접관: "고객사 지사에서 '본사 네트워크가 끊겼다'고 연락이 왔습니다. OSPF로 라우팅되고 있다는데, 어디부터 확인하시겠어요?"

OSPF 기반 환경에서 지사 통신 장애라면 가장 먼저 **OSPF Neighbor 상태**를 확인합니다. Neighbor가 정상적으로 FULL 상태인지부터 봐야 합니다.

**1단계: Neighbor 상태 확인**

```
show ip ospf neighbor
```

출력 예시 (장애 상태):
```
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.1.1.1          1   EXSTART/DR      00:00:38    10.0.0.1        GigabitEthernet0/1
```

State가 **FULL**이 아니라면 Neighbor 형성에 문제가 있는 것입니다. 위 예시처럼 EXSTART에 멈춰 있다면 DBD 교환 단계에서 실패하고 있다는 의미입니다.

**2단계: 인터페이스 레벨 OSPF 설정 확인**

```
show ip ospf interface GigabitEthernet0/1
show ip ospf interface brief
```

**3단계: OSPF 프로세스 전체 상태 확인**

```
show ip ospf
show ip protocols
```

**4단계: 실시간 Hello/DBD 교환 디버그**

```
debug ip ospf adj
debug ip ospf hello
```

> 디버그는 트래픽이 많은 운영 장비에서 주의해서 사용합니다. 확인 후 반드시 `undebug all`로 해제합니다.

---

### 면접관: "show ip ospf neighbor 쳤더니 State가 EXSTART에서 멈춰 있습니다. 원인이 뭘까요?"

EXSTART 상태에서 멈추는 가장 대표적인 원인은 **MTU 불일치**입니다.

OSPF는 Neighbor 형성 과정에서 DBD(Database Description) 패킷을 교환하는데, 이 패킷의 크기가 상대방 인터페이스 MTU보다 크면 수신 측에서 패킷을 Drop합니다. 결과적으로 DBD 교환이 완료되지 않아 EXSTART/EXCHANGE 상태에서 무한 반복합니다.

확인 방법:

```
show ip ospf interface GigabitEthernet0/1
! 출력에서 MTU 값 확인

show interface GigabitEthernet0/1 | include MTU
! 양쪽 장비의 MTU 비교
```

양쪽 출력 비교 예시:
```
! 본사 라우터
GigabitEthernet0/1 is up, line protocol is up
  Internet Address 10.0.0.1/30, Area 0, Attached via Network Statement
  ...MTU 1500...

! 지사 라우터
GigabitEthernet0/0 is up, line protocol is up
  Internet Address 10.0.0.2/30, Area 0, Attached via Network Statement
  ...MTU 9000...
```

MTU가 다르면 해결 방법은 두 가지입니다:

**방법 1: MTU를 맞춘다 (권장)**
```
! 지사 라우터
interface GigabitEthernet0/0
 ip mtu 1500
```

**방법 2: OSPF MTU 검사를 비활성화한다 (임시 조치)**
```
interface GigabitEthernet0/0
 ip ospf mtu-ignore
```

> `mtu-ignore`는 근본 해결이 아닙니다. MTU가 실제로 다르면 이후 대용량 LSA 전송 시 문제가 재발할 수 있으므로, MTU를 맞추는 것이 정답입니다.

---

### 면접관: "OSPF Neighbor State에 대해 처음부터 끝까지 설명해 주세요. 각 단계에서 뭘 교환하나요?"

OSPF Neighbor는 총 **7단계** State Machine을 거칩니다.

| State | 설명 | 교환 내용 |
|-------|------|----------|
| **Down** | 상대방에서 Hello를 받지 못한 초기 상태 | 없음 |
| **Attempt** | NBMA 환경에서만 발생. 수동 설정된 Neighbor에게 Hello를 보낸 상태 | Hello (Unicast) |
| **Init** | 상대방의 Hello를 받았으나, 그 Hello의 Neighbor List에 자신이 없는 상태 | Hello 수신 (단방향) |
| **2-Way** | 양방향 Hello 교환 완료. 상대방 Hello의 Neighbor List에 자신의 Router-ID가 포함됨. DR/BDR 선출이 이 단계에서 발생 | Hello (양방향) |
| **ExStart** | Master/Slave 결정. Router-ID가 높은 쪽이 Master. DBD의 Initial Sequence Number 협상 | DBD (빈 DBD로 협상) |
| **Exchange** | 실제 DBD 교환. 자신이 가진 LSA의 헤더(요약) 목록을 상대방에게 전달 | DBD (LSA Header 목록) |
| **Loading** | DBD에서 확인한 LSA 중 자신에게 없거나 오래된 것을 LSR로 요청하고, 상대방이 LSU로 응답 | LSR / LSU / LSAck |
| **Full** | LSDB 동기화 완료. 정상 상태 | 주기적 Hello로 유지 |

핵심 포인트:
- **2-Way에서 멈추는 것은 정상일 수 있습니다.** Broadcast/NBMA 환경에서 DROther끼리는 2-Way까지만 형성합니다. FULL은 DR 또는 BDR과만 형성합니다.
- **ExStart/Exchange에서 멈추면** MTU 불일치 또는 DBD 관련 문제입니다.
- **Loading에서 멈추면** 특정 LSA를 요청했는데 응답이 없는 것이므로, 해당 LSA를 가진 라우터에 문제가 있을 수 있습니다.

---

### 면접관: "MTU는 맞는데 그래도 Neighbor가 안 올라옵니다. Init 상태에서 멈춰 있어요. 뭘 봐야 하나요?"

Init에서 멈춰 있다면 **Hello는 한쪽만 도달하고 있다**는 뜻입니다. 상대방이 보낸 Hello에 내 Router-ID가 없으니 2-Way로 넘어가지 못하는 것입니다.

이 경우 **Hello/Dead Timer 불일치**를 의심합니다.

```
show ip ospf interface GigabitEthernet0/1
```

확인 포인트:
```
! 본사
Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

! 지사
Timer intervals configured, Hello 30, Dead 120, Wait 120, Retransmit 5
```

OSPF는 **Hello Timer와 Dead Timer가 양쪽에서 정확히 일치**해야 Neighbor를 형성합니다. 위 예시에서는 본사가 기본값(10/40)이고 지사가 변경된 값(30/120)이므로, 서로의 Hello를 받아도 Timer가 다르기 때문에 Neighbor로 인정하지 않습니다.

해결:
```
! 지사 라우터 - 타이머를 본사와 맞춤
interface GigabitEthernet0/0
 ip ospf hello-interval 10
 ip ospf dead-interval 40
```

추가로, Init에서 멈추는 다른 원인들:

**ACL이 OSPF 멀티캐스트를 차단하는 경우:**
```
! OSPF는 224.0.0.5 (AllSPFRouters), 224.0.0.6 (AllDRouters) 사용
! 인터페이스에 적용된 ACL 확인
show ip access-lists
show running-config interface GigabitEthernet0/1
```

**한쪽에서만 Passive Interface로 설정된 경우:**
```
show ip ospf interface GigabitEthernet0/1
! "No Hellos (Passive interface)" 표시 확인

! 또는
show ip protocols
! Passive Interface 목록 확인
```

---

### 면접관: "Timer도 맞고, MTU도 맞고, Passive도 아닌데 Neighbor가 Down입니다. 또 뭐가 있을까요?"

이 경우 **OSPF Network Type 불일치**를 확인합니다. 양쪽 인터페이스의 Network Type이 다르면 Hello 간격이 달라지거나 DR/BDR 선출 동작이 달라져서 Neighbor 형성에 실패할 수 있습니다.

```
show ip ospf interface GigabitEthernet0/1 | include Network Type
```

Network Type별 특성:

| Network Type | Hello/Dead | DR/BDR | 멀티캐스트/유니캐스트 |
|-------------|------------|--------|-------------------|
| **Broadcast** | 10/40 | 있음 | 멀티캐스트 |
| **Non-Broadcast (NBMA)** | 30/120 | 있음 | 유니캐스트 |
| **Point-to-Point** | 10/40 | 없음 | 멀티캐스트 |
| **Point-to-Multipoint** | 30/120 | 없음 | 멀티캐스트 |

예를 들어, 한쪽이 Broadcast(Hello 10초)이고 상대가 Point-to-Multipoint(Hello 30초)이면 Timer가 자동으로 달라지므로 Neighbor 형성이 실패합니다.

또 하나 자주 놓치는 것: **한쪽은 Point-to-Point, 한쪽은 Broadcast**인 경우, Timer는 같아도 DR 선출 동작 차이로 인해 DBD 교환에서 문제가 생길 수 있습니다.

해결:
```
interface GigabitEthernet0/1
 ip ospf network point-to-point
! 양쪽 동일하게 맞춤
```

---

### 면접관: "그럼 Area 불일치나 Authentication 불일치도 있을 수 있죠? 이것들은 어떤 증상으로 나타나나요?"

맞습니다. 이 두 가지도 OSPF Neighbor 형성 실패의 주요 원인입니다.

#### Area 불일치

양쪽 인터페이스가 서로 다른 Area에 속해 있으면 Hello를 받아도 Neighbor를 형성하지 않습니다. 로그에 다음과 같은 메시지가 나타납니다:

```
%OSPF-4-BADAREA: OSPF received a packet from 10.0.0.2, GigabitEthernet0/1
  area 1 does not match configured area 0
```

확인:
```
show ip ospf interface brief
! Area 할당 확인

show running-config | section router ospf
! network 문 확인
```

```
! 본사
router ospf 1
 network 10.0.0.0 0.0.0.3 area 0

! 지사 (잘못된 설정)
router ospf 1
 network 10.0.0.0 0.0.0.3 area 1
```

#### Authentication 불일치

OSPF Authentication이 한쪽만 설정되어 있거나, Key가 다르면 Hello를 수신해도 인증 실패로 무시합니다.

```
debug ip ospf adj
```

로그 예시:
```
%OSPF-4-NOVALIDKEY: No valid authentication key found on interface GigabitEthernet0/1
! 또는
%OSPF-4-AUTHFAIL: OSPF authentication failure on GigabitEthernet0/1
```

확인:
```
show ip ospf interface GigabitEthernet0/1
! Authentication 타입 및 Key ID 확인

show running-config interface GigabitEthernet0/1
```

Authentication 설정 예시 (양쪽 동일하게):
```
! MD5 Authentication
interface GigabitEthernet0/1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 MySecretKey

! 또는 Area 단위로
router ospf 1
 area 0 authentication message-digest
```

추가로, **Subnet Mask 불일치**도 자주 발생합니다. 한쪽이 /30이고 상대가 /24이면 Hello 패킷의 Network Mask 필드가 달라서 Neighbor를 형성하지 않습니다. (단, Point-to-Point Network Type에서는 Mask를 비교하지 않습니다.)

```
show ip ospf interface GigabitEthernet0/1
! Network Mask 확인
```

---

### 면접관: "OSPF 장애가 해결됐는데, 이런 문제가 반복되지 않으려면 어떻게 해야 하나요?"

OSPF Neighbor 장애를 사전에 방지하기 위한 체크리스트와 모니터링 전략입니다.

#### 설정 표준화 체크리스트

| 항목 | 확인 내용 | 확인 명령어 |
|------|----------|-----------|
| **MTU** | 양쪽 인터페이스 MTU 동일 | `show interface \| include MTU` |
| **Hello/Dead Timer** | 양쪽 동일 (수동 변경 시 특히 주의) | `show ip ospf interface` |
| **Network Type** | 양쪽 동일 | `show ip ospf interface \| include Network Type` |
| **Area** | 동일 링크의 양쪽 인터페이스가 같은 Area | `show ip ospf interface brief` |
| **Authentication** | 타입/Key 동일 | `show ip ospf interface` |
| **Subnet Mask** | 양쪽 동일 (P2P 제외) | `show ip interface brief` |
| **Passive Interface** | 연결 인터페이스에 설정되지 않았는지 | `show ip protocols` |

#### 모니터링

```
! SNMP Trap 설정 - Neighbor 상태 변경 시 알림
snmp-server enable traps ospf state-change
snmp-server enable traps ospf errors

! Syslog에서 OSPF 관련 메시지 모니터링
! %OSPF-5-ADJCHG: Neighbor 상태 변경 로그를 NMS에서 감시

! EEM Script - Neighbor Down 시 자동 알림
event manager applet OSPF-NEIGHBOR-DOWN
 event syslog pattern "%OSPF-5-ADJCHG.*to DOWN"
 action 1.0 syslog msg "CRITICAL: OSPF Neighbor went DOWN"
```

#### 설계 권장사항

1. **Point-to-Point 링크에서는 `ip ospf network point-to-point` 명시** - 불필요한 DR/BDR 선출을 피하고, Network Type 불일치 리스크를 줄입니다.
2. **Authentication은 Area 단위로 일괄 적용** - 인터페이스 단위 설정은 누락 가능성이 높습니다.
3. **변경 작업 시 양쪽 동시 작업** - 한쪽만 변경하면 일시적 Neighbor Down이 발생합니다.
4. **변경 전 `show ip ospf neighbor` 상태 저장** - 변경 후 비교할 기준선(Baseline)을 확보합니다.
5. **BFD(Bidirectional Forwarding Detection) 연동** - OSPF Dead Timer(기본 40초)보다 빠르게 장애를 감지할 수 있습니다.

```
! BFD 설정 예시
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3

router ospf 1
 bfd all-interfaces
```

---

### 면접관: "마지막으로, OSPF Neighbor가 Flapping(반복적으로 올라갔다 내려갔다)하는 케이스는 어떻게 접근하나요?"

Neighbor Flapping은 단순 Down보다 원인 파악이 어렵습니다. 접근 순서는 다음과 같습니다.

**1단계: Flapping 패턴 확인**

```
show logging | include OSPF
! Neighbor Up/Down 시간 패턴 분석 - 주기적인지, 불규칙한지
```

**2단계: 물리 계층 확인**

```
show interface GigabitEthernet0/1
! CRC errors, input errors, output drops 확인
! Resets 카운터 확인

show controllers GigabitEthernet0/1
! 광모듈 Rx/Tx 레벨 확인 (SFP)
```

Flapping이 주기적이면 물리 계층 불안정(케이블 불량, SFP 노후, 중간 스위치 포트 불량)을 강하게 의심합니다.

**3단계: CPU 과부하 확인**

```
show processes cpu history
show processes cpu sorted | head 10
```

CPU 사용률이 높으면 Hello 패킷 처리가 지연되어 Dead Timer 만료로 Neighbor가 Drop됩니다. 특히 BGP가 대량의 경로를 처리하는 중이거나, ACL 로깅이 과도할 때 발생합니다.

**4단계: SPF Throttle 확인**

```
show ip ospf
! SPF schedule delay, Hold time, Maximum wait time 확인
```

대규모 OSPF 환경에서 LSA가 빈번하게 변경되면 SPF 계산이 과도하게 반복되어 CPU 부하를 유발합니다. SPF Throttle 튜닝이 필요할 수 있습니다.

```
router ospf 1
 timers throttle spf 50 200 5000
! Initial delay: 50ms, Hold: 200ms, Max wait: 5000ms
```

**5단계: 구간별 패킷 캡처**

위 모든 것이 정상이면, 중간 경로(L2 스위치 등)에서 OSPF Hello 패킷이 간헐적으로 Drop되는지 패킷 캡처로 확인합니다.

```
! 라우터에서 OSPF 패킷 캡처
monitor capture CAP interface GigabitEthernet0/1 both
monitor capture CAP match ipv4 protocol ospf any any
monitor capture CAP start
! ... 재현 대기 ...
monitor capture CAP stop
show monitor capture CAP buffer brief
```

---
