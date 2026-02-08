# 01-02. OSPF 인증과 Stub Area

## 면접관: "금융권 고객사인데, 보안 감사에서 '라우팅 프로토콜 인증이 안 되어 있다'는 지적을 받았습니다. 현재 OSPF를 사용 중인데, 인증 체계를 설계해주세요."

금융권은 보안 컴플라이언스가 최우선이므로, OSPF 인증을 반드시 적용해야 합니다. OSPF는 인증 없이 운영하면, 동일 네트워크에 악의적인 라우터가 접속해서 잘못된 경로를 주입하거나 LSDB를 오염시킬 수 있습니다.

설계 방향:
- **전 구간 OSPF 인증 적용:** Area 단위 인증 또는 인터페이스 단위 인증
- **인증 방식:** MD5 인증 (최소), 가능하면 SHA-256 기반의 Key Chain 인증
- **Key Rotation 계획:** 키 유효기간 설정으로 정기적 키 교체 자동화
- **단계적 적용:** 인증 미적용 → Null 인증 → MD5 → SHA 순으로 마이그레이션

OSPF는 인증 불일치 시 Adjacency가 즉시 끊어지므로, 운영 중 적용 시에는 **양쪽 동시에** 변경하거나, 유지보수 윈도우에서 작업해야 합니다.

> 토폴로지 이미지 추가 예정

### 핵심 설정

**방법 1: 인터페이스 단위 MD5 인증 (전통 방식)**
```
interface GigabitEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 S3cur3K3y!
```

**방법 2: Area 단위 인증 (일괄 적용)**
```
router ospf 1
 area 0 authentication message-digest
!
interface GigabitEthernet0/0
 ip ospf message-digest-key 1 md5 S3cur3K3y!
!
interface GigabitEthernet0/1
 ip ospf message-digest-key 1 md5 S3cur3K3y!
```

**방법 3: Key Chain 기반 SHA-256 인증 (권장)**
```
key chain OSPF-KEY
 key 1
  key-string F1n@nc3-OSPF-2024!
  cryptographic-algorithm hmac-sha-256
  accept-lifetime 00:00:00 Jan 1 2024 00:00:00 Jul 1 2024
  send-lifetime 00:00:00 Jan 1 2024 00:00:00 Jul 1 2024
 key 2
  key-string F1n@nc3-OSPF-2024-H2!
  cryptographic-algorithm hmac-sha-256
  accept-lifetime 00:00:00 Jun 1 2024 infinite
  send-lifetime 00:00:00 Jul 1 2024 infinite
!
interface GigabitEthernet0/0
 ip ospf authentication key-chain OSPF-KEY
```

---

## 면접관: "MD5와 SHA-256 차이가 뭐고, 왜 SHA를 권장하는 거죠? 그리고 Key Chain에서 accept-lifetime과 send-lifetime을 다르게 설정한 이유는요?"

**MD5 vs SHA-256:**

| 항목 | MD5 | SHA-256 (HMAC) |
|------|-----|-----------------|
| 해시 길이 | 128-bit | 256-bit |
| 보안 수준 | 충돌 공격 취약점 알려짐 | 현재까지 안전 |
| 설정 방식 | 인터페이스 직접 설정 | Key Chain 연동 |
| 키 교체 | 수동 (양쪽 동시 변경 필요) | Key Chain으로 자동화 가능 |
| OSPF RFC | RFC 2328 (OSPFv2 기본) | RFC 5709 |

MD5는 해시 자체에 알려진 취약점이 있고, 키 교체 자동화가 안 됩니다. SHA-256은 Key Chain과 결합하면 `accept-lifetime`과 `send-lifetime`으로 키 교체를 무중단으로 할 수 있습니다.

**Lifetime 설정 전략:**

Key 1에서 Key 2로 교체할 때:
1. Key 2의 `accept-lifetime`을 Key 1의 `send-lifetime` 종료보다 **먼저** 시작 (6월 1일부터 수신 가능)
2. Key 2의 `send-lifetime`은 전환 시점(7월 1일)부터 시작
3. 이 오버랩 기간 동안 양쪽 키 모두 수신 가능 → 무중단 전환

이렇게 하면 네트워크 장비마다 시간 차이가 있어도 안전하게 키가 교체됩니다. 만약 accept와 send를 동일하게 설정하면, 한쪽이 새 키로 전환했을 때 다른 쪽이 아직 이전 키를 쓰면 Adjacency가 끊어집니다.

---

## 면접관: "인증을 적용했습니다. 그런데 고객사에서 '지사 쪽 라우터에서 외부 인터넷 경로가 너무 많이 보인다, 필요 없는 경로도 많다'는 요청이 왔어요. 이 금융 고객사 지사에 Stub Area를 적용하려고 합니다. Stub Area 종류별 차이를 설명하고, 어떤 것을 적용할지 판단해주세요."

금융 고객사 지사 환경 분석:
- 지사에서 인터넷 직접 접속 없음 (본사 프록시 경유)
- 지사에는 별도 외부 프로토콜 재분배 없음
- 지사 WAN은 단일 회선 (이중화 없음)

이 조건이면 **Totally Stubby Area**가 최적입니다.

**Stub Area 종류 비교:**

| 구분 | 허용 LSA | Default Route | ASBR 허용 |
|------|----------|---------------|-----------|
| **Normal Area** | 1, 2, 3, 4, 5 | 없음 | O |
| **Stub** | 1, 2, 3 | ABR이 Type 3으로 주입 | X |
| **Totally Stubby** | 1, 2 + Default만 | ABR이 Type 3으로 주입 | X |
| **NSSA** | 1, 2, 3, 7 | 수동 설정 필요 | O (Type 7) |
| **Totally NSSA** | 1, 2, 7 + Default만 | ABR이 Type 3으로 주입 | O (Type 7) |

**Totally Stubby 적용 시 이점:**
- 지사 라우터의 라우팅 테이블: Area 내부 경로 + Default Route만 존재
- LSDB 대폭 축소 → 메모리/CPU 절약
- SPF 재계산 범위 최소화

### 핵심 설정

```
! ABR (본사 측)
router ospf 1
 area 10 stub no-summary
 area 0 authentication message-digest
 area 10 authentication message-digest
!
! 지사 라우터
router ospf 1
 area 10 stub
```

---

## 면접관: "Totally Stubby로 설정했는데, 고객사에서 '지사에서 특정 외부 서버(파트너 은행 연동 서버)로 통신이 안 된다'는 장애 신고가 들어왔습니다. 어디부터 확인하시겠어요?"

**1단계: 지사 라우터에서 해당 목적지로의 경로 확인**
```
show ip route 203.0.113.50
show ip route 0.0.0.0
```

Totally Stubby Area이므로, 외부 목적지는 Default Route를 타고 ABR로 가야 합니다. Default Route가 없다면 ABR 설정 문제입니다.

**2단계: ABR에서 Default Route 주입 확인**
```
! ABR에서 확인
show ip ospf database summary 0.0.0.0
show ip ospf database database-summary
```

`no-summary` 옵션이 빠져서 Default Route가 주입되지 않았을 수 있습니다.

**3단계: ABR에서 해당 외부 경로 존재 확인**
```
! ABR에서
show ip route 203.0.113.50
show ip ospf database external | include 203.0.113
```

ABR 자체에 해당 외부 경로가 없으면, 더 상위(ASBR 또는 ISP 연동 구간)를 확인해야 합니다.

**4단계: 경로가 있는데 통신 안 되면 리턴 경로 확인**
```
! 파트너 은행 연동 구간 라우터에서
show ip route 10.10.0.0
traceroute 10.10.10.1
```

비대칭 라우팅 또는 방화벽 정책 누락일 수 있습니다. 금융권은 방화벽 규칙이 매우 엄격하므로, 새로 추가된 경로에 대한 방화벽 정책이 없을 가능성이 높습니다.

---

## 면접관: "파트너 은행과의 연동 때문에 지사에서 외부 경로를 재분배해야 하는 상황이 생겼어요. 그런데 이미 Totally Stubby로 설정되어 있잖아요. 어떻게 하죠?"

Stub/Totally Stubby Area에서는 ASBR이 존재할 수 없으므로, 외부 경로 재분배가 불가능합니다. 이 경우 **NSSA 또는 Totally NSSA**로 전환해야 합니다.

**Totally Stubby → Totally NSSA 전환:**

```
! 기존 설정 제거
router ospf 1
 no area 10 stub no-summary   ! ABR
!
! NSSA 적용
router ospf 1
 area 10 nssa no-summary       ! ABR - Totally NSSA
!
! 지사 라우터
router ospf 1
 no area 10 stub
 area 10 nssa
!
! 지사에서 파트너 은행 경로 재분배
router ospf 1
 redistribute static subnets route-map PARTNER-BANK
!
ip route 203.0.113.0 255.255.255.0 192.168.100.1
!
route-map PARTNER-BANK permit 10
 match ip address prefix-list BANK-ROUTES
 set metric 100
 set metric-type type-2
!
ip prefix-list BANK-ROUTES permit 203.0.113.0/24
```

이 설정의 동작:
1. 지사 ASBR이 203.0.113.0/24를 **Type 7 LSA**로 생성
2. Type 7은 NSSA 내부에서만 전파
3. ABR이 NSSA 경계에서 **Type 7 → Type 5 변환**
4. 변환된 Type 5 LSA가 Area 0과 나머지 Area로 전파
5. 외부에서 오는 Type 5 LSA는 여전히 차단 (NSSA 특성)

---

## 면접관: "Type 7에서 Type 5로 변환할 때, ABR이 여러 대 있으면 어떤 ABR이 변환을 담당하죠?"

ABR이 여러 대 있는 경우, **Router-ID가 가장 높은 ABR**이 Type 7 → Type 5 변환을 담당합니다. 이를 **NSSA Translator**라고 합니다.

동작 세부사항:
- 모든 ABR이 Type 7 LSA를 수신하지만, 실제 Type 5를 생성하는 것은 Translator 1대
- 나머지 ABR은 Type 7을 수신만 하고 변환하지 않음
- Translator ABR이 다운되면, 그 다음으로 높은 Router-ID를 가진 ABR이 역할을 이어받음

확인 명령어:
```
show ip ospf | include translat
show ip ospf database nssa-external
show ip ospf database external
```

변환 시 주의점:
- Type 7의 **Forward Address (FA)**가 Type 5에도 유지됨
- FA가 0.0.0.0이면 ABR 자신을 Next-hop으로 사용
- FA가 설정되어 있으면 해당 주소가 Area 0에서도 도달 가능해야 트래픽이 정상 전달됨

```
! Type 7 LSA의 Forward Address 확인
show ip ospf database nssa-external 203.0.113.0
  LS Type: AS NSSA External Link
  Forward Address: 192.168.100.1    ← 이 주소가 Area 0에서 도달 가능해야 함
```

---

## 면접관: "전체적으로 이 금융 고객사 OSPF 인증 + NSSA 설계를 적용할 때, 작업 순서와 주의사항을 정리해주세요."

**작업 순서 (유지보수 윈도우 내):**

**Phase 1: OSPF 인증 적용 (구간별 순차)**
1. Key Chain 사전 배포 (모든 라우터에 키 설정, 아직 인터페이스에 적용 안 함)
2. 가장 영향 적은 지사부터 한 구간씩 양쪽 동시에 인증 활성화
3. 각 구간 적용 후 Adjacency 확인: `show ip ospf neighbor`
4. 문제 시 즉시 롤백: `no ip ospf authentication key-chain`

```
! 적용 전 neighbor 상태 기록
show ip ospf neighbor
!
! 적용 후 확인 - FULL 상태인지
show ip ospf neighbor
show log | include OSPF
```

**Phase 2: Totally NSSA 전환 (Area 단위)**
1. 먼저 ABR에서 `area X nssa no-summary` 적용
2. 그 즉시 Area 내부 라우터에서 `area X nssa` 적용
3. **양쪽이 불일치하면 Adjacency가 끊어지므로 최대한 동시에 적용**
4. 적용 후 Default Route 수신 확인

```
! 전환 후 필수 확인사항
show ip ospf neighbor          ! Adjacency FULL 확인
show ip ospf database summary  ! Default Route(0.0.0.0) 확인
show ip route ospf             ! 경로 정상 확인
show ip ospf database nssa     ! Type 7 LSA 확인 (재분배 있는 경우)
```

**주의사항:**
- 인증과 Stub/NSSA 변경을 **동시에 하지 말 것** → 장애 시 원인 분리 어려움
- 변경 전 `show run`과 `show ip route ospf` 백업
- NTP 동기화 확인 (Key Chain lifetime 기반이므로 시간 정확성 필수)
- 금융권 특성상 **변경관리 승인, 롤백 계획, 고객 통보** 필수
