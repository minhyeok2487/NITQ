# 02-05. DHCP Snooping과 DAI

---

### 면접관: "고객사에서 '사무실 PC들이 이상한 IP를 받는다, IP 충돌이 계속 난다'고 신고가 왔어요. 확인해보니 누군가 개인 공유기를 네트워크에 연결해서 DHCP 서버 역할을 하고 있었어요. 이런 문제를 근본적으로 방지하려면 어떻게 하시겠어요?"

**DHCP Snooping**을 적용하겠습니다. DHCP Snooping은 스위치가 DHCP 트래픽을 감시하여 인가된 DHCP 서버의 응답만 허용하고, 비인가 DHCP 서버의 응답을 차단하는 L2 보안 기능입니다.

**설계 개념:**

```
                [정상 DHCP 서버]
                 10.1.1.10
                     │
                     │ Trusted Port (DHCP 응답 허용)
                     │
              [Core/Distribution SW]
              (DHCP Snooping 활성화)
               │            │
    Untrusted  │            │ Untrusted
               │            │
            [PC들]    [비인가 공유기] ← DHCP Offer/ACK 차단됨
```

> 토폴로지 이미지 추가 예정

#### 핵심 설정

```
! 1. DHCP Snooping 전역 활성화
ip dhcp snooping

! 2. 적용할 VLAN 지정
ip dhcp snooping vlan 10,20,30

! 3. Option 82 삽입 비활성화 (일부 DHCP 서버에서 문제 발생 가능)
no ip dhcp snooping information option

! 4. Trusted 포트 설정 (정상 DHCP 서버 방향)
interface GigabitEthernet0/48
 ip dhcp snooping trust

! 5. Untrusted 포트 Rate Limit (DHCP 스톰 방지)
interface range GigabitEthernet0/1 - 24
 ip dhcp snooping limit rate 15
 ! 초당 15개 DHCP 패킷까지 허용. 초과 시 err-disabled
```

**동작 원리:**

| DHCP 메시지 | 방향 | Trusted 포트 | Untrusted 포트 |
|------------|------|:-----------:|:-------------:|
| **Discover** | Client → Server | 허용 | 허용 |
| **Offer** | Server → Client | 허용 | **차단** |
| **Request** | Client → Server | 허용 | 허용 |
| **ACK** | Server → Client | 허용 | **차단** |

핵심: **DHCP 서버 응답(Offer, ACK)은 Trusted 포트에서만 허용**됩니다. Untrusted 포트에서 이 메시지가 수신되면 즉시 드롭합니다.

---

### 면접관: "Trusted와 Untrusted를 어떤 기준으로 나누나요? 잘못 설정하면 어떻게 되나요?"

**Trusted 포트 설정 원칙:**

```
Trusted로 설정해야 하는 포트:
├── 정상 DHCP 서버가 직접 연결된 포트
├── DHCP 서버 방향의 Uplink 포트 (Trunk 포함)
├── DHCP Relay를 수행하는 L3 스위치/라우터 방향 포트
└── 스위치 간 Trunk (DHCP 서버까지의 경로상 모든 스위치)

Untrusted로 두어야 하는 포트 (기본값):
├── 모든 Access 포트 (PC, 프린터 등)
├── 무선 AP 연결 포트
└── IP Phone 연결 포트
```

**잘못 설정한 경우의 시나리오:**

**시나리오 1: DHCP 서버 방향 Uplink를 Untrusted로 둔 경우**
- 모든 DHCP Offer/ACK가 차단됨
- 전체 사용자가 IP를 받지 못함
- 가장 흔한 실수

**시나리오 2: Access 포트를 Trusted로 설정한 경우**
- 해당 포트에 비인가 DHCP 서버가 연결되면 차단할 수 없음
- DHCP Snooping의 의미가 없어짐

**다중 스위치 환경에서의 Trusted 설정:**

```
[DHCP Server] ──── [Core SW] ──── [Distribution SW] ──── [Access SW] ──── [PC]
                      │                  │                     │
              DHCP Snooping ON    DHCP Snooping ON     DHCP Snooping ON
              Trust: Server포트   Trust: Uplink(Core)  Trust: Uplink(Dist)
                                  Untrust: Downlink    Untrust: Access포트
```

모든 스위치에서 DHCP Snooping을 활성화하고, DHCP 서버 방향의 Uplink를 Trusted로 설정해야 합니다. 어느 한 스위치라도 빠지면 해당 구간에서 DHCP가 차단됩니다.

---

### 면접관: "그런데 DHCP Snooping을 적용했더니 정상 DHCP 서버에서도 IP를 못 받는다고 합니다. 어디부터 보시겠어요?"

DHCP Snooping 적용 후 정상 DHCP가 안 되는 것은 매우 흔한 장애입니다.

**1단계: DHCP Snooping 상태 확인**

```
show ip dhcp snooping
show ip dhcp snooping statistics
```

```
! 출력 예시에서 주목할 항목:
! DHCP Snooping is enabled
! DHCP Snooping is configured on following VLANs: 10,20,30
! Insertion of option 82 is enabled     ← 문제 원인 #1
!
! Interface           Trusted   Allow    Rate limit
! ------------------  -------   -----    ----------
! GigabitEthernet0/48  yes       yes      unlimited
! GigabitEthernet0/1   no        yes      15
```

**원인 #1: Option 82 (가장 흔한 원인)**

DHCP Snooping은 기본적으로 DHCP 패킷에 Option 82 (Relay Agent Information)를 삽입합니다. 그런데 일부 DHCP 서버(특히 Windows DHCP 서버)는 Option 82가 포함된 패킷을 거부합니다.

```
! 해결: Option 82 삽입 비활성화
no ip dhcp snooping information option
```

또는 DHCP 서버 쪽에서 Option 82를 허용하도록 설정합니다.

**원인 #2: Trusted 포트 미설정**

```
! DHCP 서버 방향 포트가 Trusted인지 확인
show ip dhcp snooping | include Trust

! 수정
interface GigabitEthernet0/48
 ip dhcp snooping trust
```

**원인 #3: Rate Limit 초과로 포트가 err-disabled**

```
show interfaces status err-disabled

! DHCP 요청이 한꺼번에 몰리면 (예: 스위치 재부팅 후 모든 PC가 동시에 DHCP 요청)
! Rate Limit 초과로 포트가 err-disabled될 수 있음

! 해결 1: Rate Limit 값을 높임
interface range GigabitEthernet0/1 - 24
 ip dhcp snooping limit rate 30

! 해결 2: err-disabled 자동 복구 설정
errdisable recovery cause dhcp-rate-limit
errdisable recovery interval 300
```

**원인 #4: DHCP Snooping Binding DB 관련**

```
show ip dhcp snooping binding

! DHCP Snooping은 바인딩 테이블을 유지
! MAC, IP, VLAN, Port, Lease Time 매핑
! 스위치 재부팅 시 이 테이블이 사라지면 기존 클라이언트의 DHCP Renew가 실패할 수 있음

! 바인딩 DB를 플래시에 저장
ip dhcp snooping database flash:dhcp-snooping.db
ip dhcp snooping database write-delay 300
```

**진단 명령어 종합:**

```
show ip dhcp snooping
show ip dhcp snooping binding
show ip dhcp snooping statistics
show ip dhcp snooping database
show logging | include DHCP
debug ip dhcp snooping event    ← 운영 환경 주의
debug ip dhcp snooping packet   ← 운영 환경 주의
```

---

### 면접관: "DHCP Snooping 바인딩 테이블이 뭔가요? 이게 왜 중요한가요?"

DHCP Snooping Binding Table은 DHCP Snooping이 DHCP 트랜잭션을 관찰하면서 구축하는 **MAC-IP-VLAN-Port 매핑 데이터베이스**입니다.

**바인딩 테이블 예시:**

```
show ip dhcp snooping binding

! MacAddress         IpAddress       Lease(sec)  Type         VLAN  Interface
! -----------------  --------------- ----------  -----------  ----  ---------
! aa:bb:cc:00:01:01  10.1.10.100     86400       dhcp-snooping  10  Gi0/1
! aa:bb:cc:00:01:02  10.1.10.101     86400       dhcp-snooping  10  Gi0/2
! aa:bb:cc:00:02:01  10.1.20.100     86400       dhcp-snooping  20  Gi0/9
```

**바인딩 테이블의 중요성:**

이 테이블은 단순히 DHCP Snooping만을 위한 것이 아닙니다. 다음 보안 기능들의 **기반 데이터**로 사용됩니다:

1. **DAI (Dynamic ARP Inspection):** ARP 패킷의 소스 MAC-IP가 바인딩 테이블과 일치하는지 검증
2. **IP Source Guard:** 패킷의 소스 IP/MAC가 바인딩 테이블과 일치하는지 검증
3. **포렌식/감사:** 어떤 MAC이 어떤 IP를 받았는지 추적 가능

**의존 관계:**

```
DHCP Snooping (기반)
    │
    ├── Binding Table 생성
    │       │
    │       ├── DAI가 참조
    │       │
    │       └── IP Source Guard가 참조
    │
    └── Trusted/Untrusted 분류
```

따라서 DHCP Snooping이 정상 동작하지 않으면 DAI와 IP Source Guard도 제대로 동작하지 않습니다. 이 세 가지는 반드시 함께 이해해야 합니다.

**바인딩 테이블 관리:**

```
! 수동으로 바인딩 추가 (Static IP 서버 등)
ip dhcp snooping binding aaaa.bbbb.cccc vlan 10 10.1.10.200 interface GigabitEthernet0/5 expiry infinite

! 바인딩 테이블 저장 (재부팅 대비)
ip dhcp snooping database flash:dhcp-snooping-db
! 또는 TFTP/FTP로 외부 저장
ip dhcp snooping database tftp://10.1.99.10/dhcp-snooping-db

! 저장 주기 설정
ip dhcp snooping database write-delay 120
ip dhcp snooping database timeout 300
```

Static IP를 사용하는 서버나 프린터는 DHCP 바인딩이 자동 생성되지 않으므로, 수동으로 추가해야 DAI와 IP Source Guard가 정상 동작합니다. 이 부분을 놓치면 Static IP 디바이스의 통신이 차단됩니다.

---

### 면접관: "좋아요. 그러면 ARP Spoofing 공격도 방어하고 싶대요. DAI를 어떻게 추가하나요?"

DAI(Dynamic ARP Inspection)는 ARP 패킷을 검사하여 ARP Spoofing/Poisoning 공격을 방지합니다.

**ARP Spoofing 공격 원리:**

```
정상 상태:
[PC-A: 10.1.10.100] ── ARP Request: "10.1.10.1의 MAC은?" ──→ [Gateway]
[PC-A] ←── ARP Reply: "10.1.10.1 = 00:aa:bb:cc:dd:ee" ──── [Gateway]

공격 상태:
[Attacker: 10.1.10.50] ── Gratuitous ARP: "10.1.10.1 = [공격자 MAC]" ──→ [PC-A]
[PC-A]는 게이트웨이 MAC을 공격자 MAC으로 갱신
→ PC-A의 모든 외부 트래픽이 공격자를 경유 (Man-in-the-Middle)
```

**DAI 설계:**

DAI는 **DHCP Snooping 바인딩 테이블**을 참조하여 ARP 패킷의 Source MAC과 Source IP가 바인딩과 일치하는지 검증합니다.

#### 핵심 설정

```
! 전제: DHCP Snooping이 이미 활성화되어 있어야 함
! ip dhcp snooping
! ip dhcp snooping vlan 10,20,30

! 1. DAI 활성화 (VLAN 단위)
ip arp inspection vlan 10,20,30

! 2. Trusted 포트 설정 (DHCP Snooping과 동일한 방향)
interface GigabitEthernet0/48
 ip arp inspection trust

! 3. Rate Limit 설정 (ARP 스톰 방지)
interface range GigabitEthernet0/1 - 24
 ip arp inspection limit rate 15

! 4. 추가 검증 옵션 (권장)
ip arp inspection validate src-mac dst-mac ip
```

**추가 검증 옵션 설명:**

```
ip arp inspection validate src-mac
! → ARP 페이로드의 Source MAC과 이더넷 헤더의 Source MAC 일치 확인

ip arp inspection validate dst-mac
! → ARP Reply의 Destination MAC과 이더넷 헤더의 Destination MAC 일치 확인

ip arp inspection validate ip
! → ARP 페이로드의 Source IP가 유효한지 확인
!    0.0.0.0, 255.255.255.255, 멀티캐스트 IP 등을 차단

! 세 가지를 한 줄로 동시 적용
ip arp inspection validate src-mac dst-mac ip
```

**Static IP 디바이스를 위한 ARP ACL:**

DHCP를 사용하지 않는 디바이스는 바인딩 테이블에 없으므로 DAI가 차단합니다. ARP ACL로 예외 처리합니다.

```
! ARP ACL 생성
arp access-list STATIC-DEVICES
 permit ip host 10.1.10.200 mac host aaaa.bbbb.cccc
 permit ip host 10.1.10.201 mac host aaaa.bbbb.dddd

! DAI에 ARP ACL 적용
ip arp inspection filter STATIC-DEVICES vlan 10
```

**확인 명령어:**

```
show ip arp inspection vlan 10
show ip arp inspection statistics vlan 10
show ip arp inspection interfaces
show ip arp inspection log
```

---

### 면접관: "DAI 적용 후에 특정 PC가 통신이 안 된다고 합니다. 어떻게 진단하나요?"

**1단계: DAI 드롭 통계 확인**

```
show ip arp inspection statistics vlan 10

!  Vlan  Forwarded  Dropped  DHCP Drops  ACL Drops
!  ----  ---------  -------  ----------  ---------
!  10    1520       47       47          0

! Dropped 카운터가 증가하고 있으면 DAI가 ARP를 차단하고 있는 것
```

**2단계: 드롭된 ARP 로그 확인**

```
show ip arp inspection log

! 출력 예시:
!  Interface  Vlan  Sender MAC       Sender IP     Num  Reason
!  ---------  ----  ---------------  ------------  ---  ------
!  Gi0/5      10    aaaa.bbbb.ffff   10.1.10.200   23   DHCP Deny
```

`DHCP Deny` = 바인딩 테이블에 해당 MAC-IP 매핑이 없음

**3단계: 원인 파악**

```
! 해당 MAC의 DHCP Snooping 바인딩 확인
show ip dhcp snooping binding | include aaaa.bbbb.ffff

! 바인딩이 없는 경우:
! 1. Static IP 디바이스 → ARP ACL 추가 필요
! 2. DHCP Lease가 만료됨 → DHCP 갱신 문제 확인
! 3. 포트를 옮긴 경우 → 바인딩이 이전 포트에 남아있음
! 4. MAC이 변경된 경우 (NIC 교체) → 바인딩 갱신 필요
```

**4단계: 해결**

```
! Static IP인 경우: ARP ACL 추가
arp access-list STATIC-DEVICES
 permit ip host 10.1.10.200 mac host aaaa.bbbb.ffff

ip arp inspection filter STATIC-DEVICES vlan 10

! 또는 수동 바인딩 추가
ip dhcp snooping binding aaaa.bbbb.ffff vlan 10 10.1.10.200 interface GigabitEthernet0/5 expiry infinite

! Rate Limit 초과로 err-disabled된 경우
show interfaces status err-disabled
errdisable recovery cause arp-inspection
errdisable recovery interval 300
```

**DAI 트러블슈팅 체크리스트:**

| 증상 | 원인 | 해결 |
|------|------|------|
| 특정 PC만 안 됨 | Static IP, 바인딩 없음 | ARP ACL 또는 수동 바인딩 |
| 전체 VLAN 안 됨 | Trusted 포트 미설정 | Uplink trust 설정 |
| 간헐적 장애 | Rate Limit 초과 | Rate Limit 값 조정 |
| 재부팅 후 장애 | 바인딩 테이블 소실 | DB 저장 설정 |
| DHCP Renew 실패 | 바인딩 불일치 | 바인딩 삭제 후 재할당 |

---

### 면접관: "IP Source Guard까지 추가하면 보안이 완벽해지나요? 어떻게 구성하나요?"

IP Source Guard는 DHCP Snooping 바인딩 테이블을 기반으로 **모든 IP 패킷의 Source IP(와 MAC)를 검증**하는 기능입니다. DAI가 ARP만 검사한다면, IP Source Guard는 모든 IP 트래픽을 검사합니다.

**3단계 보안 방어 체계:**

```
DHCP Snooping   → 비인가 DHCP 서버 차단
     │
     ↓
    DAI          → ARP Spoofing 차단
     │
     ↓
IP Source Guard  → IP/MAC Spoofing 차단 (모든 IP 트래픽)
```

#### IP Source Guard 설정

```
! IP 기반 필터링 (Source IP만 검증)
interface range GigabitEthernet0/1 - 24
 ip verify source

! IP + MAC 기반 필터링 (Source IP와 MAC 모두 검증)
interface range GigabitEthernet0/1 - 24
 ip verify source port-security
```

`ip verify source`는 패킷의 Source IP가 DHCP Snooping 바인딩 테이블에 있는지 확인합니다. 없으면 드롭.

`ip verify source port-security`는 Source IP와 Source MAC 모두를 검증합니다. Port Security와 연동되어 MAC 스푸핑까지 방지합니다.

**Static IP 디바이스 처리:**

```
! IP Source Guard도 바인딩 테이블에 의존하므로
! Static IP 디바이스는 수동 바인딩 필수
ip source binding aaaa.bbbb.cccc vlan 10 10.1.10.200 interface GigabitEthernet0/5
```

**확인 명령어:**

```
show ip verify source
show ip source binding

! 출력 예시:
! IP source binding table
! MAC Addr        IP Address    Lease   Type       VLAN  Interface
! aaaa.bbbb.cccc  10.1.10.100   86400   dhcp-snoop  10   Gi0/1
! aaaa.bbbb.dddd  10.1.10.200   infinite static     10   Gi0/5
```

**전체 보안 스택 설정 요약:**

```
! ===== 전역 설정 =====
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
no ip dhcp snooping information option
ip dhcp snooping database flash:dhcp-snooping.db

ip arp inspection vlan 10,20,30
ip arp inspection validate src-mac dst-mac ip

arp access-list STATIC-DEVICES
 permit ip host 10.1.10.200 mac host aaaa.bbbb.cccc

ip arp inspection filter STATIC-DEVICES vlan 10

errdisable recovery cause dhcp-rate-limit
errdisable recovery cause arp-inspection
errdisable recovery interval 300

! ===== Trusted 포트 (Uplink / DHCP 서버 방향) =====
interface GigabitEthernet0/48
 ip dhcp snooping trust
 ip arp inspection trust

! ===== Untrusted 포트 (Access 포트) =====
interface range GigabitEthernet0/1 - 24
 switchport mode access
 switchport port-security
 switchport port-security maximum 2
 switchport port-security violation restrict
 ip dhcp snooping limit rate 15
 ip arp inspection limit rate 15
 ip verify source port-security
 spanning-tree portfast
 spanning-tree bpduguard enable

! ===== Static IP 디바이스 수동 바인딩 =====
ip source binding aaaa.bbbb.cccc vlan 10 10.1.10.200 interface GigabitEthernet0/5
ip dhcp snooping binding aaaa.bbbb.cccc vlan 10 10.1.10.200 interface GigabitEthernet0/5 expiry infinite
```

**"보안이 완벽해지나요?"에 대한 답:**

이 3단계 방어(DHCP Snooping + DAI + IP Source Guard)로 L2 레벨의 주요 공격을 방어할 수 있지만, "완벽"은 아닙니다.

**추가로 고려해야 할 사항:**

| 위협 | 방어 | 비고 |
|------|------|------|
| 비인가 DHCP | DHCP Snooping | 본 문서에서 다룸 |
| ARP Spoofing | DAI | 본 문서에서 다룸 |
| IP Spoofing | IP Source Guard | 본 문서에서 다룸 |
| MAC Flooding | Port Security | 위 설정에 포함 |
| VLAN Hopping | DTP 비활성화, Native VLAN 변경 | 02-01 문서 참조 |
| STP 공격 | BPDU Guard, Root Guard | 02-02 문서 참조 |
| 802.1X 미인증 접근 | 802.1X + RADIUS | 별도 구성 필요 |
| 무선 Rogue AP | Wireless IDS/IPS | 별도 솔루션 필요 |
| Physical Layer 공격 | 물리적 보안 | 서버룸 잠금 등 |

L2 보안은 계층적으로 적용해야 하며, 각 레이어(L2, L3, L7, 물리적)마다 적절한 방어 메커니즘을 조합하는 것이 Defense-in-Depth 전략입니다.

---
