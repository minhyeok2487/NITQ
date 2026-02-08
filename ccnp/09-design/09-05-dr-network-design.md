# 09-05. 재해복구(DR) 네트워크 설계

---

### 면접관: "고객사가 재해복구(DR) 체계를 구축하려고 합니다. 주 데이터센터는 서울, DR 사이트는 대전에 있어요. 핵심 업무 시스템의 RTO 4시간, RPO 1시간 요건입니다. 네트워크를 어떻게 설계하시겠어요?"

#### 전체 아키텍처 개요

```
[서울 주 DC]                                          [대전 DR DC]
┌────────────────────┐      WAN(전용회선)      ┌────────────────────┐
│  [Core SW]──[FW]   │ ◄══════════════════════► │  [Core SW]──[FW]   │
│      │             │   Primary: 10G MPLS      │      │             │
│  [Server Farm]     │   Backup: 1G Internet VPN│  [Server Farm]     │
│  - Web/App/DB      │                          │  - Web/App/DB      │
│  - Storage (SAN)   │ ◄──Storage Replication──►│  - Storage (SAN)   │
│      │             │                          │      │             │
│  [Users/Branch]    │                          │  (Standby)         │
└────────────────────┘                          └────────────────────┘
         │
    [GSLB/DNS]──── Internet ────[Users]
```

#### 네트워크 설계 요소

#### 1. WAN 회선 (서울 ↔ 대전)

| 회선 | 용도 | 대역폭 | 비고 |
|------|------|--------|------|
| Primary | 데이터 복제 + 관리 트래픽 | 10Gbps (MPLS 전용회선) | KT/SKB 전용망 |
| Backup | Primary 장애 시 Failover | 1Gbps (IPsec VPN over Internet) | 이중화 |

```
! 서울 DC 라우터 - MPLS 전용회선 (Primary)
interface GigabitEthernet0/0
 description TO-DAEJEON-DR-PRIMARY
 ip address 10.255.1.1 255.255.255.252
 bandwidth 10000000

! 서울 DC 라우터 - Internet VPN (Backup)
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
!
crypto map DR-VPN 10 ipsec-isakmp
 set peer 203.0.113.100
 set transform-set AES256-SHA256
 match address DR-TRAFFIC
!
interface GigabitEthernet0/1
 description TO-DAEJEON-DR-BACKUP-VPN
 ip address 203.0.113.1 255.255.255.248
 crypto map DR-VPN
```

#### 2. 라우팅 설계

- Primary/Backup 경로를 **Floating Static Route** 또는 **BGP MED/Local Preference**로 제어
- Primary 장애 시 자동으로 Backup VPN으로 Failover

```
! Floating Static Route 예시
ip route 10.200.0.0 255.255.0.0 10.255.1.2 name PRIMARY-MPLS
ip route 10.200.0.0 255.255.0.0 10.255.2.2 100 name BACKUP-VPN
! AD 100으로 Backup 경로가 Primary보다 우선순위 낮음
```

#### 3. 서울-대전 동일 서브넷 확장 (필요 시)

DR 전환 시 서버 IP를 변경하지 않으려면, 양쪽 DC에 **동일 서브넷**을 확장해야 합니다.

- **OTV (Overlay Transport Virtualization)** 또는 **VXLAN over WAN** 사용
- 이에 대해서는 뒤에서 자세히 설명하겠습니다.

> 토폴로지 이미지 추가 예정

---

### 면접관: "RPO와 RTO를 설명해주세요. 네트워크 설계에 어떤 영향을 주나요?"

#### RPO (Recovery Point Objective) - 복구 시점 목표

**"장애 발생 시 최대 얼마나 오래된 데이터까지 잃어도 되는가?"**

```
RPO = 1시간
→ 최악의 경우 최근 1시간 데이터 손실 허용
→ 최소 1시간마다 데이터 복제/백업 필요
```

- RPO 0 = 데이터 손실 불가 → **동기(Synchronous) 복제** 필요
- RPO > 0 = 일정 데이터 손실 허용 → **비동기(Asynchronous) 복제** 가능

#### RTO (Recovery Time Objective) - 복구 시간 목표

**"장애 발생 후 최대 얼마 안에 서비스를 복구해야 하는가?"**

```
RTO = 4시간
→ 장애 발생 후 4시간 이내에 DR 사이트에서 서비스 재개
→ 네트워크 전환, 서버 기동, 데이터 정합성 확인 포함
```

#### 네트워크 설계에 미치는 영향

| RPO/RTO | 복제 방식 | WAN 대역폭 | DR 구성 | 비용 |
|---------|-----------|------------|---------|------|
| RPO=0, RTO<1분 | 동기 복제 | 매우 높음 (10G+) | Active-Active | 최고 |
| RPO<1h, RTO<4h | 비동기 복제 | 높음 (1~10G) | Active-Standby (Warm) | 고 |
| RPO<24h, RTO<24h | 주기적 백업 | 보통 (100M~1G) | Cold Standby | 중 |
| RPO<7d, RTO<48h | 테이프/오프라인 백업 | 낮음 | Cold Site | 저 |

#### 이 프로젝트 (RPO 1h, RTO 4h)

- **비동기 복제 + Warm Standby** 구성이 적합
- WAN 대역폭: **데이터 변경량(Change Rate)에 따라 산정**

```
WAN 대역폭 산정 예시:
- 일일 데이터 변경량: 500GB
- RPO 1시간 → 1시간 동안 변경량: 500GB / 24h ≒ 21GB
- 21GB를 1시간 내에 전송: 21GB x 8bit / 3,600초 ≒ 47Mbps
- 오버헤드(프로토콜, 압축 효율) 고려 x 1.5: ≒ 70Mbps
- 피크 시간대 고려 x 2: ≒ 140Mbps
- 여유 포함 최소 대역폭: 200Mbps ~ 1Gbps 권장
```

---

### 면접관: "Active-Active DR과 Active-Standby DR의 차이를 설명해주세요. 각각 언제 쓰나요?"

#### Active-Standby (이 프로젝트 기본안)

```
[서울 DC - Active]                    [대전 DR - Standby]
  모든 서비스 운영 중                    서버 전원 OFF 또는 최소 가동
  실시간 데이터 복제 ──────────────►     복제 데이터 수신만
  사용자 트래픽 처리                      사용자 트래픽 없음
```

**동작**:
- 평상시 서울 DC에서만 서비스 운영
- 대전 DR은 복제 데이터를 수신하며 대기
- 서울 장애 시 → 대전 DR을 활성화 (수동 또는 반자동 전환)

**전환 과정**:
1. 장애 감지 (모니터링/알림)
2. DR 전환 결정 (경영진 승인 - 보통 여기서 시간 소모)
3. DR 서버 기동 / 데이터 정합성 확인
4. DNS/GSLB 전환 → 트래픽을 대전으로 유도
5. 서비스 재개

**RTO**: 보통 **1~8시간** (수동 절차 포함)

#### Active-Active

```
[서울 DC - Active]  ◄───── GSLB ─────► [대전 DC - Active]
  서비스 50% 처리     로드밸런싱          서비스 50% 처리
  양방향 데이터 동기 ◄─────────────────► 양방향 데이터 동기
```

**동작**:
- **양쪽 DC 모두** 서비스 운영 (트래픽 분산)
- GSLB가 사용자를 지리적으로 가까운 DC로 유도
- 데이터 양방향 동기 복제 (Conflict Resolution 필요)

**장애 시**:
- 서울 Down → GSLB가 자동으로 모든 트래픽을 대전으로 유도
- **RTO**: 거의 **0** (이미 대전이 운영 중이므로)
- 다만 대전 DC가 갑자기 100% 트래픽을 감당해야 함 → **용량 설계 중요**

#### 비교

| 구분 | Active-Standby | Active-Active |
|------|----------------|---------------|
| RTO | 1~8시간 | 수초~수분 |
| RPO | 복제 주기에 따라 | 거의 0 (동기 복제 시) |
| 비용 | 중 (DR 자원 유휴) | 고 (양쪽 풀 용량) |
| 전환 방식 | 수동/반자동 | 자동 (GSLB) |
| 데이터 동기 | 단방향 | 양방향 (복잡) |
| 적용 | 일반 기업, RTO 요건 완화 | 금융/통신, RTO 0 요구 |

#### 이 프로젝트 (RPO 1h, RTO 4h)

RTO 4시간이면 **Active-Standby**로 충분합니다. Active-Active는 비용이 거의 2배이고, 데이터 양방향 동기의 복잡도(Split-Brain, Conflict)가 높습니다. 다만 DR 서버를 **Warm Standby** (OS/앱 설치 완료, 최신 데이터 복제 완료, 전원 ON but 서비스 미오픈)로 유지하여 전환 시간을 최소화하겠습니다.

---

### 면접관: "서울 주 DC에 화재가 발생하여 전체 다운되었습니다. DR 전환 시나리오를 네트워크 관점에서 단계별로 설명해주세요."

#### DR 전환 시나리오 (Active-Standby, Warm Standby 기준)

#### Phase 0: 장애 감지 (T+0분 ~ T+10분)

```
1. 서울 DC 전체 Down → 모니터링 시스템(NMS/SIEM) Alert
2. GSLB Health Check 실패 → 서울 DC 서비스 Unreachable 확인
3. WAN 회선(MPLS + VPN) 양쪽 모두 Down 확인
4. DR 전환 여부 결정 (비상 연락 체계 가동)
```

**네트워크 확인 사항**:
- 서울 DC의 모든 BGP 세션 Down → ISP에서 서울 DC 경로(prefix) 철회
- GSLB(F5 GTM 등)가 서울 DC Health Check 실패 감지 (기본 30초 주기)

#### Phase 1: DR 네트워크 활성화 (T+10분 ~ T+30분)

```
! 대전 DR - 서비스 IP 활성화
! 서울에서 사용하던 Public IP를 대전 ISP로 BGP Announce
router bgp 65000
 network 203.0.113.0 mask 255.255.255.0
 ! 기존에는 announce 안 하고 있었음 (suppressed)
```

- 대전 DR의 방화벽 정책 활성화 (사전 설정 완료 상태)
- 서버 서비스 포트 오픈 (웹서버 80/443, 앱서버 등)

#### Phase 2: DNS/GSLB 전환 (T+30분 ~ T+1시간)

**GSLB 자동 전환** (사전 설정 시):

```
! F5 GTM GSLB 설정 예시
ltm pool WEB-POOL {
    members {
        SEOUL-VIP:443 {
            monitor http
            priority 10
        }
        DAEJEON-VIP:443 {
            monitor http
            priority 1
        }
    }
    load-balancing-mode topology
    fallback-mode priority
}
```

- 서울 VIP Health Check 실패 → GSLB가 자동으로 대전 VIP 응답
- DNS TTL 만큼의 전환 시간 소요 (TTL 300초 = 5분)
- **주의**: DNS TTL이 길면 전환이 느려짐 → DR 대비 시 **TTL을 짧게 사전 조정** (60~300초)

**수동 DNS 전환** (GSLB 미사용 시):

```
# DNS A 레코드 변경
web.company.com  A  203.0.113.10  → 변경 →  A  198.51.100.10
                  (서울 VIP)                    (대전 VIP)
```

#### Phase 3: 데이터 정합성 확인 (T+1시간 ~ T+2시간)

- 마지막 복제 시점 확인 (RPO 1시간 → 최대 1시간 데이터 손실 가능)
- DB 정합성 체크 (트랜잭션 로그 확인)
- 애플리케이션 레벨 데이터 검증

#### Phase 4: 서비스 재개 및 검증 (T+2시간 ~ T+4시간)

```
1. 대전 DR에서 서비스 오픈
2. 모니터링: 서비스 정상 여부 확인 (응답 시간, 에러율)
3. 사용자 공지 (서비스 재개 안내)
4. 잔여 이슈 처리 (일부 세션 끊김, 캐시 갱신 등)
```

#### 전체 타임라인

```
T+0     : 화재 발생, 서울 DC Down
T+5~10  : 모니터링 Alert, 장애 확인
T+15    : DR 전환 결정 (경영진 승인)
T+30    : DR 네트워크 활성화 완료
T+45    : GSLB/DNS 전환 완료, 트래픽 유입 시작
T+1~2h  : 데이터 정합성 확인
T+2~3h  : 서비스 정상 운영 확인
T+4h    : RTO 달성 ✓
```

---

### 면접관: "GSLB와 OTV에 대해 좀 더 설명해주세요. 각각 DR에서 어떤 역할을 하나요?"

#### GSLB (Global Server Load Balancing)

**DNS 기반 트래픽 분배 기술**로, 여러 DC에 분산된 서비스로 사용자를 유도합니다.

```
[User] → DNS Query: web.company.com
                  ↓
            [GSLB (F5 GTM / Citrix ADC)]
                  ↓
    ┌─────────────┼─────────────┐
    ▼                           ▼
[서울 DC VIP]            [대전 DR VIP]
203.0.113.10            198.51.100.10
  (Healthy)               (Healthy)
```

**동작 원리**:
1. GSLB가 각 DC의 VIP에 대해 **Health Check** 수행
2. DNS 쿼리가 오면, **Health 상태 + 정책**에 따라 응답할 DC 결정
3. 정책: Round-Robin, 지리적 근접성(Topology), 가중치(Ratio), 우선순위(Priority)

**DR에서의 역할**:
- 서울 DC Health Check 실패 → GSLB가 대전 DR의 VIP를 DNS 응답으로 반환
- **자동 전환**: 사람의 개입 없이 트래픽이 대전으로 유도
- **한계**: DNS TTL 동안 캐시된 응답은 여전히 서울로 향함 → TTL 단축 필요

#### OTV (Overlay Transport Virtualization)

**L2 도메인을 WAN을 통해 확장**하는 기술로, DR 전환 시 서버 IP 변경 없이 서비스를 이전할 수 있게 합니다.

```
[서울 DC]                    WAN                    [대전 DR]
VLAN 100: 10.1.100.0/24 ◄═══ OTV Tunnel ═══► VLAN 100: 10.1.100.0/24
  Server-A: 10.1.100.10                        Server-A-DR: 10.1.100.10
```

**왜 필요한가?**

DR 전환 시 가장 큰 문제 중 하나가 **IP 변경**입니다.
- 서울 서버 IP: 10.1.100.10
- 대전 DR 서버 IP: 10.2.100.10 (다른 서브넷)
- IP가 바뀌면 → DNS 변경, 방화벽 룰 변경, 애플리케이션 설정 변경... → **전환 시간 증가**

OTV로 L2를 확장하면:
- 양쪽 DC에서 **동일한 VLAN/서브넷** 사용 가능
- DR 서버가 서울 서버와 **같은 IP**로 서비스 재개
- DNS/방화벽 변경 불필요 → **전환 시간 단축**

**OTV 동작**:

```
! 서울 DC - OTV Edge 설정 (NX-OS)
feature otv

interface Overlay1
 otv join-interface Ethernet1/1
 otv control-group 239.1.1.1
 otv data-group 232.1.1.0/24
 otv extend-vlan 100,200
 no shutdown
```

- **OTV Edge Device**: 각 DC에 배치, VLAN을 OTV 캡슐화하여 WAN으로 전달
- **MAC-in-IP 캡슐화**: L2 프레임을 IP 패킷으로 캡슐화
- **STP 격리**: 한쪽 DC의 STP 이벤트가 다른 DC로 전파되지 않음
- **ARP 최적화**: 불필요한 ARP Broadcast를 WAN으로 보내지 않음 (ARP 캐시 활용)

#### OTV vs VXLAN over WAN

| 구분 | OTV | VXLAN over WAN |
|------|-----|----------------|
| 프로토콜 | MAC-in-IP | MAC-in-UDP |
| Control Plane | OTV IS-IS | BGP EVPN |
| STP 격리 | 지원 | 지원 |
| 벤더 | Cisco 전용 | 멀티벤더 |
| 트렌드 | Legacy | 현재 주류 |

최근에는 OTV 대신 **VXLAN + BGP EVPN over DCI(Data Center Interconnect)**가 더 많이 사용됩니다. 멀티벤더 환경에서도 동작하고, DC 내부 Fabric과 동일한 기술 스택을 사용할 수 있기 때문입니다.

---

### 면접관: "동기 복제와 비동기 복제의 차이를 네트워크 관점에서 설명해주세요. 서울-대전 간 거리가 영향을 주나요?"

#### 동기 복제 (Synchronous Replication)

```
[서울 DC]                        [대전 DR]
  App Write ──► Storage ──Write──► Storage
                   │                  │
                   │    ACK ◄─────────┘
                   │
              Write 완료 확인
```

- 서울에서 데이터를 쓰면, **대전에도 쓰기가 완료된 후에야** 서울에서 Write ACK 반환
- **RPO = 0**: 데이터 손실 없음 (양쪽 항상 동일)
- **단점**: WAN 지연(RTT)이 Write 성능에 직접 영향

#### 비동기 복제 (Asynchronous Replication)

```
[서울 DC]                        [대전 DR]
  App Write ──► Storage ──► Write ACK (즉시)
                   │
                   └──── 나중에 ────► Storage (복제)
```

- 서울에서 데이터를 쓰면, **즉시** Write ACK 반환
- 대전으로의 복제는 **백그라운드로 나중에** 수행
- **RPO > 0**: 복제 지연만큼 데이터 손실 가능
- **장점**: Write 성능에 WAN 지연 영향 없음

#### 서울-대전 간 거리의 영향

```
서울 ↔ 대전 직선 거리: 약 150km
광케이블 전파 속도: 약 200,000 km/s (광속의 2/3)
단방향 지연: 150km / 200,000 km/s ≒ 0.75ms
왕복(RTT): 약 1.5ms (실제 라우팅/장비 지연 포함: 3~10ms)
```

#### 동기 복제 시 영향

- 매 Write마다 RTT(3~10ms) 추가
- IOPS 영향: 로컬 Write가 0.5ms라면, 동기 복제 시 0.5ms + 10ms = 10.5ms → **IOPS 약 95% 하락**
- **서울-대전(150km)은 동기 복제가 가능한 거리**이지만, 성능 저하는 감수해야 함

#### 거리별 복제 방식 권장

| 거리 | RTT | 동기 복제 | 비동기 복제 |
|------|-----|-----------|-------------|
| ~100km | ~5ms | 가능 (성능 저하 감수) | 가능 |
| 100~300km | 5~15ms | 고성능 스토리지에서 가능 | 권장 |
| 300km+ | 15ms+ | 비현실적 (성능 심각 저하) | 필수 |

#### 이 프로젝트

- 서울-대전 약 150km, RTT 약 5~10ms
- RPO 1시간 → **비동기 복제로 충분**
- 비동기 복제 주기: 10~30분 (RPO 1시간 내 여유 확보)
- 동기 복제가 필요한 경우(RPO=0): 가능하지만, 성능 영향을 애플리케이션 팀과 협의 필요

---

### 면접관: "마지막으로, DR 전환 테스트(DR Drill)는 어떻게 계획하나요? 네트워크 관점에서 Runbook에 포함할 항목을 알려주세요."

#### DR Drill (재해복구 전환 훈련) 목적

- 실제 장애 시 **Runbook 대로 전환이 되는지 검증**
- 전환 소요 시간 측정 → RTO 달성 여부 확인
- 담당자 숙련도 향상
- **규제 요건**: 금융기관은 **연 1회 이상 DR 훈련 의무** (전자금융감독규정)

#### DR Drill 방식

#### 1. Tabletop Exercise (문서 기반 훈련)

- 실제 전환 없이, 시나리오 기반으로 **절차를 리뷰**
- 비용/리스크 최소, 실효성은 낮음

#### 2. Partial Failover (부분 전환)

- 일부 시스템만 DR로 전환하여 테스트
- 예: 웹서버만 대전으로 전환, DB는 서울 유지

#### 3. Full Failover (전체 전환)

- **실제 서비스를 DR로 완전 전환**
- 가장 실효성 높음, 리스크도 높음
- 보통 심야/주말에 수행

#### 네트워크 Runbook 체크리스트

```
[DR 전환 Runbook - 네트워크 섹션]

■ 사전 준비 (D-7)
□ DR 사이트 네트워크 장비 상태 점검 (Spine/Leaf/FW/LB)
□ WAN 회선 상태 확인 (MPLS + VPN 양쪽)
□ GSLB Health Check 상태 확인
□ DNS TTL 단축 (300초 → 60초) - 전환 1주일 전 적용
□ DR 사이트 방화벽 룰 최신화 (서울과 동일한지 비교)
□ DR 사이트 라우팅 테이블 사전 검증

■ 전환 실행 (D-Day)
□ Step 1: 서울 DC 서비스 트래픽 차단
  - 서울 FW에서 서비스 포트 차단 또는 LB에서 Pool Disable

□ Step 2: 최종 데이터 복제 완료 확인
  - 스토리지 복제 상태 확인 (RPO 이내인지)

□ Step 3: 대전 DR 네트워크 활성화
  - BGP Announce (서비스 IP prefix)
  - 방화벽 서비스 정책 활성화
  - LB VIP 활성화 + Health Check 정상 확인

□ Step 4: GSLB/DNS 전환
  - GSLB: 서울 Pool Disable → 대전 Pool Active
  - 또는 DNS A 레코드 변경

□ Step 5: 트래픽 유입 확인
  - 대전 DR FW 로그에서 인바운드 트래픽 확인
  - 서비스 응답 시간 모니터링
  - 외부에서 접속 테스트 (다양한 ISP/지역에서)

□ Step 6: 전체 기능 검증
  - 웹 로그인, 결제, API 호출 등 주요 기능 테스트
  - SSL 인증서 정상 확인
  - 세션 유지 여부 확인
```

#### 복귀 (Failback) 절차

DR Drill의 핵심은 전환뿐 아니라 **복귀(Failback)**까지 검증하는 것입니다.

```
[Failback Runbook - 네트워크 섹션]

□ Step 1: 서울 DC 네트워크 복구 확인
  - 모든 장비 정상 가동, 라우팅 테이블 정상

□ Step 2: 데이터 역복제 (대전 → 서울)
  - DR 운영 중 변경된 데이터를 서울로 복제

□ Step 3: GSLB/DNS를 서울로 복귀
  - 점진적 전환: GSLB Weight 조정 (대전 90% → 50% → 10% → 0%)

□ Step 4: 서울 DC 서비스 정상 확인

□ Step 5: DNS TTL 원복 (60초 → 300초)

□ Step 6: DR 사이트 Standby 모드 복귀
  - 복제 방향 원복 (서울 → 대전)
  - DR 서비스 포트 비활성화
```

#### DR Drill 결과 보고서에 포함할 네트워크 지표

| 지표 | 목표 | 실제 |
|------|------|------|
| 장애 감지 시간 | 5분 이내 | __ 분 |
| 네트워크 전환 완료 시간 | 30분 이내 | __ 분 |
| DNS/GSLB 전파 완료 시간 | 10분 이내 | __ 분 |
| 전체 서비스 복구 시간 (RTO) | 4시간 이내 | __ 시간 __ 분 |
| 데이터 손실 (RPO) | 1시간 이내 | __ 분 |
| Failback 완료 시간 | 4시간 이내 | __ 시간 __ 분 |

매 Drill마다 이 지표를 측정하고, **RTO/RPO를 달성하지 못한 항목**이 있으면 Runbook을 개선합니다. "DR은 테스트하지 않으면 존재하지 않는 것과 같다"는 것이 핵심 원칙입니다.
