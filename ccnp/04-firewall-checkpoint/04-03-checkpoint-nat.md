# 04-03. Checkpoint NAT (Hide NAT / Static NAT)

---

### 면접관: "고객사에서 내부 사용자들은 인터넷을 나가야 하고, DMZ에 있는 웹서버는 외부에서 접근 가능하게 해달라고 합니다. NAT 설계를 어떻게 하시겠어요?"

두 가지 요건이 있으므로 **Hide NAT**와 **Static NAT**를 함께 설계합니다.

**요건 1: 내부 사용자 → 인터넷 (다수 대 1)**
- **Hide NAT** 적용: 내부 네트워크(예: 10.10.10.0/24) 전체가 Gateway의 외부 인터페이스 IP 또는 별도 Public IP 하나로 변환되어 나갑니다.
- 여러 내부 IP가 하나의 IP 뒤에 숨는 구조 (PAT/NAPT와 동일 개념)

**요건 2: 외부 → DMZ 웹서버 (1대1)**
- **Static NAT** 적용: DMZ 웹서버(예: 172.16.1.10)에 고정 Public IP(예: 203.0.113.10)를 1:1 매핑합니다.
- 외부에서 203.0.113.10으로 접근하면 172.16.1.10으로 변환됩니다.

> 토폴로지 이미지 추가 예정

#### 핵심 설정 - SmartConsole에서 Automatic NAT 구성

```
# === Hide NAT (내부 → 인터넷) ===

# 1. Network Object 생성 또는 수정
#    Objects → New Network → Internal_Net (10.10.10.0/24)

# 2. 해당 Network Object 더블클릭 → NAT 탭
#    - "Add Automatic Address Translation rules" 체크
#    - Translation Method: "Hide"
#    - Hide behind: "Gateway" (Gateway 외부 IP 사용)
#      또는 "IP Address" → 별도 NAT용 Public IP 입력
#    → OK

# === Static NAT (외부 → DMZ 웹서버) ===

# 1. Host Object 생성: DMZ_WebServer (172.16.1.10)

# 2. 해당 Host Object 더블클릭 → NAT 탭
#    - "Add Automatic Address Translation rules" 체크
#    - Translation Method: "Static"
#    - Translate to IP Address: 203.0.113.10
#    → OK

# === Security Policy에서 허용 룰 추가 ===

# Rule: 내부 → 인터넷
#   Source: Internal_Net
#   Destination: Any (또는 External Zone)
#   Service: http, https, dns
#   Action: Accept

# Rule: 외부 → DMZ 웹서버
#   Source: Any (또는 External)
#   Destination: DMZ_WebServer (원본 IP 172.16.1.10 오브젝트 사용!)
#   Service: http, https
#   Action: Accept
#   ※ 중요: NAT 전 IP(Original)로 룰을 작성!

# 정책 설치:
# Install Policy → Gateway 선택 → Install
```

**Check Point NAT의 핵심 원칙:**
Security Policy(Rule Base)는 항상 **NAT 전의 Original IP** 기준으로 작성합니다. Check Point은 정책 매칭을 먼저 하고, 그 후에 NAT를 적용하기 때문입니다.

---

### 면접관: "Automatic NAT와 Manual NAT의 차이가 뭡니까? 우선순위는 어떻게 됩니까?"

Check Point NAT에는 두 가지 설정 방식이 있습니다.

#### Automatic NAT
- **Object의 NAT 탭**에서 설정
- 오브젝트 자체에 NAT 규칙이 내장됨
- 간편하지만, 복잡한 조건(특정 Destination에 대해서만 NAT 등)은 불가

#### Manual NAT
- **NAT Rule Base**에서 직접 룰을 작성
- Source, Destination, Service를 세밀하게 지정 가능
- 조건부 NAT가 가능 (예: 특정 목적지로 갈 때만 다른 IP로 NAT)

#### NAT 우선순위 (Rule Base 처리 순서)

```
# SmartConsole → Security Policies → NAT 탭에서 전체 NAT Rule Base 확인

# NAT Rule Base 구성 순서:
# ┌──────────────────────────────────────────┐
# │  1. Manual NAT Rules (상단)               │  ← 최우선 (관리자가 직접 배치)
# │     - 사용자가 수동으로 추가한 룰          │
# ├──────────────────────────────────────────┤
# │  2. Automatic/Static NAT Rules            │  ← 두 번째
# │     - Object NAT 탭에서 Static으로 설정   │
# ├──────────────────────────────────────────┤
# │  3. Automatic/Hide NAT Rules              │  ← 세 번째
# │     - Object NAT 탭에서 Hide로 설정       │
# ├──────────────────────────────────────────┤
# │  4. Manual NAT Rules (하단)               │  ← 마지막
# │     - "Below" 섹션에 수동 추가한 룰       │
# └──────────────────────────────────────────┘
```

**우선순위 요약:**
1. **Manual (Top)** → 가장 먼저 매칭
2. **Automatic Static** → Host 단위 1:1 매핑
3. **Automatic Hide** → Network/Range 단위 다:1 매핑
4. **Manual (Bottom)** → 예외 처리용

**실무에서 Manual NAT가 필요한 경우:**
- 특정 목적지에 대해서만 다른 IP로 NAT해야 할 때
- Port Translation이 필요할 때 (예: 외부 8080 → 내부 80)
- 동일 서버가 서비스별로 다른 Public IP를 사용해야 할 때

```
# Manual NAT Rule 예시: Port Translation
# 외부에서 203.0.113.10:8080 → 내부 172.16.1.10:80

# NAT 탭에서 Manual Rule 추가:
# Original:
#   Source: Any
#   Destination: NAT_Public_IP (203.0.113.10)
#   Service: HTTP_8080 (TCP 8080 - Custom Service 생성)
# Translated:
#   Source: Original (변경 없음)
#   Destination: DMZ_WebServer (172.16.1.10)
#   Service: http (TCP 80)
```

---

### 면접관: "Static NAT를 설정했는데 외부에서 DMZ 웹서버로 통신이 안 됩니다. 어디부터 확인하시겠어요?"

Static NAT 후 통신 불가는 실무에서 자주 발생하는 문제입니다. 순서대로 점검합니다.

#### 1단계: Security Policy 확인

```
# 가장 흔한 실수: Security Rule에서 NAT된 IP로 룰을 작성한 경우
# Check Point은 Original IP 기준이므로:

# 틀린 예:
#   Destination: 203.0.113.10 (Translated IP) → 매칭 안 됨!

# 맞는 예:
#   Destination: DMZ_WebServer (172.16.1.10, Original IP) → 정상 매칭

# SmartConsole → Logs & Monitor에서 해당 트래픽 Drop 로그 확인
```

#### 2단계: ARP Proxy 문제 확인

**이것이 가장 흔한 원인입니다.** Static NAT에서 사용하는 Public IP(203.0.113.10)에 대해 Gateway의 외부 인터페이스가 **Proxy ARP**로 응답해야 외부 라우터가 해당 IP로 패킷을 보낼 수 있습니다.

```
# Gateway Expert Mode에서 Proxy ARP 확인
fw ctl arp

# 정상 출력 예시:
# 203.0.113.10 is on eth0 (Gateway 외부 인터페이스)

# Proxy ARP가 없으면 → NAT Public IP에 대해 ARP 응답 불가 → 통신 불가

# 수동 Proxy ARP 추가 (Expert Mode)
# $FWDIR/conf/local.arp 파일 편집:
vi $FWDIR/conf/local.arp
# 추가:
# 203.0.113.10 00:1c:7f:aa:bb:cc eth0
# (Gateway 외부 인터페이스 MAC, 인터페이스명)

# ARP 테이블 갱신
fw ctl arp -n
```

**Check Point의 Automatic ARP 설정:**

```
# SmartConsole → Global Properties → NAT
# → "Automatic ARP Configuration" 체크 확인
# 이 옵션이 활성화되어 있으면 Automatic NAT 시 자동으로 Proxy ARP 설정됨

# Manual NAT의 경우 Automatic ARP가 적용되지 않으므로
# 수동으로 local.arp 파일에 추가하거나,
# NAT 오브젝트에서 "Translate on Behalf of" 설정 필요
```

#### 3단계: 라우팅 확인

```
# 외부 라우터에서 NAT Public IP 대역이 Gateway를 향하는 라우팅이 있는지 확인
# Gateway에서:
show route            # Clish
netstat -rn           # Expert Mode

# DMZ 방향 라우팅 확인:
# Gateway가 172.16.1.10 (DMZ)으로 가는 경로가 있는지 확인
```

#### 4단계: fw monitor로 패킷 추적

```
# Gateway Expert Mode
fw monitor -e "accept dst=172.16.1.10 and dport=80;"

# 정상이면 4개 체인포인트 모두 보여야 함:
# i (pre-inbound) → I (post-inbound) → o (pre-outbound) → O (post-outbound)
# i에서 dst가 203.0.113.10이고, I에서 172.16.1.10으로 변환되면 NAT 동작 정상
```

---

### 면접관: "Anti-Spoofing과 NAT가 어떤 관계가 있습니까? 실무에서 충돌하는 경우가 있나요?"

**Anti-Spoofing**은 각 인터페이스에 정의된 토폴로지(이 인터페이스 뒤에 어떤 네트워크가 있는가) 기반으로, 해당 인터페이스로 들어오는 패킷의 Source IP가 정상인지 검증합니다.

#### NAT와 Anti-Spoofing 충돌 시나리오

```
# 시나리오:
# Gateway 외부 인터페이스(eth0): 203.0.113.1/24
# 내부 인터페이스(eth1): 10.10.10.1/24
# DMZ 인터페이스(eth2): 172.16.1.1/24

# Hide NAT: 내부 10.10.10.0/24 → Gateway 외부 IP(203.0.113.1)로 변환

# 문제 상황:
# 내부 → 외부 트래픽이 Hide NAT로 203.0.113.1로 변환됨
# 응답 패킷이 돌아올 때: src=외부서버, dst=203.0.113.1
# 이 패킷이 eth0(외부)로 들어오는데, dst가 Gateway 자신의 IP이므로 정상

# BUT, 만약 Hide behind IP가 203.0.113.50 (별도 IP)이고,
# 이 IP가 eth0의 Topology에 정의되어 있지 않으면?
# → Anti-Spoofing 위반은 아님 (Source 검사이므로)
# BUT, 라우팅/ARP 문제 발생 가능
```

#### 실제 충돌 사례

```
# 사례: VPN 환경에서 NAT + Anti-Spoofing 충돌
# VPN Tunnel을 통해 들어오는 패킷의 Source가 원격지 사설 IP
# Gateway의 Internal 인터페이스 Anti-Spoofing에서 차단될 수 있음

# 해결: Anti-Spoofing 설정에서 VPN 원격 네트워크를 예외 처리
# SmartConsole → Gateway Object → Network Management → eth1
# → Topology → Anti-Spoofing → "Don't check packets from" 에 VPN 네트워크 추가
```

#### Anti-Spoofing 설정 및 확인

```
# SmartConsole에서:
# Gateway Object 더블클릭 → Network Management
# → 각 인터페이스 클릭 → Topology
#   - Internal (Leads to internal network): 내부 네트워크 정의
#   - External (Leads to internet): 외부
# → Anti-Spoofing
#   - Perform Anti-Spoofing based on interface topology
#   - Action: Prevent (차단) 또는 Detect (로그만)
#   - Spoof Tracking: Log

# Expert Mode에서 Anti-Spoofing 로그 확인
fw log -t | grep "spoofed"

# Anti-Spoofing 장애 시 임시 비활성화 (트러블슈팅용)
# SmartConsole → 해당 인터페이스 → Anti-Spoofing → Action: Detect로 변경
# → Install Policy
```

---

### 면접관: "고객사에서 내부 서버끼리 통신할 때도 NAT가 적용되는 것 같다고 합니다. 내부 간 통신에서 NAT를 제외하려면 어떻게 해야 합니까?"

이것은 **NAT Exclusion (No-NAT Rule)** 문제입니다. Hide NAT를 Network Object 전체(10.10.10.0/24)에 적용하면, 내부 → 내부 통신에도 NAT가 걸릴 수 있습니다.

#### 해결 방법 1: Manual NAT로 No-NAT Rule 추가 (가장 정석)

```
# NAT Rule Base 상단(Manual Top)에 No-NAT Rule 추가:

# Original:
#   Source: Internal_Net (10.10.10.0/24)
#   Destination: Internal_Net (10.10.10.0/24)   ← 내부끼리
#   Service: Any
# Translated:
#   Source: = Original    (변환 없음)
#   Destination: = Original (변환 없음)
#   Service: = Original   (변환 없음)

# 같은 방식으로 Internal ↔ DMZ 간 No-NAT도 추가:
# Original:
#   Source: Internal_Net
#   Destination: DMZ_Net
#   Service: Any
# Translated: 모두 Original
```

#### 해결 방법 2: Automatic NAT 설정 시 옵션 활용

```
# Network Object (Internal_Net) → NAT 탭
# Hide NAT 설정 시:
# "Hide behind Gateway" 선택한 경우:
# → Check Point이 자동으로 내부→내부는 NAT 제외 (대부분의 경우)

# 그래도 문제가 발생하면 Manual No-NAT Rule로 명시적 제외 권장
```

#### 해결 방법 3: VPN 환경에서의 NAT Exclusion

```
# VPN 터널 트래픽에 NAT가 적용되면 VPN이 깨짐
# VPN Community 설정에서 자동으로 No-NAT가 적용되지만,
# 수동으로 확인/추가가 필요한 경우:

# Manual NAT (Top):
# Original:
#   Source: Local_Encryption_Domain
#   Destination: Remote_Encryption_Domain
#   Service: Any
# Translated: 모두 Original

# 역방향도 추가:
# Original:
#   Source: Remote_Encryption_Domain
#   Destination: Local_Encryption_Domain
#   Service: Any
# Translated: 모두 Original
```

---

### 면접관: "NAT Rule Base 전체를 정리해서 한눈에 보여줄 수 있는 방법이 있습니까? 그리고 NAT 트러블슈팅 명령어를 정리해주세요."

#### NAT Rule Base 전체 확인

```
# SmartConsole → Security Policies → NAT 탭
# 전체 NAT Rule이 우선순위 순서대로 표시됨:
#   [Manual Top Rules]
#   [Automatic Static NAT Rules]
#   [Automatic Hide NAT Rules]
#   [Manual Bottom Rules]

# 각 Rule에서 확인할 컬럼:
# - Original Source / Destination / Service
# - Translated Source / Destination / Service
# - Install On (어떤 Gateway에 적용)
# - Comment (관리 메모)
```

#### NAT 트러블슈팅 명령어 모음

```
# 1. 현재 활성 NAT Rule 확인 (Gateway Expert Mode)
fw tab -t fwx_alloc -s          # NAT 세션 테이블 통계
fw tab -t fwx_alloc -u          # NAT 세션 상세 (대량 출력 주의)

# 2. 특정 연결의 NAT 변환 확인
fw ctl conntab                   # Connection Table 전체
fw tab -t connections -f         # 연결 테이블에서 NAT 매핑 확인

# 3. fw monitor로 NAT 전후 확인
fw monitor -e "accept src=10.10.10.100;"
# i: src=10.10.10.100 (Original)
# O: src=203.0.113.1  (NAT 변환 후)

# 4. Hide NAT Port 할당 확인
fw tab -t fwx_alloc -f | grep 10.10.10.100

# 5. Proxy ARP 테이블
fw ctl arp                       # 현재 ARP Proxy 목록
arp -an                          # OS 레벨 ARP 테이블

# 6. NAT 커널 디버그 (심층 분석)
fw ctl debug 0
fw ctl debug -buf 32768
fw ctl debug -m fw + xlate       # NAT 변환 디버그 활성화
fw ctl kdebug -T -f > /var/log/nat_debug.txt
# 재현 후 디버그 중지:
fw ctl debug 0

# 7. cpinfo로 전체 설정 수집 (TAC 케이스 오픈 시)
cpinfo -y all
```

**NAT 트러블슈팅 체크리스트 요약:**
1. Security Policy에서 Original IP로 룰을 작성했는가?
2. NAT Rule Base에서 매칭 순서가 올바른가? (Manual Top이 우선)
3. Static NAT 시 Proxy ARP가 설정되어 있는가?
4. 라우팅이 NAT IP에 대해 올바른가?
5. Anti-Spoofing이 NAT 트래픽을 차단하고 있지 않은가?
6. 내부 간 통신에 불필요한 NAT가 적용되고 있지 않은가? (No-NAT Rule)
