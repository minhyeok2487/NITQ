# 04-06. HSRP 기본 (FHRP)

> CCNA 200-301 레벨 네트워크 엔지니어 면접 시나리오
> 주제: FHRP 필요성, HSRP 동작 원리, Active/Standby 역할, Virtual IP 설정, Priority/Preemption, VRRP 비교, 검증 명령어, 페일오버 시나리오

---

### 면접관: "사내 네트워크에서 기본 게이트웨이 역할을 하는 라우터가 장애를 일으키면 전체 네트워크가 마비됩니다. 이 문제를 어떻게 해결하시겠습니까?"

**FHRP(First Hop Redundancy Protocol)**를 사용하여 기본 게이트웨이의 이중화를 구현합니다. 가장 대표적인 프로토콜이 Cisco의 **HSRP(Hot Standby Router Protocol)**입니다.

문제의 근본 원인은 다음과 같습니다. PC나 서버에는 기본 게이트웨이를 하나만 설정할 수 있습니다. 라우터 두 대가 있어도 PC에 설정된 게이트웨이 라우터가 다운되면, PC는 외부 통신이 불가능합니다. 수동으로 게이트웨이를 변경하기 전까지 네트워크가 중단됩니다.

HSRP는 이 문제를 해결하기 위해 두 대 이상의 라우터가 하나의 **가상 IP(Virtual IP)**를 공유합니다. PC에는 이 가상 IP를 게이트웨이로 설정하면, Active 라우터가 장애를 일으켜도 Standby 라우터가 자동으로 역할을 인계받아 통신이 유지됩니다.

```
         [Internet]
          │      │
     ┌────┴──┐ ┌─┴─────┐
     │ R1    │ │   R2   │
     │Active │ │Standby │
     │.2     │ │   .3   │
     └───┬───┘ └───┬────┘
         │         │
    ─────┴─────────┴──────  VLAN 10: 192.168.10.0/24
         │
     Virtual IP: 192.168.10.1  ← PC들의 기본 게이트웨이
         │
      [PC들]
```

---

### 면접관: "HSRP의 동작 원리를 자세히 설명해주세요. Active와 Standby 역할은 어떻게 결정됩니까?"

HSRP에서 라우터들은 다음과 같은 과정으로 역할을 결정합니다.

**HSRP 상태 변화 과정:**

| 상태 | 설명 |
|------|------|
| **Initial** | HSRP가 시작되지 않은 초기 상태 |
| **Learn** | 가상 IP를 아직 모르는 상태 |
| **Listen** | 가상 IP를 알지만 Active/Standby가 아닌 상태 |
| **Speak** | Hello 메시지를 보내며 Active/Standby 선출에 참여 |
| **Standby** | Active의 백업 역할, Active 장애 시 인계 준비 |
| **Active** | 가상 IP에 대한 패킷을 실제로 처리하는 역할 |

역할 결정 기준은 다음과 같습니다.

1. **Priority 값**: 기본값은 100이며, 값이 높은 라우터가 Active가 됩니다.
2. **Priority가 같으면**: IP 주소가 높은 라우터가 Active가 됩니다.

HSRP 라우터들은 **Hello 메시지**를 주기적으로 교환하여 상대방의 상태를 확인합니다.

| 타이머 | 기본값 | 설명 |
|--------|--------|------|
| **Hello** | 3초 | Hello 메시지 전송 주기 |
| **Hold** | 10초 | 이 시간 동안 Hello를 못 받으면 장애로 판단 |

Active 라우터는 가상 IP에 대한 ARP 요청에 **가상 MAC 주소**로 응답합니다. HSRPv1의 가상 MAC 형식은 `0000.0c07.acXX`이며, XX는 HSRP 그룹 번호입니다.

---

### 면접관: "HSRP를 설정하는 방법을 CLI로 보여주세요."

두 대의 라우터에 HSRP를 설정하는 예시입니다.

#### R1 (Active로 동작할 라우터) 설정

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.10.2 255.255.255.0
R1(config-if)# standby 1 ip 192.168.10.1
R1(config-if)# standby 1 priority 110
R1(config-if)# standby 1 preempt
R1(config-if)# standby 1 timers 1 3
R1(config-if)# exit
```

#### R2 (Standby로 동작할 라우터) 설정

```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.10.3 255.255.255.0
R2(config-if)# standby 1 ip 192.168.10.1
R2(config-if)# standby 1 priority 100
R2(config-if)# standby 1 preempt
R2(config-if)# standby 1 timers 1 3
R2(config-if)# exit
```

설정 항목을 상세히 설명하면 다음과 같습니다.

| 명령어 | 설명 |
|--------|------|
| `standby 1 ip 192.168.10.1` | HSRP 그룹 1의 가상 IP를 192.168.10.1로 설정 |
| `standby 1 priority 110` | Priority를 110으로 설정 (기본값 100보다 높으면 Active) |
| `standby 1 preempt` | 우선순위가 높은 라우터가 복구되면 Active 역할 재탈환 |
| `standby 1 timers 1 3` | Hello 1초, Hold 3초로 설정 (빠른 장애 감지) |

PC에는 기본 게이트웨이를 가상 IP인 **192.168.10.1**로 설정합니다. 실제 라우터 IP(192.168.10.2, 192.168.10.3)가 아닌 가상 IP를 사용하는 것이 핵심입니다.

---

### 면접관: "Preemption이란 무엇이고, 왜 중요합니까?"

**Preemption(선점)**은 우선순위가 높은 라우터가 장애에서 복구되었을 때, Active 역할을 다시 가져오는 기능입니다.

Preemption이 없는 경우를 시나리오로 설명하겠습니다.

```
[정상 상태]
R1 (Priority 110) = Active
R2 (Priority 100) = Standby

[R1 장애 발생]
R1 → Down
R2 (Priority 100) = Active ← 자동 승격

[R1 복구]
R1 (Priority 110) = Standby  ← Priority가 높지만 Standby로 남음!
R2 (Priority 100) = Active   ← 여전히 Active 유지
```

`standby preempt`를 설정하면 다음과 같이 동작합니다.

```
[R1 복구 + Preemption 활성화]
R1 (Priority 110) = Active  ← Priority가 높으므로 Active 재탈환!
R2 (Priority 100) = Standby ← 다시 Standby로 전환
```

Preemption이 중요한 이유는, 일반적으로 Priority를 높게 설정한 라우터는 더 좋은 성능이나 더 나은 WAN 연결을 가진 장비입니다. 장애 복구 후에도 이 라우터가 Active가 되어야 최적의 경로로 트래픽이 전달됩니다.

단, Preemption 설정 시 주의할 점은, 장비가 불안정하게 반복적으로 다운되는 상황에서는 Active 역할이 계속 전환되어(flapping) 오히려 네트워크가 불안정해질 수 있습니다. 이를 방지하기 위해 `standby 1 preempt delay minimum 60` 같이 딜레이를 설정하기도 합니다.

---

### 면접관: "HSRP 상태를 확인하는 방법과, 실제 페일오버가 정상 동작하는지 테스트하는 방법을 알려주세요."

#### 검증 명령어

```
R1# show standby
GigabitEthernet0/0 - Group 1
  State is Active
    2 state changes, last state change 01:23:45
  Virtual IP address is 192.168.10.1
  Active virtual MAC address is 0000.0c07.ac01
    Local virtual MAC address is 0000.0c07.ac01 (v1 default)
  Hello time 1 sec, hold time 3 sec
    Next hello sent in 0.512 secs
  Preemption enabled
  Active router is local
  Standby router is 192.168.10.3, priority 100 (expires in 2.784 sec)
  Priority 110 (configured 110)
  Group name is "hsrp-Gi0/0-1" (default)

R1# show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Gi0/0       1    110 P Active   local           192.168.10.3    192.168.10.1
```

#### 페일오버 테스트 절차

```
! 1단계: 현재 상태 확인
R1# show standby brief    → R1이 Active 확인
R2# show standby brief    → R2가 Standby 확인

! 2단계: PC에서 외부 연결 테스트
PC> ping 8.8.8.8          → 정상 응답 확인
PC> arp -a                → 192.168.10.1이 HSRP 가상 MAC으로 매핑 확인

! 3단계: Active 라우터(R1) 인터페이스 다운
R1(config)# interface GigabitEthernet0/0
R1(config-if)# shutdown

! 4단계: 페일오버 확인
R2# show standby brief    → R2가 Active로 전환 확인

! 5단계: PC 연결 재확인
PC> ping 8.8.8.8          → 잠시 끊겼다가 정상 응답 확인

! 6단계: R1 복구 후 Preemption 확인
R1(config-if)# no shutdown
R1# show standby brief    → R1이 다시 Active로 복귀 확인 (preempt 설정 시)
```

기본 타이머(Hello 3초, Hold 10초)에서는 페일오버에 최대 10초가 걸리며, 타이머를 1초/3초로 줄이면 약 3초 만에 페일오버됩니다. 더 빠른 전환이 필요하면 밀리초 단위의 타이머 설정도 가능합니다.

---

### 면접관: "HSRP 외에 다른 FHRP 프로토콜도 있다고 알고 있습니다. VRRP와 비교해주세요."

HSRP와 VRRP의 주요 차이를 정리하겠습니다.

| 구분 | HSRP | VRRP |
|------|------|------|
| **표준** | Cisco 독자 프로토콜 | IEEE 표준 (RFC 5798) |
| **역할 명칭** | Active / Standby | Master / Backup |
| **기본 Priority** | 100 | 100 |
| **가상 MAC** | 0000.0c07.acXX (v1) | 0000.5e00.01XX |
| **Preemption** | 기본 비활성 (수동 설정 필요) | 기본 활성화 |
| **멀티벤더** | Cisco 장비만 지원 | 모든 벤더 지원 |
| **Hello 주기** | 3초 (기본) | 1초 (기본) |
| **가상 IP = 실제 IP** | 불가 | 가능 (Master의 실제 IP를 가상 IP로 사용) |

VRRP는 표준 프로토콜이므로 멀티벤더 환경에서 유리합니다. 하지만 Cisco 전용 환경에서는 HSRP가 더 많은 기능과 안정성을 제공하여 일반적으로 선호됩니다.

참고로 Cisco에는 **GLBP(Gateway Load Balancing Protocol)**도 있습니다. HSRP와 VRRP는 Active 라우터만 트래픽을 처리하고 Standby는 대기하는 반면, GLBP는 여러 라우터가 동시에 트래픽을 분산 처리할 수 있어 부하 분산이 가능합니다.

| 프로토콜 | 부하 분산 | 활용 시나리오 |
|---------|----------|-------------|
| HSRP | Active/Standby (부하 분산 없음) | 가장 일반적인 이중화 |
| VRRP | Master/Backup (부하 분산 없음) | 멀티벤더 환경 |
| GLBP | Active-Active (부하 분산 가능) | 대역폭 활용 최대화 |

HSRP에서도 VLAN별로 Active 라우터를 다르게 설정하면 간접적인 부하 분산이 가능합니다.

```
! VLAN 10은 R1이 Active
R1(config-if)# standby 10 priority 110
R1(config-if)# standby 10 preempt

! VLAN 20은 R2가 Active
R2(config-if)# standby 20 priority 110
R2(config-if)# standby 20 preempt
```

---

### 면접관: "마지막으로, HSRP 트러블슈팅 시 자주 발생하는 문제를 정리해주세요."

HSRP 운영 시 자주 접하는 문제와 해결 방법을 정리하겠습니다.

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| 두 라우터 모두 Active | Hello 메시지 미수신 (L2 문제) | 스위치 포트, VLAN, 케이블 확인 |
| 페일오버 안 됨 | HSRP 그룹 번호 불일치 | 양쪽 그룹 번호 동일한지 확인 |
| 복구 후 Active 미전환 | Preempt 미설정 | `standby preempt` 설정 |
| 가상 IP 통신 불가 | 가상 IP 주소 불일치 | 양쪽 `standby ip` 동일한지 확인 |
| 잦은 역할 전환 (Flapping) | 불안정한 링크/장비 | 타이머 조정, preempt delay 설정 |

#### 진단 명령어

```
! HSRP 상태 요약 확인
Router# show standby brief

! 상세 HSRP 정보 확인
Router# show standby

! HSRP 이벤트 디버그
Router# debug standby events
Router# debug standby packets
```

특히 "두 라우터가 모두 Active"인 상태는 **Split-Brain** 상황으로, 가장 위험한 문제입니다. 이 상태에서는 두 라우터 모두 같은 가상 MAC으로 응답하여 네트워크에 혼란을 일으킵니다. 주로 Layer 2 연결 문제(스위치 장애, VLAN 설정 오류)로 Hello 메시지가 전달되지 않을 때 발생하므로, HSRP 문제가 의심될 때는 먼저 Layer 2 연결 상태를 확인해야 합니다.
