# 06-04. Checkpoint ClusterXL (HA / Load Sharing)

---

### 면접관: "고객사에서 Checkpoint 방화벽 이중화를 요청했어요. 현재 단일 장비로 운영 중인데 이중화 구성을 해달라고 합니다. 어떻게 설계하시겠어요?"

Checkpoint 환경에서는 **ClusterXL**이라는 내장 이중화 솔루션을 사용합니다. Cisco ASA의 Failover에 대응하는 기능입니다.

우선 고객사 요구가 단순 이중화(Active/Standby)인지 부하 분산(Active/Active)인지 확인하고, 일반적으로는 **ClusterXL High Availability (Active/Standby)** 모드로 시작하는 것을 권장합니다.

> 토폴로지 이미지 추가 예정

#### Checkpoint 아키텍처 이해 (전제 지식)

Checkpoint는 Cisco와 다르게 **Management(SmartConsole) + Gateway(실제 방화벽)** 가 분리된 구조입니다:
- **Security Management Server (SMS)**: 정책 관리, 로그 수집 (별도 서버)
- **Security Gateway (SGW)**: 실제 트래픽 처리 (이중화 대상)
- **SmartConsole**: 관리자 GUI 클라이언트

이중화는 **Security Gateway 2대를 ClusterXL로 묶는 것**이며, Management Server는 별도로 이중화(Secondary SMS 또는 HA)합니다.

#### 핵심 설계

| 구성 요소 | 설명 |
|----------|------|
| **Cluster IP** | 가상 IP (서비스용, HSRP의 Virtual IP와 유사) |
| **Sync Network** | 두 Gateway 간 상태 동기화 전용 네트워크 |
| **Cluster Mode** | High Availability (Active/Standby) |
| **CCP (Cluster Control Protocol)** | 멤버 간 통신 프로토콜 |

#### SmartConsole에서의 설정 흐름

1. **Cluster Object 생성**: SmartConsole에서 "Security Gateway/Appliance" > "Cluster" 선택
2. **Cluster Members 추가**: 두 Gateway를 Cluster Member로 등록
3. **Network 설정**:
   - External (Outside): Cluster IP + 각 Member IP
   - Internal (Inside): Cluster IP + 각 Member IP
   - Sync: 각 Member IP (Cluster IP 불필요)
4. **Topology 정의**: 각 인터페이스의 네트워크 정보
5. **Policy Install**: 정책을 Cluster에 설치

#### CLI 레벨 핵심 설정 (Gaia OS)

**Member 1 (Active 역할):**
```bash
# 네트워크 설정 (Gaia CLI 또는 WebUI)
set interface eth0 ipv4-address 203.0.113.2 mask-length 24    # External
set interface eth1 ipv4-address 10.0.1.2 mask-length 24       # Internal
set interface eth2 ipv4-address 192.168.100.1 mask-length 24  # Sync

# ClusterXL 활성화 (cpconfig에서)
cpconfig
> (7) Cluster    -> Enable ClusterXL
> Cluster Mode: New Cluster
> Cluster IP for eth0: 203.0.113.1
> Cluster IP for eth1: 10.0.1.1
```

**Member 2 (Standby 역할):**
```bash
set interface eth0 ipv4-address 203.0.113.3 mask-length 24    # External
set interface eth1 ipv4-address 10.0.1.3 mask-length 24       # Internal
set interface eth2 ipv4-address 192.168.100.2 mask-length 24  # Sync

cpconfig
> (7) Cluster    -> Enable ClusterXL
> Cluster Mode: Existing Cluster
> Cluster IP for eth0: 203.0.113.1
> Cluster IP for eth1: 10.0.1.1
```

Cluster IP(`203.0.113.1`, `10.0.1.1`)는 양쪽 Member가 **동일하게** 설정합니다. 이것이 HSRP의 Virtual IP처럼 서비스용 IP가 됩니다.

---

### 면접관: "CCP 프로토콜이 뭐예요? 어떻게 동작하는 거죠?"

**CCP(Cluster Control Protocol)**는 ClusterXL 멤버 간의 **Heartbeat 및 상태 교환을 담당하는 Checkpoint 독자 프로토콜**입니다. Cisco HSRP의 Hello 패킷 역할과 유사합니다.

#### CCP 동작 방식

| 항목 | 설명 |
|------|------|
| **프로토콜** | UDP 기반, Multicast(기본) 또는 Broadcast |
| **전송 주기** | 기본 100ms (매우 빠름) |
| **Failover 판단** | 기본 3번 연속 미응답 시 (약 300ms) |
| **전송 인터페이스** | 모든 Cluster 인터페이스에서 CCP 전송 (Sync 포함) |
| **역할** | Heartbeat, Member 상태 교환, Active/Standby 선출 |

#### CCP 모드

```bash
# CCP 모드 확인
cphaprob mcastaddr
```

| 모드 | 설명 | 사용 환경 |
|------|------|----------|
| **Multicast** (기본) | CCP를 Multicast로 전송 | L2 직접 연결, Multicast 지원 스위치 |
| **Broadcast** | CCP를 Broadcast로 전송 | Multicast 미지원 스위치 |
| **Unicast** | CCP를 Unicast로 전송 | L3 분리 환경 (드묾) |

**실무에서 흔한 문제**: 중간 스위치가 Multicast를 차단하면 CCP가 전달되지 않아서 **양쪽 모두 Active(Split-brain)**가 됩니다. 이때는 Broadcast 모드로 변경합니다:

```bash
# Broadcast 모드로 변경
cphaconf set_ccp broadcast

# 변경 확인
cphaprob mcastaddr
```

#### CCP와 Failover 타이밍

기본 설정에서 Failover 감지 시간은 약 **300ms~1초**로, Cisco ASA(기본 3초 Hello / 10초 Hold)보다 훨씬 빠릅니다. 이것이 Checkpoint ClusterXL의 강점 중 하나입니다.

---

### 면접관: "구축 후에 Failover 테스트를 했는데, Active를 수동으로 넘겼다가 원래 멤버로 복귀가 안 된다고 합니다. 어디부터 확인하시겠어요?"

#### 1단계: Cluster 상태 확인

가장 먼저 `cphaprob stat` 명령어로 전체 상태를 확인합니다:

```bash
[Expert@Member1]# cphaprob stat
```
```
Cluster Mode:           High Availability (Active/Standby)
Number                  Unique Address       Assigned Load    State
1 (local)               192.168.100.1        100%             STANDBY
2                       192.168.100.2        0%               ACTIVE
```

Member 1이 Standby이고 Member 2가 Active인 상태. 복귀가 안 된다는 것은 Member 1이 다시 Active가 안 된다는 의미입니다.

#### 2단계: Problem Notification 확인

```bash
[Expert@Member1]# cphaprob list
```
```
Device             Name             Status     Priority
---------------------------------------------------
Interface          eth0             OK         1
Interface          eth1             OK         1
Interface          eth2(Sync)       OK         0
Policy             Installed        OK         1
SIC                Trust            OK         0
HA                 Initialized      OK         0

Number of registered devices: 6
```

만약 여기서 특정 항목이 `PROBLEM`으로 표시되면 해당 원인을 해결해야 합니다:
```
Interface          eth0             PROBLEM    1    <-- 인터페이스 문제!
```

#### 3단계: 복귀 안 되는 주요 원인

**원인 1: Checkpoint는 기본적으로 자동 복귀(Preempt)가 없음**

Cisco HSRP와 달리 ClusterXL HA 모드에서는 **Priority 개념이 없고, 먼저 Active가 된 멤버가 계속 Active를 유지**합니다. "원래 멤버로 돌아가기"를 원하면 수동으로 전환해야 합니다:

```bash
# Member 1에서 Active로 전환
[Expert@Member1]# clusterXL_admin up

# 또는 Member 2에서 Standby로 전환
[Expert@Member2]# clusterXL_admin down
```

**원인 2: Problem Notification이 남아있는 경우**

이전 장애의 Problem 상태가 clear 되지 않아서 Active 전환이 차단될 수 있습니다:
```bash
# Problem 상태 확인 및 수동 해제
cphaprob list
cphaprob -d device_name -s ok report    # 상태 수동 해제 (원인 해결 후)
```

**원인 3: SIC(Secure Internal Communication) 인증 문제**

Management Server와의 SIC 통신이 깨진 경우:
```bash
cphaprob list | grep SIC
# SIC가 PROBLEM이면
cpconfig  # SIC Reset 수행
```

**원인 4: Policy가 설치되지 않은 경우**
```bash
cphaprob list | grep Policy
# Policy가 PROBLEM이면 SmartConsole에서 Policy Install 재수행
```

#### 4단계: 수동 전환 후 확인
```bash
# 전환 실행
[Expert@Member1]# clusterXL_admin up

# 상태 재확인
[Expert@Member1]# cphaprob stat
Cluster Mode:           High Availability (Active/Standby)
Number                  Unique Address       Assigned Load    State
1 (local)               192.168.100.1        100%             ACTIVE
2                       192.168.100.2        0%               STANDBY
```

---

### 면접관: "Delta Sync와 Full Sync가 뭐예요? 세션 동기화가 어떻게 동작하는 거죠?"

ClusterXL의 State Synchronization(세션 동기화)은 **두 가지 방식**으로 동작합니다.

#### Full Sync (전체 동기화)

- **발생 시점**: Standby 멤버가 처음 Cluster에 참여하거나 재시작될 때
- **동작**: Active의 **전체 Connection Table**을 Standby에 전송
- **부하**: 매우 높음 (수십만~수백만 세션을 한 번에 전송)
- **소요 시간**: 세션 수에 따라 수초~수분

```
Active ========[전체 Connection Table]========> Standby
       (수만~수백만 개 세션 일괄 전송)
```

#### Delta Sync (변경분 동기화)

- **발생 시점**: Full Sync 완료 후 상시 동작
- **동작**: 새로 생성/변경/삭제된 세션만 **실시간으로** 전송
- **부하**: 상대적으로 낮음 (변경분만 전송)
- **전송 주기**: 거의 실시간 (연결 상태 변경 즉시)

```
Active ----[새 세션 추가]----> Standby
       ----[세션 삭제]-------> Standby
       ----[상태 변경]-------> Standby
       (변경분만 지속적 전송)
```

#### Sync 네트워크 설계 시 주의사항

1. **전용 인터페이스 사용 필수**: Sync 트래픽은 양이 많으므로 데이터 인터페이스와 반드시 분리
2. **대역폭 충분히 확보**: 최소 1Gbps, 대규모 환경에서는 10Gbps 권장
3. **직접 연결 권장**: 중간에 스위치를 넣어도 되지만 전용 VLAN 사용
4. **암호화 고려**: Sync 트래픽에 세션 정보가 포함되므로 보안 네트워크에서 전송

#### Sync 상태 확인
```bash
# Sync 상태 확인
[Expert@Member1]# fw ctl pstat
```
```
Sync:
        Sync packets sent:       total: 12345678  retransmitted: 12
        Sync packets received:   total: 12345600  retransmitted: 5
        Sync queue: 0/0 (max: 0)
        Sync new-ver sent: 0
```

`retransmitted` 수치가 높으면 Sync 네트워크에 패킷 로스가 있다는 의미입니다. 네트워크 품질을 점검해야 합니다.

```bash
# 동기화된 Connection 수 비교
[Expert@Member1]# fw tab -t connections -s
[Expert@Member2]# fw tab -t connections -s
# 양쪽 수치가 거의 일치해야 정상
```

#### 동기화 되는 항목 vs 안 되는 항목

| 동기화 됨 | 동기화 안 됨 |
|----------|------------|
| TCP/UDP Connection Table | Routing Table (각 멤버 독립) |
| NAT Translation | ARP Table |
| VPN SA (IPSec IKE) | User Authentication 캐시 |
| Accounting 정보 | HTTPS Inspection 캐시 (일부) |

---

### 면접관: "고객사가 이번에는 HA 모드 말고, 양쪽 다 트래픽을 처리하는 Load Sharing 모드로 전환해달라고 합니다. 어떻게 하시겠어요?"

ClusterXL은 HA(Active/Standby) 외에 **Load Sharing** 모드를 지원합니다. Load Sharing에는 두 가지 방식이 있습니다.

#### Load Sharing 모드 비교

| 항목 | Multicast (추천) | Unicast (Pivot) |
|------|-----------------|-----------------|
| **동작** | 스위치가 트래픽을 양쪽 멤버에 Multicast | Pivot 멤버가 수신 후 분배 |
| **의존성** | 스위치의 Multicast/IGMP 지원 필요 | 스위치 의존 없음 |
| **성능** | 더 좋음 (스위치가 분배) | Pivot 멤버에 부하 집중 |
| **복잡도** | 스위치 설정 필요 | 스위치 설정 불필요 |

#### Load Sharing Multicast 모드 전환 절차

**SmartConsole에서:**
1. Cluster Object > ClusterXL > "Load Sharing - Multicast" 선택
2. Policy Install

**CLI에서 확인:**
```bash
[Expert@Member1]# cphaprob stat
```
```
Cluster Mode:           Load Sharing (Multicast)
Number                  Unique Address       Assigned Load    State
1 (local)               192.168.100.1        50%              ACTIVE
2                       192.168.100.2        50%              ACTIVE
```

양쪽 모두 **ACTIVE**이고 Load가 50%씩 분배된 것을 확인할 수 있습니다.

#### Load Sharing의 동작 원리

1. Cluster IP에 **Multicast MAC 주소**가 할당됨
2. 스위치는 Cluster IP 목적지 트래픽을 양쪽 멤버에 Multicast
3. 각 멤버가 **CCP를 통해 합의한 분배 알고리즘**에 따라, 자기 담당 트래픽만 처리하고 나머지는 드랍
4. 새 Connection은 해시 기반으로 특정 멤버에 할당되고, 해당 멤버가 세션이 끝날 때까지 책임

#### 스위치 측 설정 (Multicast 모드 시)

스위치가 Multicast MAC을 학습하도록 Static MAC 또는 IGMP Snooping 설정이 필요합니다:

```
! Cisco 스위치 예시 - Static Multicast MAC 설정
mac address-table static 0100.5e01.0101 vlan 100 interface GigabitEthernet0/1 GigabitEthernet0/2
```

#### Load 비율 조정

기본 50:50이지만, 한쪽 성능이 더 좋다면 비율을 조정할 수 있습니다:

SmartConsole > Cluster Object > Members > 각 Member의 Weight 조정
```
Member 1: Weight 70  -> 70% 트래픽 처리
Member 2: Weight 30  -> 30% 트래픽 처리
```

#### HA에서 Load Sharing으로 전환 시 주의사항

1. **서비스 중단 발생**: 모드 전환 시 Cluster 재구성이 필요하므로 **유지보수 시간에 수행**
2. **스위치 설정 사전 준비**: Multicast 모드 시 스위치 Multicast 설정이 먼저 완료되어야 함
3. **충분한 테스트**: 전환 후 양쪽 멤버의 Connection 분배가 정상인지 확인
4. **Rollback 계획**: 문제 발생 시 HA 모드로 즉시 복귀할 수 있도록 준비

---

### 면접관: "Checkpoint ClusterXL 운영하면서 자주 쓰는 명령어와 트러블슈팅 팁을 정리해 주세요."

#### 필수 확인 명령어

```bash
# 1. Cluster 전체 상태 확인 (가장 많이 사용)
cphaprob stat

# 2. 각 디바이스/서비스의 상태 상세 확인
cphaprob list

# 3. Cluster 인터페이스 상태
cphaprob -a if

# 4. CCP 통신 상태 (Heartbeat)
cphaprob mcastaddr

# 5. Sync 네트워크 상태
fw ctl pstat

# 6. Connection Table 수 확인 (동기화 비교용)
fw tab -t connections -s

# 7. Cluster 멤버 세부 정보
clusterXL_admin
```

#### Failover 수동 전환

```bash
# 현재 멤버를 Active로 전환
clusterXL_admin up

# 현재 멤버를 Standby로 전환
clusterXL_admin down

# 특정 인터페이스를 Down으로 보고 (테스트용)
cphaprob -d eth0 -t 1 -s problem report
```

#### 자주 발생하는 문제와 해결

**문제 1: Split-Brain (양쪽 모두 Active)**
```bash
# 확인
cphaprob stat
# 양쪽 모두 ACTIVE로 표시

# 원인: CCP 통신 불가 (Sync 네트워크 장애, 스위치 Multicast 차단)
# 해결:
# 1) Sync 네트워크 물리 연결 확인
# 2) CCP 모드를 Broadcast로 변경
cphaconf set_ccp broadcast
```

**문제 2: Standby 멤버가 "READY" 대신 "DOWN"**
```bash
cphaprob list
# 원인: SIC 실패, Policy 미설치, 인터페이스 문제

# SIC 재설정
cpconfig  # SIC Reset 수행 후 SmartConsole에서 Trust 재설정
```

**문제 3: Failover 후 일부 세션 끊어짐**
```bash
# Sync 상태 확인
fw ctl pstat
# retransmitted 수치가 높으면 Sync 네트워크 품질 문제

# Connection 수 비교
fw tab -t connections -s    # 양쪽에서 실행하여 비교
```

**문제 4: Policy Install 실패**
```bash
# 원인: Cluster 멤버 간 통신 불가 또는 SIC 문제
# 확인:
cphaprob list | grep -i policy
fw stat                      # 현재 적용된 정책 확인
cpstat fw -f policy          # 정책 상태 상세
```

#### Checkpoint vs Cisco ASA Failover 비교 정리

| 항목 | Checkpoint ClusterXL | Cisco ASA Failover |
|------|---------------------|-------------------|
| **Failover 감지 시간** | ~300ms (CCP 기본) | 기본 3~10초 (조정 가능) |
| **관리 방식** | SmartConsole (중앙 관리) | 장비 CLI/ASDM (개별 관리) |
| **Load Sharing** | 네이티브 지원 | Active/Active (Multi-Context 필요) |
| **설정 동기화** | SMS에서 Policy Install | Failover Link를 통해 자동 동기화 |
| **세션 동기화** | Sync 네트워크 | State Link |
| **라이선스** | Cluster 라이선스 | 기본 포함 |

이 비교표는 면접에서 "Cisco와 Checkpoint 둘 다 경험 있으시죠?"라는 질문에 대비하는 데 유용합니다.
