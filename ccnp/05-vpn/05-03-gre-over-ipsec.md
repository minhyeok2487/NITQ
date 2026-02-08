# 05-03. GRE over IPSec

---

### 면접관: "고객사에서 본사-지사 간 IPSec VPN을 쓰고 있는데, VPN 위에서 OSPF 동적 라우팅을 돌리고 싶다고 합니다. 가능한가요? 어떻게 구성하시겠어요?"

기본 IPSec VPN(Crypto Map 방식)은 **포인트-투-포인트 터널 인터페이스가 없기 때문에** 동적 라우팅 프로토콜을 직접 구동할 수 없습니다. OSPF나 EIGRP는 Next-hop과 인접 관계(Adjacency)를 맺을 인터페이스가 필요한데, Crypto Map은 물리 인터페이스에 붙는 정책일 뿐 논리적 인터페이스가 아닙니다.

이를 해결하기 위해 **GRE over IPSec** 구성을 제안드리겠습니다.

**GRE(Generic Routing Encapsulation)** 터널을 먼저 만들어서 논리적 Point-to-Point 인터페이스를 생성하고, 그 GRE 터널을 **IPSec으로 암호화**하는 방식입니다.

> 토폴로지 이미지 추가 예정

#### 설계 개요

| 항목 | 본사 (HQ) | 지사 (Branch) |
|------|-----------|---------------|
| 장비 | Cisco ISR 4331 | Cisco ISR 4221 |
| WAN IP | 203.0.113.1 | 198.51.100.1 |
| 내부 대역 | 10.1.0.0/16 | 10.2.0.0/16 |
| Tunnel IP | 172.16.0.1/30 | 172.16.0.2/30 |
| 라우팅 | OSPF Area 0 | OSPF Area 0 |

#### 핵심 설정 (HQ Router)

```
! ── GRE Tunnel 인터페이스 ──
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 198.51.100.1
 tunnel mode gre ip
 ip ospf 1 area 0
 ip mtu 1400
 ip tcp adjust-mss 1360

! ── OSPF 설정 ──
router ospf 1
 network 10.1.0.0 0.0.255.255 area 0
 network 172.16.0.0 0.0.0.3 area 0

! ── IPSec으로 GRE 터널 보호 ──
! 방법 1: Crypto Map 방식
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key C0mpl3x!Key address 198.51.100.1

crypto ipsec transform-set TSET-GRE esp-aes 256 esp-sha256-hmac
 mode transport

! GRE 트래픽(Protocol 47)만 암호화
access-list 101 permit gre host 203.0.113.1 host 198.51.100.1

crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set TSET-GRE
 match address 101

interface GigabitEthernet0/0/0
 ip address 203.0.113.1 255.255.255.0
 crypto map CMAP-GRE
```

#### 핵심 설정 (Branch Router)

```
interface Tunnel0
 ip address 172.16.0.2 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 203.0.113.1
 tunnel mode gre ip
 ip ospf 1 area 0
 ip mtu 1400
 ip tcp adjust-mss 1360

router ospf 1
 network 10.2.0.0 0.0.255.255 area 0
 network 172.16.0.0 0.0.0.3 area 0

crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key C0mpl3x!Key address 203.0.113.1

crypto ipsec transform-set TSET-GRE esp-aes 256 esp-sha256-hmac
 mode transport

access-list 101 permit gre host 198.51.100.1 host 203.0.113.1

crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TSET-GRE
 match address 101

interface GigabitEthernet0/0/0
 ip address 198.51.100.1 255.255.255.0
 crypto map CMAP-GRE
```

---

### 면접관: "GRE와 IPSec의 근본적인 차이점이 뭔가요? 왜 GRE만 쓰면 안 되고, IPSec만 쓰면 안 되는 건가요?"

#### GRE vs IPSec 핵심 비교

| 항목 | GRE | IPSec |
|------|-----|-------|
| 목적 | **터널링(Encapsulation)** | **보안(Encryption)** |
| 암호화 | X (없음) | O (기밀성 + 무결성 + 인증) |
| 멀티캐스트 지원 | **O** | X (기본 IPSec은 유니캐스트만) |
| 브로드캐스트 지원 | **O** | X |
| 동적 라우팅 | **O** (터널 인터페이스 존재) | X (Crypto Map 방식에서는 불가) |
| 프로토콜 유연성 | 모든 L3 프로토콜 캡슐화 가능 | IP만 |
| 오버헤드 | 24 bytes (GRE 헤더) | 50~57+ bytes (ESP 헤더) |
| IP Protocol | 47 | 50 (ESP) / 51 (AH) |

#### 왜 둘 다 필요한가

**GRE만 쓰면**: 터널은 만들어지고 OSPF도 돌아가지만, 데이터가 **평문으로 인터넷을 통과**합니다. 보안이 없습니다.

**IPSec만 쓰면 (Crypto Map)**: 데이터는 암호화되지만, 터널 인터페이스가 없어서 **동적 라우팅을 돌릴 수 없습니다**. 또한 멀티캐스트/브로드캐스트를 전달하지 못합니다.

**GRE over IPSec**: GRE가 터널링과 라우팅을 담당하고, IPSec이 그 GRE 패킷을 암호화합니다. 두 프로토콜의 장점을 결합한 것입니다.

#### 패킷 구조 (GRE over IPSec, Transport Mode)

```
[New IP Header] [ESP Header] [GRE Header] [Original IP Header] [Payload] [ESP Trailer]
                ← IPSec 암호화 범위 (Transport Mode) →

! Transport Mode: 원래 IP 헤더(WAN IP)를 유지하고 GRE+Payload만 암호화
! Tunnel Mode:    새로운 IP 헤더를 추가하고 원래 IP 헤더+GRE+Payload 전체를 암호화
```

---

### 면접관: "구성했는데, VPN 통신은 되는데 특정 애플리케이션에서 패킷이 잘리거나 느려지는 현상이 발생합니다. 어디부터 확인하시겠어요?"

이 증상은 거의 확실하게 **MTU/Fragmentation 문제**입니다. GRE over IPSec에서 가장 흔하고 가장 까다로운 문제입니다.

#### 문제 원인

일반 이더넷 MTU는 **1500 bytes**입니다. GRE over IPSec을 적용하면 헤더가 추가되면서 실제 전달 가능한 페이로드가 줄어듭니다.

```
원래 MTU:        1500 bytes
- GRE 헤더:     - 24 bytes
- ESP 헤더:     - 50~57 bytes (AES-256 + SHA-256 기준)
────────────────────────────
남은 페이로드:    ~1419~1426 bytes
```

1500 bytes 패킷이 Tunnel 인터페이스에 도착하면 캡슐화 후 1500을 초과하므로 **Fragmentation**이 발생합니다. DF(Don't Fragment) 비트가 설정되어 있으면 패킷이 그냥 **드롭**됩니다.

#### 진단 순서

```
! 1단계: Tunnel 인터페이스의 현재 MTU 확인
show interface Tunnel0 | include MTU

! 2단계: 실제 통과 가능한 MTU 테스트 (상대방 내부 IP로)
ping 10.2.1.1 size 1500 df-bit source 10.1.1.1
ping 10.2.1.1 size 1400 df-bit source 10.1.1.1
! → 1400은 되고 1500은 안 되면 MTU 문제 확인

! 3단계: Fragment 카운터 확인
show interface Tunnel0 | include fragment
show crypto ipsec sa | include frag
```

#### 해결 방법

```
! ── 방법 1: Tunnel MTU 조정 ──
interface Tunnel0
 ip mtu 1400

! ── 방법 2: TCP MSS Clamping (TCP 트래픽에 가장 효과적) ──
interface Tunnel0
 ip tcp adjust-mss 1360

! ── 방법 3: PMTUD(Path MTU Discovery) 활용 ──
! DF 비트가 있는 패킷이 MTU를 초과하면 ICMP "Fragmentation Needed" 반환
! → 출발지가 패킷 크기를 줄임
! 주의: 중간 장비에서 ICMP를 차단하면 PMTUD 실패 (Black Hole)

! ICMP Unreachable 허용 확인 (중요!)
interface GigabitEthernet0/0/0
 no ip unreachables   ← 이게 설정되어 있으면 PMTUD 실패!
```

#### MTU vs MSS 정리

| 항목 | MTU | MSS |
|------|-----|-----|
| 계층 | L3 (IP) | L4 (TCP) |
| 의미 | 인터페이스가 전달 가능한 최대 패킷 크기 | TCP 세그먼트의 최대 페이로드 크기 |
| 관계 | - | MSS = MTU - 20(IP) - 20(TCP) = MTU - 40 |
| 설정 위치 | `ip mtu` | `ip tcp adjust-mss` |
| UDP 적용 | O | X (TCP 전용) |

**실무 권장값**: `ip mtu 1400` + `ip tcp adjust-mss 1360`으로 설정하면 대부분의 GRE over IPSec 환경에서 문제가 해결됩니다.

---

### 면접관: "아까 설정에서 Transport Mode를 쓰셨는데, Tunnel Mode와 무슨 차이가 있나요? 어떤 상황에서 어떤 걸 써야 하나요?"

#### IPSec Tunnel Mode vs Transport Mode

| 항목 | Tunnel Mode | Transport Mode |
|------|-------------|----------------|
| IP 헤더 | **새 IP 헤더 추가** (원본 IP도 암호화) | 원본 IP 헤더 유지 |
| 암호화 범위 | 원본 IP 헤더 + Payload 전체 | Payload만 (L4 이상) |
| 오버헤드 | 크다 (IP 헤더 20B 추가) | 작다 |
| 사용 사례 | **순수 IPSec VPN** (Crypto Map) | **GRE over IPSec** |

#### GRE over IPSec에서 Transport Mode를 쓰는 이유

GRE 터널이 이미 **새로운 IP 헤더(WAN IP)**를 추가하기 때문에, IPSec에서 또다시 Tunnel Mode로 IP 헤더를 추가하면 **이중 캡슐화**가 됩니다. 불필요하게 오버헤드가 증가합니다.

```
! Tunnel Mode (비효율적 - 이중 캡슐화):
[New IPSec IP][ESP][New GRE IP][GRE][Original IP][Payload]
 ← 20 bytes 낭비 →

! Transport Mode (효율적):
[GRE IP][ESP][GRE][Original IP][Payload]
 ← IP 헤더 재사용 →
```

#### 언제 어떤 모드를 쓰는가

| 구성 | 권장 IPSec Mode | 이유 |
|------|----------------|------|
| 순수 IPSec (Crypto Map) | **Tunnel Mode** | 원본 내부 IP를 숨기기 위해 새 IP 헤더 필요 |
| GRE over IPSec | **Transport Mode** | GRE가 이미 외부 IP 헤더를 제공 |
| VTI (Virtual Tunnel Interface) | **Tunnel Mode** | GRE 없이 IPSec 터널 인터페이스 사용 |

```
! Transport Mode 설정
crypto ipsec transform-set TSET-GRE esp-aes 256 esp-sha256-hmac
 mode transport

! Tunnel Mode 설정 (기본값)
crypto ipsec transform-set TSET-IPSEC esp-aes 256 esp-sha256-hmac
 mode tunnel
```

---

### 면접관: "GRE over IPSec 구성에서 Recursive Routing 문제가 생긴다고 들었는데, 그게 뭔가요?"

**Recursive Routing**은 GRE 터널 구성 시 발생할 수 있는 심각한 라우팅 루프 문제입니다.

#### 문제 상황

GRE 터널의 Destination(상대방 WAN IP)으로 가는 경로가 **터널 인터페이스 자체를 통과**하게 되면 무한 루프가 발생합니다.

```
! 문제 시나리오:
! Tunnel Destination: 198.51.100.1
! 라우팅 테이블에 Default Route가 Tunnel0을 가리킴

! 패킷 흐름:
! 1. 10.1.1.1 → 10.2.1.1 패킷 발생
! 2. GRE 캡슐화: Source 203.0.113.1, Dest 198.51.100.1
! 3. 198.51.100.1로 가는 경로 조회 → Default Route → Tunnel0 ?!
! 4. 다시 GRE 캡슐화... → 무한 루프 → Tunnel Down!

%TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing
```

#### 발생 조건

OSPF를 터널 위에서 돌리면, 상대방으로부터 Default Route(0.0.0.0/0)를 학습하게 될 수 있습니다. 이 Default Route가 터널의 Destination IP에 대한 경로도 Tunnel 인터페이스로 잡아버리면 Recursive Routing이 발생합니다.

#### 해결 방법

```
! ── 방법 1: Static Route로 터널 Destination을 물리 인터페이스로 고정 (가장 권장) ──
ip route 198.51.100.1 255.255.255.255 203.0.113.254
! (203.0.113.254 = WAN 게이트웨이)

! ── 방법 2: tunnel route-via 명령 (IOS 15.x 이상) ──
interface Tunnel0
 tunnel route-via GigabitEthernet0/0/0

! ── 방법 3: OSPF에서 터널 Destination 네트워크 필터링 ──
router ospf 1
 distribute-list prefix PL-NO-DEFAULT in

ip prefix-list PL-NO-DEFAULT seq 5 deny 0.0.0.0/0
ip prefix-list PL-NO-DEFAULT seq 10 permit 0.0.0.0/0 le 32
```

#### 실무 Best Practice

```
! 터널 Destination에 대한 Static Route는 반드시 설정
ip route 198.51.100.1 255.255.255.255 203.0.113.254

! OSPF에서 Default Route 전파 시 주의
! → OSPF로 Default Route를 받더라도, 터널 Destination은 Static이 우선 (AD 1 vs 110)
! → Administrative Distance 차이로 보호됨
```

---

### 면접관: "GRE over IPSec 말고 VTI(Virtual Tunnel Interface)라는 것도 있다고 들었는데, 차이점이 뭔가요? 요즘은 어떤 걸 더 많이 쓰나요?"

#### GRE over IPSec vs VTI 비교

| 항목 | GRE over IPSec | VTI (IPSec VTI) |
|------|---------------|-----------------|
| 터널 인터페이스 | `tunnel mode gre ip` | `tunnel mode ipsec ipv4` |
| GRE 오버헤드 | O (24 bytes) | **X (없음)** |
| 동적 라우팅 | O | **O** |
| 멀티캐스트 | O | O (최신 IOS) |
| ACL 필요 | O (GRE 트래픽 매칭) | **X (자동)** |
| Crypto Map 필요 | O | **X (IPSec Profile 사용)** |
| 설정 복잡도 | 높음 | **낮음** |
| MTU 효율 | 낮음 (GRE+ESP 오버헤드) | **높음 (ESP 오버헤드만)** |

#### VTI 설정 예시

```
! ── IKEv2 + VTI (현재 권장 방식) ──
crypto ikev2 proposal PROP-VTI
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-VTI
 proposal PROP-VTI

crypto ikev2 keyring KR-VTI
 peer BRANCH
  address 198.51.100.1
  pre-shared-key C0mpl3x!Key

crypto ikev2 profile PROF-VTI
 match identity remote address 198.51.100.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-VTI

crypto ipsec transform-set TSET-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── IPSec Profile (Crypto Map 대신 사용) ──
crypto ipsec profile IPSEC-PROF-VTI
 set transform-set TSET-VTI
 set ikev2-profile PROF-VTI
 set pfs group14

! ── VTI 인터페이스 ──
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 198.51.100.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF-VTI
 ip ospf 1 area 0
 ip mtu 1440
 ip tcp adjust-mss 1400
```

#### 요즘 추세

**신규 구축이라면 VTI를 권장**합니다:
- GRE 오버헤드가 없어서 MTU 효율이 좋습니다
- ACL과 Crypto Map이 불필요해서 설정이 간결합니다
- IKEv2 + VTI 조합이 현재 Cisco 권장 Best Practice입니다

**GRE over IPSec을 써야 하는 경우**:
- Non-IP 프로토콜(IPX 등)을 터널링해야 할 때 (매우 드뭄)
- 레거시 장비(오래된 IOS)에서 VTI 미지원 시
- DMVPN 구성 시 (DMVPN은 mGRE 기반)

```
! 현재 터널 모드 확인
show interface Tunnel0 | include Tunnel mode
! 출력 예: Tunnel protocol/transport GRE/IP (VTI면: IPSEC/IP)
```
