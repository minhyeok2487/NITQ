# 01-05. ARP와 MAC 주소 테이블

> CCNA 200-301 수준의 네트워크 엔지니어 면접 시나리오입니다.
> ARP 동작 과정, ARP 테이블, 스위치 MAC 주소 테이블, Gratuitous ARP, 트러블슈팅 명령어 등을 다룹니다.

---

### 면접관: "ARP(Address Resolution Protocol)가 무엇이고, 왜 필요한지 설명해 주세요."

ARP는 IP 주소(3계층 논리적 주소)를 MAC 주소(2계층 물리적 주소)로 변환하는 프로토콜입니다. OSI 모델에서 2~3계층 사이에서 동작합니다.

**ARP가 필요한 이유:**

네트워크에서 데이터를 실제로 전송하려면 목적지의 MAC 주소가 필요합니다. 우리가 통신할 때 IP 주소는 알고 있지만, 해당 IP 주소를 가진 장비의 MAC 주소는 모르는 경우가 대부분입니다. ARP는 이 문제를 해결합니다.

예를 들어, PC-A(192.168.1.10)가 PC-B(192.168.1.20)에게 데이터를 보내려면 다음이 필요합니다.

```
전송에 필요한 정보:
- 출발지 IP: 192.168.1.10     (알고 있음)
- 목적지 IP: 192.168.1.20     (알고 있음)
- 출발지 MAC: AA:BB:CC:11:22:33  (자신의 MAC, 알고 있음)
- 목적지 MAC: ??:??:??:??:??:??  (모름 → ARP로 확인!)
```

ARP가 없다면 IP 주소만으로는 같은 네트워크 내에서도 실제 프레임을 전달할 수 없습니다.

---

### 면접관: "ARP의 동작 과정을 Request와 Reply 단계로 자세히 설명해 주세요."

ARP는 **ARP Request (요청)**와 **ARP Reply (응답)** 두 단계로 동작합니다.

**시나리오:** PC-A(192.168.1.10)가 PC-B(192.168.1.20)의 MAC 주소를 알아내는 과정

#### ARP Request (브로드캐스트)

```
PC-A → 네트워크 전체 (브로드캐스트)

"192.168.1.20의 MAC 주소를 알려주세요!"

ARP Request 프레임:
+-----------------+-------------------+
| 출발지 MAC      | AA:BB:CC:11:22:33 |
| 목적지 MAC      | FF:FF:FF:FF:FF:FF | (브로드캐스트)
| 출발지 IP       | 192.168.1.10      |
| 목적지 IP       | 192.168.1.20      |
| Opcode          | 1 (Request)       |
+-----------------+-------------------+
```

- ARP Request는 **브로드캐스트**(FF:FF:FF:FF:FF:FF)로 전송됩니다.
- 같은 네트워크의 모든 장비가 이 프레임을 수신합니다.
- 해당 IP를 가지지 않은 장비는 이 요청을 무시합니다.

#### ARP Reply (유니캐스트)

```
PC-B → PC-A (유니캐스트)

"192.168.1.20은 저입니다. 제 MAC 주소는 DD:EE:FF:44:55:66입니다."

ARP Reply 프레임:
+-----------------+-------------------+
| 출발지 MAC      | DD:EE:FF:44:55:66 |
| 목적지 MAC      | AA:BB:CC:11:22:33 | (유니캐스트)
| 출발지 IP       | 192.168.1.20      |
| 목적지 IP       | 192.168.1.10      |
| Opcode          | 2 (Reply)         |
+-----------------+-------------------+
```

- ARP Reply는 **유니캐스트**로 요청한 장비에게만 전송됩니다.
- PC-A는 받은 정보를 ARP 캐시(테이블)에 저장합니다.

#### 전체 흐름 요약

```
PC-A                    스위치                   PC-B
 |                        |                       |
 |  ARP Request           |                       |
 |  (브로드캐스트)         |  모든 포트로 전달       |
 |  =====================>|=======================>|
 |                        |                       |
 |                        |      ARP Reply        |
 |                        |      (유니캐스트)      |
 |<======================|<=======================|
 |                        |                       |
 | [ARP 캐시에 저장]       |                       |
 | 192.168.1.20 →         |                       |
 |  DD:EE:FF:44:55:66     |                       |
```

---

### 면접관: "ARP 테이블과 스위치의 MAC 주소 테이블은 어떻게 다른가요?"

이 두 테이블은 역할과 위치가 다릅니다. 혼동하기 쉬운 개념이므로 명확히 구분하겠습니다.

| 항목 | ARP 테이블 | MAC 주소 테이블 |
|------|-----------|----------------|
| **위치** | 라우터, PC, L3 장비 | L2 스위치 |
| **매핑 정보** | IP 주소 ↔ MAC 주소 | MAC 주소 ↔ 포트 번호 |
| **목적** | IP로 MAC을 찾기 위함 | 프레임을 어느 포트로 보낼지 결정 |
| **학습 방법** | ARP Request/Reply | 수신 프레임의 출발지 MAC 학습 |
| **계층** | L2~L3 (IP↔MAC) | L2 (MAC↔포트) |
| **타임아웃** | 보통 120~300초 | 보통 300초 (5분) |

#### ARP 테이블 확인

```
! 라우터에서 ARP 테이블 확인
Router# show arp
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  192.168.1.1             -   aabb.cc11.2233  ARPA   Gi0/0
Internet  192.168.1.10           5   ddee.ff44.5566  ARPA   Gi0/0
Internet  192.168.1.20           3   1122.3344.5566  ARPA   Gi0/0

! Windows PC에서 ARP 캐시 확인
C:\> arp -a
Interface: 192.168.1.10
  Internet Address    Physical Address    Type
  192.168.1.1         aa-bb-cc-11-22-33   dynamic
  192.168.1.20        dd-ee-ff-44-55-66   dynamic
```

#### MAC 주소 테이블 확인

```
! 스위치에서 MAC 주소 테이블 확인
Switch# show mac address-table
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    aabb.cc11.2233    DYNAMIC     Gi0/1
   1    ddee.ff44.5566    DYNAMIC     Gi0/2
   1    1122.3344.5566    DYNAMIC     Gi0/3
```

스위치는 프레임을 수신할 때, 출발지 MAC 주소와 수신 포트를 MAC 주소 테이블에 기록합니다. 이후 해당 MAC 주소로 향하는 프레임이 오면, 테이블을 참조하여 해당 포트로만 전달합니다 (유니캐스트 포워딩).

만약 목적지 MAC 주소가 테이블에 없으면, 수신 포트를 제외한 모든 포트로 프레임을 전송합니다 (Unknown Unicast Flooding).

---

### 면접관: "ARP 캐시에 타임아웃이 있는 이유는 무엇인가요? 그리고 정적 ARP 항목은 언제 사용하나요?"

**ARP 캐시 타임아웃이 필요한 이유:**

1. **IP-MAC 매핑 변경 대응**: 장비의 NIC가 교체되거나 IP 주소가 변경되면 기존 ARP 항목이 더 이상 유효하지 않습니다. 타임아웃이 없으면 잘못된 매핑 정보로 통신 장애가 발생합니다.

2. **메모리 관리**: 더 이상 통신하지 않는 장비의 정보를 계속 유지하면 메모리가 낭비됩니다.

3. **네트워크 변경 반영**: DHCP 환경에서는 IP 주소가 주기적으로 변경될 수 있으므로, ARP 캐시도 주기적으로 갱신되어야 합니다.

**일반적인 타임아웃 값:**
- Cisco IOS: 240분 (4시간)
- Windows: 15~45초 (사용하지 않으면)
- Linux: 60초 ~ 수분

```
! Cisco에서 ARP 타임아웃 설정
Router(config)# interface GigabitEthernet0/0
Router(config-if)# arp timeout 300

! ARP 캐시 수동 삭제
Router# clear arp-cache
```

**정적 ARP 항목 사용 사례:**

정적 ARP는 수동으로 IP-MAC 매핑을 설정하는 것으로, 다음과 같은 경우에 사용합니다.

- **보안 강화**: ARP 스푸핑 공격을 방지하기 위해 게이트웨이 등 중요 장비의 ARP를 정적으로 설정
- **특수 환경**: ARP를 지원하지 않는 장비와 통신할 때

```
! 정적 ARP 설정
Router(config)# arp 192.168.1.100 ddee.ff44.5566 arpa

! Windows에서 정적 ARP 설정
C:\> arp -s 192.168.1.1 aa-bb-cc-11-22-33

! ARP 테이블에서 static 항목 확인
Router# show arp
Internet  192.168.1.100    -    ddee.ff44.5566  ARPA   Gi0/0
```

정적 항목은 타임아웃이 적용되지 않아 수동으로 삭제하기 전까지 유지됩니다.

---

### 면접관: "Gratuitous ARP란 무엇인가요? 어떤 상황에서 사용되나요?"

Gratuitous ARP는 자신의 IP 주소에 대해 자기 자신에게 보내는 ARP Request입니다. 즉, 출발지 IP와 목적지 IP가 동일한 ARP 요청입니다.

```
Gratuitous ARP 프레임:
+-----------------+-------------------+
| 출발지 MAC      | AA:BB:CC:11:22:33 |
| 목적지 MAC      | FF:FF:FF:FF:FF:FF | (브로드캐스트)
| 출발지 IP       | 192.168.1.10      |
| 목적지 IP       | 192.168.1.10      | (자기 자신!)
| Opcode          | 1 (Request)       |
+-----------------+-------------------+
```

**Gratuitous ARP의 사용 목적:**

1. **IP 주소 중복 감지**: 장비가 부팅되거나 IP를 설정할 때 Gratuitous ARP를 보내서, 같은 IP를 사용하는 다른 장비가 있는지 확인합니다. 만약 응답이 오면 IP가 중복되었다는 의미입니다.

2. **ARP 캐시 갱신**: 다른 장비의 ARP 캐시를 강제로 업데이트합니다. NIC 교체 등으로 MAC 주소가 변경된 경우 유용합니다.

3. **HSRP/VRRP 장애 전환**: 게이트웨이 이중화 프로토콜에서 Active 장비가 변경되면, 새로운 Active 장비가 Gratuitous ARP를 보내서 트래픽을 자신에게로 전환합니다.

4. **가상화 환경**: VM이 다른 호스트로 마이그레이션될 때 Gratuitous ARP를 전송하여 스위치의 MAC 테이블을 갱신합니다.

```
! Cisco에서 IP 중복 감지 시 로그 메시지
%IP-4-DUPADDR: Duplicate address 192.168.1.10 on GigabitEthernet0/0,
sourced by aabb.cc44.5566
```

---

### 면접관: "같은 네트워크가 아닌 다른 네트워크로 통신할 때 ARP는 어떻게 동작하나요?"

다른 네트워크로 통신할 때 ARP는 **기본 게이트웨이(Default Gateway)**의 MAC 주소를 요청합니다. 목적지 호스트의 MAC 주소를 직접 알아내는 것이 아닙니다.

**시나리오:** PC-A(192.168.1.10/24)가 서버(10.0.0.100/24)로 통신하는 경우

```
PC-A (192.168.1.10)
   |
   |  GW: 192.168.1.1
   |
[라우터] (192.168.1.1 / 10.0.0.1)
   |
   |
서버 (10.0.0.100)
```

#### 통신 과정

**1단계: PC-A가 목적지가 다른 네트워크임을 판단**

```
PC-A IP:    192.168.1.10 / 255.255.255.0
목적지 IP:  10.0.0.100

192.168.1.10 AND 255.255.255.0 = 192.168.1.0 (자신의 네트워크)
10.0.0.100   AND 255.255.255.0 = 10.0.0.0    (목적지 네트워크)

→ 다른 네트워크! 기본 게이트웨이로 보내야 함
```

**2단계: PC-A가 게이트웨이(192.168.1.1)의 MAC 주소를 ARP로 요청**

```
ARP Request: "192.168.1.1의 MAC 주소를 알려주세요!"
ARP Reply (라우터): "저의 MAC 주소는 00:11:22:33:44:55입니다"
```

**3단계: PC-A가 프레임을 게이트웨이의 MAC 주소로 전송**

```
프레임 헤더:
  출발지 MAC: PC-A의 MAC
  목적지 MAC: 라우터의 MAC (00:11:22:33:44:55)

IP 헤더:
  출발지 IP: 192.168.1.10
  목적지 IP: 10.0.0.100  (최종 목적지는 변하지 않음)
```

**4단계: 라우터가 목적지 네트워크에서 ARP 수행**

라우터는 서버(10.0.0.100)의 MAC 주소를 알아내기 위해 해당 네트워크에서 ARP를 수행하고, 프레임의 MAC 주소를 변경하여 서버에게 전달합니다.

**핵심 포인트:** 라우터를 통과할 때마다 L2 헤더(MAC 주소)는 변경되지만, L3 헤더(IP 주소)는 변경되지 않습니다(NAT 제외).

---

### 면접관: "ARP 관련 트러블슈팅을 할 때 어떤 명령어를 사용하고, 어떤 문제들이 발생할 수 있나요?"

ARP 관련 문제는 네트워크에서 흔히 발생하며, 체계적인 접근이 필요합니다.

#### 주요 트러블슈팅 명령어

```
! Cisco 라우터/L3 스위치
Router# show arp                        # ARP 테이블 전체 확인
Router# show arp | include 192.168.1    # 특정 네트워크 ARP 필터링
Router# clear arp-cache                 # ARP 캐시 전체 삭제

! Cisco L2 스위치
Switch# show mac address-table                          # MAC 테이블 전체
Switch# show mac address-table address aabb.cc11.2233  # 특정 MAC 검색
Switch# show mac address-table interface Gi0/1          # 특정 포트의 MAC
Switch# show mac address-table vlan 10                  # 특정 VLAN의 MAC
Switch# clear mac address-table dynamic                 # 동적 MAC 항목 삭제

! Windows
C:\> arp -a                   # ARP 캐시 확인
C:\> arp -d *                 # ARP 캐시 전체 삭제
C:\> arp -d 192.168.1.1      # 특정 항목 삭제

! Linux
$ arp -n                      # ARP 캐시 확인
$ ip neigh show               # ARP 캐시 확인 (최신 명령어)
$ ip neigh flush all          # ARP 캐시 삭제
```

#### 자주 발생하는 ARP 관련 문제

| 문제 | 증상 | 원인 | 해결 방법 |
|------|------|------|----------|
| IP 중복 | 간헐적 통신 장애 | 동일 IP를 두 장비가 사용 | show arp에서 MAC 주소 확인, 중복 장비 찾기 |
| ARP 스푸핑 | 트래픽 탈취 | 악의적 ARP Reply | Dynamic ARP Inspection(DAI) 설정 |
| Incomplete ARP | 특정 IP 통신 불가 | 상대방 미응답/다운 | 물리적 연결 확인, IP 설정 확인 |
| ARP 테이블 가득 참 | 성능 저하 | 대규모 네트워크 | 서브넷 분할, ARP 타이머 조정 |

```
! Incomplete ARP 확인 (상대방이 응답하지 않는 경우)
Router# show arp
Internet  192.168.1.50     0    Incomplete      ARPA

! 이 경우 다음을 확인:
! 1. 해당 IP의 장비가 켜져 있는지
! 2. 케이블이 연결되어 있는지
! 3. IP 주소 설정이 올바른지
! 4. 방화벽이 ARP를 차단하고 있지 않은지
```

#### ARP 보안: Dynamic ARP Inspection (DAI)

```
! DAI 설정 (스위치에서)
Switch(config)# ip arp inspection vlan 10
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# ip arp inspection trust    ! 신뢰할 수 있는 포트(업링크)

Switch# show ip arp inspection vlan 10
```

DAI는 DHCP Snooping 바인딩 테이블을 참조하여 유효하지 않은 ARP 패킷을 차단합니다. ARP 스푸핑 공격으로부터 네트워크를 보호하는 중요한 보안 기능입니다.
