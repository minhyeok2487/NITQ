# 05-05. SSL VPN / Remote Access VPN

---

## 면접관: "고객사에서 재택근무를 확대하면서, 직원 200명이 집에서 사내 시스템에 접속해야 합니다. 어떻게 구성하시겠어요?"

재택근무 원격 접속에는 **Cisco AnyConnect SSL VPN**을 제안드리겠습니다.

Site-to-Site VPN과 달리 Remote Access VPN은 **불특정 다수의 사용자가 다양한 네트워크 환경(가정, 카페, 호텔)**에서 접속하므로, **SSL/TLS 기반**이 가장 적합합니다. HTTPS(TCP 443)를 사용하기 때문에 대부분의 네트워크 환경에서 차단 없이 접속 가능합니다.

> 토폴로지 이미지 추가 예정

### 설계 개요

| 항목 | 설정 |
|------|------|
| VPN 장비 | Cisco ASA 5525-X 또는 Firepower |
| VPN 방식 | AnyConnect SSL VPN |
| 클라이언트 | Cisco AnyConnect Secure Mobility Client |
| 인증 | AD/LDAP 연동 + Local Fallback |
| IP 할당 | VPN Pool: 10.99.0.0/24 |
| 접속 포트 | TCP 443 (SSL/TLS) |
| 동시 접속 | 200명 (라이선스 확인 필수) |

### 핵심 설정 (Cisco ASA)

```
! ── SSL VPN 활성화 ──
webvpn
 enable outside
 anyconnect enable
 anyconnect image disk0:/anyconnect-win-4.10.xxxx.pkg
 anyconnect image disk0:/anyconnect-macos-4.10.xxxx.pkg
 tunnel-group-list enable

! ── VPN IP Pool ──
ip local pool VPN-POOL 10.99.0.1-10.99.0.254 mask 255.255.255.0

! ── Group Policy ──
group-policy GP-REMOTE-ACCESS internal
group-policy GP-REMOTE-ACCESS attributes
 vpn-tunnel-protocol ssl-client
 dns-server value 10.1.1.10
 default-domain value company.local
 split-tunnel-policy tunnelspecified
 split-tunnel-network-list value ACL-SPLIT-TUNNEL
 address-pools value VPN-POOL
 anyconnect keep-installer installed
 anyconnect dpd-interval client 30
 anyconnect dpd-interval gateway 30

! ── Split Tunnel ACL (사내 대역만 VPN 경유) ──
access-list ACL-SPLIT-TUNNEL standard permit 10.1.0.0 255.255.0.0
access-list ACL-SPLIT-TUNNEL standard permit 10.2.0.0 255.255.0.0
access-list ACL-SPLIT-TUNNEL standard permit 192.168.100.0 255.255.255.0

! ── Tunnel Group (Connection Profile) ──
tunnel-group TG-REMOTE-ACCESS type remote-access
tunnel-group TG-REMOTE-ACCESS general-attributes
 default-group-policy GP-REMOTE-ACCESS
 address-pool VPN-POOL
 authentication-server-group AD-SERVERS
 authentication-server-group (outside) LOCAL
tunnel-group TG-REMOTE-ACCESS webvpn-attributes
 group-alias "Company VPN" enable
 group-url https://vpn.company.com/remote enable

! ── NAT Exemption (VPN 트래픽은 NAT 제외) ──
nat (inside,outside) source static OBJ-INTERNAL OBJ-INTERNAL destination static OBJ-VPN-POOL OBJ-VPN-POOL no-proxy-arp route-lookup

! ── VPN 트래픽 라우팅 ──
! 내부 장비에서 VPN Pool(10.99.0.0/24)에 대한 return route 필요
! ASA에서 자동 추가되지만, 내부 L3 스위치에도 Static Route 필요
```

---

## 면접관: "SSL VPN과 IPSec VPN의 차이점을 설명해주세요. 왜 Remote Access에서는 SSL VPN을 더 많이 쓰나요?"

### SSL VPN vs IPSec Remote Access VPN

| 항목 | SSL VPN | IPSec VPN |
|------|---------|-----------|
| 프로토콜 | **TLS/SSL (TCP 443)** | IKE(UDP 500) + ESP(IP 50) |
| NAT 통과 | **문제 없음** (HTTPS니까) | NAT-T 필요 (UDP 4500) |
| 방화벽 통과 | **거의 항상 가능** | 기업 방화벽에서 차단 빈번 |
| 클라이언트 | 웹 브라우저 또는 AnyConnect | 전용 클라이언트 필수 |
| 설치 | AnyConnect 자동 배포 가능 | 사전 설치 필요 |
| 세분화 접근제어 | **Group Policy, DAP로 세밀한 제어** | 상대적으로 제한적 |
| 인증 | **다양 (LDAP, RADIUS, SAML, Certificate, MFA)** | PSK, Certificate |
| 접속 환경 제약 | **거의 없음** | 일부 환경(호텔, 공항) 차단 |
| 성능 | TCP overhead로 약간 느림 | **UDP 기반으로 빠름** |
| Clientless 모드 | **O (웹 포털)** | X |

### SSL VPN을 선호하는 이유

1. **접속 환경의 다양성**: 재택, 카페, 호텔, 공항 등 어디서든 **TCP 443만 열려 있으면** 접속 가능. IPSec은 UDP 500/4500과 ESP Protocol 50이 필요해서 제한적인 네트워크에서 차단됨

2. **배포 용이성**: 사용자가 웹 브라우저로 VPN 포털에 접속하면 AnyConnect가 **자동 다운로드 및 설치**됨. 별도 설치 과정 불필요

3. **세분화 접근제어**: DAP(Dynamic Access Policy)로 접속 환경(OS, 패치 수준, 안티바이러스 유무)에 따라 **접근 수준을 동적으로 변경** 가능

4. **DTLS 지원**: AnyConnect는 초기 연결을 TLS(TCP)로 맺은 후, **DTLS(UDP 443)**로 전환하여 성능 저하를 최소화. TCP-over-TCP 문제 회피

```
! DTLS 활성화 확인
show webvpn dtls

! AnyConnect 접속 상태 확인
show vpn-sessiondb anyconnect
```

---

## 면접관: "특정 사용자가 AnyConnect로 접속이 안 된다고 합니다. 다른 사용자는 잘 되는 상황인데, 어디부터 확인하시겠어요?"

### 진단 순서

#### 1단계: 기본 접속 확인

```
! ASA에서 현재 VPN 세션 확인
show vpn-sessiondb anyconnect

! 해당 사용자가 접속 시도한 기록 확인
show logging | include <username>
```

#### 2단계: 인증 실패 여부 확인

```
! 실시간 로그 확인
logging enable
logging timestamp
logging class auth console debugging

! 또는 syslog에서 확인
show logging | include authentication
! "authentication rejected" → 인증 서버 문제
! "session could not be established" → Group Policy 문제
```

#### 3단계: 주요 원인별 체크리스트

| # | 증상 | 가능한 원인 | 확인/해결 |
|---|------|------------|-----------|
| 1 | 로그인 자체 실패 | AD 계정 잠김/비밀번호 만료 | AD에서 계정 상태 확인 |
| 2 | 로그인 성공 후 끊김 | VPN Pool IP 소진 | `show ip local pool VPN-POOL` |
| 3 | 접속 되지만 사내 접근 불가 | Group Policy의 ACL 제한 | `show run group-policy` |
| 4 | 특정 사이트에서만 안 됨 | MTU 문제 또는 ISP 차단 | 다른 네트워크에서 테스트 |
| 5 | 인증서 오류 | SSL 인증서 만료/불일치 | ASA 인증서 확인 |
| 6 | AnyConnect 설치 실패 | 클라이언트 OS 호환성 | AnyConnect 버전 확인 |
| 7 | 라이선스 초과 | 동시 접속 수 초과 | `show vpn-sessiondb license-summary` |

#### 4단계: 상세 디버그 (필요 시)

```
! 특정 사용자에 대해서만 디버그 (운영 환경에서 안전)
debug webvpn anyconnect 255
debug aaa authentication
debug aaa authorization

! 확인 후 반드시 디버그 해제
undebug all
```

#### 5단계: VPN Pool 상태 확인

```
show ip local pool VPN-POOL
! 할당된 IP 수 / 남은 IP 수 확인
! 200명인데 /24(254개)면 여유 있지만, Peak 시간에 부족할 수 있음

! Pool 확장 필요 시
ip local pool VPN-POOL 10.99.0.1-10.99.1.254 mask 255.255.254.0
```

---

## 면접관: "Split Tunnel이란 뭔가요? Full Tunnel과 비교해서 장단점을 설명해주세요."

### Split Tunnel vs Full Tunnel

```
! ── Full Tunnel ──
! 모든 트래픽이 VPN을 경유
! 사용자 PC → [VPN 터널] → ASA → 인터넷/사내 시스템

! ── Split Tunnel ──
! 사내 대역만 VPN 경유, 나머지(인터넷)는 로컬로 직접 나감
! 사용자 PC → [VPN 터널] → ASA → 사내 시스템
! 사용자 PC → [로컬 인터넷] → YouTube, Google 등
```

### 비교

| 항목 | Full Tunnel | Split Tunnel |
|------|-------------|--------------|
| 보안 | **높음** (모든 트래픽 검사) | 인터넷 트래픽은 미검사 |
| 대역폭 부하 | **높음** (YouTube 등도 VPN 경유) | **낮음** (사내 트래픽만) |
| 사용자 체감 | 느림 (인터넷 속도 저하) | **빠름** |
| VPN 장비 부하 | 높음 | **낮음** |
| Compliance | 금융, 공공기관 요구 | 일반 기업 |

### 설정

```
! ── Full Tunnel ──
group-policy GP-FULL-TUNNEL attributes
 split-tunnel-policy tunnelall

! ── Split Tunnel (지정된 네트워크만 VPN 경유) ──
group-policy GP-SPLIT-TUNNEL attributes
 split-tunnel-policy tunnelspecified
 split-tunnel-network-list value ACL-SPLIT-TUNNEL

access-list ACL-SPLIT-TUNNEL standard permit 10.1.0.0 255.255.0.0
access-list ACL-SPLIT-TUNNEL standard permit 10.2.0.0 255.255.0.0

! ── Exclude (특정 네트워크만 VPN 제외, 나머지 VPN 경유) ──
group-policy GP-SPLIT-EXCLUDE attributes
 split-tunnel-policy excludespecified
 split-tunnel-network-list value ACL-SPLIT-EXCLUDE

! 예: Zoom/Teams 트래픽만 로컬 인터넷으로 빼기
access-list ACL-SPLIT-EXCLUDE standard permit host 13.107.64.0 255.255.192.0
```

### 실무 권장

**일반 기업**: Split Tunnel 권장. 200명이 Full Tunnel이면 ASA의 Throughput과 인터넷 대역폭에 심각한 부하 발생.

**금융/공공**: Full Tunnel + 내부 프록시 서버 경유. 보안 감사 요구사항 충족.

---

## 면접관: "Group Policy와 DAP(Dynamic Access Policy)의 차이점은 뭔가요? 어떤 상황에서 DAP를 써야 하죠?"

### Group Policy

Group Policy는 **VPN 사용자 그룹에 공통 적용되는 정적 정책**입니다.

```
! 예: 일반 직원용 Group Policy
group-policy GP-EMPLOYEE attributes
 vpn-tunnel-protocol ssl-client
 split-tunnel-policy tunnelspecified
 split-tunnel-network-list value ACL-EMPLOYEE-NETS
 vpn-session-timeout 480
 vpn-idle-timeout 30
 address-pools value POOL-EMPLOYEE

! 예: IT 관리자용 Group Policy
group-policy GP-IT-ADMIN attributes
 vpn-tunnel-protocol ssl-client
 split-tunnel-policy tunnelall
 vpn-session-timeout none
 address-pools value POOL-ADMIN
 webvpn
  anyconnect profiles value PROF-ADMIN type user
```

**Group Policy 적용 순서** (위가 우선):
1. DAP (Dynamic Access Policy)
2. User Attributes (사용자별 설정)
3. Group Policy (Connection Profile에 연결)
4. Default Group Policy (DfltGrpPolicy)

### DAP (Dynamic Access Policy)

DAP는 **접속 시점의 조건을 실시간 평가**하여 정책을 동적으로 적용합니다. Group Policy가 "누구인가"에 따른 정책이라면, DAP는 "어떤 상태로 접속하는가"에 따른 정책입니다.

### DAP 평가 기준

| 기준 | 예시 |
|------|------|
| AAA 속성 | LDAP 그룹 멤버십, RADIUS 속성 |
| Endpoint 속성 | OS 종류, 안티바이러스 설치 여부, 방화벽 상태 |
| 인증 방식 | 인증서 유무, MFA 사용 여부 |
| 접속 시간 | 업무 시간 / 야간 |
| 접속 위치 | 특정 IP 대역 |

### DAP 설정 예시 (ASDM에서 주로 설정)

```
! CLI에서는 DAP XML을 직접 수정하기보다 ASDM 권장
! 하지만 개념 이해를 위한 예시:

! 시나리오: 안티바이러스가 없는 PC로 접속 시 제한된 접근만 허용

! DAP Record 1: AV 있는 경우 → Full Access
! Endpoint Condition: AntiVirus.ActiveScan = "Installed"
! Action: ACL = ACL-FULL-ACCESS

! DAP Record 2: AV 없는 경우 → 웹 포털만 허용
! Endpoint Condition: AntiVirus.ActiveScan != "Installed"
! Action: ACL = ACL-WEB-ONLY, AnyConnect = Terminate

! DAP Record 순서/우선순위: Priority 값으로 결정
```

### DAP가 필요한 실무 시나리오

1. **BYOD 환경**: 회사 PC vs 개인 PC에 다른 정책 적용
2. **보안 컴플라이언스**: 패치 미적용 PC는 패치 서버만 접근 허용
3. **부서별 접근제어**: AD 그룹 + 접속 환경 조합으로 세밀한 제어
4. **외부 협력사**: 인증서 없는 접속은 제한된 리소스만 허용

---

## 면접관: "고객사가 보안을 강화하기 위해 2FA(Two-Factor Authentication)를 추가하고 싶다고 합니다. 어떻게 구성하나요?"

### 2FA 구성 방식

Remote Access VPN에서 2FA는 **RADIUS 서버를 통해 MFA 솔루션과 연동**하는 것이 일반적입니다.

### 대표적인 2FA 솔루션

| 솔루션 | 연동 방식 | 특징 |
|--------|-----------|------|
| **Cisco Duo** | RADIUS Proxy (Duo Authentication Proxy) | Cisco 생태계, Push 알림 |
| RSA SecurID | RADIUS | 하드웨어/소프트웨어 토큰 |
| Microsoft Azure MFA | RADIUS (NPS Extension) | Azure AD 통합 |
| Google Authenticator | RADIUS + TOTP | 무료, 간단 |

### Cisco Duo 연동 구성 (가장 흔한 구성)

```
! ── 구성 아키텍처 ──
! AnyConnect → ASA → Duo Auth Proxy (RADIUS) → Duo Cloud → 사용자 모바일 Push

! ── ASA: RADIUS 서버 그룹 설정 ──
aaa-server DUO-MFA protocol radius
aaa-server DUO-MFA (inside) host 10.1.1.50
 key DuoR@diusKey!
 timeout 60
 retry-interval 5
 authentication-port 1812

! ── Tunnel Group에 2차 인증 추가 ──
tunnel-group TG-REMOTE-ACCESS general-attributes
 authentication-server-group AD-SERVERS
 secondary-authentication-server-group DUO-MFA
 ! 1차: AD 인증 (ID/PW)
 ! 2차: Duo RADIUS (Push 알림)

! ── 또는 단일 RADIUS로 AD+Duo 통합 인증 ──
! Duo Auth Proxy가 AD 인증 + Duo Push를 모두 처리
tunnel-group TG-REMOTE-ACCESS general-attributes
 authentication-server-group DUO-MFA
```

### Duo Authentication Proxy 설정 (서버 측)

```ini
; authproxy.cfg (Duo Auth Proxy 설정 파일)

[ad_client]
host=10.1.1.10
service_account_username=svc_duo
service_account_password=P@ssword
search_dn=DC=company,DC=local

[radius_server_auto]
ikey=DI_xxxxxxxxxxxxxxxxxxxx
skey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
api_host=api-xxxxxxxx.duosecurity.com
radius_ip_1=10.1.1.1
radius_secret_1=DuoR@diusKey!
client=ad_client
failmode=safe
port=1812
```

### 사용자 경험 흐름

```
1. 사용자가 AnyConnect에서 ID/PW 입력
2. ASA → AD 서버: 1차 인증 (ID/PW 확인)
3. ASA → Duo Auth Proxy: 2차 인증 요청
4. Duo Auth Proxy → Duo Cloud: Push 알림 전송
5. 사용자 모바일에 Push 알림 도착 → "Approve" 터치
6. Duo Cloud → Duo Auth Proxy → ASA: 인증 성공
7. VPN 터널 수립

! 소요 시간: 약 5~15초 (Push 응답에 따라)
! Timeout 설정 여유 있게 (60초 권장)
```

### 2FA 장애 대비

```
! Duo 서버 장애 시 Failover 설정
aaa-server DUO-MFA (inside) host 10.1.1.50
 timeout 60

aaa-server DUO-MFA (inside) host 10.1.1.51
 timeout 60

! Duo Auth Proxy의 failmode=safe 설정:
! safe = Duo Cloud 연결 불가 시 인증 허용 (가용성 우선)
! secure = Duo Cloud 연결 불가 시 인증 거부 (보안 우선)
! 환경에 따라 선택. 보안 감사 요구 시 secure 권장
```

### 전체 인증 흐름 정리

```
[AnyConnect Client]
      |
      v
[Cisco ASA] ─── Primary Auth ───> [AD/LDAP Server]
      |                              (ID + Password)
      |
      └──── Secondary Auth ───> [Duo Auth Proxy] ───> [Duo Cloud]
                                  (RADIUS)              (Push/SMS/Token)
                                                            |
                                                            v
                                                    [User's Mobile]
                                                    (Approve/Deny)
```
