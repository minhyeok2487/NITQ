# 08-04. QoS 기본 개념

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 시나리오

---

### 면접관: "네트워크에서 QoS가 왜 필요한지, 구체적인 예시를 들어 설명해 주세요."

QoS(Quality of Service)는 네트워크에서 **트래픽 유형별로 차등화된 서비스 품질**을 제공하는 기술입니다.

예를 들어, 회사에서 100Mbps WAN 회선을 사용하고 있는데, 다음과 같은 트래픽이 동시에 발생한다고 가정합니다.

- 임원 화상회의 (VoIP + Video)
- ERP 시스템 접속 (업무용 데이터)
- 직원 유튜브 시청 (일반 인터넷)
- 대용량 파일 백업 (FTP)

QoS가 없으면 모든 트래픽이 **Best Effort** 방식으로 처리되어, 파일 백업이 대역폭을 독점하면 화상회의 음질이 끊기고 ERP가 느려집니다.

QoS를 적용하면 트래픽을 분류하고 우선순위를 부여할 수 있습니다.

| 트래픽 유형 | QoS 요구사항 | 우선순위 |
|-------------|-------------|----------|
| **VoIP 음성** | 지연 < 150ms, 지터 < 30ms, 손실 < 1% | 최우선 |
| **화상회의** | 지연 < 200ms, 지터 < 30ms | 높음 |
| **ERP/업무 데이터** | 적정 대역폭 보장 | 중간 |
| **일반 인터넷** | Best Effort | 낮음 |
| **파일 백업** | 잔여 대역폭 사용 | 최하위 |

이렇게 분류하면 회선이 혼잡해도 화상회의 품질은 유지되고, 잔여 대역폭으로 파일 백업이 진행됩니다.

---

### 면접관: "QoS의 주요 메커니즘을 설명해 주세요. 어떤 단계로 QoS가 동작합니까?"

QoS는 다음 **다섯 가지 메커니즘**이 순차적으로 동작합니다.

#### 1. Classification (분류)
트래픽을 식별하여 그룹으로 나누는 단계입니다.
- **ACL 기반**: 출발지/목적지 IP, 포트 번호로 분류
- **NBAR (Network Based Application Recognition)**: 애플리케이션 단위로 분류
- **기존 마킹 기반**: 이미 표시된 DSCP/CoS 값으로 분류

#### 2. Marking (마킹)
분류된 트래픽에 우선순위 표시를 하는 단계입니다.
- **Layer 2**: CoS (Class of Service) - 802.1Q VLAN 태그의 3비트 (0~7)
- **Layer 3**: DSCP (Differentiated Services Code Point) - IP 헤더의 6비트 (0~63)
- **Legacy**: IP Precedence - IP 헤더의 3비트 (0~7)

#### 3. Queuing (큐잉)
마킹된 트래픽을 여러 큐에 배치하고 순서대로 전송하는 단계입니다.
- **LLQ (Low Latency Queuing)**: 음성 트래픽을 위한 우선 큐
- **CBWFQ (Class-Based Weighted Fair Queuing)**: 클래스별 대역폭 할당
- **WFQ (Weighted Fair Queuing)**: 가중치 기반 공정 큐잉

#### 4. Policing (폴리싱)
트래픽이 약속된 속도를 초과하면 **즉시 드롭**하는 단계입니다.
- 주로 인바운드(수신) 방향에서 사용합니다.
- ISP가 고객의 과다 트래픽을 제한할 때 사용합니다.

#### 5. Shaping (셰이핑)
트래픽이 약속된 속도를 초과하면 **버퍼에 저장 후 지연 전송**하는 단계입니다.
- 주로 아웃바운드(송신) 방향에서 사용합니다.
- 폴리싱보다 부드러운 트래픽 흐름을 제공합니다.

| 비교 | Policing | Shaping |
|------|----------|---------|
| **초과 트래픽 처리** | 드롭 | 버퍼링 후 지연 전송 |
| **적용 방향** | 인바운드/아웃바운드 | 주로 아웃바운드 |
| **지연 발생** | 없음 | 있음 (버퍼링) |
| **트래픽 흐름** | 버스트 발생 | 부드러운 흐름 |

---

### 면접관: "DSCP 값에 대해 좀 더 자세히 설명해 주세요. 주요 DSCP 값과 용도를 알려 주십시오."

DSCP(Differentiated Services Code Point)는 IP 헤더의 ToS(Type of Service) 필드 중 **상위 6비트**를 사용하여 64단계(0~63)의 우선순위를 표현합니다.

#### 주요 DSCP 값과 용도

| DSCP 값 | 이름 | Per-Hop Behavior | 용도 |
|---------|------|------------------|------|
| **46** | EF (Expedited Forwarding) | 최우선 전달 | VoIP 음성 |
| **34** | AF41 | 높은 우선순위 | 화상회의 비디오 |
| **26** | AF31 | 중간 우선순위 | 스트리밍 미디어 |
| **18** | AF21 | 보통 우선순위 | 트랜잭션 데이터 |
| **10** | AF11 | 낮은 우선순위 | 일반 대량 데이터 |
| **0** | BE (Best Effort) | 기본값 | 특별 처리 없음 |
| **8** | CS1 (Scavenger) | 최하위 | 불필요 트래픽 |

#### AF (Assured Forwarding) 클래스 구조

AF는 4개 클래스 x 3개 드롭 우선순위로 구성됩니다.

| | 낮은 드롭 확률 | 중간 드롭 확률 | 높은 드롭 확률 |
|---|---|---|---|
| **Class 4** | AF41 (34) | AF42 (36) | AF43 (38) |
| **Class 3** | AF31 (26) | AF32 (28) | AF33 (30) |
| **Class 2** | AF21 (18) | AF22 (20) | AF23 (22) |
| **Class 1** | AF11 (10) | AF12 (12) | AF13 (14) |

클래스 번호가 높을수록 우선순위가 높고, 드롭 확률 번호가 높을수록 혼잡 시 먼저 버려집니다.

또한 Layer 2 환경에서는 **CoS(Class of Service)** 값을 사용합니다. 802.1Q 태그의 PCP(Priority Code Point) 3비트로 0~7까지 표현합니다.

| CoS | 용도 |
|-----|------|
| **5** | VoIP 음성 |
| **4** | 화상회의 비디오 |
| **3** | Call Signaling |
| **0** | Best Effort |

---

### 면접관: "Trust Boundary(신뢰 경계)라는 개념이 있던데, 이것이 QoS에서 왜 중요합니까?"

Trust Boundary는 **네트워크에서 QoS 마킹을 신뢰하기 시작하는 지점**입니다.

일반 PC나 사용자 장비가 보낸 DSCP/CoS 마킹을 무조건 신뢰하면, 악의적인 사용자가 자신의 트래픽에 EF(46)를 마킹하여 최우선 처리를 받을 수 있습니다. 이를 방지하기 위해 Trust Boundary를 설정합니다.

#### Trust Boundary 적용 원칙

```
[PC] --- [Access Switch] --- [Distribution Switch] --- [WAN Router]
              ^
              |
        Trust Boundary
     (여기서부터 마킹 신뢰)
```

1. **엔드포인트(PC) 마킹은 신뢰하지 않음**: Access Switch에서 들어오는 트래픽의 DSCP를 0으로 재설정합니다.
2. **IP Phone 마킹은 신뢰**: Cisco IP Phone은 음성 트래픽에 EF(46)를 마킹하는데, CDP/LLDP로 IP Phone이 확인되면 해당 포트에서 마킹을 신뢰합니다.
3. **네트워크 장비 간 마킹은 신뢰**: Distribution 이상 구간에서는 이미 설정된 마킹을 신뢰합니다.

#### Access Switch에서 Trust 설정 예시

```
! IP Phone이 연결된 포트: CoS/DSCP 신뢰
Switch(config-if)# mls qos trust cos
Switch(config-if)# mls qos trust device cisco-phone

! 일반 PC 포트: 마킹을 신뢰하지 않음 (기본 동작)
Switch(config-if)# no mls qos trust
```

---

### 면접관: "QoS 모델 세 가지를 비교해서 설명해 주세요. Best Effort, IntServ, DiffServ의 차이는 무엇입니까?"

네트워크 QoS 모델은 세 가지로 분류됩니다.

#### 1. Best Effort (최선형)
- QoS가 전혀 없는 기본 모델입니다.
- 모든 패킷을 동등하게 FIFO(First In First Out)로 처리합니다.
- 인터넷의 기본 동작 방식입니다.
- 장점: 설정 불필요, 단순함
- 단점: 품질 보장 없음

#### 2. IntServ (Integrated Services)
- **RSVP(Resource Reservation Protocol)**를 사용하여 경로상의 모든 라우터에 **자원을 예약**합니다.
- 각 플로우(flow)마다 대역폭, 지연, 지터를 보장합니다.
- 장점: 확실한 QoS 보장
- 단점: **확장성 부족** (모든 라우터가 모든 플로우 상태를 유지해야 함)

#### 3. DiffServ (Differentiated Services)
- 트래픽을 **클래스별로 분류하고 마킹**하여 PHB(Per-Hop Behavior)에 따라 처리합니다.
- 각 라우터가 DSCP 마킹만 보고 독립적으로 처리합니다.
- 장점: **확장성 우수**, 실무에서 가장 많이 사용
- 단점: 절대적인 보장은 아닌 상대적 우선순위

| 비교 항목 | Best Effort | IntServ | DiffServ |
|-----------|-------------|---------|----------|
| **QoS 보장** | 없음 | 절대적 보장 | 상대적 보장 |
| **확장성** | 해당 없음 | 낮음 | 높음 |
| **복잡도** | 없음 | 매우 높음 | 중간 |
| **상태 관리** | 없음 | 플로우별 상태 유지 | 클래스별 처리 |
| **실무 사용** | 기본 | 거의 미사용 | 가장 많이 사용 |

현재 대부분의 엔터프라이즈 네트워크는 **DiffServ 모델**을 채택하고 있습니다.

---

### 면접관: "Cisco에서 QoS를 설정할 때 MQC(Modular QoS CLI)를 사용한다고 들었습니다. 기본 구조를 설명하고 간단한 예시를 보여 주세요."

MQC(Modular QoS CLI)는 Cisco에서 QoS를 설정하는 표준 프레임워크입니다. 세 가지 요소로 구성됩니다.

#### MQC 3단계 구조

| 단계 | 명령어 | 역할 |
|------|--------|------|
| **1. Class-map** | `class-map` | 트래픽 분류 (어떤 트래픽인가?) |
| **2. Policy-map** | `policy-map` | 정책 정의 (어떻게 처리할 것인가?) |
| **3. Service-policy** | `service-policy` | 정책 적용 (어디에 적용할 것인가?) |

#### 실제 설정 예시: WAN 인터페이스에 QoS 적용

시나리오: 100Mbps WAN 회선에서 VoIP에 우선순위, ERP에 30% 대역폭 보장, 나머지는 Best Effort로 처리합니다.

```
! 1단계: Class-map (트래픽 분류)
Router(config)# class-map match-all VOICE
Router(config-cmap)# match dscp ef
Router(config-cmap)# exit

Router(config)# class-map match-all ERP-DATA
Router(config-cmap)# match access-group 101
Router(config-cmap)# exit

! ACL로 ERP 서버 트래픽 식별
Router(config)# access-list 101 permit tcp any host 10.1.1.100 eq 443

! 2단계: Policy-map (정책 정의)
Router(config)# policy-map WAN-QOS
Router(config-pmap)# class VOICE
Router(config-pmap-c)# priority 10000
Router(config-pmap-c)# exit
Router(config-pmap)# class ERP-DATA
Router(config-pmap-c)# bandwidth percent 30
Router(config-pmap-c)# exit
Router(config-pmap)# class class-default
Router(config-pmap-c)# fair-queue
Router(config-pmap-c)# exit
Router(config-pmap)# exit

! 3단계: Service-policy (정책 적용)
Router(config)# interface GigabitEthernet 0/1
Router(config-if)# service-policy output WAN-QOS
```

#### 설정 확인 명령어

```
Router# show policy-map interface GigabitEthernet 0/1

  Service-policy output: WAN-QOS
    Class-map: VOICE (match-all)
      0 packets, 0 bytes
      Match: dscp ef (46)
      Priority: 10000 kbps, burst bytes 250000

    Class-map: ERP-DATA (match-all)
      0 packets, 0 bytes
      Match: access-group 101
      Bandwidth 30%, 30000 kbps

    Class-map: class-default (match-any)
      0 packets, 0 bytes
      Match: any
      Fair-queue
```

---

### 면접관: "VoIP 트래픽에 priority 명령어를 사용했는데, bandwidth와 어떤 차이가 있습니까?"

**priority**와 **bandwidth**는 모두 대역폭을 할당하지만 동작 방식이 다릅니다.

#### priority (LLQ - Low Latency Queuing)
- 해당 클래스의 트래픽을 **항상 먼저** 전송합니다.
- 다른 큐보다 절대적으로 우선합니다.
- 지정된 대역폭을 초과하면 초과분은 **폴리싱(드롭)**됩니다.
- 음성처럼 **지연에 매우 민감한** 트래픽에 사용합니다.

#### bandwidth (CBWFQ - Class-Based Weighted Fair Queuing)
- 해당 클래스에 **최소 대역폭을 보장**합니다.
- 혼잡하지 않으면 다른 클래스도 잔여 대역폭을 사용할 수 있습니다.
- 초과 트래픽은 드롭이 아니라 잔여 대역폭에서 공정하게 배분됩니다.
- 데이터 트래픽처럼 **일정 대역폭만 보장하면 되는** 트래픽에 사용합니다.

| 비교 항목 | priority | bandwidth |
|-----------|----------|-----------|
| **큐 처리 순서** | 항상 최우선 | 가중치 기반 |
| **초과 트래픽** | 폴리싱(드롭) | 잔여 대역폭 공유 |
| **지연** | 최소 | 상대적 |
| **적용 대상** | 음성, 실시간 트래픽 | 데이터, 비실시간 트래픽 |

주의할 점은 priority 큐에 너무 많은 대역폭을 할당하면 다른 트래픽이 기아(starvation) 상태에 빠질 수 있으므로, 일반적으로 **전체 대역폭의 33% 이하**로 설정하는 것을 권장합니다.
