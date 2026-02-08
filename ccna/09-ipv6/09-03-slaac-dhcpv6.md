# 09-03. SLAAC와 DHCPv6

> 개요
IPv6에서 호스트가 자동으로 주소를 획득하는 SLAAC, Stateless DHCPv6, Stateful DHCPv6의 동작 원리와 설정, 그리고 NDP와 DAD 메커니즘에 대한 면접 시나리오입니다.

---

### 면접관: "IPv6 환경에서 호스트가 IP 주소를 자동으로 받는 방법이 여러 가지가 있다고 들었습니다. SLAAC에 대해 먼저 설명해 주시겠습니까?"

네, SLAAC(Stateless Address Autoconfiguration)는 IPv6에서 호스트가 DHCP 서버 없이도 스스로 IPv6 주소를 구성할 수 있는 메커니즘입니다.

**SLAAC 동작 과정:**

```
1. 호스트가 네트워크에 연결됨
   → Link-Local 주소 자동 생성 (FE80::...)

2. Router Solicitation (RS) 전송
   출발지: 호스트의 Link-Local 주소
   목적지: FF02::2 (All Routers 멀티캐스트)
   "이 링크에 라우터가 있습니까?"

3. 라우터가 Router Advertisement (RA) 응답
   출발지: 라우터의 Link-Local 주소
   목적지: FF02::1 (All Nodes 멀티캐스트) 또는 요청한 호스트
   내용: 프리픽스 정보, 프리픽스 길이, 기본 게이트웨이 등

4. 호스트가 RA에서 받은 프리픽스 + 자체 생성한 인터페이스 ID로 주소 구성
   프리픽스: 2001:DB8:AAAA:1::/64 (RA에서 수신)
   인터페이스 ID: EUI-64 또는 랜덤 생성
   최종 주소: 2001:DB8:AAAA:1:xxxx:xxxx:xxxx:xxxx/64

5. DAD(Duplicate Address Detection) 수행
   → 주소 충돌이 없으면 주소 사용 시작
```

SLAAC의 장점은 DHCP 서버 인프라가 필요 없다는 것입니다. 라우터의 RA 메시지만으로 호스트가 완전한 IPv6 통신을 할 수 있습니다.

라우터 측 기본 설정:

```
Router(config)# ipv6 unicast-routing
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:DB8:AAAA:1::1/64
Router(config-if)# no shutdown
```

`ipv6 unicast-routing`이 활성화되면 라우터는 자동으로 RA 메시지를 주기적으로 전송합니다.

---

### 면접관: "SLAAC만으로는 DNS 서버 정보를 받을 수 없다고 알고 있습니다. 이 문제는 어떻게 해결하나요?"

맞습니다. 기본 SLAAC만으로는 DNS 서버 주소나 도메인 이름 같은 추가 정보를 제공할 수 없습니다. 이를 해결하는 방법이 바로 Stateless DHCPv6입니다.

**세 가지 자동 주소 할당 방식 비교:**

| 항목 | SLAAC | Stateless DHCPv6 | Stateful DHCPv6 |
|------|-------|-------------------|-----------------|
| IP 주소 할당 | SLAAC (호스트 자체 생성) | SLAAC (호스트 자체 생성) | DHCPv6 서버 |
| 기본 게이트웨이 | RA에서 제공 | RA에서 제공 | RA에서 제공 |
| DNS 서버 | 제공 안 함 | DHCPv6 서버 | DHCPv6 서버 |
| 도메인 이름 | 제공 안 함 | DHCPv6 서버 | DHCPv6 서버 |
| 주소 추적 | 어려움 | 어려움 | 가능 (서버에 기록) |
| RA 플래그 | M=0, O=0 | M=0, O=1 | M=1 |

**Stateless DHCPv6:**
- IP 주소는 여전히 SLAAC로 자체 생성
- DNS, 도메인 이름 등 추가 정보만 DHCPv6 서버에서 수신
- RA의 O 플래그(Other Configuration)가 1로 설정됨

**Stateful DHCPv6:**
- IP 주소를 포함한 모든 정보를 DHCPv6 서버가 관리
- IPv4의 DHCP와 가장 유사한 방식
- RA의 M 플래그(Managed Address Configuration)가 1로 설정됨
- 기본 게이트웨이는 여전히 RA에서 제공 (DHCPv6에서는 기본 게이트웨이를 제공하지 않음)

중요한 점은 Stateful DHCPv6에서도 기본 게이트웨이는 RA를 통해 전달된다는 것입니다. 이것이 IPv4 DHCP와의 큰 차이점입니다.

---

### 면접관: "RA 메시지의 M 플래그와 O 플래그를 설정하는 방법을 보여주시겠습니까?"

네, 라우터에서 RA 플래그를 설정하는 방법을 각 시나리오별로 보여드리겠습니다.

**시나리오 1: SLAAC Only (M=0, O=0) - 기본 설정**

```
Router(config)# ipv6 unicast-routing
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:DB8:1::1/64
Router(config-if)# no shutdown
! 기본적으로 M=0, O=0이므로 추가 설정 불필요
```

**시나리오 2: SLAAC + Stateless DHCPv6 (M=0, O=1)**

```
! 라우터 설정 (RA에서 O 플래그 활성화)
Router(config-if)# ipv6 nd other-config-flag

! DHCPv6 서버 설정 (Stateless 모드)
Router(config)# ipv6 dhcp pool STATELESS-POOL
Router(config-dhcpv6)# dns-server 2001:DB8:1::FFFF
Router(config-dhcpv6)# domain-name example.com
Router(config-dhcpv6)# exit
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 dhcp server STATELESS-POOL
```

**시나리오 3: Stateful DHCPv6 (M=1)**

```
! 라우터 설정 (RA에서 M 플래그 활성화)
Router(config-if)# ipv6 nd managed-config-flag

! DHCPv6 서버 설정 (Stateful 모드)
Router(config)# ipv6 dhcp pool STATEFUL-POOL
Router(config-dhcpv6)# address prefix 2001:DB8:1:1::/64
Router(config-dhcpv6)# dns-server 2001:DB8:1::FFFF
Router(config-dhcpv6)# domain-name example.com
Router(config-dhcpv6)# exit
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 dhcp server STATEFUL-POOL
```

확인 명령어:

```
Router# show ipv6 dhcp pool
Router# show ipv6 dhcp binding
Router# show ipv6 interface GigabitEthernet0/0 | include ND
```

---

### 면접관: "NDP(Neighbor Discovery Protocol)에 대해 설명해 주세요. IPv4의 ARP와 어떻게 다른가요?"

NDP는 IPv6에서 ARP를 대체하는 프로토콜이며, 그 이상의 기능도 수행합니다. ICMPv6 위에서 동작합니다.

**NDP의 5가지 주요 메시지:**

| 메시지 | 유형 | ICMPv6 타입 | 용도 |
|--------|------|------------|------|
| Router Solicitation (RS) | 멀티캐스트 | 133 | 라우터 발견 요청 |
| Router Advertisement (RA) | 멀티캐스트 | 134 | 프리픽스, 게이트웨이 정보 제공 |
| Neighbor Solicitation (NS) | 멀티캐스트 | 135 | MAC 주소 확인 (ARP 대체), DAD |
| Neighbor Advertisement (NA) | 유니캐스트 | 136 | NS에 대한 응답 |
| Redirect | 유니캐스트 | 137 | 더 좋은 경로 알림 |

**IPv4 ARP vs IPv6 NDP 비교:**

| 항목 | IPv4 ARP | IPv6 NDP |
|------|----------|----------|
| 프로토콜 | 독립 프로토콜 (L2/L3) | ICMPv6 기반 |
| 주소 해석 | ARP Request (브로드캐스트) | NS (멀티캐스트) |
| 트래픽 범위 | 전체 브로드캐스트 도메인 | Solicited-Node 멀티캐스트 |
| 라우터 발견 | 별도 프로토콜 필요 | NDP에 내장 (RS/RA) |
| 주소 자동 설정 | DHCP 필요 | SLAAC 내장 |
| 중복 주소 탐지 | Gratuitous ARP (선택) | DAD (필수) |

NDP가 멀티캐스트를 사용하기 때문에 ARP 브로드캐스트보다 네트워크 효율이 좋습니다. 특히 대규모 네트워크에서 브로드캐스트 트래픽이 줄어드는 이점이 있습니다.

```
Router# show ipv6 neighbors
IPv6 Address            Age  Link-layer Addr  State  Interface
2001:DB8:1::10          5    0011.2233.4455   REACH  Gi0/0
FE80::211:22FF:FE33:4455 3   0011.2233.4455   STALE  Gi0/0
```

---

### 면접관: "DAD(Duplicate Address Detection)는 어떻게 동작하나요? 왜 필요한가요?"

DAD는 호스트가 새로운 IPv6 주소를 사용하기 전에 해당 주소가 이미 네트워크에서 사용되고 있지 않은지 확인하는 과정입니다.

**DAD 동작 과정:**

```
1. 호스트가 새 IPv6 주소(예: 2001:DB8:1::10)를 구성하려 함

2. Neighbor Solicitation (NS) 전송
   출발지: :: (Unspecified, 아직 주소를 사용할 수 없으므로)
   목적지: Solicited-Node Multicast (FF02::1:FF00:0010)
   대상 주소: 2001:DB8:1::10
   "이 주소를 사용하는 장비가 있습니까?"

3-A. 응답 없음 → 주소가 고유함 → 주소 사용 시작
3-B. Neighbor Advertisement 응답 수신 → 주소 충돌 → 주소 사용 불가
```

**DAD가 필요한 이유:**
- SLAAC에서 EUI-64를 사용하면 MAC 주소 기반이라 충돌 가능성이 낮지만, 수동 설정이나 랜덤 생성 시 충돌 가능
- 가상 환경(VM)에서 MAC 주소가 중복될 수 있음
- 주소 충돌은 통신 장애를 유발하므로 사전에 방지해야 함

**DAD 관련 설정:**

```
! DAD 시도 횟수 설정 (기본값: 1)
Router(config-if)# ipv6 nd dad attempts 3

! DAD 비활성화 (권장하지 않음)
Router(config-if)# ipv6 nd dad attempts 0
```

DAD는 Link-Local 주소 생성 시에도 수행됩니다. 따라서 인터페이스가 활성화될 때 약간의 지연이 발생할 수 있습니다.

---

### 면접관: "클라이언트 측에서 DHCPv6를 사용하도록 설정하는 방법도 설명해 주시겠습니까?"

클라이언트(라우터를 클라이언트로 사용하는 경우)에서 DHCPv6를 설정하는 방법을 보여드리겠습니다.

**Stateless DHCPv6 클라이언트 설정:**

SLAAC로 주소를 받되, DNS 등 추가 정보를 DHCPv6에서 받는 경우:

```
Client-Router(config)# interface GigabitEthernet0/0
Client-Router(config-if)# ipv6 address autoconfig
Client-Router(config-if)# ipv6 enable
Client-Router(config-if)# no shutdown
```

`ipv6 address autoconfig` 명령어는 SLAAC를 통해 주소를 자동 구성하며, RA의 O 플래그가 1이면 자동으로 DHCPv6에서 추가 정보를 요청합니다.

**Stateful DHCPv6 클라이언트 설정:**

```
Client-Router(config)# interface GigabitEthernet0/0
Client-Router(config-if)# ipv6 address dhcp
Client-Router(config-if)# ipv6 enable
Client-Router(config-if)# no shutdown
```

`ipv6 address dhcp`는 DHCPv6 서버로부터 주소를 받겠다는 의미입니다.

**DHCPv6 메시지 교환 과정:**

```
클라이언트                          서버
    |                                |
    |--- Solicit (멀티캐스트) -------->|
    |                                |
    |<-------- Advertise ------------|
    |                                |
    |--- Request ------------------>|
    |                                |
    |<-------- Reply ---------------|
```

IPv4 DHCP의 DORA(Discover, Offer, Request, Ack)와 유사하지만 이름이 다릅니다. Solicit-Advertise-Request-Reply로 진행됩니다.

확인 명령어:

```
! 클라이언트 측
Client-Router# show ipv6 dhcp interface GigabitEthernet0/0

! 서버 측
Server-Router# show ipv6 dhcp pool
Server-Router# show ipv6 dhcp binding
```

---

### 면접관: "실무에서 SLAAC, Stateless DHCPv6, Stateful DHCPv6 중 어떤 것을 선택해야 할지 기준이 있나요?"

각 방식의 선택 기준을 정리하겠습니다.

**SLAAC Only 적합한 경우:**
- 소규모 네트워크 또는 가정용 네트워크
- DNS 정보가 필요 없거나 수동으로 설정 가능한 환경
- 최대한 단순한 인프라를 원할 때
- DHCP 서버 구축이 어려운 환경

**Stateless DHCPv6 적합한 경우:**
- DNS 서버 정보를 중앙에서 관리하고 싶지만, IP 주소 할당은 관리할 필요 없을 때
- SLAAC의 편의성을 유지하면서 추가 정보가 필요할 때
- 중간 규모 네트워크

**Stateful DHCPv6 적합한 경우:**
- IP 주소 할당을 중앙에서 관리하고 추적해야 할 때
- 보안 정책상 어떤 장비에 어떤 주소가 할당되었는지 기록이 필요할 때
- 대규모 기업 네트워크
- 특정 주소 범위를 특정 장비에 할당해야 할 때

실무에서 가장 많이 사용되는 조합은 Stateless DHCPv6입니다. IP 주소는 SLAAC로 자동 생성하되, DNS와 도메인 정보는 DHCPv6 서버에서 중앙 관리하는 방식이 관리 부담과 편의성의 균형이 좋기 때문입니다.

다만 보안이 중요한 기업 환경에서는 Stateful DHCPv6를 사용하여 주소 할당 이력을 관리하는 것이 권장됩니다.
