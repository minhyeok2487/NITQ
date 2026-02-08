# 04-06. Application Control과 URL Filtering

---

## 면접관: "고객사에서 사내 네트워크에서 토렌트, 유튜브 같은 스트리밍, 개인 클라우드 스토리지 사용을 차단해달라고 합니다. 어떻게 설계하시겠어요?"

단순 포트 기반 차단으로는 불가능합니다. 토렌트, 유튜브, Dropbox 등은 모두 **HTTP/HTTPS(TCP 80/443)** 을 사용하므로, 포트를 차단하면 정상 웹 서비스도 같이 차단됩니다. 이런 경우 Check Point의 **Application Control Blade**를 활성화하여 **애플리케이션 레벨(L7)** 에서 식별하고 차단해야 합니다.

### 설계 방향

| 차단 대상 | Application Control 카테고리/앱 | 비고 |
|-----------|-------------------------------|------|
| 토렌트 | BitTorrent, uTorrent, Vuze 등 → "P2P File Sharing" 카테고리 | 프로토콜 자체 식별 |
| 유튜브/스트리밍 | YouTube, Netflix, Twitch → "Streaming Media" 카테고리 | HTTPS Inspection 필요 |
| 개인 클라우드 | Dropbox, Google Drive(Personal) → "File Storage and Sharing" 카테고리 | 업무용 구분 가능 |

> 토폴로지 이미지 추가 예정

### 핵심 설정 - Application Control Blade 활성화 및 정책 구성

```
# === 1. Application Control Blade 활성화 ===
# SmartConsole → Gateway Object 더블클릭
# → General Properties → Network Security 탭
# → "Application Control" 체크 → OK

# === 2. Application Control Policy Layer 확인 ===
# Security Policies → Policy 탭
# → "Application Control" Layer가 자동 생성됨
# (또는 기존 Security Policy에 Unified Policy로 통합)

# === 3. Application Control Rule 구성 ===

# Rule 1: P2P 전체 차단
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications:
#     → Applications/Categories → "P2P File Sharing" 카테고리 선택
#     (BitTorrent, eMule, uTorrent 등 수십 개 앱 포함)
#   Action: Drop
#   Track: Log

# Rule 2: Streaming Media 차단
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications:
#     → Applications → "YouTube" 직접 선택
#     → Categories → "Streaming Media" 카테고리 추가
#   Action: Drop
#   Track: Log

# Rule 3: 개인 클라우드 스토리지 차단
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications:
#     → Applications → "Dropbox", "Google Drive"
#     → Categories → "File Storage and Sharing"
#   Action: Drop
#   Track: Log

# Rule 4: 나머지 웹 트래픽 허용
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications: Any
#   Action: Accept
#   Track: Log

# === 4. 정책 설치 ===
# Install Policy → Gateway 선택 → Install
```

**Application Control의 애플리케이션 DB:**
- Check Point은 **AppWiki**라는 애플리케이션 데이터베이스를 유지 (1만+ 앱 등록)
- 자동 업데이트를 통해 새로운 앱/버전이 지속 추가됨
- SmartConsole에서 `Objects` → `Applications/Categories`에서 검색 가능

---

## 면접관: "Application Control이 애플리케이션을 어떻게 식별하는 겁니까? 그리고 HTTPS 트래픽은 암호화되어 있는데 어떻게 내용을 보나요?"

### 애플리케이션 식별 방법

Application Control은 여러 기법을 조합하여 애플리케이션을 식별합니다.

| 식별 기법 | 설명 | 예시 |
|-----------|------|------|
| **SNI (Server Name Indication)** | TLS Handshake의 SNI 필드에서 도메인 확인 | `*.youtube.com` → YouTube |
| **Certificate CN** | 서버 인증서의 Common Name 확인 | `*.dropbox.com` → Dropbox |
| **IP 기반** | 알려진 서비스의 IP 대역 | Netflix CDN IP 대역 |
| **URL Pattern** | HTTP Host 헤더 또는 URL 패턴 매칭 | `Host: torrent.xxx` |
| **Protocol Signature** | 프로토콜 고유 패턴(바이트 시퀀스) 분석 | BitTorrent 프로토콜 시그니처 |
| **Behavioral** | 통신 패턴(연결 수, 포트, 패킷 크기 등) | P2P 특유의 다수 Peer 연결 |

### HTTPS와 Application Control

```
# HTTPS 트래픽에서 Application Control이 할 수 있는 것:
#
# [HTTPS Inspection 없이도 가능]
# - SNI 필드 분석: TLS ClientHello에서 도메인 확인 (암호화 전)
# - 서버 인증서 CN 분석: TLS Handshake에서 확인 (암호화 전)
# - IP 기반 매칭: 목적지 IP로 서비스 식별
# → "유튜브 접속 자체"는 SNI만으로 차단 가능!
#
# [HTTPS Inspection 필요]
# - URL 전체 경로 분석 (예: youtube.com/watch?v=xxx)
# - HTTP Body 내 콘텐츠 분석
# - 파일 업로드/다운로드 내용 검사
# - DLP(Data Loss Prevention) 적용
# → "유튜브 내 특정 카테고리 영상만 차단" 같은 세밀한 제어에 필요
```

**결론:**
- 대부분의 앱 차단(유튜브 전체 차단, 토렌트 차단 등)은 **SNI 기반으로 HTTPS Inspection 없이 가능**
- 세밀한 URL 기반 제어, 콘텐츠 검사가 필요하면 **HTTPS Inspection 필수**

---

## 면접관: "그런데 특정 앱(예: Dropbox)이 차단 정책을 넣었는데도 계속 동작합니다. 왜 그런 거고 어떻게 해결하시겠어요?"

Application Control 차단이 안 되는 경우는 여러 원인이 있습니다.

### 원인 1: 이미 수립된 세션

```
# Application Control은 새 세션에 대해 적용됨
# 이미 연결된 세션은 차단되지 않을 수 있음

# 해결: 기존 연결 테이블 클리어 (서비스 영향 주의!)
fw tab -t connections -x    # 전체 연결 테이블 클리어 (주의!)

# 또는 특정 IP 연결만 확인
fw tab -t connections -f | grep <Client_IP>
# → 기존 세션이 유지되고 있는지 확인

# 정책 Install 후 일정 시간이 지나면 기존 세션도 만료되어 차단 적용
```

### 원인 2: Rule Base 순서 문제

```
# Application Control Layer에서 룰 순서 확인
# 예시 (잘못된 순서):
# Rule 1: Any → Any → HTTPS → Accept    ← 이 룰이 먼저 매칭!
# Rule 2: Any → Any → Dropbox → Drop    ← 절대 도달 안 됨

# 해결: 구체적인(Application 지정) 룰을 위에 배치
# Rule 1: Any → Any → Dropbox → Drop    ← 먼저 매칭
# Rule 2: Any → Any → HTTPS → Accept    ← Dropbox 외 HTTPS 허용
```

### 원인 3: 앱이 다른 프로토콜/도메인 사용

```
# 일부 앱은 차단을 우회하기 위해:
# - 다른 도메인으로 통신 (CDN, 서브도메인)
# - QUIC(UDP 443) 프로토콜 사용
# - Domain Fronting 기법 사용

# 해결 1: 카테고리 전체를 차단 (개별 앱보다 카테고리 기반)
# "File Storage and Sharing" 카테고리 → Dropbox + 유사 서비스 모두 포함

# 해결 2: QUIC 프로토콜 차단
# QUIC(UDP 443)를 차단하면 브라우저가 HTTPS(TCP 443)로 Fallback
# → Application Control이 더 효과적으로 식별 가능
# Rule: Any → Any → QUIC(UDP 443) → Drop

# 해결 3: Custom Application 생성
# SmartConsole → Objects → Applications → New Application
# → 해당 앱이 사용하는 도메인/URL 패턴 수동 등록
```

### 원인 4: Application DB 미업데이트

```
# Application Control 시그니처가 오래되면 새 앱/변종 식별 불가

# 업데이트 확인:
# SmartConsole → Manage & Settings → Blades → Application Control
# → "Update Application and URL Filtering Database" 확인
# → "Schedule automatic update" 활성화

# 수동 업데이트 (Gateway Expert Mode)
# Online 업데이트
fwm fetch_uf_local    # URL Filtering DB 갱신

# 또는 SmartConsole에서:
# Gateway Object → Application Control → "Update Now"
```

### 원인 5: Fail-Open 설정

```
# Application Control Blade가 부하 초과 시 검사를 건너뛰는 설정

# SmartConsole → Gateway Object → Application Control
# → Advanced → "Fail mode"
#   - "Fail-open": 검사 불가 시 트래픽 허용 (기본값)
#   - "Fail-close": 검사 불가 시 트래픽 차단

# 트래픽이 많은 환경에서 Fail-open이면 일부 앱이 차단을 피할 수 있음
# → 하드웨어 성능 확인 및 필요 시 Fail-close 검토
```

---

## 면접관: "HTTPS Inspection을 도입하려고 합니다. 인증서는 어떻게 배포하고, 주의할 점은 뭡니까?"

HTTPS Inspection(SSL Inspection)은 Gateway가 **중간에서 HTTPS를 복호화**하여 내용을 검사하는 기능입니다. 본질적으로 **Man-in-the-Middle(MITM)** 방식이므로 인증서 관리가 핵심입니다.

### HTTPS Inspection 동작 원리

```
# [Client] ←--HTTPS--→ [Gateway] ←--HTTPS--→ [Server]
#
# 1. Client가 Server로 HTTPS 요청
# 2. Gateway가 중간에서 가로채서:
#    a. Gateway ↔ Server 간 별도 TLS 세션 수립 (서버의 실제 인증서 확인)
#    b. Gateway가 서버 인증서를 모방한 "대체 인증서" 생성
#       (Gateway의 CA 인증서로 서명)
#    c. Client ↔ Gateway 간 TLS 세션 수립 (대체 인증서 사용)
# 3. Gateway가 복호화된 트래픽을 검사 (Application Control, IPS, AV 등)
# 4. 검사 후 다시 암호화하여 각각에게 전달
#
# ※ Client가 Gateway의 CA 인증서를 신뢰해야 경고 없이 동작
```

### HTTPS Inspection 설정

```
# === 1. HTTPS Inspection 활성화 ===
# SmartConsole → Gateway Object → HTTPS Inspection
# → "Enable HTTPS Inspection" 체크

# === 2. CA 인증서 생성 또는 가져오기 ===
# 옵션 A: Check Point이 자체 생성 (간편)
# → "Create" → CA 이름, 유효기간 설정
# → 이 CA 인증서를 모든 Client PC에 배포해야 함

# 옵션 B: 기존 Enterprise CA 인증서 가져오기 (권장)
# → "Import" → 기업 내부 CA 인증서 (PFX/P12) 가져오기
# → 이미 도메인 PC에 Enterprise CA가 신뢰되어 있으면 별도 배포 불필요!

# === 3. HTTPS Inspection Policy 구성 ===
# SmartConsole → Security Policies → HTTPS Inspection 탭

# Rule 1: 금융/의료 사이트 예외 (법적 사유)
#   Source: Any
#   Destination: Financial Services, Health 카테고리
#   Action: Bypass
#   Track: Log

# Rule 2: 내부 → 외부 HTTPS 검사
#   Source: Internal_Net
#   Destination: Any
#   Action: Inspect
#   Track: Log

# === 4. Install Policy ===
```

### CA 인증서 배포 방법

```
# 방법 1: Active Directory GPO (도메인 환경, 가장 효율적)
# 1. SmartConsole에서 CA 인증서 Export (Base64 .cer 형식)
# 2. AD 서버에서:
#    Group Policy Management → 새 GPO 생성
#    Computer Configuration → Policies → Windows Settings
#    → Security Settings → Public Key Policies
#    → Trusted Root Certification Authorities
#    → Import → CA 인증서 가져오기
# 3. GPO를 전체 OU에 적용
# 4. Client PC에서 gpupdate /force 또는 다음 로그인 시 자동 적용

# 방법 2: MDM (모바일 장비)
# Intune, JAMF 등 MDM 솔루션을 통해 인증서 프로파일 배포

# 방법 3: 수동 설치 (소규모 / 비도메인 환경)
# Client PC에서:
# 1. CA 인증서 파일 더블클릭
# 2. "인증서 설치" → "로컬 머신" → "모든 인증서를 다음 저장소에 저장"
#    → "신뢰할 수 있는 루트 인증 기관" 선택 → 완료

# 방법 4: 브라우저별 설정 (Firefox는 별도 관리)
# Firefox는 Windows 인증서 저장소를 사용하지 않으므로 별도 설정:
# Firefox → Settings → Privacy & Security → Certificates
# → View Certificates → Authorities → Import → CA 인증서
# 또는 GPO로 Firefox 정책 배포
```

### 주의사항

```
# 1. 법적/윤리적 이슈
#    - HTTPS Inspection은 사실상 통신 감청
#    - 반드시 사전 고지 필요 (사내 보안 정책, 개인정보 처리방침)
#    - 금융 거래, 의료 정보 등 민감 사이트는 Bypass 권장

# 2. 인증서 Pinning 문제
#    - 일부 앱(은행 앱, 특정 보안 프로그램)은 Certificate Pinning 사용
#    - HTTPS Inspection 시 인증서가 변경되므로 앱이 동작하지 않음
#    - 해결: 해당 앱/도메인을 Bypass Rule에 추가

# 3. 성능 영향
#    - HTTPS 복호화/재암호화는 CPU 집약적 작업
#    - 트래픽 양에 따라 Gateway 성능 30~50% 저하 가능
#    - 하드웨어 스펙(특히 SSL Acceleration 카드) 확인 필요

# 4. 인증서 만료 관리
#    - Inspection용 CA 인증서 유효기간 모니터링
#    - 만료 시 모든 Client에서 인증서 오류 발생
```

---

## 면접관: "고객사에서 추가로 특정 URL 카테고리(도박, 성인 사이트)도 차단해달라고 합니다. URL Filtering은 Application Control과 뭐가 다른 겁니까?"

### Application Control vs URL Filtering

| 구분 | Application Control | URL Filtering |
|------|-------------------|---------------|
| **식별 대상** | 애플리케이션(앱) | URL/도메인/카테고리 |
| **식별 단위** | 앱 단위 (예: YouTube 전체) | URL 단위 (예: youtube.com/특정경로) |
| **데이터베이스** | AppWiki (앱 시그니처) | URL Filtering DB (도메인/URL 카테고리) |
| **카테고리 예시** | P2P, Streaming, Social Network | Gambling, Adult Content, Malware |
| **라이선스** | Application Control Blade | URL Filtering Blade (별도 또는 NGTX 포함) |
| **사용 목적** | 앱 사용 제어 | 웹사이트 접근 제어 |

**핵심 차이:** Application Control은 "어떤 앱인가", URL Filtering은 "어떤 사이트/카테고리인가"를 기준으로 제어합니다. 실무에서는 **두 Blade를 함께 활성화**하여 사용하는 것이 일반적입니다.

### URL Filtering 설정

```
# === 1. URL Filtering Blade 활성화 ===
# SmartConsole → Gateway Object → General Properties
# → "URL Filtering" 체크 → OK

# === 2. URL Filtering Rule 구성 ===
# Security Policies → Application Control / URL Filtering Layer

# Rule 1: 도박 사이트 차단
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications:
#     → Categories → "Gambling" 선택
#   Action: Drop
#   Track: Log

# Rule 2: 성인 사이트 차단
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications:
#     → Categories → "Pornography / Sexually Explicit" 선택
#     → Categories → "Nudity" 추가
#   Action: Drop
#   Track: Log

# Rule 3: 악성코드 유포 사이트 차단
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications:
#     → Categories → "Malware", "Phishing", "Spam" 선택
#   Action: Drop
#   Track: Log + Alert

# Rule 4: 나머지 허용
#   Source: Internal_Net
#   Destination: Any
#   Service & Applications: Any
#   Action: Accept
#   Track: Log

# === 3. Install Policy ===
```

### URL Filtering 카테고리 관리

```
# 카테고리 조회:
# SmartConsole → Objects → Applications/Categories
# → 검색창에 "Gambling", "Adult" 등 검색
# → 각 카테고리에 포함된 사이트/도메인 확인

# 특정 URL의 카테고리 확인:
# 방법 1: SmartConsole → Logs → 해당 로그에서 "Application/Category" 필드 확인
# 방법 2: Check Point URL Categorization 웹사이트에서 직접 조회
#   https://www.checkpoint.com/urlcat/

# Custom URL 추가 (카테고리에 없는 사이트 수동 등록):
# SmartConsole → Objects → Applications → New Application/Site
# → URL List에 특정 도메인/URL 패턴 추가
# → 해당 Custom Object를 Rule에 사용

# 예시: 특정 도메인 수동 차단
# New Application/Site:
#   Name: Blocked_Custom_Sites
#   URLs:
#     - *.gambling-site.com
#     - *.unauthorized-streaming.net
#   → Rule에서 이 오브젝트를 Drop
```

---

## 면접관: "URL Filtering에서 차단 시 사용자에게 '이 사이트는 차단되었습니다' 같은 페이지를 보여줄 수 있습니까?"

네, Check Point의 **UserCheck** 기능을 사용하면 차단 페이지(Block Page)를 사용자에게 표시할 수 있습니다.

### UserCheck 설정

```
# === 1. UserCheck 활성화 ===
# SmartConsole → Gateway Object → General Properties
# → Other → "UserCheck" 체크
# → UserCheck Portal 설정:
#   - Portal Address: Gateway의 IP 또는 FQDN
#   - Certificate: UserCheck용 인증서 (자동 생성 또는 Import)

# === 2. Rule의 Action을 "Drop" 대신 특수 Action 사용 ===

# Action 종류:
# - Drop: 단순 차단 (사용자에게 아무 알림 없음, 타임아웃)
# - Reject: 차단 + TCP RST 전송 (빠른 실패 알림)
# - Ask: 경고 페이지 표시 + 사용자가 "계속 진행" 선택 가능
# - Inform: 정보 페이지 표시 + 자동 허용 (경고만)
# - Block (UserCheck): 차단 페이지 표시 (사유 표시, 사용자 확인)

# Rule 설정:
#   Source: Internal_Net
#   Destination: Any
#   Applications: Gambling 카테고리
#   Action: "Ask" 또는 사용자 정의 UserCheck 선택
#   Track: Log

# === 3. UserCheck 메시지 커스터마이징 ===
# SmartConsole → Manage & Settings → Blades → UserCheck
# → "UserCheck Interaction Objects"

# Block 메시지 예시 (한글 지원):
# 제목: "접근이 차단되었습니다"
# 본문: "해당 웹사이트는 사내 보안 정책에 의해 차단되었습니다."
#       "카테고리: [CATEGORY]"
#       "사유: 회사 보안 정책 위반"
#       "문의: 정보보안팀 (내선 1234)"
# 회사 로고 삽입 가능

# Ask 메시지 예시:
# 제목: "이 사이트를 방문하시겠습니까?"
# 본문: "해당 사이트([URL])는 [CATEGORY] 카테고리에 해당합니다."
#       "업무상 필요한 경우 '계속' 버튼을 클릭하세요."
#       "접속 기록은 보안 로그에 남습니다."
# [계속 진행] [취소] 버튼
```

### UserCheck 주의사항

```
# 1. HTTPS 사이트에서의 UserCheck:
#    - HTTPS Inspection 없이는 Block Page가 표시되지 않을 수 있음
#    - SNI 기반 차단 시: 브라우저에 "연결 실패" 에러만 표시
#    - HTTPS Inspection 활성화 시: Block Page 정상 표시

# 2. UserCheck Portal 접근성:
#    - Client → Gateway UserCheck Portal (TCP 443) 통신이 가능해야 함
#    - Internal DNS에서 UserCheck Portal FQDN 등록 권장

# 3. HTTP Redirect 방식:
#    - 차단 시 Gateway가 HTTP 302 Redirect로 UserCheck Portal로 안내
#    - HTTPS의 경우 Redirect 불가 → HTTPS Inspection 필요

# 4. 성능:
#    - UserCheck는 추가 HTTP 세션을 생성하므로 Gateway 부하 증가
#    - 대량 사용자 환경에서는 Gateway 스펙 확인
```

---

## 면접관: "Application Control과 URL Filtering을 운영하면서 '이 사이트/앱은 왜 차단되나요?'라는 사용자 문의가 많이 올 텐데, 효율적으로 대응하는 방법이 있습니까?"

실무에서 가장 많이 받는 문의이며, 체계적인 대응 프로세스가 필요합니다.

### 1단계: 즉시 확인 (SmartConsole)

```
# Logs & Monitor에서 해당 사용자의 차단 로그 검색
# 필터: src:<User_IP> AND action:Drop AND time:last_1_hour

# 로그 상세에서 확인할 항목:
# - Rule Name: 어떤 룰에 의해 차단되었는지
# - Application: 식별된 애플리케이션
# - Category: URL Filtering 카테고리
# - URL: 접근 시도한 URL (HTTPS Inspection 활성 시)
# - Matched Category: 매칭된 카테고리
```

### 2단계: 오탐(False Positive) 확인

```
# 해당 URL/앱이 잘못 분류된 경우:

# Check Point URL 카테고리 재확인:
# https://www.checkpoint.com/urlcat/ 에서 URL 입력
# → 현재 분류 카테고리 확인

# 오분류인 경우:
# → Check Point에 재분류 요청 (위 사이트에서 "Request to change" 제출)
# → 임시로 해당 URL을 Whitelist에 추가

# 임시 예외 처리 (White List):
# SmartConsole → Security Policies → Application Control
# → 최상단에 예외 Rule 추가:
# Rule 0 (최상단):
#   Source: Internal_Net (또는 특정 사용자 그룹)
#   Destination: Any
#   Applications: 문제의 앱/URL (Custom Object로 생성)
#   Action: Accept
#   Track: Log
#   Comment: "오탐 - 재분류 요청 중 (Ticket #12345)"
```

### 3단계: 업무 필요성에 따른 예외 처리

```
# 사용자가 업무상 필요하다고 주장하는 경우:

# 방법 1: 사용자/그룹 기반 예외
# Identity Awareness Blade 활성화 시:
# Rule:
#   Source: Marketing_Team (AD Group)
#   Applications: Social Networks (Facebook, Instagram 등)
#   Action: Accept
#   Track: Log
# → 마케팅 팀만 SNS 접근 허용

# 방법 2: 시간 기반 제어
# Rule의 Time 컬럼 사용:
#   Applications: Streaming Media
#   Action: Accept
#   Time: "Lunch_Time" (12:00-13:00)
# → 점심시간에만 스트리밍 허용

# 방법 3: Limit (대역폭 제한)
# Action: "Limit" 선택
# → 해당 앱의 대역폭을 제한 (예: YouTube 1Mbps)
# → 완전 차단 대신 속도 제한으로 절충
```

### 운영 자동화 팁

```
# 1. UserCheck "Ask" 모드 활용:
#    차단 대신 "Ask"로 설정하면:
#    - 사용자가 직접 "업무상 필요" 클릭하여 일시 허용
#    - 로그에 사용자 확인 기록 남음
#    - 보안팀 문의 감소, 감사 시 근거 확보

# 2. 주간/월간 리포트:
#    SmartEvent → Application Usage Report
#    → 가장 많이 차단된 앱/카테고리 Top 10
#    → 오탐 패턴 파악 → 정책 튜닝

# 3. Application Control DB 자동 업데이트 확인:
#    SmartConsole → Manage & Settings → Blades
#    → Application Control & URL Filtering
#    → "Automatic Update" 스케줄 확인
#    → 업데이트 실패 시 수동:
cpstop FWD
cpstart FWD
# 또는
contract_util update_all    # Expert Mode에서 DB 업데이트
```

### 전체 아키텍처 요약

```
# Check Point 콘텐츠 보안 Blade 스택:
#
# [L3/L4] Firewall (기본 정책) ──────────────── 포트/IP 기반 차단
#              ↓
# [L7]   Application Control ────────────────── 앱 식별/차단
#              ↓
# [L7]   URL Filtering ─────────────────────── URL 카테고리 차단
#              ↓
# [L7]   HTTPS Inspection ──────────────────── 암호화 트래픽 검사
#              ↓
# [L7]   IPS (Intrusion Prevention) ────────── 취약점 공격 차단
#              ↓
# [L7]   Anti-Virus / Anti-Bot ──────────────── 악성코드/C&C 차단
#              ↓
# [L7]   Threat Emulation / Extraction ─────── 샌드박스/파일 무해화
#
# ※ 각 Blade는 독립적으로 활성화 가능
# ※ NGTX 라이선스: 위 모든 Blade 포함
# ※ 성능: Blade를 많이 활성화할수록 Gateway 부하 증가
#    → 사전 Sizing 필수 (Check Point Sizing Tool 활용)
```
