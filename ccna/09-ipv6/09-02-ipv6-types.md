# 09-02. IPv6 주소 유형 (Link-Local/Global Unicast)

> 개요
IPv6의 다양한 주소 유형(Link-Local, Global Unicast, Unique Local, Multicast, Anycast)과 각 주소의 목적, 범위, 자동 생성 메커니즘에 대한 면접 시나리오입니다.

---

### 면접관: "IPv6 네트워크를 설계하려고 합니다. IPv6에는 여러 종류의 주소 유형이 있다고 알고 있는데, 각 유형의 특징과 용도를 설명해 주시겠습니까?"

네, IPv6의 주요 주소 유형을 정리해서 설명드리겠습니다.

| 주소 유형 | 프리픽스 | 범위 | 용도 |
|-----------|---------|------|------|
| Global Unicast | 2000::/3 | 인터넷 전체 | 공인 주소, 인터넷 통신 |
| Link-Local | FE80::/10 | 로컬 링크만 | 같은 링크 내 통신, 라우팅 프로토콜 |
| Unique Local | FC00::/7 (FD00::/8 실제 사용) | 조직 내부 | IPv4 사설 주소와 유사 |
| Multicast | FF00::/8 | 범위에 따라 다름 | 그룹 통신 |
| Loopback | ::1/128 | 호스트 자신 | 자기 자신과의 통신 |
| Unspecified | ::/128 | - | 주소가 없음을 의미 |
| Anycast | 별도 프리픽스 없음 | 여러 노드에 동일 주소 | 가장 가까운 노드로 전달 |

IPv4와 가장 큰 차이점은 브로드캐스트 주소가 없다는 것입니다. IPv6에서는 브로드캐스트 대신 멀티캐스트를 사용합니다.

---

### 면접관: "Link-Local 주소에 대해 더 자세히 설명해 주세요. 왜 필요하고, 어떻게 자동으로 생성되나요?"

Link-Local 주소는 IPv6에서 매우 중요한 역할을 하는 주소입니다.

**Link-Local 주소 특징:**
- 프리픽스: FE80::/10 (실질적으로 FE80:0000:0000:0000::/64)
- 범위: 해당 링크(세그먼트) 내에서만 유효
- 라우터가 이 주소를 가진 패킷을 절대 다른 링크로 전달하지 않음
- IPv6를 활성화하면 인터페이스에 자동으로 생성됨

**Link-Local 주소가 필요한 이유:**

1. **라우팅 프로토콜의 Next-Hop**: OSPFv3, EIGRPv6 등에서 네이버 관계 수립과 next-hop 주소로 Link-Local을 사용합니다
2. **NDP(Neighbor Discovery Protocol)**: Router Solicitation, Router Advertisement, Neighbor Solicitation, Neighbor Advertisement 메시지가 Link-Local 주소를 사용합니다
3. **글로벌 주소 없이도 로컬 통신 가능**: 글로벌 주소가 설정되지 않아도 같은 링크 내 장비 간 통신이 가능합니다

**자동 생성 과정:**

```
1. 인터페이스에서 IPv6 활성화
   Router(config-if)# ipv6 enable
   또는
   Router(config-if)# ipv6 address 2001:DB8::1/64

2. Link-Local 주소 자동 생성 (EUI-64 기반)
   FE80:: + 인터페이스 ID (EUI-64)

   예: MAC 00:11:22:33:44:55
   Link-Local: FE80::211:22FF:FE33:4455
```

수동으로 지정할 수도 있습니다:

```
Router(config-if)# ipv6 address FE80::1 link-local
```

실무에서는 라우터의 Link-Local 주소를 기억하기 쉽게 수동 설정하는 경우가 많습니다. 예를 들어 모든 라우터 인터페이스에 FE80::1, FE80::2 등으로 설정하면 관리가 편해집니다.

---

### 면접관: "Global Unicast 주소는 IPv4의 공인 주소에 해당한다고 볼 수 있나요? 구조를 설명해 주세요."

맞습니다. Global Unicast Address(GUA)는 인터넷에서 라우팅 가능한 고유한 주소로, IPv4의 공인 주소와 유사한 역할을 합니다.

**Global Unicast 주소 구조 (2000::/3):**

```
|<-- 48비트 -->|<-- 16비트 -->|<-------- 64비트 -------->|
| Global       | Subnet      | Interface                 |
| Routing      | ID          | ID                        |
| Prefix       |             |                           |
|<---------- 네트워크 프리픽스 -------->|<--- 호스트 부분 --->|
|<---------------- /64 ----------------->|
```

- **Global Routing Prefix (48비트)**: ISP가 조직에 할당하는 부분. 인터넷에서 해당 조직의 네트워크를 식별합니다
- **Subnet ID (16비트)**: 조직 내부에서 서브넷을 구분하는 데 사용. 최대 65,536개의 서브넷 생성 가능
- **Interface ID (64비트)**: 개별 호스트를 식별

설정 예시:

```
Router(config)# ipv6 unicast-routing
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:DB8:AAAA:1::1/64
Router(config-if)# no shutdown
```

여기서 `2001:DB8:AAAA`가 Global Routing Prefix, `1`이 Subnet ID, `::1`이 인터페이스 ID에 해당합니다.

참고로 `2001:DB8::/32`는 문서화 목적으로 예약된 주소 범위이며, 실제 인터넷에서는 사용하지 않습니다. 시험이나 교육 자료에서 자주 사용됩니다.

---

### 면접관: "Unique Local 주소는 IPv4의 사설 주소(10.x, 172.16.x, 192.168.x)와 같은 개념인가요?"

유사하지만 완전히 동일하지는 않습니다.

**Unique Local Address (ULA) 특징:**
- 프리픽스: FC00::/7 (실제로는 FD00::/8을 사용)
- 조직 내부에서만 사용하며 인터넷으로 라우팅되지 않음
- IPv4 사설 주소(RFC 1918)와 유사한 역할

**Unique Local 주소 구조:**

```
| 7비트  | 1비트 | 40비트          | 16비트    | 64비트        |
| FC00:: | L비트  | Global ID      | Subnet ID | Interface ID |
```

- L비트 = 1이면 FD00::/8 (로컬 할당, 실무에서 사용)
- L비트 = 0이면 FC00::/8 (중앙 할당, 현재 미정의)

**IPv4 사설 주소와의 차이점:**

| 항목 | IPv4 사설 주소 | IPv6 Unique Local |
|------|---------------|-------------------|
| NAT 필요 여부 | 인터넷 접속 시 NAT 필수 | NAT 불필요 (GUA 병행 사용) |
| 주소 충돌 가능성 | 높음 (같은 대역 사용) | 낮음 (40비트 랜덤 Global ID) |
| 실무 활용도 | 매우 높음 | 상대적으로 낮음 |

IPv6에서는 하나의 인터페이스에 여러 IPv6 주소를 가질 수 있기 때문에, GUA와 ULA를 동시에 사용할 수 있습니다. 외부 통신에는 GUA를, 내부 통신에는 ULA를 사용하는 구성이 가능합니다.

하지만 실무에서는 IPv6의 주소 공간이 충분하기 때문에 ULA 없이 GUA만으로 운영하는 경우도 많습니다.

---

### 면접관: "멀티캐스트 주소에 대해 설명해 주세요. 특히 Solicited-Node Multicast는 어떤 역할을 하나요?"

IPv6 멀티캐스트 주소는 FF00::/8 프리픽스를 사용하며, 여러 중요한 그룹 주소가 있습니다.

**주요 멀티캐스트 주소:**

| 주소 | 용도 |
|------|------|
| FF02::1 | 링크 내 모든 노드 (All Nodes) |
| FF02::2 | 링크 내 모든 라우터 (All Routers) |
| FF02::5 | OSPF 라우터 |
| FF02::6 | OSPF DR 라우터 |
| FF02::9 | RIPng 라우터 |
| FF02::A | EIGRP 라우터 |
| FF02::1:FFxx:xxxx | Solicited-Node Multicast |

**Solicited-Node Multicast 주소:**

이 주소는 IPv6에서 매우 중요한 역할을 합니다. IPv4의 ARP 브로드캐스트를 대체하는 효율적인 메커니즘입니다.

생성 규칙:
```
FF02::1:FF + 유니캐스트 주소의 하위 24비트

예: 유니캐스트 주소가 2001:DB8::11:22FF:FE33:4455인 경우
    Solicited-Node Multicast = FF02::1:FF33:4455
```

**동작 과정 (IPv4 ARP 대체):**

1. 호스트 A가 호스트 B의 MAC 주소를 알고 싶을 때
2. IPv4에서는 ARP 브로드캐스트를 전체 네트워크에 전송
3. IPv6에서는 호스트 B의 Solicited-Node Multicast 주소로 Neighbor Solicitation 전송
4. 해당 멀티캐스트 그룹에 가입한 호스트(보통 호스트 B만)만 처리

이 방식의 장점은 불필요한 트래픽을 크게 줄인다는 것입니다. 브로드캐스트는 모든 호스트가 처리해야 하지만, Solicited-Node Multicast는 해당 주소를 가진 호스트만 처리하면 됩니다.

---

### 면접관: "Anycast 주소는 어떤 상황에서 사용하나요? 일반 Unicast와 어떻게 다릅니까?"

Anycast는 동일한 주소를 여러 노드에 할당하고, 가장 가까운(라우팅 메트릭이 가장 낮은) 노드로 패킷을 전달하는 방식입니다.

**Anycast의 특징:**
- 별도의 주소 형식이 없습니다. 일반 유니캐스트 주소와 동일한 형식
- 여러 장비에 같은 주소를 설정하면 Anycast로 동작
- 라우터는 라우팅 테이블 기반으로 가장 가까운 목적지를 선택

**사용 시나리오:**
1. **DNS 서비스**: 여러 DNS 서버에 같은 Anycast 주소를 부여하여 사용자가 가장 가까운 DNS 서버에 접근
2. **CDN**: 콘텐츠 전달 네트워크에서 지리적으로 분산된 서버들에 활용
3. **기본 게이트웨이 이중화**: 여러 라우터에 같은 주소를 설정

라우터 인터페이스에 Anycast를 설정하는 방법:

```
Router(config-if)# ipv6 address 2001:DB8:1:1::1/64 anycast
```

CCNA 수준에서는 Anycast의 개념과 용도를 이해하는 것이 중요하며, 복잡한 설계까지는 요구되지 않습니다.

---

### 면접관: "EUI-64 방식을 사용하여 인터페이스에 IPv6 주소를 설정하는 방법을 보여주시고, 보안상 주의할 점을 알려주세요."

EUI-64를 이용한 설정과 보안 고려사항을 설명드리겠습니다.

**EUI-64 설정:**

```
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 address 2001:DB8:AAAA:1::/64 eui-64
```

이렇게 설정하면 프리픽스(2001:DB8:AAAA:1::/64)는 관리자가 지정하고, 인터페이스 ID(하위 64비트)는 MAC 주소 기반으로 자동 생성됩니다.

**EUI-64 변환 과정 재확인:**

```
MAC 주소: 00:1A:2B:3C:4D:5E

1단계: MAC 주소를 반으로 나눔
   00:1A:2B | 3C:4D:5E

2단계: 가운데에 FF:FE 삽입
   00:1A:2B:FF:FE:3C:4D:5E

3단계: 7번째 비트(U/L 비트) 반전
   00 = 0000 0000 → 0000 0010 = 02
   최종: 02:1A:2B:FF:FE:3C:4D:5E

최종 IPv6 주소: 2001:DB8:AAAA:1:021A:2BFF:FE3C:4D5E
```

**보안 주의사항:**

1. **장치 추적 가능성**: EUI-64로 생성된 인터페이스 ID에는 MAC 주소가 포함되어 있어, 네트워크를 이동해도 장치를 추적할 수 있습니다
2. **제조사 식별**: OUI(상위 24비트)로 장비 제조사를 파악할 수 있습니다
3. **프라이버시 확장**: RFC 4941에서 정의한 임시 주소(Privacy Extensions)를 사용하면 랜덤 인터페이스 ID를 생성하여 추적을 어렵게 할 수 있습니다

서버나 네트워크 장비처럼 고정 주소가 필요한 경우에는 EUI-64나 수동 설정을 사용하고, 클라이언트 장비에서는 프라이버시 확장을 활성화하는 것이 일반적인 모범 사례입니다.

확인 명령어:

```
Router# show ipv6 interface GigabitEthernet0/0
Router# show ipv6 interface brief
```

이렇게 하면 인터페이스에 할당된 Link-Local 주소와 Global Unicast 주소를 모두 확인할 수 있습니다.
