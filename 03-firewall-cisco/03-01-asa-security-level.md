# 03-01. ASA 기본 구성과 Security Level

---

## 면접관: "신규 고객사 본사에 Cisco ASA 방화벽을 도입하기로 했습니다. 내부 네트워크, 외부 인터넷, 그리고 DMZ에 웹서버가 있는 구조인데요. 기본 인터페이스 설계를 어떻게 하시겠어요?"

우선 ASA의 인터페이스를 3개 영역으로 나누어 설계하겠습니다.

- **Inside (GigabitEthernet0/1)**: 내부 사용자 네트워크 (192.168.10.0/24)
- **Outside (GigabitEthernet0/0)**: 외부 인터넷 연결 (ISP로부터 할당받은 공인 IP)
- **DMZ (GigabitEthernet0/2)**: 웹서버 등 외부에 공개해야 하는 서버 (172.16.1.0/24)

각 인터페이스에 Security Level을 부여하여 트래픽 흐름의 기본 정책을 정합니다. Security Level은 0~100 사이의 값으로, **높은 레벨에서 낮은 레벨로의 트래픽은 기본 허용**, **낮은 레벨에서 높은 레벨로의 트래픽은 기본 차단**됩니다.

> 토폴로지 이미지 추가 예정

### 핵심 설정

```
! 인터페이스 기본 구성
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.10.1 255.255.255.0
 no shutdown

interface GigabitEthernet0/2
 nameif dmz
 security-level 50
 ip address 172.16.1.1 255.255.255.0
 no shutdown

! 기본 라우팅
route outside 0.0.0.0 0.0.0.0 203.0.113.254
```

---

## 면접관: "Security Level 값을 왜 Inside 100, DMZ 50, Outside 0으로 설정하셨나요? 다른 값으로 하면 안 되나요?"

값 자체는 관례적인 것이고, 반드시 100/50/0이어야 하는 것은 아닙니다. 중요한 것은 **상대적인 크기 관계**입니다.

ASA의 Security Level 동작 원리는 다음과 같습니다.

| 트래픽 방향 | Security Level 비교 | 기본 동작 |
|---|---|---|
| Inside → Outside | 100 → 0 (높→낮) | **허용** (ACL 없이도) |
| Inside → DMZ | 100 → 50 (높→낮) | **허용** |
| DMZ → Outside | 50 → 0 (높→낮) | **허용** |
| Outside → Inside | 0 → 100 (낮→높) | **차단** (ACL 필요) |
| Outside → DMZ | 0 → 50 (낮→높) | **차단** (ACL 필요) |
| DMZ → Inside | 50 → 100 (낮→높) | **차단** (ACL 필요) |

예를 들어 Inside를 80, DMZ를 40, Outside를 0으로 설정해도 동작 논리는 동일합니다. 다만 100/50/0은 업계에서 보편적으로 사용하는 값이라 운영팀 간 의사소통과 문서화 측면에서 유리합니다.

한 가지 주의할 점은, **동일한 Security Level 간 트래픽은 기본적으로 차단**됩니다. 이 부분은 별도 설정이 필요합니다.

---

## 면접관: "그런데 외부에서 DMZ 웹서버(172.16.1.10)에 접속이 안 된다고 합니다. 어디부터 확인하시겠어요?"

Outside(0) → DMZ(50)는 낮은 Security Level에서 높은 Security Level로의 트래픽이므로 **ACL이 없으면 기본 차단**됩니다. 단계별로 확인하겠습니다.

**1단계: ACL 존재 여부 확인**

```
show running-config access-list
show access-list outside_access_in
```

Outside 인터페이스에 인바운드 ACL이 적용되어 있는지 확인합니다.

**2단계: NAT 설정 확인**

```
show nat
show nat detail
show xlate
```

외부에서 DMZ 서버에 접근하려면 Static NAT 또는 Static PAT가 설정되어 있어야 합니다. NAT가 없으면 외부에서 172.16.1.10이라는 사설 IP로 접근할 방법이 없습니다.

**3단계: ACL 및 NAT 설정 추가**

```
! Static NAT: 외부 공인 IP로 DMZ 웹서버 매핑
object network OBJ_DMZ_WEB
 host 172.16.1.10
 nat (dmz,outside) static 203.0.113.10

! ACL: 외부에서 웹서버 80/443 포트 허용
access-list outside_access_in extended permit tcp any host 203.0.113.10 eq www
access-list outside_access_in extended permit tcp any host 203.0.113.10 eq https
access-group outside_access_in in interface outside
```

**4단계: 연결 확인**

```
show xlate | include 172.16.1.10
show conn address 172.16.1.10
packet-tracer input outside tcp 8.8.8.8 12345 203.0.113.10 80
```

특히 `packet-tracer`는 ASA에서 트래픽이 각 단계(ACL, NAT, routing 등)를 통과하는지 시뮬레이션해주는 매우 강력한 트러블슈팅 도구입니다. 실제 패킷을 보내지 않고도 문제 지점을 정확히 파악할 수 있습니다.

---

## 면접관: "방금 말씀하신 것 중에 Stateful Inspection이라는 개념이 있는데, ASA에서 어떻게 동작하는지 설명해주시겠어요?"

ASA는 **Stateful Packet Inspection (SPI)** 방화벽입니다. 이것은 단순히 패킷의 헤더(IP, 포트)만 보는 것이 아니라, **연결 상태(Connection State)를 추적**한다는 의미입니다.

동작 방식을 예로 설명하겠습니다.

1. Inside 사용자(192.168.10.100)가 Outside 웹서버(8.8.8.8:80)로 TCP SYN을 보냄
2. ASA가 이 연결을 **conn table**에 기록 (출발지 IP/Port, 목적지 IP/Port, 프로토콜, 상태)
3. 웹서버의 응답 패킷(SYN-ACK)이 돌아올 때, ASA는 conn table을 조회하여 **이미 허용된 세션의 응답임을 확인**하고 통과시킴
4. Outside → Inside 방향의 ACL이 없어도, 기존 세션에 대한 응답은 자동 허용

```
! 현재 연결 상태 테이블 확인
show conn
show conn count
show conn detail

! 출력 예시
TCP outside 8.8.8.8:80 inside 192.168.10.100:54321, idle 0:00:05, bytes 15234, flags UIO
```

flags의 의미:
- **U**: Up (연결 활성)
- **I**: Inbound data
- **O**: Outbound data
- **B**: Initial SYN from outside (외부 발신)

Stateful Inspection의 핵심 이점은 **리턴 트래픽에 대해 별도 ACL을 만들 필요가 없다**는 것입니다. 전통적인 Stateless ACL(라우터 ACL)에서는 양방향 모두 permit 규칙을 작성해야 했지만, ASA에서는 세션 테이블이 이를 자동 처리합니다.

UDP나 ICMP처럼 connectionless 프로토콜도 ASA는 pseudo-session을 생성하여 추적합니다. 예를 들어 DNS 쿼리(UDP 53)를 보내면, 일정 시간(기본 2분) 동안 응답을 기다리는 pseudo-session이 생성됩니다.

---

## 면접관: "ASA의 conn table 엔트리 수에 한계가 있지 않나요? 대규모 환경에서 문제가 되진 않나요?"

맞습니다. ASA 모델별로 최대 동시 연결(concurrent connections) 수에 차이가 있습니다.

| 모델 | 최대 동시 연결 수 |
|---|---|
| ASA 5506-X | 50,000 |
| ASA 5508-X | 100,000 |
| ASA 5516-X | 250,000 |
| ASA 5525-X | 500,000 |
| ASA 5545-X | 750,000 |
| ASA 5555-X | 1,000,000 |

conn table이 가득 차면 신규 연결이 거부될 수 있으므로, 운영 시 모니터링이 중요합니다.

```
! 현재 연결 수 확인
show conn count
! 출력 예시: 12045 in use, 45000 most used

! 리소스 사용률 확인
show resource usage
show cpu usage
show memory
```

대규모 환경에서는 다음과 같은 대응을 합니다.

- **timeout 조정**: 불필요하게 오래 유지되는 세션 정리
- **connection limit 설정**: MPF(Modular Policy Framework)를 통해 호스트별 최대 연결 수 제한
- **dead connection detection**: idle 세션 정리

```
! 타임아웃 조정 예시
timeout conn 1:00:00
timeout half-closed 0:10:00
timeout udp 0:02:00
timeout icmp 0:00:02
timeout xlate 3:00:00
```

---

## 면접관: "고객사에서 DMZ에 서버가 2대 더 추가되는데, 서로 간에 통신이 필요하다고 합니다. 그런데 같은 DMZ 영역인데 통신이 안 된다네요. 어떻게 하시겠어요?"

같은 인터페이스(DMZ)에 연결된 서버들 간의 통신, 즉 **Same Security Level 간 트래픽**은 ASA에서 기본적으로 차단됩니다.

이를 해결하는 방법은 두 가지입니다.

### 방법 1: same-security-traffic permit (권장)

```
! 동일 Security Level 간 트래픽 허용
same-security-traffic permit inter-interface

! 동일 인터페이스 내 트래픽 허용 (hairpin/u-turn)
same-security-traffic permit intra-interface
```

- **inter-interface**: Security Level이 같은 **서로 다른 인터페이스** 간 트래픽 허용
- **intra-interface**: **같은 인터페이스**로 들어와서 같은 인터페이스로 나가는 트래픽 허용 (예: VPN 트래픽이 같은 outside 인터페이스로 hairpin)

DMZ 서버들이 같은 GigabitEthernet0/2에 연결되어 있다면 `intra-interface`가 필요하고, 만약 DMZ를 여러 서브인터페이스로 나눠서 같은 Security Level 50을 부여했다면 `inter-interface`가 필요합니다.

### 방법 2: 서브인터페이스로 분리 후 각각 다른 Security Level 부여

```
interface GigabitEthernet0/2.1
 vlan 101
 nameif dmz-web
 security-level 50
 ip address 172.16.1.1 255.255.255.0

interface GigabitEthernet0/2.2
 vlan 102
 nameif dmz-db
 security-level 40
 ip address 172.16.2.1 255.255.255.0
```

이 경우 dmz-web(50) → dmz-db(40)는 높→낮이므로 기본 허용되고, 반대 방향은 ACL이 필요합니다. 보안 관점에서 DMZ 내에서도 웹서버와 DB서버의 보안 등급을 다르게 가져가고 싶다면 이 방법이 더 세밀한 제어가 가능합니다.

---

## 면접관: "마지막으로, 이 고객사에서 ASA를 이중화하고 싶다고 합니다. Active/Standby Failover를 구성한다면 Security Level과 관련해서 추가로 고려할 사항이 있을까요?"

ASA Failover를 구성할 때 Security Level 자체는 Active/Standby 간 동일하게 복제되므로 별도 고려사항은 없습니다. 하지만 Failover 구성 시 인터페이스 설계와 관련된 중요한 포인트가 있습니다.

**Failover 전용 인터페이스 구성**

```
! Failover 링크 설정 (전용 인터페이스 권장)
interface GigabitEthernet0/3
 description LAN Failover Link
 no shutdown

failover
failover lan unit primary
failover lan interface failover-link GigabitEthernet0/3
failover key *****
failover link failover-link GigabitEthernet0/3

! Failover 인터페이스 IP (Primary / Standby)
failover interface ip failover-link 10.0.0.1 255.255.255.252 standby 10.0.0.2
```

**Failover와 관련된 주의사항**:

1. **Failover 링크**는 별도 인터페이스를 사용하는 것을 권장합니다. Security Level이 부여되는 데이터 인터페이스와 분리해야 합니다.
2. **Stateful Failover**를 구성하면 conn table, NAT xlate table이 Standby로 복제되어 failover 시 기존 세션이 유지됩니다.
3. 각 데이터 인터페이스에는 **Active IP와 Standby IP** 두 개를 설정해야 합니다.

```
! 데이터 인터페이스에 Standby IP 추가
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.10.1 255.255.255.0 standby 192.168.10.2

! Stateful Failover 활성화
failover link state-link GigabitEthernet0/3
failover interface ip state-link 10.0.0.1 255.255.255.252 standby 10.0.0.2
```

```
! Failover 상태 확인
show failover
show failover state
```

Failover 상태에서 Standby 장비가 Active로 전환되면, 모든 인터페이스의 nameif, security-level, ACL, NAT 설정이 동일하게 적용되므로 트래픽 정책에는 변함이 없습니다.
