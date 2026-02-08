# 09-02. DMZ 설계

---

## 면접관: "고객사가 자체 웹서비스를 외부에 공개해야 합니다. 웹서버, API 서버, DB 서버가 있고, 일반 사용자가 인터넷에서 접속합니다. 내부 업무망과 분리하면서 서비스를 공개하려면 어떻게 설계해야 하나요?"

**DMZ(Demilitarized Zone)**를 별도로 구성하여, 외부에 공개할 서버를 내부 업무망과 분리된 네트워크 세그먼트에 배치하겠습니다.

### 설계 옵션: Single Firewall vs Dual Firewall

#### Option A: Single Firewall (3-Leg)

```
              Internet
                 |
            [Router/ISP]
                 |
          +-----------+
          |  Firewall  |
          +-----------+
         /      |       \
    Outside   DMZ     Inside
   (Untrust) (DMZ)   (Trust)
```

- 방화벽 하나에 인터페이스 3개: Outside, DMZ, Inside
- **장점**: 비용 절감, 관리 단순
- **단점**: 방화벽 단일 장애점(SPOF), 방화벽 자체가 침투되면 모든 존 노출

```
! ASA Single FW 3-Leg 구성 예시
interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 203.0.113.1 255.255.255.248
!
interface GigabitEthernet0/1
 nameif dmz
 security-level 50
 ip address 172.16.1.1 255.255.255.0
!
interface GigabitEthernet0/2
 nameif inside
 security-level 100
 ip address 10.1.1.1 255.255.255.0
```

#### Option B: Dual Firewall (Sandwich)

```
              Internet
                 |
            [Router/ISP]
                 |
          +-----------+
          | FW-Outer  |  (Vendor A)
          +-----------+
                |
            [ DMZ ]
                |
          +-----------+
          | FW-Inner  |  (Vendor B)
          +-----------+
                |
           [Inside LAN]
```

- **외부 FW**: Internet ↔ DMZ 트래픽 제어
- **내부 FW**: DMZ ↔ Inside 트래픽 제어
- **장점**: 심층 방어(Defense in Depth), 벤더 다양화로 동일 취약점 동시 피격 방지
- **단점**: 비용 2배, 운영 복잡도 증가

### 권장: 이 고객에게는?

- 웹서비스 규모와 보안 요구 수준에 따라 결정합니다.
- **일반 기업**: Single FW (HA 이중화) + 3-Leg → 비용 효율적
- **금융/공공기관**: Dual FW → 규제 요건 충족, 감사 대응

### 서버 배치

| 존 | 서버 | 이유 |
|----|------|------|
| DMZ | 웹서버, Reverse Proxy | 외부 직접 접근 필요 |
| DMZ | API 서버 (Front-end) | 외부 요청 수신 |
| Inside | DB 서버 | 절대 외부 직접 접근 불가 |
| Inside | API 서버 (Back-end) | 내부 로직 처리 |

> 토폴로지 이미지 추가 예정

---

## 면접관: "기본적인 질문인데, 왜 DMZ를 따로 두나요? 방화벽 룰로 특정 포트만 열면 내부망에 서버를 둬도 되지 않나요?"

핵심은 **침투 후 횡이동(Lateral Movement) 차단**입니다.

### 내부망에 웹서버를 두면 발생하는 문제

1. **외부에서 내부망으로 직접 경로가 생김**: 방화벽에서 TCP 443만 열어도, 해당 서버가 뚫리면 공격자는 **내부망 네트워크에 발판(foothold)**을 확보합니다.
2. **횡이동 가능**: 내부망의 다른 서버(DB, 파일서버, AD)와 같은 서브넷에 있으므로, 방화벽을 거치지 않고 직접 접근 가능합니다.
3. **방화벽 룰의 한계**: 인바운드는 443만 열었더라도, 웹서버에서 내부 다른 서버로의 **아웃바운드 통신은 제어하기 어렵습니다** (같은 서브넷이면 방화벽을 경유하지 않으니까).

### DMZ를 두면?

- 웹서버가 DMZ에 있으므로, 웹서버가 뚫려도 공격자는 **DMZ 안에 갇힘**
- DMZ → Inside 통신은 **내부 FW(또는 FW의 DMZ→Inside 룰)**을 반드시 거침
- DMZ → Inside 룰: "DMZ 웹서버 → 내부 DB서버의 TCP 3306만 허용" 같은 최소 권한 원칙 적용

```
! ASA: DMZ에서 Inside로 DB 접근만 허용
access-list DMZ-TO-INSIDE extended permit tcp host 172.16.1.10 host 10.1.1.50 eq 3306
access-list DMZ-TO-INSIDE extended deny ip any any log
!
access-group DMZ-TO-INSIDE in interface dmz
```

즉, DMZ는 **"뚫려도 괜찮은(expendable) 영역"이 아니라, "뚫려도 내부망을 보호할 수 있는 완충 지대"**입니다.

---

## 면접관: "좋습니다. 그러면 실제로 DMZ에 있는 웹서버가 해킹당했다고 가정합시다. 공격자가 웹서버 셸을 탈취했어요. 이때 내부망 DB 서버는 어떻게 보호되나요?"

### 방어 계층 분석

**공격자 현재 위치**: DMZ 웹서버 (172.16.1.10) 셸 확보

#### 1차 방어: 방화벽 룰 (Network Level)

```
DMZ → Inside 허용 룰:
- 172.16.1.10 → 10.1.1.50:3306 (MySQL)  ← 이것만 허용
- 나머지 전부 차단 (deny all)
```

- 공격자가 내부망 스캔(10.1.1.0/24)을 시도해도, 방화벽에서 **전부 차단**
- DB 서버의 3306 포트로만 접근 가능 → **SQL Injection 등 애플리케이션 레벨 공격은 가능**

#### 2차 방어: DB 서버 자체 보안 (Host Level)

- DB 접근 계정: 최소 권한 (SELECT만 부여, FILE/LOAD 권한 제거)
- DB 서버 방화벽(iptables/firewalld): 172.16.1.10에서만 3306 접근 허용
- DB 서버에 SSH는 관리용 점프 서버(Bastion)에서만 허용

#### 3차 방어: 모니터링/탐지 (Detection Level)

- **IPS/IDS**: DMZ↔Inside 구간에 인라인 또는 미러링으로 배치
- 비정상 쿼리 패턴 탐지 (대량 SELECT, UNION-based injection 등)
- **SIEM**: 방화벽 로그 + DB 접근 로그 상관 분석

#### 4차 방어: 마이크로세그멘테이션

- DMZ 내부에서도 웹서버 ↔ API 서버 간 통신을 방화벽 룰로 제어
- **Zero Trust 원칙**: DMZ 안이라고 해서 서버 간 자유 통신을 허용하지 않음

### 핵심 포인트

DMZ 설계의 가치는 공격자가 웹서버를 탈취해도 **내부망 진입 경로가 극도로 제한**된다는 것입니다. 방화벽 룰 하나만이 아니라 다층 방어(Defense in Depth)로 보호합니다.

---

## 면접관: "Reverse Proxy와 WAF를 DMZ에 배치한다고 했는데, 어떤 역할이고 어디에 배치하나요?"

### Reverse Proxy

외부 사용자의 HTTP/HTTPS 요청을 **대신 받아서 내부(또는 DMZ) 웹서버로 전달**하는 프록시입니다.

```
[Client] → [FW] → [Reverse Proxy (DMZ)] → [Web Server (DMZ/Inside)]
```

**역할**:
- **서버 IP 은닉**: 클라이언트는 Reverse Proxy의 IP만 알고, 실제 웹서버 IP는 모름
- **SSL Offloading**: Reverse Proxy에서 TLS 종단 → 웹서버 부하 감소
- **캐싱 및 로드밸런싱**: 정적 콘텐츠 캐시, 다수 웹서버로 분산
- **대표 솔루션**: Nginx, HAProxy, F5 BIG-IP

### WAF (Web Application Firewall)

HTTP/HTTPS 트래픽을 L7 수준에서 검사하여 **웹 공격(SQL Injection, XSS, CSRF 등)을 탐지/차단**합니다.

**배치 위치**: Reverse Proxy 앞 또는 뒤 (보통 Reverse Proxy와 통합 또는 직전 배치)

```
[Client] → [FW] → [WAF] → [Reverse Proxy/LB] → [Web Server]
```

```
! 일반적인 트래픽 흐름 (DMZ 내부)
1. Client HTTPS 요청 → 외부 FW 통과 (TCP 443 허용)
2. WAF: SQL Injection/XSS 패턴 검사 → 정상이면 통과, 악성이면 차단
3. Reverse Proxy: SSL 종단 → 적절한 웹서버로 전달
4. 웹서버: 요청 처리 → DB 서버(Inside) 조회 필요 시 FW 통과
```

### WAF vs 일반 방화벽 차이

| 구분 | 일반 방화벽 (L3/L4) | WAF (L7) |
|------|---------------------|----------|
| 검사 대상 | IP, Port, Protocol | HTTP Header, URL, Body, Cookie |
| 차단 예시 | "80번 포트 차단" | "UNION SELECT 패턴 차단" |
| SQL Injection | 탐지 불가 | 탐지/차단 가능 |
| DDoS (HTTP Flood) | 제한적 | Layer 7 Rate Limiting 가능 |

---

## 면접관: "WAF나 로드밸런서를 배치할 때 1-Arm과 2-Arm 방식이 있잖아요. 차이점과 각각 언제 쓰는지 설명해주세요."

### 2-Arm (Inline / Bridge Mode)

```
[Client] ─── [WAF/LB] ─── [Server]
             In    Out
```

- 트래픽이 **반드시** WAF/LB를 통과 (인라인)
- 인바운드와 아웃바운드 **모두** WAF/LB를 경유
- **장점**: 모든 트래픽을 확실히 검사, 양방향 모니터링 가능
- **단점**: WAF/LB가 죽으면 서비스 중단 (SPOF) → Bypass 기능 또는 HA 필수
- **사용 시나리오**: 보안 요구가 높은 환경, 모든 트래픽 검사 필수

### 1-Arm (One-Arm / DSR Mode)

```
[Client] ──→ [WAF/LB] ──→ [Server]
                              │
[Client] ←────────────────────┘  (Direct Server Return)
```

- 클라이언트 요청은 WAF/LB를 거치지만, 서버 응답은 **WAF/LB를 거치지 않고 직접 반환**
- LB의 경우 DSR(Direct Server Return) 모드
- **장점**: LB 대역폭 부하 감소 (응답 트래픽이 안 거침, 보통 응답이 요청보다 10배 이상 큼)
- **단점**: 응답 트래픽 검사 불가, 세션 추적 제한
- **사용 시나리오**: 대용량 트래픽 처리, 응답 검사가 불필요한 경우

### 비교 요약

| 구분 | 1-Arm | 2-Arm |
|------|-------|-------|
| 트래픽 경로 | 요청만 경유 | 양방향 경유 |
| 대역폭 부하 | 낮음 | 높음 |
| 보안 검사 | 요청만 | 양방향 |
| 서버 설정 | DSR용 루프백 설정 필요 | 기본 게이트웨이만 변경 |
| HA 중요도 | 장애 시 일부 우회 가능 | 장애 시 서비스 중단 |

WAF는 요청/응답 모두 검사해야 하므로 보통 **2-Arm**, LB는 성능이 중요하면 **1-Arm(DSR)**을 선택합니다.

---

## 면접관: "잘 설명했습니다. 이제 고객사에서 '기존 웹서비스 외에 모바일 앱 API, 파트너 전용 B2B API도 공개해야 한다'고 합니다. DMZ를 어떻게 확장하시겠어요?"

### 멀티 DMZ (Multiple DMZ Zones) 설계

서비스 성격과 보안 등급이 다른 서버 그룹을 **별도 DMZ 존**으로 분리합니다.

```
                    Internet
                       |
                  [FW - Outside]
                  /     |     \
          [DMZ-Web] [DMZ-API] [DMZ-B2B]
                  \     |     /
                  [FW - Inside]
                       |
                  [Inside LAN]
```

### 존별 설계

| 존 | 용도 | 접근 주체 | 보안 수준 |
|----|------|-----------|-----------|
| DMZ-Web | 일반 사용자 웹서비스 | 불특정 다수 (인터넷) | 높음 |
| DMZ-API | 모바일 앱 API | 앱 사용자 (인터넷) | 높음 + API 인증 |
| DMZ-B2B | 파트너 전용 API | 특정 파트너 IP만 | 최고 (IP 화이트리스트) |

### 방화벽 룰 설계

```
! DMZ-Web: 인터넷 전체에서 443만 허용
access-list OUTSIDE-TO-DMZ-WEB permit tcp any host 203.0.113.10 eq 443

! DMZ-API: 인터넷 전체에서 443 허용 (API Gateway에서 인증 처리)
access-list OUTSIDE-TO-DMZ-API permit tcp any host 203.0.113.20 eq 443

! DMZ-B2B: 파트너 IP에서만 접근 허용
access-list OUTSIDE-TO-DMZ-B2B permit tcp host 198.51.100.0 255.255.255.0 host 203.0.113.30 eq 443

! 존 간 통신 차단 (DMZ-Web → DMZ-B2B 불가)
access-list DMZ-WEB-TO-DMZ-B2B deny ip 172.16.1.0 255.255.255.0 172.16.3.0 255.255.255.0
```

### 추가 고려사항

1. **API Gateway 배치**: DMZ-API 앞단에 API Gateway (Kong, AWS API Gateway 등) 배치
   - Rate Limiting, API Key 인증, OAuth 2.0 처리
2. **mTLS (Mutual TLS)**: B2B 파트너 통신에는 서버/클라이언트 양방향 인증서 검증
3. **존 간 격리**: 한 DMZ가 침투되어도 다른 DMZ로 횡이동 불가
4. **로깅**: 각 존별 독립 로그 수집 → SIEM에서 존별 이상 탐지

### 물리적 구현

방화벽이 충분한 인터페이스를 제공하면 VLAN/서브인터페이스로 존을 분리합니다.

```
! ASA: 서브인터페이스로 멀티 DMZ 구현
interface GigabitEthernet0/1.10
 vlan 10
 nameif dmz-web
 security-level 50
 ip address 172.16.1.1 255.255.255.0
!
interface GigabitEthernet0/1.20
 vlan 20
 nameif dmz-api
 security-level 50
 ip address 172.16.2.1 255.255.255.0
!
interface GigabitEthernet0/1.30
 vlan 30
 nameif dmz-b2b
 security-level 60
 ip address 172.16.3.1 255.255.255.0
```

B2B 존의 security-level을 60으로 높인 것은 파트너 통신이 일반 DMZ보다 더 신뢰할 수 있는 트래픽이기 때문입니다. 하지만 security-level만으로 보안을 판단하지 말고, **명시적 ACL이 항상 우선**합니다.
