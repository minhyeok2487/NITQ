# 06-01. HSRP / VRRP / GLBP

---

### 면접관: "고객사에서 코어 스위치 1대가 장애 나면서 전체 사무실 네트워크가 30분 동안 다운됐대요. 원인을 보니까 PC들의 기본 게이트웨이가 장애 난 스위치 IP였던 거예요. 이런 상황 어떻게 해결하시겠어요?"

네, 전형적인 게이트웨이 단일 장애점(SPOF) 문제입니다. 현재 구조는 PC들이 하나의 물리 스위치 IP를 기본 게이트웨이로 사용하고 있어서, 해당 장비가 죽으면 모든 트래픽이 빠져나갈 수가 없는 거죠.

해결 방안으로 **HSRP(Hot Standby Router Protocol)**를 적용해서 두 대의 L3 스위치가 하나의 **가상 IP(Virtual IP)**를 공유하도록 설계하겠습니다. PC들은 가상 IP를 게이트웨이로 사용하므로, Active 장비가 죽어도 Standby 장비가 즉시 역할을 이어받아서 서비스 중단을 최소화할 수 있습니다.

> 토폴로지 이미지 추가 예정

#### 핵심 설계

- **DSW1** (Active) / **DSW2** (Standby) 두 대의 L3 스위치
- VLAN 10 (사무실): 가상 IP `10.10.10.1`을 HSRP로 공유
- PC 게이트웨이: `10.10.10.1` (가상 IP)
- DSW1 Priority 110 (Active), DSW2 Priority 기본값 100 (Standby)

#### 핵심 설정

**DSW1 (Active 역할):**
```
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt
 standby 10 timers 1 3
```

**DSW2 (Standby 역할):**
```
interface Vlan10
 ip address 10.10.10.3 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 100
 standby 10 preempt
 standby 10 timers 1 3
```

핵심은 PC들이 `.1`(가상 IP)만 바라보면 되고, 실제 누가 Active인지는 HSRP가 자동으로 관리해주는 겁니다. Hello 타이머를 1초, Hold 타이머를 3초로 설정해서 장애 감지 시간을 단축했습니다.

---

### 면접관: "HSRP 말고 VRRP도 있고 GLBP도 있잖아요. 차이점이 뭐예요? 왜 HSRP를 선택하셨어요?"

세 프로토콜 모두 게이트웨이 이중화라는 목적은 같지만, 중요한 차이가 있습니다.

| 항목 | HSRP | VRRP | GLBP |
|------|------|------|------|
| **표준** | Cisco 독자 | IEEE RFC 5798 | Cisco 독자 |
| **역할 명칭** | Active / Standby | Master / Backup | AVG / AVF |
| **가상 MAC** | `0000.0c9f.fxxx` (v1) / `0000.0c9f.fxxx` (v2) | `0000.5e00.01xx` | `0007.b400.xxyy` |
| **가상 IP = 실제 IP** | 불가 | 가능 (Master가 자기 IP를 VIP로 사용 가능) | 불가 |
| **부하 분산** | 불가 (VLAN 분리로 우회) | 불가 | **네이티브 지원** |
| **Preempt 기본값** | 비활성 | **활성** | 비활성 |
| **Multicast 주소** | 224.0.0.2 (v1) / 224.0.0.102 (v2) | 224.0.0.18 | 224.0.0.102 |

HSRP를 선택한 이유는 **고객사 환경이 전부 Cisco 장비**이기 때문입니다. Cisco 환경에서는 HSRP가 가장 안정적이고, TAC 지원도 원활합니다. 만약 멀티벤더 환경이었다면 VRRP를 권장했을 것이고, 부하 분산이 필요했다면 GLBP를 고려했을 겁니다.

VRRP에서 주의할 점은 **Preempt가 기본 활성화**라는 겁니다. HSRP는 명시적으로 `preempt`를 켜줘야 하는데, VRRP는 기본으로 켜져 있어서 의도치 않은 전환이 발생할 수 있습니다.

---

### 면접관: "좋아요. 그런데 구축 후에 고객사에서 연락이 왔어요. DSW1 업링크가 끊어졌는데 HSRP Active가 DSW2로 안 넘어간대요. 어디부터 확인하시겠어요?"

이건 전형적인 **HSRP Tracking 미설정** 문제입니다. HSRP는 기본적으로 해당 VLAN 인터페이스의 상태만 모니터링하거든요. 업링크(예: 코어 라우터로 가는 인터페이스)가 죽어도 VLAN 인터페이스 자체는 살아 있으니까 Active를 유지하는 겁니다.

결과적으로 DSW1이 Active인데 업링크가 끊어져서, 트래픽이 DSW1로 왔다가 빠져나갈 길이 없는 **블랙홀** 상태가 됩니다.

#### 확인 순서

**1단계: HSRP 상태 확인**
```
DSW1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State   Active          Standby         Virtual IP
Vl10        10   110 P Active  local           10.10.10.3      10.10.10.1
```
- DSW1이 여전히 Active인 것을 확인

**2단계: 업링크 상태 확인**
```
DSW1# show ip interface brief | include GigabitEthernet0/1
GigabitEthernet0/1     172.16.1.1      YES NVRAM  down         down
```
- 업링크가 down인데도 Active를 유지하고 있음

**3단계: Tracking 설정 확인**
```
DSW1# show standby vlan 10
  Track object 1 state Down decrement 20
```
- Tracking이 아예 없거나, decrement 값이 부족해서 Priority가 Standby보다 여전히 높은 상태

#### 해결: Interface Tracking 설정

```
! DSW1
track 1 interface GigabitEthernet0/1 line-protocol

interface Vlan10
 standby 10 track 1 decrement 20
```

이렇게 하면 업링크(`Gi0/1`)가 다운되면 Priority가 110에서 90으로 떨어지고, DSW2(Priority 100)가 Active를 가져갑니다. **decrement 값은 반드시 두 장비의 Priority 차이(여기서는 10)보다 크게 설정**해야 합니다.

추가로 IP SLA Tracking을 쓰면 더 정밀한 모니터링이 가능합니다:

```
ip sla 1
 icmp-echo 172.16.1.254
 frequency 5
ip sla schedule 1 life forever start-time now

track 1 ip sla 1 reachability

interface Vlan10
 standby 10 track 1 decrement 20
```

이렇게 하면 업링크 인터페이스가 UP이더라도 실제 넥스트홉까지 통신이 안 되면 Failover가 동작합니다.

---

### 면접관: "Preempt를 설정했다고 했는데, 정확히 어떻게 동작하는 거예요? Preempt 없으면 어떻게 되죠?"

Preempt는 **"내 Priority가 더 높으면 Active를 강제로 뺏어오겠다"**는 설정입니다.

**Preempt가 없는 경우의 시나리오:**
1. DSW1(Priority 110)이 Active, DSW2(Priority 100)가 Standby
2. DSW1 장애 발생 -> DSW2가 Active 승격
3. DSW1 복구 -> **DSW2가 계속 Active 유지** (Priority가 낮은데도)
4. DSW1은 Standby로 남음

이유는 HSRP의 기본 동작이 **"현재 Active가 살아있으면 건드리지 않는다"**이기 때문입니다. 안정성을 위한 설계죠. Active 전환 자체가 순간적인 트래픽 끊김을 유발하니까요.

**Preempt가 있는 경우:**
1. DSW1(Priority 110)이 Active, DSW2(Priority 100)가 Standby
2. DSW1 장애 발생 -> DSW2가 Active 승격
3. DSW1 복구 -> DSW1이 **Active를 다시 가져옴** (Priority가 더 높으므로)

실무에서는 `preempt delay` 를 함께 설정하는 것을 권장합니다:

```
standby 10 preempt delay minimum 30 reload 60
```

- `minimum 30`: Preempt 조건 충족 후 30초 대기 후 Active 전환 (라우팅 테이블 수렴 시간 확보)
- `reload 60`: 장비 리부팅 후 60초 대기 후 Active 전환 (OSPF 등 라우팅 프로토콜이 완전히 수렴할 때까지 대기)

이 딜레이 없이 바로 Preempt 하면, 라우팅이 아직 수렴 안 된 상태에서 Active를 가져와서 오히려 블랙홀이 발생할 수 있습니다.

---

### 면접관: "이해했어요. 그런데 고객사가 '지금 구조는 1대는 놀고 있는 거 아니냐, 양쪽 스위치 다 트래픽 처리하게 할 수 없냐'고 물어봐요. 어떻게 하시겠어요?"

두 가지 방법이 있습니다.

#### 방법 1: HSRP VLAN별 Active 분산 (권장, 안정적)

VLAN 단위로 Active 장비를 다르게 지정해서 양쪽 모두 트래픽을 처리하게 합니다.

```
! DSW1 - VLAN 10은 Active, VLAN 20은 Standby
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 110
 standby 10 preempt delay minimum 30

interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 90
 standby 20 preempt delay minimum 30
```

```
! DSW2 - VLAN 10은 Standby, VLAN 20은 Active
interface Vlan10
 ip address 10.10.10.3 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.1
 standby 10 priority 100
 standby 10 preempt delay minimum 30

interface Vlan20
 ip address 10.10.20.3 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 110
 standby 20 preempt delay minimum 30
```

> 토폴로지 이미지 추가 예정

이 방식은 **STP Root Bridge도 HSRP Active와 일치시키는 것이 중요**합니다. 그렇지 않으면 HSRP Active는 DSW1인데, STP 때문에 트래픽이 DSW2로 돌아가는 비효율이 발생합니다:

```
! DSW1 - VLAN 10의 STP Root
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 8192

! DSW2 - VLAN 20의 STP Root
spanning-tree vlan 10 priority 8192
spanning-tree vlan 20 priority 4096
```

#### 방법 2: GLBP (부하 분산 네이티브 지원)

GLBP는 하나의 가상 IP에 대해 **여러 장비가 동시에 트래픽을 처리**합니다.

```
! DSW1
interface Vlan10
 ip address 10.10.10.2 255.255.255.0
 glbp 10 ip 10.10.10.1
 glbp 10 priority 110
 glbp 10 preempt
 glbp 10 load-balancing round-robin

! DSW2
interface Vlan10
 ip address 10.10.10.3 255.255.255.0
 glbp 10 ip 10.10.10.1
 glbp 10 priority 100
 glbp 10 preempt
 glbp 10 load-balancing round-robin
```

GLBP의 역할 구조:
- **AVG (Active Virtual Gateway)**: Priority가 높은 DSW1이 담당. ARP 응답을 관리
- **AVF (Active Virtual Forwarder)**: DSW1, DSW2 모두 AVF로 동작. 실제 트래픽 전달

PC가 게이트웨이 `10.10.10.1`에 ARP 요청을 보내면, AVG가 **라운드로빈으로 다른 가상 MAC 주소를 응답**해서 자연스럽게 부하를 분산합니다.

| 부하 분산 방식 | 설명 |
|---|---|
| `round-robin` | ARP 응답마다 다른 AVF의 MAC을 번갈아 응답 |
| `weighted` | 가중치 기반 분산 |
| `host-dependent` | 출발지 MAC 기준으로 항상 같은 AVF 할당 |

다만 GLBP는 Cisco 전용이고 HSRP보다 복잡도가 높아서, 실무에서는 **방법 1(VLAN별 HSRP Active 분산)**을 더 많이 씁니다. VLAN이 2~3개만 되어도 충분히 부하 분산 효과를 얻을 수 있거든요.

---

### 면접관: "HSRP version 1과 version 2는 뭐가 다른 거예요? 왜 v2로 설정하셨어요?"

| 항목 | HSRPv1 | HSRPv2 |
|------|--------|--------|
| **Group 번호 범위** | 0~255 | 0~**4095** |
| **가상 MAC 형식** | `0000.0c07.acXX` | `0000.0c9f.f000` ~ `0000.0c9f.ffff` |
| **Multicast 주소** | 224.0.0.2 (all routers) | **224.0.0.102** (HSRP 전용) |
| **IPv6 지원** | 불가 | **지원** |
| **타이머** | 초 단위 | **밀리초 단위 가능** |

v2를 선택한 핵심 이유:

1. **Group 번호를 VLAN 번호와 일치시킬 수 있습니다.** v1은 255까지만 가능해서 VLAN 300 이상이면 Group 번호를 별도로 관리해야 하는 불편함이 있습니다. v2는 4095까지 지원하므로 `standby 300 ip x.x.x.x` 형태로 직관적인 관리가 가능합니다.

2. **Multicast 주소가 분리되어 있습니다.** v1은 `224.0.0.2` (All Routers)를 사용해서 다른 라우팅 프로토콜 트래픽과 섞일 수 있지만, v2는 전용 주소 `224.0.0.102`를 사용합니다.

3. **밀리초 타이머로 더 빠른 Failover가 가능합니다:**
```
standby 10 timers msec 200 msec 750
```
이러면 200ms마다 Hello, 750ms 안에 응답 없으면 Failover. 기본 3/10초 대비 훨씬 빠릅니다.

주의할 점은 **양쪽 장비의 HSRP 버전이 반드시 일치해야** 합니다. 한쪽이 v1이고 다른 쪽이 v2이면 서로 Hello를 인식하지 못해서 양쪽 모두 Active가 되는 **Dual-Active 장애**가 발생합니다.

---

### 면접관: "마지막으로, HSRP 이중화를 운영하면서 주의해야 할 점이나 Best Practice가 있다면?"

실무 운영에서 자주 겪는 이슈와 Best Practice를 정리하면:

#### 1. HSRP + STP 일치 (가장 중요)
```
! HSRP Active = STP Root Bridge 반드시 일치
! DSW1이 VLAN 10 HSRP Active라면:
spanning-tree vlan 10 priority 4096
```
불일치하면 트래픽이 Standby 장비를 경유하는 **우회 경로(suboptimal path)** 가 발생해서 불필요한 레이턴시가 추가됩니다.

#### 2. HSRP Authentication
```
! 같은 VLAN에 비인가 장비가 HSRP에 참여하는 것을 방지
standby 10 authentication md5 key-string HSRP_SECRET_KEY
```

#### 3. 다중 인터페이스 Tracking
```
track 1 interface GigabitEthernet0/1 line-protocol
track 2 interface GigabitEthernet0/2 line-protocol
track 10 list boolean and
 object 1
 object 2

interface Vlan10
 standby 10 track 10 decrement 20
```
업링크가 여러 개일 때, **모든 업링크가 다운되어야 Failover** 하려면 `boolean and`, **하나라도 다운되면 Failover** 하려면 `boolean or`를 사용합니다.

#### 4. 검증 명령어 모음
```
show standby brief                    ! 전체 HSRP 상태 요약
show standby vlan 10                  ! 특정 VLAN 상세 정보
show standby vlan 10 detail           ! Tracking, 타이머 등 상세
show track                            ! Track 오브젝트 상태
debug standby events                  ! HSRP 이벤트 디버깅 (주의: 운영 중 사용 자제)
show standby delay                    ! Preempt 딜레이 확인
```

#### 5. Failover 테스트 절차
구축 완료 후 반드시 아래 테스트를 수행합니다:
1. Active 장비의 VLAN 인터페이스 shutdown -> Standby 승격 확인
2. Active 장비의 업링크 shutdown -> Tracking에 의한 Failover 확인
3. Active 장비 전원 OFF -> 물리 장애 시뮬레이션
4. 복구 후 Preempt에 의한 Active 복귀 확인
5. 각 단계에서 **PC의 ping 끊김 시간** 측정 (목표: 3초 이내)

이 다섯 가지만 제대로 챙기면 HSRP 이중화 운영에서 대부분의 문제를 예방할 수 있습니다.
