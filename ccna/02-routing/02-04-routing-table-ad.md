# 02-04. 라우팅 테이블과 Administrative Distance

> CCNA 200-301 라우팅 기초 면접 시나리오
> 주제: 라우팅 테이블 구조, AD 값, Longest Prefix Match, 최적 경로 선택, 메트릭과 AD 차이

---

### 면접관: "show ip route 명령어의 출력을 보고, 라우팅 테이블의 구조를 상세히 읽는 방법을 설명해주세요."

라우팅 테이블 출력의 각 요소를 분석해보겠습니다.

```
R1# show ip route
Codes: L - local, C - connected, S - static,
       O - OSPF, D - EIGRP, B - BGP
       * - candidate default

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C        10.0.12.0/30 is directly connected, GigabitEthernet0/1
L        10.0.12.1/32 is directly connected, GigabitEthernet0/1
O        10.0.23.0/30 [110/20] via 10.0.12.2, 00:15:32, GigabitEthernet0/1
O        10.0.34.0/30 [110/30] via 10.0.12.2, 00:15:32, GigabitEthernet0/1
      192.168.10.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.10.0/24 is directly connected, GigabitEthernet0/0
L        192.168.10.1/32 is directly connected, GigabitEthernet0/0
O     192.168.20.0/24 [110/20] via 10.0.12.2, 00:15:32, GigabitEthernet0/1
D     192.168.30.0/24 [90/2570240] via 10.0.12.2, 00:10:45, GigabitEthernet0/1
```

각 항목을 분석하겠습니다:

**1. 경로 소스 코드 (첫 번째 문자)**
- `C` = Connected (직접 연결)
- `L` = Local (자신의 IP, /32)
- `S` = Static (정적 라우트)
- `O` = OSPF 학습 경로
- `D` = EIGRP 학습 경로
- `*` = Candidate Default Route

**2. 네트워크/프리픽스**
- `192.168.20.0/24`: 목적지 네트워크와 프리픽스 길이

**3. [AD/Metric]**
- `[110/20]`: 대괄호 안의 첫 번째 숫자가 Administrative Distance, 두 번째가 메트릭
- OSPF 경로이므로 AD=110, Cost=20

**4. via (Next-hop)**
- `via 10.0.12.2`: 패킷을 전달할 다음 홉 주소

**5. 타이머**
- `00:15:32`: 해당 경로를 학습한 이후 경과 시간

**6. 출구 인터페이스**
- `GigabitEthernet0/1`: 패킷이 나가는 인터페이스

**7. "variably subnetted" 표기**
- `10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks`: 10.0.0.0 대역에 서로 다른 서브넷 마스크를 가진 4개의 서브넷이 존재한다는 의미 (VLSM 사용)

---

### 면접관: "Administrative Distance(AD)란 무엇이며, 주요 라우팅 소스별 AD 값을 말씀해주세요. AD가 왜 필요합니까?"

Administrative Distance(AD)는 라우팅 정보의 신뢰도를 나타내는 값입니다. 값이 낮을수록 더 신뢰할 수 있는 소스로 간주됩니다.

AD가 필요한 이유는 동일한 목적지에 대해 여러 라우팅 소스에서 경로 정보를 받을 수 있기 때문입니다. 예를 들어 192.168.30.0/24 네트워크로의 경로를 OSPF와 EIGRP 모두에서 학습했다면, 라우터는 어떤 소스를 신뢰할지 결정해야 합니다. 이때 AD 값을 비교하여 더 낮은 AD를 가진 경로를 라우팅 테이블에 설치합니다.

**주요 라우팅 소스별 AD 값:**

| 라우팅 소스 | AD 값 | 설명 |
|-----------|-------|------|
| Connected | 0 | 직접 연결된 네트워크 (가장 신뢰) |
| Static | 1 | 관리자가 수동 설정 |
| eBGP | 20 | 외부 BGP |
| EIGRP (Internal) | 90 | EIGRP 내부 경로 |
| OSPF | 110 | OSPF 경로 |
| IS-IS | 115 | IS-IS 경로 |
| RIP | 120 | RIP 경로 |
| EIGRP (External) | 170 | EIGRP 외부 경로 |
| iBGP | 200 | 내부 BGP |
| Unknown/Unreachable | 255 | 사용 불가 (라우팅 테이블에 설치 안 됨) |

중요한 포인트는 AD는 **서로 다른 라우팅 프로토콜 간** 경로를 비교할 때 사용된다는 것입니다. 같은 프로토콜 내에서는 AD가 동일하므로, **메트릭**을 사용하여 최적 경로를 결정합니다.

```
! 동일 목적지에 OSPF(AD=110)와 EIGRP(AD=90) 경로가 있으면
! AD가 낮은 EIGRP 경로가 라우팅 테이블에 설치됨

R1# show ip route 192.168.30.0
Routing entry for 192.168.30.0/24
  Known via "eigrp 100", distance 90, metric 2570240
  ...
```

---

### 면접관: "Longest Prefix Match가 무엇인지 설명하고, 라우터가 패킷을 전달할 때 이 규칙을 어떻게 적용하는지 예시를 들어 설명해주세요."

Longest Prefix Match는 라우터가 목적지 IP에 대해 라우팅 테이블에서 가장 구체적인(프리픽스 길이가 가장 긴) 경로를 선택하는 규칙입니다. 이것은 AD나 메트릭보다 **항상 우선** 적용됩니다.

예시를 들어 설명하겠습니다. 라우팅 테이블에 다음 경로가 있다고 가정합니다:

```
S     10.0.0.0/8 [1/0] via 192.168.1.1
O     10.1.0.0/16 [110/20] via 192.168.1.2
D     10.1.1.0/24 [90/2570240] via 192.168.1.3
S     10.1.1.128/25 [1/0] via 192.168.1.4
```

**목적지 IP가 10.1.1.200일 때:**

| 경로 | 프리픽스 길이 | 일치 여부 | 선택 |
|------|-------------|----------|------|
| 10.0.0.0/8 | /8 | 일치 | |
| 10.1.0.0/16 | /16 | 일치 | |
| 10.1.1.0/24 | /24 | 일치 | |
| 10.1.1.128/25 | /25 | 일치 (10.1.1.128~255 범위) | 선택됨 |

10.1.1.128/25가 가장 긴 프리픽스(/25)를 가지므로, AD가 1인 Static 경로가 선택됩니다. 여기서 주목할 점은 EIGRP 경로(10.1.1.0/24, AD=90)가 AD로만 보면 OSPF보다 유리하지만, 더 구체적인 /25 Static 경로가 존재하므로 해당 패킷은 192.168.1.4로 전달됩니다.

**목적지 IP가 10.1.1.50일 때:**

| 경로 | 프리픽스 길이 | 일치 여부 | 선택 |
|------|-------------|----------|------|
| 10.0.0.0/8 | /8 | 일치 | |
| 10.1.0.0/16 | /16 | 일치 | |
| 10.1.1.0/24 | /24 | 일치 | 선택됨 |
| 10.1.1.128/25 | /25 | 불일치 (10.1.1.50은 범위 밖) | |

10.1.1.50은 10.1.1.128/25 범위(128~255)에 속하지 않으므로, 그 다음으로 긴 /24 경로인 EIGRP 경로로 192.168.1.3에 전달됩니다.

이처럼 라우터의 경로 선택 우선순위는 다음과 같습니다:

**1순위: Longest Prefix Match (프리픽스 길이)**
**2순위: Administrative Distance (라우팅 소스 신뢰도)**
**3순위: Metric (같은 프로토콜 내 최적 경로)**

---

### 면접관: "메트릭(Metric)과 Administrative Distance(AD)의 차이를 명확하게 설명해주세요. 둘은 어떤 상황에서 각각 사용됩니까?"

메트릭과 AD는 모두 최적 경로 선택에 관여하지만, 적용 범위와 목적이 다릅니다.

**Administrative Distance (AD):**
- **역할**: 서로 다른 라우팅 소스(프로토콜) 간 경로의 신뢰도를 비교
- **비교 대상**: OSPF 경로 vs EIGRP 경로 vs Static 경로 등
- **값의 의미**: 낮을수록 더 신뢰할 수 있는 소스
- **예시**: OSPF(AD=110)와 RIP(AD=120)이 같은 목적지를 알려주면 OSPF 선택

**메트릭 (Metric):**
- **역할**: 같은 라우팅 프로토콜 내에서 여러 경로 중 최적 경로를 선택
- **비교 대상**: OSPF 경로 A vs OSPF 경로 B (같은 프로토콜끼리)
- **값의 의미**: 프로토콜마다 기준이 다름
- **예시**: OSPF 경로 Cost 20 vs OSPF 경로 Cost 30이면 Cost 20 선택

**프로토콜별 메트릭:**

| 프로토콜 | 메트릭 기준 | 설명 |
|---------|-----------|------|
| OSPF | Cost | Reference BW / Interface BW |
| EIGRP | 복합 메트릭 | 대역폭, 지연 (기본), 신뢰도, 부하 (선택) |
| RIP | Hop Count | 경유하는 라우터 수 (최대 15) |
| Connected | 0 | 직접 연결이므로 메트릭 없음 |
| Static | 0 | 관리자 설정이므로 메트릭 없음 |

핵심 정리를 하면:

```
동일 목적지, 다른 프로토콜 → AD 비교 (낮은 AD 선택)
동일 목적지, 같은 프로토콜 → Metric 비교 (낮은 Metric 선택)
동일 목적지, 다른 프리픽스 → Longest Prefix Match (긴 프리픽스 선택)
```

---

### 면접관: "라우팅 테이블에 동일한 목적지에 대해 같은 메트릭을 가진 경로가 여러 개 있으면 어떻게 됩니까?"

같은 라우팅 프로토콜에서 동일한 목적지에 대해 동일한 메트릭을 가진 경로가 여러 개 존재하면, **Equal-Cost Multi-Path (ECMP)** 로드 밸런싱이 수행됩니다.

```
R1# show ip route 192.168.30.0
Routing entry for 192.168.30.0/24
  Known via "ospf 1", distance 110, metric 20, type intra area
  Last update from 10.0.13.2 on GigabitEthernet0/2, 00:05:12 ago
  Routing Descriptor Blocks:
  * 10.0.12.2, from 2.2.2.2, 00:05:12 ago, via GigabitEthernet0/1
  * 10.0.13.2, from 3.3.3.3, 00:05:12 ago, via GigabitEthernet0/2
```

위 출력에서 192.168.30.0/24로 가는 OSPF 경로가 2개 있고, 둘 다 Cost가 20으로 동일합니다. 이 경우 라우터는 두 경로를 모두 라우팅 테이블에 설치하고 트래픽을 분산합니다.

**OSPF의 ECMP 특징:**
- 기본적으로 최대 4개의 Equal-Cost 경로를 설치합니다 (IOS에 따라 다름).
- `maximum-paths` 명령어로 변경할 수 있습니다.

```
R1(config)# router ospf 1
R1(config-router)# maximum-paths 8
! 최대 8개까지 Equal-Cost 경로 허용
```

**로드 밸런싱 방식:**
Cisco 라우터는 기본적으로 CEF(Cisco Express Forwarding)를 사용하며, Per-Destination 또는 Per-Packet 로드 밸런싱을 수행합니다. 기본값은 Per-Destination으로, 같은 목적지로의 패킷은 같은 경로를 사용합니다. 이는 패킷 순서가 뒤바뀌는 것을 방지하기 위함입니다.

---

### 면접관: "다음 시나리오를 분석해주세요. R1의 라우팅 테이블에 172.16.10.0/24에 대한 경로가 3개 있습니다. 어떤 경로가 선택됩니까?"

```
시나리오:
S     172.16.10.0/24 [1/0] via 10.0.0.1       ← Static (AD=1)
O     172.16.10.0/24 [110/30] via 10.0.0.2     ← OSPF (AD=110)
D     172.16.10.0/24 [90/2570240] via 10.0.0.3 ← EIGRP (AD=90)
```

먼저 이 세 경로는 모두 같은 프리픽스(/24)를 가지므로 Longest Prefix Match는 동일합니다.

다음으로 AD를 비교합니다:
- Static: AD = 1
- EIGRP: AD = 90
- OSPF: AD = 110

**Static 경로(AD=1)가 선택**됩니다. AD가 가장 낮기 때문입니다.

실제로 라우팅 테이블에는 하나의 경로만 설치됩니다:

```
R1# show ip route 172.16.10.0
Routing entry for 172.16.10.0/24
  Known via "static", distance 1, metric 0
  Routing Descriptor Blocks:
  * 10.0.0.1
```

나머지 EIGRP와 OSPF 경로는 라우팅 테이블에 나타나지 않지만, 각 프로토콜의 데이터베이스에는 보관되어 있습니다. Static 경로가 사라지면(예: next-hop 인터페이스가 down) EIGRP 경로(AD=90)가 자동으로 라우팅 테이블에 설치됩니다.

이 원리를 활용한 것이 바로 Floating Static Route입니다. 정적 라우트의 AD를 동적 라우팅 프로토콜보다 높게 설정하면, 동적 경로가 사라졌을 때만 정적 경로가 활성화되어 백업 역할을 합니다.

---

### 면접관: "마지막으로, 라우팅 테이블 관련 트러블슈팅 시 주로 사용하는 명령어와 접근 방법을 설명해주세요."

라우팅 테이블 트러블슈팅의 체계적인 접근 방법을 설명하겠습니다.

**1단계: 라우팅 테이블에 목적지 경로가 존재하는지 확인**
```
R1# show ip route
R1# show ip route 192.168.30.0
R1# show ip route 192.168.30.0 255.255.255.0 longer-prefixes
```

**2단계: 특정 프로토콜의 경로만 필터링하여 확인**
```
R1# show ip route static
R1# show ip route ospf
R1# show ip route eigrp
R1# show ip route connected
```

**3단계: 특정 목적지까지의 경로 추적**
```
R1# traceroute 192.168.30.1
! 패킷이 어느 경로를 통해 전달되는지 홉별로 확인
```

**4단계: CEF 테이블 확인 (실제 포워딩 결정)**
```
R1# show ip cef 192.168.30.0
! 실제 CEF가 어떤 인터페이스로 포워딩하는지 확인
```

**흔한 문제 패턴과 해결:**

| 증상 | 가능한 원인 | 확인 방법 |
|------|-----------|----------|
| 목적지 경로 없음 | 라우팅 프로토콜 미설정 | show ip protocols |
| 잘못된 경로로 전달 | AD/Metric 문제, 더 구체적인 경로 존재 | show ip route [목적지] |
| 비대칭 경로 | 양방향 경로 불일치 | 양쪽 라우터에서 show ip route |
| 간헐적 연결 불가 | 플래핑 경로 | show logging, debug ip routing |

핵심은 "패킷의 관점에서 생각하기"입니다. 출발지에서 목적지까지, 그리고 목적지에서 출발지까지 양방향 모두 경로가 올바른지 확인해야 합니다. 한쪽 방향만 확인하는 것은 초보 엔지니어가 자주 하는 실수입니다.
