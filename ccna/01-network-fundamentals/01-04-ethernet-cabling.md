# 01-04. 이더넷과 케이블링 (UTP/STP/광케이블)

> CCNA 200-301 수준의 네트워크 엔지니어 면접 시나리오입니다.
> 이더넷 표준, UTP 카테고리, 다이렉트/크로스오버 케이블, 광케이블, 커넥터 타입, Half/Full Duplex 등을 다룹니다.

---

### 면접관: "이더넷(Ethernet)의 주요 표준과 속도에 대해 설명해 주세요. 현재 실무에서 많이 사용되는 표준은 무엇인가요?"

이더넷은 IEEE 802.3 표준 기반의 LAN 기술로, 발전 과정에 따라 여러 속도의 표준이 존재합니다.

| 표준 이름 | IEEE 표준 | 속도 | 케이블 타입 | 최대 거리 |
|-----------|----------|------|------------|----------|
| Ethernet | 802.3 | 10 Mbps | UTP Cat3 | 100m |
| Fast Ethernet | 802.3u | 100 Mbps | UTP Cat5 | 100m |
| Gigabit Ethernet | 802.3ab | 1 Gbps | UTP Cat5e/6 | 100m |
| Gigabit Ethernet (광) | 802.3z | 1 Gbps | 광섬유 | 550m~5km |
| 10 Gigabit Ethernet | 802.3an | 10 Gbps | UTP Cat6a | 100m |
| 10 Gigabit Ethernet (광) | 802.3ae | 10 Gbps | 광섬유 | 300m~40km |

**현재 실무에서는:**
- **액세스 레이어**: Gigabit Ethernet (1 Gbps)이 표준적으로 사용됩니다. PC와 스위치 간 연결에 가장 많이 쓰입니다.
- **업링크/백본**: 10 Gigabit Ethernet 또는 그 이상을 사용합니다.
- **서버 환경**: 10 Gbps ~ 25 Gbps가 일반적이며, 데이터센터에서는 40 Gbps, 100 Gbps도 사용합니다.

이더넷의 표기법에서 `10BASE-T`와 같은 형식을 사용하는데, 여기서 10은 속도(Mbps), BASE는 베이스밴드 전송, T는 트위스티드 페어 케이블을 의미합니다.

---

### 면접관: "UTP 케이블의 카테고리별 차이점을 설명해 주세요. 실무에서 어떤 카테고리를 주로 사용하나요?"

UTP(Unshielded Twisted Pair, 비차폐 꼬임 쌍선) 케이블은 카테고리(Category)에 따라 지원하는 대역폭과 속도가 다릅니다.

| 카테고리 | 최대 대역폭 | 지원 속도 | 주요 용도 |
|---------|-----------|----------|----------|
| Cat 3 | 16 MHz | 10 Mbps | 전화선 (현재 거의 사용 안 함) |
| Cat 5 | 100 MHz | 100 Mbps | Fast Ethernet (레거시) |
| **Cat 5e** | 100 MHz | **1 Gbps** | Gigabit Ethernet (많이 사용) |
| **Cat 6** | 250 MHz | **1 Gbps** (10G@55m) | Gigabit Ethernet (권장) |
| **Cat 6a** | 500 MHz | **10 Gbps** | 10 Gigabit Ethernet |
| Cat 7 | 600 MHz | 10 Gbps | STP, 데이터센터 |
| Cat 8 | 2000 MHz | 25/40 Gbps | 데이터센터 (30m 제한) |

**실무에서의 선택:**
- **일반 사무실**: Cat 5e 또는 Cat 6가 가장 많이 사용됩니다. 비용 대비 성능이 우수합니다.
- **신규 구축**: Cat 6을 권장합니다. 향후 10G 업그레이드 가능성을 고려하기 위함입니다.
- **10G 필요 시**: Cat 6a를 사용해야 100m 거리까지 10 Gbps를 보장합니다.

**UTP vs STP:**

| 구분 | UTP | STP |
|------|-----|-----|
| 차폐 | 없음 | 호일/브레이드 차폐 |
| EMI 내성 | 낮음 | 높음 |
| 비용 | 저렴 | 비쌈 |
| 유연성 | 우수 | 다소 뻣뻣함 |
| 용도 | 일반 사무실 | 공장, EMI 환경 |

UTP가 가장 보편적으로 사용되며, 전자기 간섭(EMI)이 심한 환경에서만 STP를 고려합니다.

---

### 면접관: "다이렉트(Straight-through) 케이블과 크로스오버(Crossover) 케이블의 차이점을 설명하고, 각각 언제 사용하는지 말씀해 주세요."

두 케이블의 핵심 차이는 내부 배선 방식입니다.

**다이렉트(Straight-through) 케이블:**
- 양쪽 커넥터의 핀 배열이 동일합니다 (T568B - T568B 또는 T568A - T568A).
- **서로 다른 종류**의 장비를 연결할 때 사용합니다.

**크로스오버(Crossover) 케이블:**
- 양쪽 커넥터의 핀 배열이 다릅니다 (T568A - T568B).
- 1,2번 핀과 3,6번 핀이 교차됩니다.
- **같은 종류**의 장비를 연결할 때 사용합니다.

| 연결 | 케이블 타입 |
|------|-----------|
| PC ↔ 스위치 | 다이렉트 |
| PC ↔ 라우터 | 다이렉트 |
| 스위치 ↔ 라우터 | 다이렉트 |
| 스위치 ↔ 스위치 | 크로스오버 |
| 라우터 ↔ 라우터 | 크로스오버 |
| PC ↔ PC | 크로스오버 |

```
T568B 핀 배열:
Pin 1: 흰주 (Tx+)
Pin 2: 주황 (Tx-)
Pin 3: 흰녹 (Rx+)
Pin 4: 파랑
Pin 5: 흰파
Pin 6: 녹색 (Rx-)
Pin 7: 흰갈
Pin 8: 갈색
```

**실무 참고 사항:**

현재 대부분의 장비는 **Auto-MDI/MDIX** 기능을 지원하여, 어떤 케이블을 연결하든 자동으로 핀 배열을 감지하고 조정합니다. 따라서 최신 장비에서는 다이렉트 케이블로 통일하여 사용하는 것이 일반적입니다.

```
! Cisco 스위치에서 Auto-MDIX 확인/설정
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# mdix auto

Switch# show interfaces GigabitEthernet0/1
  Auto-MDIX : On
```

---

### 면접관: "광케이블(Fiber Optic)에 대해 설명해 주세요. 싱글모드와 멀티모드의 차이는 무엇인가요?"

광케이블은 전기 신호 대신 빛(광 신호)을 이용하여 데이터를 전송하는 케이블입니다. UTP에 비해 장거리 전송이 가능하고 EMI의 영향을 받지 않습니다.

| 특성 | 멀티모드 (MMF) | 싱글모드 (SMF) |
|------|---------------|---------------|
| 코어 직경 | 50 또는 62.5 마이크론 | 8~10 마이크론 |
| 광원 | LED 또는 VCSEL | 레이저 |
| 전송 거리 | ~550m (1G), ~400m (10G) | 수 km ~ 수십 km |
| 비용 | 상대적으로 저렴 | 상대적으로 고가 |
| 색상 (일반적) | 주황색 또는 아쿠아 | 노란색 |
| 용도 | 건물 내 백본, 데이터센터 | 캠퍼스 간 연결, WAN |
| 대역폭 | 상대적으로 낮음 | 매우 높음 |

**멀티모드**는 코어 직경이 넓어서 여러 빛의 경로(모드)가 존재하며, 모드 간 도착 시간 차이(모드 분산)로 인해 장거리 전송에 제한이 있습니다.

**싱글모드**는 코어 직경이 매우 좁아서 하나의 빛 경로만 통과하므로, 모드 분산이 없어 수십 km까지 전송할 수 있습니다.

#### 광케이블 커넥터 타입

| 커넥터 | 특징 | 용도 |
|--------|------|------|
| **LC** (Lucent Connector) | 소형, 래치 타입 | SFP 모듈, 가장 많이 사용 |
| **SC** (Subscriber Connector) | 사각형, 푸시풀 타입 | 데이터센터 패치 패널 |
| **ST** (Straight Tip) | 원형, 바요넷 잠금 | 구형 장비, 레거시 |
| **MPO/MTP** | 다심(12~24심) 다중 연결 | 40G/100G 고밀도 연결 |

```
! Cisco 스위치에서 SFP 모듈 정보 확인
Switch# show interfaces transceiver
Switch# show inventory
```

---

### 면접관: "Half Duplex와 Full Duplex의 차이를 설명하고, 현재 네트워크에서는 어떤 방식을 사용하나요?"

**Half Duplex (반이중):**
- 한 번에 한 방향으로만 데이터를 전송할 수 있습니다.
- 송신과 수신을 동시에 할 수 없습니다.
- 충돌(Collision)이 발생할 수 있어 CSMA/CD 메커니즘이 필요합니다.
- 허브(Hub)에 연결된 환경에서 사용됩니다.

**Full Duplex (전이중):**
- 송신과 수신을 동시에 할 수 있습니다.
- 충돌이 발생하지 않습니다.
- 스위치(Switch)에 연결된 환경에서 사용됩니다.
- 실제 대역폭이 2배로 활용됩니다 (1Gbps Full Duplex = 동시에 양방향 1Gbps).

| 항목 | Half Duplex | Full Duplex |
|------|-------------|-------------|
| 동시 통신 | 불가 | 가능 |
| 충돌 발생 | 가능 | 없음 |
| CSMA/CD | 필요 | 불필요 |
| 연결 장비 | 허브 | 스위치 |
| 효율 | 낮음 | 높음 |

**현재 네트워크에서는** 대부분 Full Duplex를 사용합니다. 허브는 거의 사용되지 않으며, 스위치 기반 환경에서는 각 포트가 독립적인 충돌 도메인이므로 Full Duplex가 기본입니다.

#### CSMA/CD (Carrier Sense Multiple Access / Collision Detection)

Half Duplex 환경에서 충돌을 처리하는 메커니즘입니다.

1. **Carrier Sense**: 전송 전에 회선이 사용 중인지 감지합니다.
2. **Multiple Access**: 여러 장비가 동일 매체를 공유합니다.
3. **Collision Detection**: 충돌 발생 시 감지하고, 잼 시그널을 보낸 후 랜덤 시간 대기 후 재전송합니다.

```
! 인터페이스 Duplex 설정 및 확인
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# duplex full
Switch(config-if)# speed 1000

Switch# show interfaces GigabitEthernet0/1
  Full-duplex, 1000Mb/s
```

---

### 면접관: "Auto-negotiation이란 무엇인가요? 그리고 Duplex mismatch 문제에 대해 설명해 주세요."

**Auto-negotiation**은 연결된 두 장비가 자동으로 최적의 속도(Speed)와 이중 방식(Duplex)을 협상하는 IEEE 802.3u 기능입니다.

**동작 과정:**
1. 양쪽 장비가 자신이 지원하는 속도와 duplex 능력을 광고합니다.
2. 양쪽이 모두 지원하는 가장 높은 속도와 duplex 모드를 선택합니다.
3. 협상된 설정으로 링크가 활성화됩니다.

**Duplex Mismatch 문제:**

한쪽이 수동 설정(예: Full Duplex)이고, 다른 쪽이 Auto-negotiation인 경우 문제가 발생할 수 있습니다.

```
예시 상황:
스위치 포트: speed 100, duplex full (수동 설정)
PC NIC: auto-negotiation

결과:
- PC는 상대방이 auto-negotiation에 응답하지 않으므로
- 속도는 100Mbps를 감지하여 맞추지만
- Duplex는 기본값인 Half Duplex로 설정됨
- 스위치: Full Duplex / PC: Half Duplex → Duplex Mismatch!
```

**Duplex mismatch 증상:**
- 속도가 비정상적으로 느림
- Late collision, CRC 에러 증가
- FCS 에러 카운터 증가
- 간헐적인 연결 끊김

#### 트러블슈팅 방법

```
! 인터페이스 상태 확인
Switch# show interfaces GigabitEthernet0/1
  Full-duplex, 100Mb/s, media type is 10/100/1000BaseTX
  Input errors: 0, CRC: 0, frame: 0

! 에러 카운터에 late collisions가 증가하면 Duplex mismatch 의심
  Late collision: 523

! 해결: 양쪽 모두 auto로 설정하거나, 양쪽 모두 동일하게 수동 설정
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# duplex auto
Switch(config-if)# speed auto
```

**모범 사례:** 양쪽 모두 auto-negotiation을 사용하거나, 양쪽 모두 동일한 값으로 수동 설정하는 것이 좋습니다. 한쪽만 수동으로 설정하면 Duplex mismatch가 발생할 수 있습니다.

---

### 면접관: "네트워크 케이블링과 관련된 실무 트러블슈팅 경험이 있나요? 물리 계층 문제를 어떻게 진단하시나요?"

물리 계층 문제 진단은 네트워크 엔지니어의 기본 역량입니다. 다음과 같은 체계적 접근 방법을 사용합니다.

#### 1단계: 물리적 확인

- 케이블이 양쪽에 확실하게 꽂혀 있는지 확인
- LED 상태 확인 (Link/Activity LED)
- 케이블 손상 여부 육안 점검
- 케이블 테스터로 핀 연결 상태 확인

#### 2단계: 인터페이스 상태 확인

```
Switch# show interfaces status
Port    Name   Status       Vlan  Duplex Speed Type
Gi0/1          connected    1     a-full a-1000 10/100/1000BaseTX
Gi0/2          notconnect   1     auto   auto   10/100/1000BaseTX
Gi0/3          err-disabled 1     auto   auto   10/100/1000BaseTX
```

| 상태 | 의미 | 조치 |
|------|------|------|
| connected | 정상 연결 | - |
| notconnect | 물리적 미연결 | 케이블 확인 |
| err-disabled | 에러로 비활성화 | 원인 확인 후 복구 |
| disabled | 관리자가 shutdown | no shutdown |

#### 3단계: 에러 카운터 분석

```
Switch# show interfaces GigabitEthernet0/1
  5 minute input rate 1000 bits/sec, 2 packets/sec
  Input errors: 15, CRC: 10, frame: 5, overrun: 0
  Output errors: 0, collisions: 0, late collision: 0
```

| 에러 타입 | 원인 | 조치 |
|-----------|------|------|
| CRC errors | 케이블 불량, EMI 간섭 | 케이블 교체 |
| Collisions | Half Duplex 환경 | Full Duplex 확인 |
| Late collisions | Duplex mismatch, 케이블 길이 초과 | 설정 확인, 케이블 길이 확인 |
| Input errors | 다양한 물리적 문제 | 케이블, 포트, 트랜시버 순서로 점검 |
| Runts | 최소 프레임 크기 미만 | 충돌 또는 NIC 문제 |
| Giants | 최대 프레임 크기 초과 | MTU 설정 확인 |

```
! 에러 카운터 초기화 후 모니터링
Switch# clear counters GigabitEthernet0/1
! 시간 경과 후 다시 확인하여 에러가 증가하는지 확인
Switch# show interfaces GigabitEthernet0/1
```

이러한 단계별 접근을 통해 물리 계층 문제를 체계적으로 진단하고 해결할 수 있습니다.
