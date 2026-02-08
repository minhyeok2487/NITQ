# 06-02. Cisco ASA Failover (Active/Standby)

---

## 면접관: "고객사 데이터센터에 방화벽이 ASA 1대인데, 이게 단일 장애점이라 이중화를 해달라고 합니다. 어떻게 설계하시겠어요?"

네, ASA가 단일 장비로 운영되면 그 장비 하나가 죽었을 때 전체 인터넷 및 내부 서비스 통신이 끊어지므로, **ASA Active/Standby Failover** 구성을 제안하겠습니다.

두 대의 ASA를 **Failover Link**로 직접 연결하고, 하나는 Active로 트래픽을 처리하고 다른 하나는 Standby로 대기하다가 Active 장애 시 즉시 승격하는 구조입니다.

> 토폴로지 이미지 추가 예정

### 핵심 설계 포인트

- **Failover Link**: 두 ASA 간 상태 정보 교환용 전용 링크 (Heartbeat)
- **State Link**: 세션 테이블 동기화용 링크 (Stateful Failover 시 필요)
- Active ASA의 IP와 Standby ASA의 IP를 별도로 할당하되, **가상 IP(Active IP)**를 공유
- Failover 시 Active IP와 MAC 주소가 Standby로 이전 -> 주변 장비 설정 변경 불필요

### 핵심 설정

**Primary ASA (Active 역할):**
```
! Failover 기본 설정
failover
failover lan unit primary
failover lan interface FAILOVER GigabitEthernet0/3
failover polltime unit 1 holdtime 3
failover key FAILOVER_SECRET
failover replication http
failover link STATE GigabitEthernet0/4
failover interface ip FAILOVER 192.168.100.1 255.255.255.252 standby 192.168.100.2
failover interface ip STATE 192.168.200.1 255.255.255.252 standby 192.168.200.2

! Outside 인터페이스
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.0 standby 203.0.113.2

! Inside 인터페이스
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 10.0.1.1 255.255.255.0 standby 10.0.1.2

! DMZ 인터페이스
interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 172.16.1.1 255.255.255.0 standby 172.16.1.2
```

**Secondary ASA (Standby 역할):**
```
failover
failover lan unit secondary
failover lan interface FAILOVER GigabitEthernet0/3
failover key FAILOVER_SECRET
failover interface ip FAILOVER 192.168.100.1 255.255.255.252 standby 192.168.100.2
```

Secondary는 설정을 최소한만 넣으면, Failover Link를 통해 **Primary의 설정이 자동으로 동기화**됩니다. 이 점이 ASA Failover의 큰 장점입니다.

---

## 면접관: "Stateless Failover와 Stateful Failover 차이가 뭐예요? 고객사한테 뭘 권장하시겠어요?"

이 차이가 실무에서 굉장히 중요합니다.

### Stateless Failover (기본값)
- Failover 발생 시 **모든 기존 세션이 끊어집니다**
- 새로운 Active ASA는 빈 Connection Table로 시작
- 사용자는 TCP 세션을 처음부터 다시 맺어야 함
- 웹 브라우징 같은 짧은 세션은 사용자가 거의 못 느낌
- 하지만 **VPN 터널, 대용량 파일 전송, DB 세션** 등은 끊어짐

### Stateful Failover
- **State Link**를 통해 Active의 Connection Table이 실시간으로 Standby에 복제
- Failover 발생해도 **기존 세션이 유지**됨
- 동기화되는 정보:
  - NAT Translation Table
  - TCP Connection States
  - HTTP Connection States (`failover replication http` 설정 시)
  - IPSec VPN SA (Security Associations)
  - AnyConnect VPN Sessions
  - DHCP Server Lease 정보

동기화 **안 되는** 정보:
- User Authentication Table (uauth)
- Routing Table (각 ASA가 독립적으로 관리)
- ARP Table
- SIP Signaling Sessions (일부 버전)

고객사에는 **반드시 Stateful Failover를 권장**합니다. 특히 VPN을 사용하는 환경이라면 세션이 끊어지면 사용자가 다시 인증해야 하므로 업무 영향이 큽니다. 앞서 설정에서 `failover link STATE` 부분이 바로 Stateful Failover를 위한 State Link 설정입니다.

---

## 면접관: "좋아요. 그런데 구축 후에 Failover 테스트를 했는데, Active가 넘어갔을 때 기존 세션이 끊어진다고 고객이 컴플레인 했어요. Stateful로 설정했는데 왜 그런 거죠?"

이 문제는 몇 가지 원인이 있을 수 있습니다. 순서대로 확인하겠습니다.

### 1단계: Failover 상태 확인
```
show failover
```
```
Failover On
Failover unit Primary
Failover LAN Interface: FAILOVER GigabitEthernet0/3 (up)
Failover State Link: STATE GigabitEthernet0/4 (up)    <-- 여기 확인!
  Unit Poll frequency 1 seconds, holdtime 3 seconds
  Interface Poll frequency 5 seconds, holdtime 25 seconds
  Stateful Failover Logical Update Statistics
    Link : STATE GigabitEthernet0/4 (up)
    Stateful Obj    xmit       xerr       rcv        rerr
    General         1234       0          1230       0
    sys cmd         567        0          567        0
    ...
```

**State Link가 down이면** Stateful 동기화가 안 됩니다. 물리적 케이블 연결 또는 인터페이스 설정을 확인해야 합니다.

### 2단계: State Link 동기화 확인
```
show failover state
```
```
               State          Last Failure Reason
This host   -  Primary
              Active           None
Other host  -  Secondary
              Standby Ready    None

====Configuration State===
        Sync Done
====Communication State===
        Mac set
```

`Sync Done`이 아니라 `Sync Not Started`나 오류가 있으면 문제입니다.

### 3단계: Connection 복제 여부 확인
```
! Active ASA에서
show conn count
12345 in use, 50000 most used

! Standby ASA에서 (Failover exec 사용)
failover exec standby show conn count
12340 in use, 50000 most used    <-- Active와 거의 일치해야 함
```

수치가 크게 차이 나면 State Link의 대역폭 부족이나 패킷 드랍이 원인일 수 있습니다.

### 4단계: 흔한 원인과 해결

**원인 1: `failover link` 미설정**
```
! Failover LAN Interface만 설정하고 State Link를 빠뜨린 경우
! 해결: State Link 추가
failover link STATE GigabitEthernet0/4
failover interface ip STATE 192.168.200.1 255.255.255.252 standby 192.168.200.2
```

**원인 2: HTTP 복제 미설정**
```
! HTTP 세션은 기본적으로 복제 안 됨
! 해결:
failover replication http
```

**원인 3: State Link 대역폭 부족**
트래픽이 많은 환경에서 State Link가 100Mbps면 복제가 밀릴 수 있습니다. **최소 1Gbps 전용 링크** 사용을 권장합니다.

---

## 면접관: "Failover Link와 State Link는 반드시 분리해야 하나요? 하나로 합쳐도 되죠?"

물리적으로 하나의 인터페이스에 Failover Link와 State Link를 **합칠 수 있습니다**. 설정도 간단합니다:

```
failover lan interface FO_LINK GigabitEthernet0/3
failover link FO_LINK GigabitEthernet0/3
failover interface ip FO_LINK 192.168.100.1 255.255.255.252 standby 192.168.100.2
```

하지만 **실무에서는 분리를 강력히 권장**합니다. 이유는:

| 항목 | Failover Link | State Link |
|------|--------------|------------|
| **역할** | Heartbeat, 장비 상태 교환, 설정 동기화 | Connection Table 실시간 복제 |
| **트래픽량** | 매우 적음 (Heartbeat 패킷) | **매우 많음** (모든 세션 정보) |
| **장애 영향** | Link 끊어지면 **Split-brain** 위험 | Link 끊어지면 **Stateful 동기화 중단** |

하나로 합치면 State Link의 대량 트래픽이 Failover Heartbeat를 지연시켜서, 정상인데 장애로 오판하는 **False Failover**가 발생할 수 있습니다.

또한 Failover Link가 끊어지면 **양쪽 모두 Active(Split-brain)**가 되는 최악의 상황이 올 수 있는데, 이때 State Link라도 살아있으면 문제를 조기에 감지할 수 있습니다. 분리하면 이런 상호 백업 효과가 있습니다.

최소 권장 구성:
- Failover Link: 전용 GigabitEthernet 1개
- State Link: 전용 GigabitEthernet 1개
- 고가용성 구성 시: 각각 EtherChannel로 이중화

---

## 면접관: "Monitored Interface가 뭐예요? 어떤 인터페이스를 모니터링해야 하죠?"

ASA Failover는 **장비 자체의 장애뿐 아니라 인터페이스 단위의 장애도 감지**할 수 있습니다. 이것이 Interface Health Monitoring입니다.

### 동작 원리
1. ASA는 각 Monitored Interface에서 주기적으로 **Hello 패킷**을 교환 (기본 5초)
2. Hold Time(기본 25초) 내에 응답이 없으면 해당 인터페이스를 **Failed**로 판단
3. Failed 인터페이스 수가 **Threshold 이상**이면 Failover 트리거

### 설정
```
! 인터페이스 모니터링 활성화 (기본적으로 nameif가 설정된 인터페이스는 자동 모니터링)
monitor-interface outside
monitor-interface inside
monitor-interface dmz

! 모니터링 비활성화 (관리 인터페이스 등)
no monitor-interface management

! 폴링 주기 조정
failover polltime interface 3 holdtime 10
```

### 모니터링 대상 선정 기준

**반드시 모니터링:**
- `outside` (인터넷 방향) - 이게 죽으면 외부 통신 불가
- `inside` (내부 네트워크 방향) - 이게 죽으면 내부 통신 불가

**선택적 모니터링:**
- `dmz` - DMZ 서버가 중요한 경우
- 기타 데이터 인터페이스

**모니터링 제외:**
- `management` - 관리 인터페이스 장애로 Failover 발생하면 안 됨
- Failover Link / State Link 자체 인터페이스

### 확인 명령어
```
show monitor-interface
```
```
        This host: Primary - Active
            Interface outside (203.0.113.1): Normal
            Interface inside (10.0.1.1): Normal
            Interface dmz (172.16.1.1): Normal
        Other host: Secondary - Standby Ready
            Interface outside (203.0.113.2): Normal
            Interface inside (10.0.1.2): Normal
            Interface dmz (172.16.1.2): Normal
```

만약 특정 인터페이스에서 장애가 감지되면:
```
            Interface outside (203.0.113.1): Failed (Waiting)
```

이때 `interface-policy` 설정으로 몇 개 인터페이스가 Failed여야 Failover를 트리거할지 정할 수 있습니다:
```
! 1개라도 Failed면 Failover (기본값)
failover interface-policy 1

! 또는 비율로 설정
failover interface-policy 50%
```

---

## 면접관: "고객사가 Failover 후 원래 장비가 복구되면 자동으로 다시 Active가 되게 해달라고 해요. 가능한가요?"

네, ASA에서도 **Preempt** 기능을 사용할 수 있습니다. 기본적으로 ASA Failover는 **Preempt가 비활성화**되어 있어서, 한번 Failover가 발생하면 원래 Primary가 복구되어도 Standby로 남습니다.

### Preempt 설정
```
! Primary ASA에서
failover preempt
```

하지만 **실무에서는 Preempt를 권장하지 않습니다.** 이유는:

1. **Failover 자체가 순간 끊김을 유발**: Stateful이라도 100% 무중단은 아닙니다. 불필요한 전환은 줄이는 것이 좋습니다.

2. **장애 원인 미해결 시 Flapping**: Primary의 근본 문제가 해결 안 된 상태에서 Preempt로 Active를 가져오면, 다시 장애 -> Failover -> 복구 -> Preempt의 **Flapping**이 발생합니다.

3. **유지보수 시 불편**: Secondary에서 작업하려고 Failover 시켰는데, Primary가 Preempt로 Active를 다시 가져가버리는 상황

### 권장 대안: 수동 전환
```
! Active를 Standby로 강제 전환 (현재 Active ASA에서)
failover active

! 또는 특정 장비를 Active로 지정
no failover active
```

고객사에게는 이렇게 설명합니다: "자동 복귀보다는 장애 원인을 확인한 후 수동으로 전환하는 것이 안전합니다. 자동 복귀 시 장애가 반복되면 오히려 서비스 가용성이 떨어집니다."

그래도 고객이 꼭 원한다면, **Preempt Delay**를 충분히 주는 것을 권장합니다:
```
failover preempt 300
```
이렇게 하면 Primary가 복구된 후 **5분(300초) 대기** 후에 Active를 가져옵니다. 이 시간 동안 라우팅 수렴과 안정성 확인이 가능합니다.

---

## 면접관: "ASA Failover 운영 중에 알아두면 좋은 검증/관리 명령어 정리해 주세요."

실무에서 자주 사용하는 명령어를 상황별로 정리하겠습니다.

### 상태 확인
```
show failover                          ! 전체 Failover 상태 (가장 많이 사용)
show failover state                    ! 간략한 상태 요약
show failover history                  ! Failover 이력 (언제, 왜 전환됐는지)
show failover statistics               ! Failover 통계
show failover interface                ! 인터페이스별 Failover 상태
show monitor-interface                 ! 모니터링 인터페이스 상태
```

### 설정 동기화 확인
```
write standby                          ! 수동으로 설정 동기화 강제 실행
show running-config failover           ! Failover 관련 설정 확인
```

### Stateful 동기화 확인
```
show conn count                        ! Active의 Connection 수
failover exec standby show conn count  ! Standby의 Connection 수 (비교용)
show failover statistics               ! 동기화 패킷 통계
```

### 수동 전환
```
failover active                        ! 현재 장비를 Active로 전환
no failover active                     ! 현재 장비를 Standby로 전환
failover exec standby failover active  ! Standby 장비를 Active로 전환
```

### 장애 대응
```
failover reset                         ! Failed 상태 초기화
debug fover ?                          ! Failover 디버그 (운영 중 주의)
debug fover fail                       ! Failover 실패 원인 디버깅
```

### Failover 테스트 체크리스트
1. `show failover` - 양쪽 상태 정상 확인
2. Active ASA에서 `no failover active` -> Standby 승격 확인
3. 기존 세션 유지 여부 확인 (Stateful)
4. `failover active` -> 원복
5. Active 인터페이스 `shutdown` -> Interface Monitoring에 의한 Failover 확인
6. Failover Link 케이블 분리 -> **이 테스트는 반드시 유지보수 시간에!** (Split-brain 위험)
7. 각 테스트에서 failover 소요 시간 측정 및 문서화
