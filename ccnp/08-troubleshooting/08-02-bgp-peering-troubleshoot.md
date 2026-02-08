# 08-02. BGP Peering 장애

---

### 면접관: "고객사에서 '인터넷이 갑자기 끊겼다'고 연락이 왔습니다. 이 고객사는 ISP 두 곳과 BGP로 연결되어 있습니다. 어디부터 확인하시겠어요?"

인터넷 전체가 끊겼다면 **양쪽 ISP와의 BGP Peering 상태**부터 확인합니다. BGP가 Established 상태가 아니면 경로를 주고받지 못하므로 인터넷 통신이 불가능합니다.

**1단계: BGP Neighbor 상태 확인**

```
show ip bgp summary
```

출력 예시 (장애 상태):
```
Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
203.0.113.1     4 64500       0       0        1    0    0 never    Idle
198.51.100.1    4 64501       0       0        1    0    0 00:05:23 Active
```

핵심 확인 포인트:
- **Idle**: BGP 프로세스가 연결 시도조차 하지 않는 상태
- **Active**: TCP 연결을 시도하고 있으나 실패하는 상태 (이름과 달리 비정상)
- **State/PfxRcd** 열에 숫자가 나오면 Established이며 해당 숫자만큼 경로를 수신 중

**2단계: 기본 연결성 확인**

```
ping 203.0.113.1 source GigabitEthernet0/0
! BGP Neighbor IP로 Ping 테스트 - Source 인터페이스 지정 필수

show ip route 203.0.113.1
! Neighbor IP까지의 라우팅 경로 존재 여부
```

**3단계: TCP 179 포트 연결 확인**

```
telnet 203.0.113.1 179 /source-interface GigabitEthernet0/0
! TCP 179 연결이 되는지 직접 테스트
```

---

### 면접관: "show ip bgp summary에서 한쪽은 Idle, 한쪽은 Active입니다. Active 상태의 Neighbor부터 보겠습니다. TCP 179 연결이 안 됩니다. 원인이 뭘까요?"

BGP는 TCP 포트 179를 사용하여 Peer를 형성합니다. TCP 연결이 안 된다면 크게 세 가지를 의심합니다.

#### 1. ACL/Firewall에서 TCP 179 차단

```
show ip access-lists
show running-config | include access-group
! 인터페이스에 적용된 ACL 확인

! 특히 ISP 방향 인터페이스의 inbound ACL에서
! TCP 179를 permit하는지 확인
```

ACL이 원인이면 다음과 같이 수정:
```
ip access-list extended TO-ISP
 permit tcp host 10.1.1.1 host 203.0.113.1 eq bgp
 permit tcp host 203.0.113.1 eq bgp host 10.1.1.1
 ! 양방향 모두 허용 (BGP는 어느 쪽이든 연결을 시작할 수 있음)
```

#### 2. 중간 경로에서의 차단

ISP와 고객 장비 사이에 방화벽이 있다면, 해당 방화벽에서 TCP 179를 허용하는지 확인합니다. 특히 **Stateful Firewall**은 BGP 세션이 끊어진 후 재연결 시 기존 세션 테이블과 충돌할 수 있습니다.

#### 3. 물리/L2 구간 장애

```
show interface GigabitEthernet0/0
! line protocol up/down 확인
! input/output 카운터 변화 확인

show arp | include 203.0.113.1
! ARP 해석이 되는지 확인 (직접 연결 시)
```

---

### 면접관: "BGP State Machine 전체를 설명해 주세요. 각 상태에서 뭘 하고, 어디서 실패할 수 있는지까지 포함해서요."

BGP Finite State Machine은 **6단계**로 구성됩니다.

| State | 동작 | 실패 원인 |
|-------|------|----------|
| **Idle** | 초기 상태. Start 이벤트를 기다림. TCP 연결 시작 전 | `neighbor shutdown` 설정, 라우팅 테이블에 Neighbor IP 없음, 이전 연결 실패 후 대기 |
| **Connect** | TCP 3-way Handshake 시도 중 | TCP 연결 실패 (ACL, 방화벽, 라우팅 문제) |
| **Active** | TCP 연결이 실패하여 재시도 중. Listen 상태로 대기 | **가장 흔한 문제 상태.** TCP 179가 차단되었거나, Source IP가 잘못됨 |
| **OpenSent** | TCP 연결 성공. OPEN 메시지를 보냄. 상대방의 OPEN을 기다림 | OPEN 메시지의 AS Number 불일치, BGP Version 불일치 |
| **OpenConfirm** | 상대방의 OPEN을 받고 Keepalive를 보냄. 상대방의 Keepalive를 기다림 | Keepalive가 오지 않음 (중간에서 Drop) |
| **Established** | 정상 상태. UPDATE, KEEPALIVE, NOTIFICATION 메시지 교환 | Hold Timer 만료, NOTIFICATION 수신 (에러) |

중요한 점:

- **Idle과 Active를 왔다 갔다 하면**: TCP 연결 자체가 안 되는 것 (네트워크/ACL 문제)
- **OpenSent에서 Idle로 돌아가면**: OPEN 메시지 파라미터 불일치 (AS 번호 등)
- **Established에서 갑자기 Idle로 가면**: Hold Timer 만료 또는 상대방이 NOTIFICATION을 보낸 것

```
! 디버그로 상태 전이 확인
debug ip bgp
debug ip bgp events

! 특정 Neighbor만 디버그
debug ip bgp 203.0.113.1
```

---

### 면접관: "TCP 179 문제는 해결했습니다. 그런데 이번엔 Idle 상태의 다른 Neighbor를 보니, update-source 설정이 빠져 있네요. 이게 왜 중요한가요?"

BGP Neighbor 설정 시, 특히 **iBGP**에서는 물리 인터페이스 IP가 아닌 **Loopback 인터페이스**를 Neighbor 주소로 사용하는 것이 일반적입니다. Loopback은 물리 인터페이스와 달리 Down되지 않으므로, 한 경로가 끊어져도 다른 경로로 BGP 세션을 유지할 수 있기 때문입니다.

문제는 BGP가 TCP 연결을 시작할 때 **기본적으로 아웃바운드 인터페이스의 IP를 Source로 사용**한다는 것입니다. Neighbor 설정이 상대방 Loopback IP로 되어 있는데, 내 TCP Source가 물리 인터페이스 IP이면, 상대방은 이 연결을 자신의 BGP Neighbor 설정과 매칭할 수 없어 거부합니다.

```
! 잘못된 설정 (Source가 물리 인터페이스 IP로 나감)
router bgp 65000
 neighbor 10.255.0.1 remote-as 65000

! 올바른 설정
router bgp 65000
 neighbor 10.255.0.1 remote-as 65000
 neighbor 10.255.0.1 update-source Loopback0
```

**eBGP에서의 ebgp-multihop:**

eBGP는 기본적으로 TTL=1로 패킷을 보냅니다. 직접 연결(directly connected)이 아닌 경우, 중간에 홉이 있으면 TTL이 0이 되어 패킷이 Drop됩니다.

```
! Loopback끼리 eBGP Peering 시
router bgp 65000
 neighbor 203.0.113.100 remote-as 64500
 neighbor 203.0.113.100 update-source Loopback0
 neighbor 203.0.113.100 ebgp-multihop 2
! TTL을 2 이상으로 설정

! 또는 직접 연결된 eBGP에서 Loopback 사용 시
neighbor 203.0.113.100 disable-connected-check
```

---

### 면접관: "BGP Peering은 잘 올라왔는데, iBGP Neighbor에서 경로를 받아도 라우팅 테이블에 안 올라옵니다. Next-hop이 도달 불가라고 나오네요. 이건 뭔가요?"

iBGP의 핵심 동작 중 하나인 **Next-Hop 전달 규칙** 때문입니다.

eBGP에서 받은 경로를 iBGP Peer에게 전달할 때, **기본적으로 Next-Hop을 변경하지 않습니다.** 즉 eBGP Neighbor의 IP(ISP 라우터 IP)가 그대로 Next-Hop으로 전달됩니다.

```
show ip bgp
! Next Hop 열 확인

show ip bgp 0.0.0.0/0
! 특정 경로의 상세 정보 - Next Hop 및 도달성 확인
```

출력 예시:
```
BGP routing table entry for 0.0.0.0/0
  203.0.113.1 from 10.255.0.1 (10.255.0.1)
    Origin IGP, localpref 100, valid, internal
    ! "best"가 없음 - Next Hop에 도달할 수 없어서 best로 선정 안 됨
```

내부 라우터가 203.0.113.1(ISP IP)까지의 경로를 모르기 때문에, 이 BGP 경로를 사용할 수 없는 것입니다.

**해결 방법 1: Next-Hop-Self 설정 (권장)**
```
router bgp 65000
 neighbor 10.255.0.2 next-hop-self
! iBGP Neighbor에게 경로 전달 시, Next-Hop을 자신의 IP로 변경
```

**해결 방법 2: ISP 연결 서브넷을 IGP에 포함**
```
router ospf 1
 network 203.0.113.0 0.0.0.3 area 0
! ISP 연결 서브넷을 OSPF에 포함시켜 내부 라우터도 도달 가능하게 함
! 보안상 권장하지 않음
```

**해결 방법 3: Static Route 재분배**
```
ip route 203.0.113.0 255.255.255.252 GigabitEthernet0/0
router ospf 1
 redistribute static subnets
```

실무에서는 **next-hop-self**가 가장 깔끔한 해결책입니다.

---

### 면접관: "iBGP에서 경로 전파 문제가 나왔으니 말인데, iBGP는 왜 Full Mesh가 필요하고, Route Reflector는 어떻게 이 문제를 해결하나요?"

iBGP에는 **Split Horizon Rule**이 있습니다: **iBGP Peer로부터 받은 경로는 다른 iBGP Peer에게 다시 전달하지 않습니다.** 이것은 라우팅 루프를 방지하기 위한 규칙입니다.

이 때문에 모든 iBGP 라우터가 서로 직접 Peering을 맺어야(Full Mesh) 경로를 빠짐없이 받을 수 있습니다. 라우터가 N대이면 연결 수는 **N(N-1)/2**로, 라우터가 많아지면 기하급수적으로 증가합니다.

#### Route Reflector의 동작

Route Reflector(RR)는 iBGP Split Horizon Rule의 예외를 만들어 줍니다.

**RR의 경로 전달 규칙:**

| 경로 출처 | 전달 대상 | 동작 |
|----------|----------|------|
| Client로부터 수신 | 모든 Client + Non-Client | **Reflect (전달)** |
| Non-Client로부터 수신 | 모든 Client만 | **Reflect (전달)** |
| eBGP로부터 수신 | 모든 Client + Non-Client | **Reflect (전달)** |

```
! Route Reflector 설정
router bgp 65000
 neighbor 10.255.0.2 remote-as 65000
 neighbor 10.255.0.2 route-reflector-client
 neighbor 10.255.0.3 remote-as 65000
 neighbor 10.255.0.3 route-reflector-client
 ! 10.255.0.2와 10.255.0.3은 이 RR의 Client
```

RR은 Reflect한 경로에 다음 속성을 추가하여 루프를 방지합니다:

- **ORIGINATOR_ID**: 경로를 최초로 광고한 라우터의 Router-ID. 자신의 Router-ID와 같으면 Drop.
- **CLUSTER_LIST**: 경로가 거쳐온 RR Cluster ID 목록. 자신의 Cluster ID가 이미 있으면 Drop.

```
show ip bgp 10.0.0.0/8
! ORIGINATOR_ID와 CLUSTER_LIST 확인
```

#### 설계 시 주의사항

- **RR은 최소 2대** 구성하여 이중화합니다.
- RR은 경로 선택에 영향을 줄 수 있으므로 (Optimal Path가 아닌 RR이 선택한 Best Path만 전달), **RR 배치를 신중하게** 해야 합니다.
- 대규모 네트워크에서는 **계층적 RR** 구조를 고려합니다.

---

### 면접관: "BGP 장애를 사전에 방지하고 빠르게 감지하려면 어떻게 해야 하나요?"

#### 모니터링 필수 항목

```
! BGP Neighbor 상태 변경 로깅
router bgp 65000
 bgp log-neighbor-changes
! 기본적으로 활성화되어 있지만 반드시 확인

! SNMP Trap
snmp-server enable traps bgp

! 수신 경로 수 모니터링 (ISP가 갑자기 경로를 철회하면 인터넷 끊김)
show ip bgp summary
! PfxRcd 값을 주기적으로 기록하고 임계값 모니터링
```

#### Prefix Limit 설정 (경로 폭주 방지)

```
router bgp 65000
 neighbor 203.0.113.1 maximum-prefix 500000 80 restart 5
 ! 50만 개 초과 시 Peering 종료, 80% 도달 시 경고, 5분 후 재시도
```

#### BFD 연동 (빠른 장애 감지)

```
router bgp 65000
 neighbor 203.0.113.1 fall-over bfd
! BGP Hold Timer(기본 180초) 대신 BFD(ms 단위)로 빠르게 장애 감지
```

#### 운영 체크리스트

| 항목 | 주기 | 확인 내용 |
|------|------|----------|
| BGP Neighbor 상태 | 실시간 | Established 유지 여부 |
| 수신 Prefix 수 | 매시간 | 급격한 증감 여부 |
| BGP 테이블 메모리 | 매일 | 메모리 부족으로 인한 경로 Drop 가능성 |
| NOTIFICATION 로그 | 실시간 | Hold Timer 만료, UPDATE 에러 등 |
| eBGP Peer 변경 이력 | 변경 시 | ISP 측 유지보수 스케줄 확인 |

#### 설계 차원의 방지책

1. **Loopback 기반 Peering** + **update-source** + **next-hop-self**를 iBGP 표준 템플릿으로 정합니다.
2. **Peer-Group 또는 Template** 사용으로 설정 일관성을 유지합니다.
3. **Graceful Restart** 활성화로 BGP 프로세스 재시작 시 경로 유지를 보장합니다.

```
router bgp 65000
 bgp graceful-restart
```

4. **Route-Map으로 수신/송신 경로 필터링**을 철저히 합니다 (특히 eBGP).

```
router bgp 65000
 neighbor 203.0.113.1 route-map FROM-ISP in
 neighbor 203.0.113.1 route-map TO-ISP out

route-map TO-ISP permit 10
 match ip address prefix-list MY-NETWORKS
! 자사 네트워크만 광고, 실수로 Transit이 되는 것 방지
```

---
