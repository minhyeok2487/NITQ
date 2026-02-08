# 02-05. Floating Static Route

> CCNA 200-301 라우팅 기초 면접 시나리오
> 주제: Floating Static Route 개념, AD 변경, 백업 경로 구성, 페일오버, ISP 백업 시나리오

---

### 면접관: "Floating Static Route가 무엇인지 설명해주세요. 일반 Static Route와 어떤 차이가 있습니까?"

Floating Static Route는 Administrative Distance(AD) 값을 기본값(1)보다 높게 설정한 정적 라우트입니다. 동적 라우팅 프로토콜의 경로가 정상일 때는 라우팅 테이블에 나타나지 않다가, 동적 경로가 사라졌을 때 "떠오르듯(floating)" 라우팅 테이블에 설치되어 백업 경로 역할을 합니다.

일반 Static Route와의 차이를 비교하겠습니다:

| 항목 | 일반 Static Route | Floating Static Route |
|------|-----------------|---------------------|
| AD 값 | 1 (기본값) | 수동으로 높게 설정 (예: 200) |
| 용도 | 주 경로(Primary) | 백업 경로(Backup) |
| 라우팅 테이블 | 항상 설치됨 (더 구체적 경로 없는 한) | 주 경로가 사라졌을 때만 설치됨 |
| 설정 예시 | `ip route 10.0.0.0 255.0.0.0 1.1.1.1` | `ip route 10.0.0.0 255.0.0.0 2.2.2.2 200` |

핵심 원리는 AD 값 비교입니다. 동일한 목적지에 대해 AD가 낮은 경로가 우선이므로, OSPF(AD=110)가 정상일 때는 Floating Static(AD=200)보다 우선합니다. OSPF 경로가 사라지면 Floating Static이 유일한 경로가 되어 자동으로 라우팅 테이블에 설치됩니다.

---

### 면접관: "Floating Static Route의 설정 문법을 보여주고, AD 값을 어떻게 지정하는지 설명해주세요. AD 값은 보통 얼마로 설정합니까?"

설정 문법은 일반 정적 라우트 끝에 AD 값을 추가하는 것입니다:

```
ip route [목적지] [마스크] [next-hop 또는 interface] [AD 값]
```

실제 설정 예시:

```
! 일반 Static Route (AD=1, 기본값)
R1(config)# ip route 192.168.30.0 255.255.255.0 10.0.12.2

! Floating Static Route (AD=200으로 설정)
R1(config)# ip route 192.168.30.0 255.255.255.0 10.0.13.2 200
```

AD 값 설정 기준은 백업하려는 주 경로의 AD보다 높게 설정하는 것입니다:

| 주 경로 | 주 경로 AD | Floating Static AD 권장값 |
|--------|-----------|------------------------|
| OSPF | 110 | 111 ~ 254 |
| EIGRP | 90 | 91 ~ 254 |
| RIP | 120 | 121 ~ 254 |
| Static | 1 | 2 ~ 254 |

실무에서는 200 또는 250처럼 충분히 높은 값을 사용하는 경우가 많습니다. 다만 255는 사용하면 안 됩니다. AD 255는 "도달 불가(unreachable)"를 의미하여 라우팅 테이블에 절대 설치되지 않습니다.

OSPF 백업 용도로 가장 흔하게 사용하는 설정:

```
! 주 경로: OSPF (AD=110)
R1(config)# router ospf 1
R1(config-router)# network 192.168.30.0 0.0.0.255 area 0

! 백업 경로: Floating Static (AD=200)
R1(config)# ip route 192.168.30.0 255.255.255.0 10.0.99.1 200
```

---

### 면접관: "구체적인 시나리오를 제시하겠습니다. 회사 본사와 지사를 연결하는 주 회선이 전용선(OSPF)이고, 백업으로 인터넷 VPN을 사용합니다. 이 환경에서 Floating Static Route를 어떻게 구성합니까?"

네트워크 구성을 먼저 정리하겠습니다:

```
        [전용선 - OSPF]
본사 R1 ==================== 지사 R2
 |   10.0.12.0/30                  |
 |                                 |
 |      [인터넷 VPN - 백업]        |
 +---- ISP1 ----Internet---- ISP2 -+
       203.0.113.0/30     198.51.100.0/30

본사 LAN: 192.168.10.0/24
지사 LAN: 192.168.20.0/24
VPN 터널: 172.16.0.0/30
```

**본사 R1 설정:**

```
! === 주 경로: 전용선을 통한 OSPF ===
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 192.168.10.0 0.0.0.255 area 0
R1(config-router)# network 10.0.12.0 0.0.0.3 area 0
R1(config-router)# exit

! === 백업 경로: VPN 터널을 통한 Floating Static ===
! OSPF AD(110)보다 높은 200으로 설정
R1(config)# ip route 192.168.20.0 255.255.255.0 172.16.0.2 200
```

**지사 R2 설정:**

```
! === 주 경로: 전용선을 통한 OSPF ===
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 192.168.20.0 0.0.0.255 area 0
R2(config-router)# network 10.0.12.0 0.0.0.3 area 0
R2(config-router)# exit

! === 백업 경로: VPN 터널을 통한 Floating Static ===
R2(config)# ip route 192.168.10.0 255.255.255.0 172.16.0.1 200
```

**동작 원리:**

1. **정상 상태**: OSPF가 전용선을 통해 경로를 학습합니다. 192.168.20.0/24 경로의 AD는 110입니다. Floating Static(AD=200)은 라우팅 테이블에 나타나지 않습니다.

2. **전용선 장애 발생**: 전용선이 끊기면 OSPF 네이버가 Dead Timer 만료 후 다운됩니다. OSPF 경로가 라우팅 테이블에서 삭제됩니다.

3. **자동 페일오버**: OSPF 경로가 사라지면 Floating Static Route(AD=200)가 라우팅 테이블에 자동 설치됩니다. 트래픽이 VPN 터널을 통해 전달됩니다.

4. **전용선 복구**: 전용선이 복구되면 OSPF 네이버가 다시 형성되고, OSPF 경로(AD=110)가 다시 설치됩니다. Floating Static(AD=200)은 다시 라우팅 테이블에서 사라집니다.

---

### 면접관: "페일오버가 발생하는 데 걸리는 시간은 얼마나 됩니까? 이 시간을 줄일 수 있는 방법이 있습니까?"

페일오버 시간은 주 경로가 라우팅 테이블에서 제거되는 데 걸리는 시간에 의존합니다.

**OSPF의 경우:**
- 인터페이스가 물리적으로 down 되면 즉시 OSPF 경로가 제거됩니다. 이 경우 페일오버는 거의 즉각적(1초 미만)입니다.
- 상대측 장비 장애 등으로 인터페이스는 up이지만 OSPF Hello가 끊긴 경우, Dead Timer가 만료될 때까지 기다려야 합니다. 이더넷의 기본 Dead Timer는 40초이므로 최대 40초의 페일오버 시간이 발생합니다.

**페일오버 시간을 줄이는 방법:**

```
! 방법 1: OSPF Hello/Dead Timer 조정
! 양쪽 라우터 모두 동일하게 설정해야 함
R1(config)# interface GigabitEthernet 0/1
R1(config-if)# ip ospf hello-interval 1
R1(config-if)# ip ospf dead-interval 3
! Hello 1초, Dead 3초로 설정 → 최대 3초 내 감지
```

```
! 방법 2: BFD (Bidirectional Forwarding Detection) 사용
! 밀리초 단위로 장애 감지 가능 (CCNA 범위 외이지만 개념 이해)
R1(config)# interface GigabitEthernet 0/1
R1(config-if)# bfd interval 50 min_rx 50 multiplier 3
R1(config)# router ospf 1
R1(config-router)# bfd all-interfaces
```

```
! 방법 3: IP SLA + Track을 사용한 정적 라우트 감시
! 대상 IP에 대한 ping 기반 모니터링
R1(config)# ip sla 1
R1(config-ip-sla)# icmp-echo 10.0.12.2
R1(config-ip-sla-echo)# frequency 5
R1(config-ip-sla-echo)# exit
R1(config)# ip sla schedule 1 life forever start-time now

R1(config)# track 1 ip sla 1 reachability

! Track과 연동된 정적 라우트 (Track 실패 시 경로 제거)
R1(config)# ip route 192.168.20.0 255.255.255.0 10.0.12.2 track 1
```

IP SLA + Track 방식은 CCNA 범위에서도 다루는 내용으로, next-hop의 도달 가능성을 주기적으로 확인하여 더 빠른 페일오버를 실현할 수 있습니다.

---

### 면접관: "Floating Static Route가 실제로 동작하는지 검증하는 방법을 단계별로 설명해주세요."

검증은 다음 단계로 수행합니다:

**1단계: 정상 상태에서 라우팅 테이블 확인**

```
R1# show ip route

! OSPF 경로만 보임, Floating Static는 보이지 않아야 함
O     192.168.20.0/24 [110/20] via 10.0.12.2, 01:23:45, GigabitEthernet0/1
```

**2단계: Floating Static Route가 설정되어 있는지 running-config에서 확인**

```
R1# show running-config | include ip route
ip route 192.168.20.0 255.255.255.0 172.16.0.2 200
```

라우팅 테이블에는 보이지 않지만, running-config에는 설정이 존재해야 합니다.

**3단계: 주 경로 장애 시뮬레이션**

```
! 테스트 환경에서 전용선 인터페이스를 shutdown
R1(config)# interface GigabitEthernet 0/1
R1(config-if)# shutdown
```

**4단계: 페일오버 확인**

```
R1# show ip route

! OSPF 경로가 사라지고 Floating Static이 나타남
S     192.168.20.0/24 [200/0] via 172.16.0.2

R1# show ip route 192.168.20.0
Routing entry for 192.168.20.0/24
  Known via "static", distance 200, metric 0
  Routing Descriptor Blocks:
  * 172.16.0.2
```

**5단계: 연결성 테스트**

```
R1# ping 192.168.20.1 source 192.168.10.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.20.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5)

R1# traceroute 192.168.20.1
! VPN 터널 경유 경로가 표시되어야 함
```

**6단계: 주 경로 복구 및 페일백 확인**

```
R1(config)# interface GigabitEthernet 0/1
R1(config-if)# no shutdown

! 잠시 대기 후 라우팅 테이블 확인
R1# show ip route 192.168.20.0
Routing entry for 192.168.20.0/24
  Known via "ospf 1", distance 110, metric 20
  ...
! OSPF 경로가 다시 설치되고, Floating Static은 사라짐
```

---

### 면접관: "ISP 이중화 환경에서 Floating Static Route를 활용하는 실제 시나리오를 설명해주세요."

인터넷 이중화 환경에서의 Floating Static Route 활용은 매우 일반적인 실무 패턴입니다.

```
                    [Internet]
                    /        \
              ISP-A            ISP-B
         203.0.113.1       198.51.100.1
              |                  |
         Gi0/0(Primary)    Gi0/1(Backup)
              \                /
               [  회사 R1  ]
                    |
              [내부 네트워크]
              192.168.0.0/16
```

**설정:**

```
! 주 ISP (ISP-A) - 기본 Default Route
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1

! 백업 ISP (ISP-B) - Floating Static Default Route
R1(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.1 10
```

여기서 AD를 10으로 설정한 이유는, 주 경로가 일반 Static(AD=1)이므로 백업은 1보다 높으면서 불필요하게 높지 않은 값을 선택한 것입니다.

**IP SLA를 결합한 고급 설정:**

단순 Floating Static만으로는 ISP-A의 인터페이스는 up이지만 ISP-A 내부에서 인터넷 연결이 끊긴 경우를 감지할 수 없습니다. IP SLA를 추가하면 이 문제를 해결할 수 있습니다.

```
! ISP-A를 통한 인터넷 연결 상태를 DNS 서버 ping으로 확인
R1(config)# ip sla 1
R1(config-ip-sla)# icmp-echo 8.8.8.8 source-interface GigabitEthernet0/0
R1(config-ip-sla-echo)# frequency 10
R1(config-ip-sla-echo)# threshold 1000
R1(config-ip-sla-echo)# timeout 2000
R1(config-ip-sla-echo)# exit
R1(config)# ip sla schedule 1 life forever start-time now

! Track 객체 생성
R1(config)# track 1 ip sla 1 reachability

! 주 ISP Default Route에 Track 연동
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 1

! 백업 ISP Floating Static Default Route
R1(config)# ip route 0.0.0.0 0.0.0.0 198.51.100.1 10
```

이렇게 설정하면 ISP-A를 통해 8.8.8.8에 도달할 수 없을 때 Track 1이 down이 되고, 주 경로가 라우팅 테이블에서 제거됩니다. 그러면 Floating Static Route(AD=10)가 활성화되어 ISP-B를 통해 인터넷에 접속합니다.

---

### 면접관: "Floating Static Route를 사용할 때 주의해야 할 점이나 한계가 있습니까?"

주요 주의사항과 한계는 다음과 같습니다:

**1. 인터페이스 상태에 의존하는 한계**

Floating Static Route는 주 경로가 라우팅 테이블에서 제거될 때만 활성화됩니다. 앞서 말씀드린 것처럼 물리 인터페이스는 up이지만 실제 네트워크 연결이 끊긴 경우(블랙홀 라우팅)에는 자동으로 페일오버되지 않습니다. 이를 해결하려면 IP SLA + Track을 반드시 병행해야 합니다.

**2. 양방향 설정 필수**

Floating Static Route도 일반 정적 라우트와 마찬가지로 양방향 설정이 필요합니다. 백업 경로를 한쪽에만 설정하면 비대칭 라우팅이 발생하여 반환 트래픽이 다른 경로로 돌아와 방화벽 등에서 차단될 수 있습니다.

**3. 수렴 시간 고려**

주 경로의 프로토콜에 따라 장애 감지 시간이 다릅니다:
- 인터페이스 down: 거의 즉시
- OSPF Dead Timer: 기본 40초
- EIGRP Hold Timer: 기본 15초

실무에서는 이 수렴 시간이 비즈니스 요구사항(SLA)을 충족하는지 확인해야 합니다.

**4. NAT/PAT 환경에서의 추가 고려**

ISP 이중화 환경에서 NAT를 사용한다면, 페일오버 시 출발지 IP가 바뀌므로 기존 세션이 모두 끊어집니다. 이는 Floating Static Route의 한계가 아니라 NAT 자체의 특성이지만, 운영 시 인지하고 있어야 합니다.

**5. 확장성 문제**

백업 경로가 많아지면 관리가 복잡해집니다. 대규모 환경에서는 동적 라우팅 프로토콜의 이중화(OSPF 다중 경로, EIGRP Feasible Successor 등)를 사용하는 것이 더 적합합니다. Floating Static Route는 소규모 환경이나 ISP 연결 백업에 가장 적합합니다.

```
! 현재 Floating Static Route 설정 확인 (running-config에서만 보임)
R1# show running-config | include ip route
ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 1
ip route 0.0.0.0 0.0.0.0 198.51.100.1 10

! Track 상태 확인
R1# show track 1
Track 1
  IP SLA 1 reachability
  Reachability is Up
    3 changes, last change 00:45:12
  Latest operation return code: OK
```
