# 08-02. GRE Tunnel 기본

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 시나리오

---

### 면접관: "본사와 지사 간에 인터넷을 통해 내부 라우팅 프로토콜(OSPF)을 동작시키고 싶다는 요구사항이 있습니다. 어떻게 구현하시겠습니까?"

인터넷 구간에서 OSPF 같은 IGP(Interior Gateway Protocol)를 직접 동작시키는 것은 불가능합니다. OSPF는 멀티캐스트(224.0.0.5, 224.0.0.6)를 사용하는데, ISP는 고객의 멀티캐스트 트래픽을 전달하지 않기 때문입니다.

이 문제를 해결하기 위해 **GRE(Generic Routing Encapsulation) Tunnel**을 사용합니다. GRE 터널은 두 라우터 사이에 가상의 Point-to-Point 링크를 만들어, 그 위에서 어떤 프로토콜이든 캡슐화하여 전달할 수 있습니다.

GRE 터널 위에 OSPF neighbor를 맺으면, 인터넷을 거치더라도 내부 라우팅 프로토콜이 정상적으로 동작합니다.

#### GRE 터널의 핵심 특징

| 항목 | 설명 |
|------|------|
| **프로토콜 번호** | IP Protocol 47 |
| **캡슐화** | 원본 패킷을 GRE 헤더 + 새 IP 헤더로 감쌈 |
| **오버헤드** | 24바이트 (새 IP 헤더 20바이트 + GRE 헤더 4바이트) |
| **암호화** | 없음 (GRE 자체는 암호화 기능 없음) |
| **멀티캐스트 지원** | 지원 (라우팅 프로토콜 동작 가능) |
| **브로드캐스트 지원** | 지원 |

---

### 면접관: "그러면 GRE 터널을 실제로 설정하는 방법을 보여 주시겠습니까? 본사 라우터와 지사 라우터 양쪽 설정을 모두 보여 주세요."

네, 다음과 같은 토폴로지를 가정하겠습니다.

- 본사(HQ): 공인 IP 203.0.113.1, 내부 네트워크 192.168.1.0/24
- 지사(Branch): 공인 IP 198.51.100.1, 내부 네트워크 192.168.2.0/24
- 터널 네트워크: 10.0.0.0/30

#### 본사(HQ) 라우터 설정

```
HQ(config)# interface tunnel 0
HQ(config-if)# ip address 10.0.0.1 255.255.255.252
HQ(config-if)# tunnel source 203.0.113.1
HQ(config-if)# tunnel destination 198.51.100.1
HQ(config-if)# tunnel mode gre ip
HQ(config-if)# no shutdown
HQ(config-if)# exit
```

#### 지사(Branch) 라우터 설정

```
Branch(config)# interface tunnel 0
Branch(config-if)# ip address 10.0.0.2 255.255.255.252
Branch(config-if)# tunnel source 198.51.100.1
Branch(config-if)# tunnel destination 203.0.113.1
Branch(config-if)# tunnel mode gre ip
Branch(config-if)# no shutdown
Branch(config-if)# exit
```

설정의 핵심 포인트는 다음과 같습니다.

1. **tunnel source**: 자신의 물리적 공인 IP 또는 인터페이스 이름을 지정합니다.
2. **tunnel destination**: 상대방의 물리적 공인 IP를 지정합니다.
3. **ip address**: 터널 인터페이스에 부여하는 논리적 주소로, 양쪽이 같은 서브넷이어야 합니다.
4. 양쪽의 source와 destination이 **교차**되어야 합니다.

---

### 면접관: "GRE 터널 위에서 OSPF를 동작시키려면 추가로 어떤 설정이 필요합니까?"

GRE 터널 위에 OSPF를 동작시키려면, 터널 인터페이스의 네트워크를 OSPF에 포함시키면 됩니다.

#### 본사(HQ) OSPF 설정

```
HQ(config)# router ospf 1
HQ(config-router)# network 10.0.0.0 0.0.0.3 area 0
HQ(config-router)# network 192.168.1.0 0.0.0.255 area 0
```

#### 지사(Branch) OSPF 설정

```
Branch(config)# router ospf 1
Branch(config-router)# network 10.0.0.0 0.0.0.3 area 0
Branch(config-router)# network 192.168.2.0 0.0.0.255 area 0
```

이렇게 설정하면 GRE 터널을 통해 OSPF Hello 패킷(멀티캐스트 224.0.0.5)이 교환되고, neighbor가 형성됩니다. 이후 양쪽의 내부 네트워크 경로(192.168.1.0/24, 192.168.2.0/24)가 OSPF를 통해 자동으로 교환됩니다.

주의할 점은 **터널 인터페이스의 OSPF 네트워크 타입**이 기본적으로 Point-to-Point로 동작한다는 것입니다. 따라서 DR/BDR 선출 과정이 없고, neighbor 형성이 빠릅니다.

---

### 면접관: "GRE 오버헤드가 24바이트라고 하셨는데, 이것이 실제 운영에서 어떤 문제를 일으킬 수 있습니까?"

GRE 캡슐화는 원본 패킷에 24바이트를 추가합니다.

```
[새 IP 헤더: 20바이트] [GRE 헤더: 4바이트] [원본 IP 패킷]
```

일반 이더넷의 MTU가 1500바이트이므로, GRE 터널을 통과하는 패킷의 실제 가용 크기는 **1500 - 24 = 1476바이트**가 됩니다.

이로 인해 발생하는 문제는 **MTU/단편화(Fragmentation) 문제**입니다.

1. 원본 패킷이 1500바이트인 경우, GRE 헤더를 붙이면 1524바이트가 되어 물리 인터페이스의 MTU를 초과합니다.
2. DF(Don't Fragment) 비트가 설정되어 있으면 패킷이 드롭되고 ICMP "Fragmentation Needed" 메시지가 반환됩니다.
3. 이 문제로 인해 ping은 되지만 대용량 파일 전송이 안 되는 현상이 발생할 수 있습니다.

#### 해결 방법

```
! 방법 1: 터널 인터페이스의 MTU를 낮춤
HQ(config-if)# ip mtu 1476

! 방법 2: TCP MSS 조정 (TCP 트래픽에 대해)
HQ(config-if)# ip tcp adjust-mss 1436

! 방법 3: 터널에서 Path MTU Discovery 활용
HQ(config-if)# tunnel path-mtu-discovery
```

실무에서는 **ip tcp adjust-mss 1436**을 가장 많이 사용합니다. 1436인 이유는 1476(GRE MTU) - 40(TCP/IP 헤더) = 1436이기 때문입니다.

---

### 면접관: "GRE 터널이 정상적으로 동작하는지 어떻게 확인합니까?"

다음 명령어들로 GRE 터널 상태를 확인할 수 있습니다.

#### 터널 인터페이스 상태 확인

```
HQ# show interface tunnel 0
Tunnel0 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 10.0.0.1/30
  MTU 17916 bytes, BW 100 Kbit/sec, DLY 50000 usec
  Tunnel source 203.0.113.1, destination 198.51.100.1
  Tunnel protocol/transport GRE/IP
  Tunnel TTL 255
  Keepalive not set
```

핵심 확인 사항은 다음과 같습니다.

| 확인 항목 | 정상 상태 | 비정상 시 원인 |
|-----------|-----------|----------------|
| **Tunnel0 is up** | up/up | tunnel destination까지 경로 없음 |
| **Tunnel source** | 올바른 공인 IP | source 설정 오류 |
| **Tunnel destination** | 상대방 공인 IP | destination 설정 오류 |
| **Tunnel protocol** | GRE/IP | tunnel mode 설정 오류 |

#### 추가 확인 명령어

```
! 터널을 통한 연결 테스트
HQ# ping 10.0.0.2 source 10.0.0.1

! 라우팅 테이블에서 터널 경로 확인
HQ# show ip route
  O    192.168.2.0/24 [110/1001] via 10.0.0.2, Tunnel0

! OSPF neighbor 확인 (터널 위에서 OSPF 사용 시)
HQ# show ip ospf neighbor
Neighbor ID  Pri  State     Dead Time  Address     Interface
198.51.100.1  0   FULL/  -  00:00:38   10.0.0.2    Tunnel0
```

터널이 up/down인 경우 가장 흔한 원인은 **tunnel destination까지의 라우팅 경로가 없는 것**입니다. 물리 인터페이스를 통한 공인 IP 간 통신이 먼저 가능해야 합니다.

---

### 면접관: "GRE는 암호화가 없다고 하셨는데, 보안이 필요하면 어떻게 합니까? GRE와 IPSec의 차이를 비교해 주세요."

GRE 자체에는 암호화 기능이 없으므로, 보안이 필요하면 **GRE over IPSec** 조합을 사용합니다.

| 비교 항목 | GRE | IPSec | GRE over IPSec |
|-----------|-----|-------|----------------|
| **캡슐화** | 있음 | 있음 | 이중 캡슐화 |
| **암호화** | 없음 | AES, 3DES 등 | IPSec이 담당 |
| **무결성** | 없음 | HMAC-SHA 등 | IPSec이 담당 |
| **멀티캐스트** | 지원 | 미지원 | GRE가 담당 |
| **라우팅 프로토콜** | 동작 가능 | 동작 불가 (터널 모드) | GRE가 담당 |
| **오버헤드** | 24바이트 | 50~60바이트 | 60~80바이트 |

GRE over IPSec을 사용하면 GRE가 멀티캐스트와 라우팅 프로토콜 전달을 담당하고, IPSec이 암호화와 무결성을 담당하여 **각각의 장점을 결합**할 수 있습니다.

순수 IPSec 터널만으로는 멀티캐스트 트래픽을 전달할 수 없기 때문에, OSPF나 EIGRP 같은 라우팅 프로토콜을 터널 위에서 동작시키려면 반드시 GRE를 함께 사용해야 합니다.

---

### 면접관: "GRE 터널에서 라우팅 루프가 발생할 수 있다고 들었습니다. 어떤 상황에서 발생하고 어떻게 방지합니까?"

GRE 터널에서 라우팅 루프는 **재귀 라우팅(Recursive Routing)** 문제로 발생합니다.

예를 들어, 터널 destination인 198.51.100.1까지의 경로가 터널 인터페이스 자체를 통해 학습되는 경우입니다.

```
! 문제 상황: 터널 destination 경로가 터널을 가리킴
O    198.51.100.0/24 [110/1001] via 10.0.0.2, Tunnel0
```

이렇게 되면 터널 패킷을 보내려면 터널을 타야 하고, 터널을 타려면 터널 패킷을 보내야 하는 무한 루프에 빠집니다. 결과적으로 터널이 flapping(up/down 반복)합니다.

#### 방지 방법

1. **정적 경로로 터널 destination 고정**: 터널 destination까지의 경로를 물리 인터페이스를 통한 정적 경로로 지정합니다.

```
HQ(config)# ip route 198.51.100.1 255.255.255.255 203.0.113.2
```

2. **라우팅 프로토콜에서 터널 destination 네트워크 제외**: OSPF에서 터널 destination이 포함된 네트워크를 광고하지 않도록 설정합니다.

3. **터널 인터페이스의 대역폭 조정**: 터널 인터페이스의 metric을 높여 물리 경로보다 우선순위를 낮춥니다.

```
HQ(config-if)# interface tunnel 0
HQ(config-if)# bandwidth 100
```

실무에서는 **방법 1(정적 경로)**이 가장 확실하고 많이 사용됩니다. 터널 destination에 대한 경로는 항상 물리 인터페이스를 통하도록 보장하는 것이 핵심입니다.
