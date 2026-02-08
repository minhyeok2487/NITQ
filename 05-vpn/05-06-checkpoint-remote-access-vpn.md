# 05-06. Checkpoint Remote Access VPN

---

## 면접관: "고객사가 방화벽을 Checkpoint으로 운영 중인데, 재택근무 확대로 원격 VPN 접속이 필요합니다. Checkpoint 환경에서 Remote Access VPN은 어떻게 구성하시겠어요?"

Checkpoint 환경에서는 **Endpoint Security VPN**(구 SecuRemote/SecureClient) 클라이언트를 사용한 Remote Access VPN을 구성하겠습니다.

Checkpoint Remote Access VPN은 **IPSec 기반**이 기본이며, 클라이언트가 IKE 협상을 수행합니다. Cisco AnyConnect가 SSL 기반인 것과 다르게, Checkpoint은 전통적으로 **IPSec 기반 Remote Access**를 주력으로 제공합니다.

> 토폴로지 이미지 추가 예정

### 설계 개요

| 항목 | 설정 |
|------|------|
| Gateway | Checkpoint Security Gateway R81.20 |
| Management | Checkpoint SmartConsole (Smart Management Server) |
| 클라이언트 | Endpoint Security VPN (E88.x) |
| 인증 | AD 연동 + 인증서 (Optional) |
| IP 할당 | Office Mode (10.99.0.0/24) |
| Encryption | IKEv2 + AES-256 + SHA-256 |
| 동시 접속 | 200명 (라이선스: Mobile Access Blade 또는 Remote Access VPN Blade) |

### 핵심 설정 - SmartConsole

```
1. ── Gateway에 IPSec VPN Blade 활성화 ──
   SmartConsole > Gateway Object > General Properties
   > Network Security > "IPSec VPN" Blade 활성화

2. ── Remote Access Community 생성 ──
   SmartConsole > VPN Communities > Remote Access
   > "RemoteAccess" Community 확인 (기본 제공)
   > Participating Gateways에 해당 GW 추가

3. ── 사용자 인증 설정 ──
   SmartConsole > Manage & Settings > Blades > Mobile Access
   > Authentication: LDAP Account Unit 연동

   또는:
   Gateway Object > VPN Clients > Authentication
   > "LDAP users" 또는 "All users" 선택
   > Authentication Method: Check Point Password / RADIUS / Certificate

4. ── Office Mode IP Pool 설정 ──
   Gateway Object > VPN Clients > Office Mode
   > "Allow Office Mode" 체크
   > Method: "Manual" 또는 "DHCP"
   > IP Pool: 10.99.0.0 ~ 10.99.0.254 / Subnet Mask: 255.255.255.0

5. ── Encryption 설정 ──
   Gateway Object > VPN Clients > Encryption
   > IKE Phase 1: AES-256 / SHA-256 / Group 14
   > IKE Phase 2: AES-256 / SHA-256 / PFS Group 14

6. ── Access Rule 추가 ──
   Security Policy > Rule Base
   > Source: RemoteAccess (VPN Community)
   > Destination: 사내 네트워크 객체
   > Service: 필요한 서비스 (또는 Any)
   > Action: Accept
   > VPN: RemoteAccess Community

7. ── Policy 설치 ──
   Install Policy > 해당 Gateway 선택
```

---

## 면접관: "Office Mode라는 게 나왔는데, 이게 뭔가요? 왜 필요한 거죠?"

### Office Mode란

Office Mode는 Checkpoint Remote Access VPN의 **IP 할당 메커니즘**입니다. VPN 접속 시 클라이언트에게 **사내 네트워크 대역의 IP를 할당**하여, 마치 사무실에 있는 것처럼 통신할 수 있게 합니다.

Cisco AnyConnect의 "VPN Pool"과 동일한 개념입니다.

### Office Mode가 없을 때의 문제

```
! Office Mode OFF:
! 사용자 PC의 ISP IP(예: 59.10.x.x)로 사내 서버에 접근
! → 사내 방화벽/서버는 외부 IP로 인식
! → ACL, 보안 정책에서 차단될 수 있음
! → NAT 문제, 라우팅 문제 발생

! Office Mode ON:
! 사용자 PC가 10.99.0.50 (사내 대역) 할당받음
! → 사내 서버에서는 내부 IP로 인식
! → 기존 보안 정책, ACL 그대로 적용 가능
! → 라우팅도 사내 네트워크와 동일하게 동작
```

### Office Mode IP 할당 방식

| 방식 | 설명 | 장점 |
|------|------|------|
| **Manual (IP Pool)** | Gateway에서 직접 IP 대역 지정 | 간단, 독립적 |
| **DHCP** | 사내 DHCP 서버에서 할당 | DNS/WINS 자동 설정, 기존 IPAM 연동 |
| **RADIUS** | RADIUS 서버에서 IP 전달 | 사용자별 고정 IP 가능 |
| **ipassignment.conf** | Gateway 파일에서 세부 제어 | 고급 커스터마이징 |

### Manual IP Pool 설정 상세

```
! SmartConsole:
Gateway Object > VPN Clients > Office Mode

 [x] Allow Office Mode
 [x] Offer Office Mode to group: "All Users" 또는 특정 그룹

 Allocate IP from:
 (o) Manual:
     IP Address: 10.99.0.0
     Mask: 255.255.255.0
     First IP: 10.99.0.1
     Last IP: 10.99.0.254

 Anti-Spoofing:
 [x] "Perform Anti-Spoofing on Office Mode addresses"
 ! → Office Mode IP가 Gateway 뒤쪽 Topology에 포함되어야 Anti-Spoofing 통과

 Optional:
 DNS Server: 10.1.1.10
 DNS Domain: company.local
 WINS Server: (필요 시)
```

### 중요 주의사항

```
! 1. Anti-Spoofing 설정
! Office Mode IP 대역(10.99.0.0/24)이 Gateway의 Internal Topology에
! 포함되어야 함. 그렇지 않으면 Anti-Spoofing에 의해 패킷 드롭

! Gateway Object > Network Management > Topology
! Internal Network에 10.99.0.0/24 포함 확인

! 2. 사내 장비의 Return Route
! 사내 L3 스위치/라우터에서 10.99.0.0/24에 대한
! Static Route가 Checkpoint Gateway를 가리켜야 함
! ip route 10.99.0.0 255.255.255.0 <checkpoint-gw-internal-ip>

! 3. NAT 설정
! Office Mode IP가 사내→외부 통신 시 NAT되지 않도록 주의
! VPN 트래픽에 대한 NAT Exemption 필요할 수 있음
```

---

## 면접관: "사용자가 사무실 밖의 특수한 네트워크 환경(호텔, 공항 등)에서 접속이 안 된다고 합니다. 일반 가정에서는 잘 됩니다. 원인이 뭘까요?"

호텔, 공항 등의 네트워크는 대부분 **HTTP/HTTPS(TCP 80/443)만 허용**하고 나머지 프로토콜을 차단합니다. Checkpoint Endpoint Security VPN은 기본적으로 **IPSec(UDP 500, ESP Protocol 50)**을 사용하므로 이런 환경에서 차단됩니다.

이 문제를 해결하는 것이 **Visitor Mode**입니다.

### Visitor Mode란

Visitor Mode는 IPSec VPN 트래픽을 **TCP 443으로 캡슐화**하여 전송하는 Checkpoint의 기능입니다. 제한적인 네트워크에서도 VPN 접속이 가능하게 합니다.

```
! 일반 모드 (Normal Mode):
! [IP][ESP][Encrypted Payload] → UDP 500 / ESP Protocol 50
! → 호텔 방화벽에서 차단

! Visitor Mode:
! [IP][TCP 443][IPSec Tunnel 전체] → TCP 443
! → HTTPS 트래픽처럼 보이므로 통과
```

### Visitor Mode 동작 원리

1. 클라이언트가 **TCP 443으로 Gateway에 연결** (SSL Handshake X, 단순 TCP 터널)
2. TCP 443 세션 내부에서 **IKE 협상 수행**
3. IPSec 터널이 TCP 443 위에서 동작
4. Gateway 측에서 TCP 캡슐화를 벗기고 일반 IPSec으로 처리

### Visitor Mode 설정

```
! ── Gateway 측 (SmartConsole) ──
Gateway Object > VPN Clients > Visitor Mode

 [x] "Support Visitor Mode" 체크
 Visitor Mode Service: TCP 443 (기본)
 ! 또는 다른 포트 지정 가능 (TCP 4433 등)

 [x] "Allocate dedicated port for Visitor Mode"
 ! → Gateway의 외부 인터페이스에서 TCP 443 수신 대기

! Policy 설치 후 적용

! ── 클라이언트 측 (Endpoint Security VPN) ──
! VPN Site 설정 > Connection Tab
! [x] "Use Visitor Mode" 체크
! 또는 자동 감지: 일반 연결 실패 시 자동으로 Visitor Mode 전환

! 사용자에게 안내:
! "호텔 등에서 접속 안 되면 Endpoint Security VPN > Site 속성 > Visitor Mode 체크"
```

### Visitor Mode 주의사항

| 항목 | 설명 |
|------|------|
| 성능 | TCP-over-TCP 구조이므로 **성능 저하** (TCP Meltdown 가능) |
| 포트 충돌 | Gateway에서 HTTPS Management(SmartConsole)도 TCP 443 사용 → **포트 분리 필요** |
| 프록시 환경 | 일부 호텔은 HTTPS도 프록시 경유 → Proxy 설정 지원 필요 |
| SSL Inspection | HTTPS DPI(Deep Packet Inspection)를 하는 환경에서는 Visitor Mode도 차단 가능 |

### 포트 충돌 해결

```
! SmartConsole Management와 Visitor Mode가 모두 TCP 443을 쓰는 경우:
! 방법 1: Visitor Mode를 다른 포트(443 외)로 변경
!   → 하지만 443이 아니면 제한적 네트워크에서 차단될 수 있음

! 방법 2: Gateway에 인터페이스/IP가 2개 있다면 분리
!   → WAN IP 1: SmartConsole 관리용 (TCP 443)
!   → WAN IP 2: Visitor Mode VPN용 (TCP 443)

! 방법 3: Management Port를 변경
!   Gateway Object > Management > Communication
!   Management Port를 TCP 4434 등으로 변경
```

---

## 면접관: "고객사에 Checkpoint Gateway가 본사와 DR 사이트에 2대 있는데, 사용자가 자동으로 가까운 게이트웨이로 접속하게 할 수 있나요?"

네, Checkpoint의 **MEP(Multiple Entry Points)** 기능으로 구현할 수 있습니다.

### MEP란

MEP는 여러 Checkpoint Gateway가 Remote Access VPN의 Entry Point로 동작할 때, 클라이언트가 **최적의 Gateway를 자동 선택**하도록 하는 기능입니다.

### MEP 모드

| 모드 | 설명 | 사용 시나리오 |
|------|------|---------------|
| **First to respond** | 모든 GW에 동시 연결 시도, **가장 먼저 응답한 GW**에 연결 | 자동 최적 선택 (가장 흔한 설정) |
| **Primary-Backup** | Primary GW 먼저 시도, 실패 시 Backup GW로 Failover | 명확한 Active-Standby 구조 |
| **Load distribution** | Random 또는 Round-robin으로 분산 | 대규모 사용자 부하 분산 |

### MEP 설정

```
! ── SmartConsole ──
1. 두 Gateway 모두 Remote Access Community에 추가
   VPN Communities > RemoteAccess > Participating Gateways
   > Gateway-HQ (203.0.113.1) 추가
   > Gateway-DR (203.0.113.2) 추가

2. MEP 설정
   Gateway Object > VPN Clients > Office Mode
   ! 각 GW에 서로 다른 Office Mode Pool 권장
   ! GW-HQ: 10.99.1.0/24
   ! GW-DR: 10.99.2.0/24

3. Global Properties > VPN > Advanced
   ! "Enable load distribution for Multiple Entry Point configurations"
   ! MEP Mode 선택:
   ! - First to respond (권장)
   ! - Primary-Backup
   ! - Random selection
```

### MEP Gateway Selection 동작 (First to Respond)

```
! 클라이언트(Endpoint Security VPN)에 VPN Site 설정:
! Site: company-vpn.com
! Gateway 목록: 203.0.113.1, 203.0.113.2

! 접속 시:
! 1. 클라이언트가 양쪽 GW에 동시에 IKE negotiation 시작
! 2. 먼저 응답한 GW와 터널 수립
! 3. 보통 네트워크적으로 가까운 GW가 먼저 응답
! 4. 해당 GW 장애 시 자동으로 다른 GW로 재접속

! Primary-Backup 모드:
! 1. 클라이언트가 Primary GW(203.0.113.1)에만 먼저 시도
! 2. Timeout 발생 시 Backup GW(203.0.113.2)로 시도
! 3. Primary 복구 시 자동 전환 (또는 유지 - 설정에 따라)
```

### Probing 설정 (Primary-Backup 모드)

```
! Primary GW 상태를 확인하는 Probing 방식:

! SmartConsole > VPN Communities > RemoteAccess > Participating Gateways
! Gateway Priority:
!   GW-HQ: Priority 1 (Primary)
!   GW-DR: Priority 2 (Backup)

! Probing Method:
! - IKE probing: IKE negotiation으로 GW 생존 확인
! - HTTP probing: HTTP GET 요청으로 확인
! - Ongoing probing: 연결 중에도 주기적으로 Primary 상태 확인
!   → Primary 복구 시 자동 전환 가능
```

### DNS 기반 GW 선택 (추가 옵션)

```
! DNS Round Robin 또는 GeoDNS를 활용한 방법:

! vpn.company.com → 203.0.113.1 (GTM: 한국 사용자)
! vpn.company.com → 203.0.113.2 (GTM: 해외 사용자)

! F5 GTM, AWS Route 53, Cloudflare 등 활용 가능
! 클라이언트가 DNS Resolve 결과로 가까운 GW에 접속
```

---

## 면접관: "고객사가 향후 모바일 기기(스마트폰, 태블릿)에서도 접속할 수 있도록 확장하고 싶다고 합니다. Checkpoint에서 가능한가요?"

네, Checkpoint의 **Mobile Access Blade**를 활성화하면 모바일 기기 지원이 가능합니다.

### Checkpoint Remote Access 라이선스 구조

| Blade | 용도 | 클라이언트 |
|-------|------|-----------|
| **IPSec VPN Blade** | PC용 IPSec Remote Access | Endpoint Security VPN |
| **Mobile Access Blade** | 모바일 + SSL VPN + Web Portal | **Capsule Connect** (iOS/Android), 브라우저 |

### Mobile Access Blade 특징

| 항목 | IPSec VPN Blade | Mobile Access Blade |
|------|----------------|-------------------|
| 프로토콜 | IPSec | **SSL/TLS** |
| PC 클라이언트 | Endpoint Security VPN | Endpoint Security VPN (SSL 모드) |
| 모바일 클라이언트 | X | **Capsule Connect** (iOS/Android) |
| Web Portal | X | **O (Clientless)** |
| 모바일 OS | X | **iOS, Android, ChromeOS** |
| ActiveSync | X | **O (Exchange 연동)** |
| Per-App VPN | X | **O** |
| 라이선스 | GW에 포함 | **추가 라이선스 필요** |

### Mobile Access Blade 설정

```
1. ── Blade 활성화 ──
   SmartConsole > Gateway Object > General Properties
   > Mobile Access Blade 활성화

2. ── SSL VPN Portal 설정 ──
   SmartConsole > Manage & Settings > Blades > Mobile Access
   > Portal Settings:
     - Portal URL: https://vpn.company.com
     - Certificate: 정식 SSL 인증서 적용
     - Customization: 회사 로고, 색상 등

3. ── Mobile Access 정책 설정 ──
   SmartConsole > Mobile Access Policy
   > Applications:
     - Native Applications (Full Layer 3 VPN)
     - Web Applications (특정 내부 웹 서비스)
     - File Shares (SMB/CIFS)
     - Mail (Exchange ActiveSync)

4. ── Capsule Connect 앱 배포 ──
   iOS: App Store에서 "Check Point Capsule Connect" 다운로드
   Android: Google Play에서 다운로드

   앱 설정:
   - Site: vpn.company.com
   - Authentication: Username + Password (+ 2FA)
```

### Mobile Access와 기존 IPSec VPN 공존 구성

```
! 같은 Gateway에서 두 가지 모두 운영 가능:

! 1. PC 사용자: Endpoint Security VPN (IPSec)
!    → 기존 Remote Access Community 사용
!    → Office Mode IP Pool: 10.99.1.0/24

! 2. 모바일 사용자: Capsule Connect (SSL)
!    → Mobile Access Blade 사용
!    → Office Mode IP Pool: 10.99.2.0/24 (분리 권장)

! Pool 분리 이유:
! - 접속 소스 식별 용이
! - 보안 정책 차별화 (모바일은 제한된 접근만)
! - 장애 시 원인 분석 용이

! Security Policy 예시:
! Rule 1: Source=RemoteAccess(PC) / Dest=All_Internal / Action=Accept
! Rule 2: Source=MobileAccess(Mobile) / Dest=Mail_Servers+Web_Portal / Action=Accept
! Rule 3: Source=MobileAccess(Mobile) / Dest=All_Internal / Action=Drop + Log
```

### Per-App VPN (모바일 전용 고급 기능)

```
! Per-App VPN: 특정 업무용 앱만 VPN을 경유하도록 설정
! 개인 앱(YouTube, SNS)은 VPN 미경유

! SmartConsole > Mobile Access > Per-App VPN Settings
! Allowed Apps:
!   - com.company.erp (사내 ERP 앱)
!   - com.microsoft.outlook (Outlook)
!   - com.company.intranet (사내 포털)

! 장점:
! - 개인 트래픽이 회사 VPN을 경유하지 않음 (프라이버시)
! - VPN 대역폭 절약
! - MDM(Mobile Device Management) 연동으로 관리 강화
```

---

## 면접관: "Cisco AnyConnect와 Checkpoint Remote Access VPN을 비교하면 어떤 차이가 있나요? 고객사에 어떤 것을 권장하시겠어요?"

### Cisco AnyConnect vs Checkpoint Remote Access VPN

| 항목 | Cisco AnyConnect | Checkpoint Remote Access |
|------|-----------------|------------------------|
| 기본 프로토콜 | **SSL/TLS (TCP 443)** | **IPSec (UDP 500/ESP)** |
| SSL VPN | 기본 제공 | Mobile Access Blade 추가 필요 |
| 제한적 네트워크 | 기본적으로 잘 통과 | **Visitor Mode 별도 설정** 필요 |
| 모바일 지원 | AnyConnect (iOS/Android) | Capsule Connect (추가 Blade) |
| DTLS | **O (성능 최적화)** | X |
| Clientless VPN | O (WebVPN Portal) | O (Mobile Access Portal) |
| 2FA 연동 | RADIUS, SAML, Duo | RADIUS, Certificate, DynamicID |
| DAP (동적 정책) | **O (강력)** | Compliance Check (MEP 조건부) |
| 관리 | ASDM / FMC | SmartConsole |
| Split Tunnel | Group Policy에서 ACL | Encryption Domain 또는 Office Mode 설정 |
| 라이선스 | **AnyConnect Plus/Apex** | IPSec VPN Blade (기본) + Mobile Access (추가) |

### 권장 기준

| 고객사 환경 | 권장 솔루션 | 이유 |
|-------------|------------|------|
| Cisco 방화벽 사용 중 | **AnyConnect** | 네이티브 통합, 관리 용이 |
| Checkpoint 방화벽 사용 중 | **Checkpoint RA VPN** | 기존 인프라 활용, 추가 투자 최소 |
| 혼재 환경 (양쪽 모두) | 기존 GW 기반으로 선택 | 별도 VPN 집중기 고려 |
| 모바일 중심 | **AnyConnect** | SSL 기본, 모바일 지원 우수 |
| 고보안 환경 | **Checkpoint** | IPSec 기반, Endpoint Compliance 강력 |
| 다양한 네트워크 환경 | **AnyConnect** | SSL/DTLS 기본, NAT/방화벽 통과 우수 |

**실무 관점에서의 결론**: 고객사가 이미 Checkpoint을 운영 중이라면 Checkpoint Remote Access VPN을 구축하는 것이 비용과 관리 측면에서 유리합니다. 다만, 모바일 접속과 다양한 네트워크 환경 대응이 중요하다면 Mobile Access Blade를 반드시 추가하시고, Visitor Mode도 기본 활성화해두시는 것을 권장합니다. 새 프로젝트에서 장비 선정부터 할 수 있다면, Remote Access VPN 관점에서는 AnyConnect가 SSL 기반 특성상 더 유연한 것이 사실입니다.
