# 04-02. Security Policy와 Rule Base

---

## 면접관: "신규 고객사에서 Check Point 방화벽을 들여놓고 기본 보안 정책을 수립해달라고 합니다. 처음부터 어떻게 Rule Base를 구성하시겠어요?"

Check Point 보안 정책 수립은 **기본 Rule Base 프레임워크**를 먼저 잡고, 그 위에 업무 요건에 맞는 세부 룰을 추가하는 방식으로 진행합니다.

기본 Rule Base는 최소한 다음 구조로 구성합니다.

| 순번 | Rule 이름 | Source | Destination | Service | Action | Track |
|------|-----------|--------|-------------|---------|--------|-------|
| 1 | **Admin Access** | 관리자 PC | Management / Gateway | ssh, https, SmartConsole | Accept | Log |
| 2 | **Stealth Rule** | Any | Gateway(방화벽 자체) | Any | Drop | Log |
| 3 | **내부→외부 허용** | Internal_Net | External | http, https, dns | Accept | Log |
| 4 | **DNS 허용** | Internal_Net | DNS_Server | dns | Accept | Log |
| 5 | **(업무별 세부 룰)** | ... | ... | ... | Accept | Log |
| N-1 | **Noise Rule** | Any | Any | NetBIOS, Broadcast 등 | Drop | None |
| N | **Cleanup Rule** | Any | Any | Any | Drop | Log |

> 토폴로지 이미지 추가 예정

### 핵심 설정 - SmartConsole에서 Rule Base 구성

```
# SmartConsole에서:
# 1. Security Policies → Policy → Standard Policy에서 Rule 추가

# Rule 추가: 상단 메뉴 Rules → Add Rule → Above / Below

# Stealth Rule 설정:
#   Source: Any
#   Destination: Gateway Cluster 또는 개별 Gateway 오브젝트
#   Service & Applications: Any
#   Action: Drop
#   Track: Log

# Cleanup Rule 설정 (반드시 Rule Base 최하단):
#   Source: Any
#   Destination: Any
#   Service & Applications: Any
#   Action: Drop
#   Track: Log

# 정책 설치:
#   Install Policy → 대상 Gateway 선택 → Install
```

**Stealth Rule**은 방화벽 자체를 향한 불필요한 접근을 차단하는 룰이고, **Cleanup Rule**은 위의 모든 룰에 매칭되지 않은 트래픽을 명시적으로 Drop하면서 로그를 남기는 룰입니다. Cleanup Rule이 없으면 Implicit Drop에 의해 차단되지만, 기본적으로 로그가 남지 않아 트러블슈팅이 어렵습니다.

---

## 면접관: "Rule Base에서 룰 처리 순서가 어떻게 됩니까? 패킷이 들어오면 어떤 순서로 매칭됩니까?"

Check Point의 패킷 처리 순서는 다음과 같습니다.

**1. Anti-Spoofing 검사**
인터페이스에 정의된 토폴로지 기반으로 Source IP 위조 여부를 먼저 검사합니다.

**2. Rule Base 매칭 (Top-Down)**
Rule Base의 **위에서 아래로 순서대로** 매칭합니다. 첫 번째로 매칭되는 룰의 Action을 적용하고, 더 이상 아래 룰을 검사하지 않습니다.

**3. Implicit Rules (암묵적 룰)**
Rule Base에 명시하지 않아도 Check Point이 기본으로 적용하는 룰들이 있습니다. 이들은 Rule Base의 맨 위(Before), 맨 아래(After), 또는 마지막(Last)에 위치합니다.

```
# SmartConsole에서 Implicit Rules 확인:
# Menu → Global Properties → Firewall → Implied Rules

# 주요 Implicit Rules:
# - Accept Control Connections (Management-Gateway 간 SIC 통신)  → Before Last
# - Accept SmartConsole connections                              → Before Last
# - Accept ICMP requests (옵션)                                  → Before Last
# - Drop All (Default Drop)                                      → Last
```

**핵심 포인트:**
- Stealth Rule보다 Admin Access Rule이 위에 있어야 관리자가 방화벽에 접근 가능
- Cleanup Rule은 Implicit Drop보다 먼저 매칭되므로 로그를 남길 수 있음
- Rule 수가 많아질수록 상위 룰에서 매칭되는 트래픽이 많도록 배치하는 것이 성능에 유리

---

## 면접관: "그런데 특정 트래픽이 의도한 룰이 아니라 다른 룰에 매칭되어 차단되고 있습니다. 어떻게 확인하시겠어요?"

이것은 실무에서 가장 많이 발생하는 정책 문제입니다. **어떤 룰에 매칭되었는지** 확인하는 것이 핵심입니다.

### 1단계: SmartView Tracker(Log)에서 확인

```
# SmartConsole → Logs & Monitor 탭

# 해당 트래픽의 로그 검색:
# Source / Destination / Service 기준으로 필터링

# 로그 상세 정보에서 확인할 항목:
# - Rule Number: 몇 번째 룰에 매칭되었는지
# - Rule Name: 어떤 룰인지
# - Action: Accept / Drop / Reject
# - Policy Name: 어떤 정책에 의한 것인지
```

### 2단계: fw monitor로 패킷 흐름 추적

```
# Gateway Expert Mode에서
fw monitor -e "accept src=10.10.10.100 and dst=8.8.8.8;"

# 출력에서 확인할 체인 포인트:
# i (pre-inbound)  → I (post-inbound)  → o (pre-outbound)  → O (post-outbound)
# i만 보이고 I가 안 보이면 → Inbound에서 Drop된 것 (Security Policy 차단)
# I까지 보이고 o가 안 보이면 → 라우팅 문제 가능성
```

### 3단계: Rule Hit Count 확인

```
# SmartConsole에서:
# Policy → 각 Rule 우측에 Hit Count 컬럼 확인
# Hit Count가 높은 룰이 의도치 않게 해당 트래픽을 먹고 있을 수 있음

# Hit Count 초기화:
# Rule 우클릭 → Reset Hit Count
# → 초기화 후 해당 트래픽 재발생시켜서 어디에 카운트 되는지 확인
```

### 4단계: Rule Base 순서 점검

```
# 흔한 실수 패턴:
# Rule 5: Any → Any → HTTP → Drop    (광범위한 차단)
# Rule 8: ServerA → Any → HTTP → Accept  (이 룰은 절대 매칭 안 됨)
# → Rule 5가 먼저 매칭되어 ServerA 트래픽도 차단

# 해결: 더 구체적인(Specific) 룰을 위에, 광범위한(General) 룰을 아래에 배치
```

---

## 면접관: "Implicit Rules에 대해 좀 더 자세히 설명해주세요. 실무에서 이걸 건드려야 할 때가 있습니까?"

Implicit Rules는 Check Point이 자체 관리 통신을 위해 기본 적용하는 숨겨진 룰입니다. SmartConsole의 **Global Properties → Firewall** 에서 확인하고 제어할 수 있습니다.

### 주요 Implicit Rules

| Implicit Rule | 기본 상태 | 위치 | 설명 |
|---------------|-----------|------|------|
| Accept Control Connections | Enabled | Before Last | Management ↔ Gateway SIC 통신 허용 |
| Accept Remote Access Control Connections | Enabled | Before Last | VPN RA 접속 허용 |
| Accept SmartConsole Connections | Enabled | Before Last | SmartConsole → Management 접속 |
| Accept ICMP Requests | Disabled | Before Last | ICMP Echo 허용 |
| Accept outgoing packets from Gateway | Disabled | Before Last | Gateway 자체 발신 패킷 허용 |
| Log Implied Rules | Disabled | - | Implicit Rule 로그 기록 여부 |

**실무에서 건드려야 하는 경우:**

1. **보안 감사에서 Implicit Rule 허용이 문제될 때:**
   "Accept Control Connections"을 끄고, Rule Base에서 SIC 포트(TCP 18191, 18210 등)를 명시적으로 허용하는 룰을 추가합니다. 더 세밀한 제어가 가능하지만, 실수하면 Management-Gateway 통신이 끊길 수 있어 위험합니다.

2. **ICMP 관련:**
   기본적으로 ICMP가 차단되어 있어 ping이 안 됩니다. "Accept ICMP Requests"를 켜면 전체 허용이 되므로, 보통은 Rule Base에서 필요한 구간만 ICMP를 허용하는 것이 좋습니다.

3. **Gateway 자체 발신 트래픽:**
   Gateway에서 NTP, DNS, Syslog 등 외부 서비스에 접근해야 할 때, Implicit Rule로 전체 허용보다는 Rule Base에서 명시적으로 허용하는 것을 권장합니다.

```
# Implicit Rule 로그 활성화 (트러블슈팅 시 유용)
# SmartConsole → Global Properties → Firewall
# → "Log Implied Rules" 체크

# Implicit Rule을 Rule Base에서 보기:
# SmartConsole → Security Policies → 상단 View 메뉴
# → "Implied Rules" 체크하면 Rule Base에 Implicit Rule이 표시됨
```

---

## 면접관: "고객사에서 부서별로 보안 정책을 분리해서 관리하고 싶다고 합니다. Policy Layer와 Inline Layer 개념을 설명하고, 어떻게 구성하시겠어요?"

Check Point R80 이후 도입된 **Policy Layer** 와 **Inline Layer** 는 복잡한 정책을 체계적으로 관리하기 위한 기능입니다.

### Policy Layer (Ordered Layer)

Policy Layer는 **독립적인 Rule Base**를 여러 겹으로 쌓는 것입니다. 패킷은 **모든 Layer를 순서대로 통과**해야 하며, 모든 Layer에서 Accept되어야 최종 Allow됩니다.

```
[Layer 1: Network Security]     → Accept
         ↓
[Layer 2: Application Control]  → Accept
         ↓
[최종 결과: Allow]

# 하나의 Layer라도 Drop이면 → 최종 Drop
```

### Inline Layer

Inline Layer는 특정 룰의 **하위 룰셋**으로, 해당 룰에 매칭된 트래픽만 Inline Layer 내부에서 추가 검사합니다. 하나의 룰을 펼치면 안에 세부 Rule Base가 있는 구조입니다.

```
# 예시: 부서별 정책 분리

Rule 1: Admin Access → Accept
Rule 2: Stealth Rule → Drop
Rule 3: 개발팀 (Source: Dev_Net) → [Inline Layer: 개발팀 세부 정책]
    ├─ 3.1: Dev_Net → GitHub → HTTPS → Accept
    ├─ 3.2: Dev_Net → Internal_Server → SSH → Accept
    └─ 3.3: Dev_Net → Any → Any → Drop
Rule 4: 영업팀 (Source: Sales_Net) → [Inline Layer: 영업팀 세부 정책]
    ├─ 4.1: Sales_Net → CRM_Server → HTTPS → Accept
    ├─ 4.2: Sales_Net → Internet → HTTP/HTTPS → Accept
    └─ 4.3: Sales_Net → Any → Any → Drop
Rule 5: Cleanup → Drop
```

### SmartConsole 구성 방법

```
# Policy Layer 추가:
# Security Policies → Policy 우클릭 → Add Layer Above / Below
# 또는 Policy 탭 상단 → "+" 버튼으로 Layer 추가

# Inline Layer 추가:
# 특정 Rule의 Action 셀 클릭 → "Inline Layer" 선택
# → 해당 Rule 내부에 새로운 Sub-Rule Base가 생성됨
# → 하위 룰 추가/편집

# 권한 분리 (Multi-Admin 환경):
# SmartConsole → Manage & Settings → Permissions & Administrators
# → 각 Layer에 대한 Read/Write/Publish 권한을 관리자별로 분리 가능
```

**실무 활용 시나리오:**
- **Ordered Layer**: 네트워크 보안팀이 L3/L4 정책 관리, 보안관제팀이 Application Control 정책 관리 → 역할 분리
- **Inline Layer**: 부서별 정책을 깔끔하게 분리하면서도 하나의 Policy Package로 관리

---

## 면접관: "정책을 Install 했는데 'Policy installation failed'라고 뜹니다. 원인과 해결 방법을 알려주세요."

Policy Install 실패는 여러 원인이 있으며, 체계적으로 점검합니다.

### 1단계: 에러 메시지 확인

```
# SmartConsole에서 Install Policy 실패 시 표시되는 에러 메시지를 먼저 확인
# 하단 Validation 창에 구체적 오류가 표시됨

# 주요 에러 유형:
# - "SIC is not established" → SIC 문제
# - "Policy verification failed" → 룰 자체에 오류
# - "Connection to gateway failed" → 네트워크/Management-Gateway 통신 문제
# - "Object referenced in rule is invalid" → 오브젝트 설정 오류
```

### 2단계: Management-Gateway 통신 확인

```
# Management Server에서
cpstat mg                          # Management 상태
cpwd_admin list                    # 데몬 상태

# Gateway에서
cpstat fw                          # Firewall 상태
cpstat os -f ifconfig              # 인터페이스 상태

# SIC 상태 확인
cpconfig
# → SIC 상태 확인 (Initialized / Trust established)
```

### 3단계: Policy Verification

```
# SmartConsole에서:
# Policy → Verify Policy (Install 전 사전 검증)
# 오류/경고 항목 확인 후 수정

# 흔한 Verification 실패 원인:
# - 삭제된 오브젝트를 참조하는 룰이 있음
# - Network Object에 IP가 비어있음
# - Gateway의 Topology가 미설정됨
# - Anti-Spoofing 설정과 룰이 충돌
```

### 4단계: Gateway 측 로그 확인

```
# Gateway Expert Mode에서
tail -f $FWDIR/log/install_policy.elg
# Policy Install 시 실시간 로그 확인

tail -f /var/log/messages
# OS 레벨 로그

# Management Server Expert Mode에서
tail -f $FWDIR/log/fwm.elg
# Management 데몬 로그
```

### 5단계: 강제 정책 설치 (최후 수단)

```
# Management Server Expert Mode에서
fwm load $FWDIR/conf/Standard_Policy.W <gateway_object_name>

# 정책 언로드 (긴급 상황 - 모든 트래픽 허용 상태가 됨!)
fw unloadlocal      # Gateway에서 로컬 정책 제거 (주의!)

# 기본 정책 로드 (초기 정책)
fw fetch localhost   # Gateway에서 Management로부터 정책 재다운로드
```

**실무 팁:**
- Policy Install 전에 항상 **Verify**를 먼저 실행하는 습관
- 대규모 변경 시 SmartConsole에서 **Session** 기능을 활용하여, 문제 발생 시 **Discard** 가능
- Publish하기 전에는 다른 관리자에게 영향을 주지 않으므로, 충분히 검증 후 Publish → Install
