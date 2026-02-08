# 05-02. IPSec Site-to-Site VPN (Cisco-Checkpoint)

---

## 면접관: "고객사가 최근 인수합병을 해서, 본사는 Checkpoint 방화벽이고 지사는 Cisco ASA를 쓰고 있습니다. 이 두 장비 간 Site-to-Site VPN을 연결해야 하는데, 이기종 간 VPN 구성 경험 있으세요? 어떻게 접근하시겠어요?"

이기종 VPN은 실무에서 매우 자주 마주치는 상황입니다. 핵심은 **양쪽이 합의할 수 있는 공통 파라미터를 정확히 맞추는 것**입니다.

IPSec은 표준 프로토콜(RFC 기반)이기 때문에 벤더가 달라도 VPN 연결이 가능합니다. 다만 각 벤더마다 **기본값, 용어, 동작 방식**이 다르기 때문에 사전에 꼼꼼하게 파라미터를 합의해야 합니다.

> 토폴로지 이미지 추가 예정

### 양쪽 파라미터 합의 시트 (사전 작성 필수)

| 항목 | Checkpoint (본사) | Cisco ASA (지사) |
|------|-------------------|------------------|
| Peer IP | 203.0.113.1 | 198.51.100.1 |
| Internal Network | 10.1.0.0/16 | 10.2.0.0/16 |
| IKE Version | IKEv1 Main Mode | IKEv1 Main Mode |
| Phase 1 Encryption | AES-256 | AES-256 |
| Phase 1 Hash | SHA-256 | SHA-256 |
| Phase 1 DH Group | Group 14 (2048-bit) | Group 14 |
| Phase 1 Lifetime | 86400초 | 86400초 |
| Authentication | Pre-Shared Key | Pre-Shared Key |
| Phase 2 Encryption | AES-256 | AES-256 |
| Phase 2 Hash | SHA-256 | SHA-256 |
| Phase 2 PFS | Group 14 | Group 14 |
| Phase 2 Lifetime | 3600초 | 3600초 |

### 핵심 설정 - Cisco ASA 측

```
! ── IKEv1 Phase 1 ──
crypto ikev1 policy 10
 authentication pre-share
 encryption aes-256
 hash sha
 group 14
 lifetime 86400

! ── Phase 1 Tunnel Group ──
tunnel-group 203.0.113.1 type ipsec-l2l
tunnel-group 203.0.113.1 ipsec-attributes
 ikev1 pre-shared-key C0mpl3x!Key

! ── Phase 2 Transform Set ──
crypto ipsec ikev1 transform-set TSET-CP esp-aes-256 esp-sha-hmac

! ── Interesting Traffic (Checkpoint의 Encryption Domain과 Mirror) ──
access-list ACL-VPN-CP extended permit ip 10.2.0.0 255.255.0.0 10.1.0.0 255.255.0.0

! ── Crypto Map ──
crypto map CMAP-OUTSIDE 10 match address ACL-VPN-CP
crypto map CMAP-OUTSIDE 10 set peer 203.0.113.1
crypto map CMAP-OUTSIDE 10 set ikev1 transform-set TSET-CP
crypto map CMAP-OUTSIDE 10 set pfs group14
crypto map CMAP-OUTSIDE 10 set security-association lifetime seconds 3600

crypto map CMAP-OUTSIDE interface outside

! ── NAT Exemption (필수!) ──
nat (inside,outside) source static OBJ-NET-10.2 OBJ-NET-10.2 destination static OBJ-NET-10.1 OBJ-NET-10.1 no-proxy-arp route-lookup
```

### 핵심 설정 - Checkpoint SmartConsole 측

```
1. SmartConsole > VPN Communities > Star Community 또는 Meshed Community 생성
2. Participants:
   - Center Gateway: 본사 Checkpoint GW
   - Satellite Gateway: "Interoperable Device"로 Cisco ASA 등록

3. Interoperable Device 생성:
   - New > Network Object > Interoperable Device
   - Name: BRANCH-ASA
   - IP: 198.51.100.1
   - VPN Domain (Encryption Domain): 10.2.0.0/16

4. VPN Community 설정:
   - Encryption: Phase 1 - AES-256 / SHA-256 / Group 14
   - Encryption: Phase 2 - AES-256 / SHA-256
   - Phase 1 SA Lifetime: 1440 minutes (= 86400초)
   - Phase 2 SA Lifetime: 3600초
   - PFS: Group 14 체크

5. Shared Secret:
   - VPN Community > Participant의 Shared Secret에 동일한 PSK 입력

6. Policy 설치 (Install Policy)
```

---

## 면접관: "Checkpoint에서 'Encryption Domain'이라는 용어가 나왔는데, 이게 정확히 뭔가요? Cisco의 어떤 개념에 대응되죠?"

**Encryption Domain**은 Checkpoint 용어로, **VPN을 통해 보호할 내부 네트워크 범위**를 의미합니다.

### 벤더별 용어 매핑

| 개념 | Checkpoint | Cisco ASA | Cisco IOS |
|------|-----------|-----------|-----------|
| 보호 대상 네트워크 | **Encryption Domain** | **Interesting Traffic (ACL)** | **Interesting Traffic (ACL)** |
| VPN 정책 | **VPN Community** | **Crypto Map** | **Crypto Map** |
| 상대방 장비 등록 | **Interoperable Device** | **Tunnel Group** | **crypto isakmp key ... address** |

### Checkpoint Encryption Domain의 특성

Checkpoint에서는 **게이트웨이 객체의 Topology 탭**에서 VPN Domain을 정의합니다.

- **Derived from topology**: 게이트웨이 뒤에 정의된 모든 네트워크를 자동으로 VPN Domain에 포함
- **Manual**: 관리자가 직접 Network Group으로 지정

**중요 포인트**: Checkpoint은 Encryption Domain을 **네트워크 객체(Object) 단위**로 정의하고, Cisco는 **ACL 단위**로 정의합니다. 이 차이가 이기종 VPN 장애의 가장 큰 원인입니다.

Checkpoint 측 Encryption Domain과 Cisco 측 ACL이 **정확히 Mirror 관계**여야 합니다:

```
! Cisco ASA ACL:
!   Source: 10.2.0.0/16 (Cisco 내부)
!   Destination: 10.1.0.0/16 (Checkpoint 내부)

! Checkpoint Encryption Domain:
!   Local: 10.1.0.0/16
!   Remote (Interoperable Device의 VPN Domain): 10.2.0.0/16
```

---

## 면접관: "VPN 구성을 했는데 Phase 1은 올라오는데 Phase 2가 안 맺어집니다. 디버그를 보니 'INVALID_ID_INFORMATION'이 나옵니다. 어디를 보시겠어요?"

이기종 VPN에서 Phase 2 실패의 가장 흔한 원인은 **Proxy ID(Encryption Domain) 불일치**입니다.

### 원인 분석

Phase 2 협상 시 양쪽은 **Proxy ID**(어떤 트래픽을 보호할지)를 교환합니다. 이 Proxy ID가 정확히 일치하지 않으면 `INVALID_ID_INFORMATION`이 발생합니다.

### 장애 시나리오별 원인

#### 시나리오 1: ACL이 Mirror가 아닌 경우

```
! Cisco ASA:
access-list ACL-VPN permit ip 10.2.0.0 255.255.0.0 10.1.0.0 255.255.0.0
! → Proxy ID: 10.2.0.0/16 ↔ 10.1.0.0/16

! Checkpoint Encryption Domain:
! Local: 10.1.0.0/16
! Remote: 10.2.1.0/24 (실수로 /24로 설정)
! → Proxy ID: 10.1.0.0/16 ↔ 10.2.1.0/24

! 결과: Proxy ID 불일치 → Phase 2 실패
```

#### 시나리오 2: Supernet 문제 (가장 흔함)

Checkpoint에서 Encryption Domain에 여러 서브넷을 넣으면, Checkpoint이 이를 **Supernet으로 합치거나 개별 SA로 분리**하는데, Cisco 측 ACL과 맞지 않으면 실패합니다.

```
! Checkpoint Encryption Domain에 여러 네트워크:
! 10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24
! → Checkpoint이 각각 별도 Phase 2 SA를 만들려고 시도

! Cisco ASA ACL:
access-list ACL-VPN permit ip 10.2.0.0 255.255.0.0 10.1.0.0 255.255.0.0
! → Cisco는 10.1.0.0/16 하나로 Proxy ID 제안

! 결과: Proxy ID 불일치
```

### 해결 방법

```
! ── 방법 1: Cisco ASA ACL을 Checkpoint Encryption Domain과 정확히 매칭 ──
access-list ACL-VPN permit ip 10.2.0.0 255.255.0.0 10.1.1.0 255.255.255.0
access-list ACL-VPN permit ip 10.2.0.0 255.255.0.0 10.1.2.0 255.255.255.0
access-list ACL-VPN permit ip 10.2.0.0 255.255.0.0 10.1.3.0 255.255.255.0

! ── 방법 2: Checkpoint에서 Encryption Domain을 Supernet으로 통합 ──
! SmartConsole에서 Network Group 대신 10.1.0.0/16 단일 네트워크 객체 사용

! ── 확인 명령어 (Cisco ASA) ──
show crypto isakmp sa
show crypto ipsec sa
debug crypto ikev1 127
debug crypto ipsec 127
```

### Checkpoint 측 확인

```
# SmartView Tracker 또는 SmartLog에서 VPN 관련 로그 확인
# Expert Mode CLI:
fw ctl zdebug drop | grep -i encrypt
vpn debug trunc
vpn debug on TDERROR_ALL_ALL=5
```

---

## 면접관: "고객사 환경에 NAT 장비가 중간에 있을 수도 있다고 합니다. NAT-Traversal은 어떻게 처리하나요?"

IPSec 패킷이 NAT 장비를 통과하면 **ESP 프로토콜(IP Protocol 50)의 헤더가 변조**되어 무결성 검증에 실패합니다. 이를 해결하는 것이 **NAT-T(NAT Traversal)**입니다.

### NAT-T 동작 원리

1. IKE Phase 1 협상 시, 양쪽이 **NAT-D(NAT Discovery) 페이로드**를 교환
2. IP + Port의 해시값을 비교하여 중간에 NAT가 있는지 탐지
3. NAT가 감지되면, **ESP 패킷을 UDP 4500으로 캡슐화**하여 전송
4. UDP 헤더가 있으므로 NAT 장비가 정상적으로 포트 변환 가능

### 설정

```
! ── Cisco ASA (기본적으로 NAT-T 활성화됨) ──
! 확인:
show running-config crypto | include nat-traversal
! 기본값: crypto isakmp nat-traversal 20

! 명시적 설정 (비활성화 되어 있다면):
crypto isakmp nat-traversal 20

! ── Cisco IOS Router ──
crypto isakmp nat-traversal
```

### Checkpoint 측 NAT-T 설정

```
1. SmartConsole > Gateway Properties > VPN > NAT-T 설정
   - "Support NAT traversal" 체크 (기본 활성화)
   - UDP Encapsulation 활성화

2. Global Properties > VPN > Advanced
   - "Enable NAT-T" 체크
   - NAT-T Keep-alive interval: 20초
```

### 방화벽/ACL에서 허용해야 할 포트

| 프로토콜 | 포트 | 용도 |
|----------|------|------|
| UDP | 500 | IKE 협상 |
| UDP | 4500 | NAT-T (ESP over UDP) |
| ESP | Protocol 50 | IPSec (NAT 없을 때) |
| AH | Protocol 51 | IPSec AH (NAT 환경 사용 불가) |

**주의**: AH(Authentication Header) 모드는 전체 IP 헤더를 인증하므로 NAT 환경에서 절대 사용할 수 없습니다. NAT 환경에서는 반드시 **ESP만 사용**해야 합니다.

---

## 면접관: "Checkpoint SmartConsole에서 Cisco를 Interoperable Device로 등록한다고 했는데, 일반 Gateway 객체와 뭐가 다른 건가요?"

### Checkpoint 객체 유형 비교

| 항목 | Check Point Gateway | Interoperable Device |
|------|--------------------|--------------------|
| 용도 | Checkpoint 장비 등록 | **타사(3rd-party) 장비 등록** |
| SIC 통신 | O (Secure Internal Communication) | X |
| SmartConsole 관리 | O | X (상대방 벤더 콘솔로 관리) |
| Anti-Spoofing | 자동 설정 | 수동 설정 필요 |
| VPN Domain | Topology에서 자동 도출 가능 | **반드시 수동 지정** |
| 인증 방식 | 인증서 (기본) | **Pre-Shared Key (일반적)** |

### Interoperable Device 생성 상세 절차

```
1. SmartConsole > New > Network Object > More > Interoperable Device

2. General Properties:
   - Name: BRANCH-CISCO-ASA
   - IP Address: 198.51.100.1 (Cisco ASA WAN IP)

3. Topology 탭:
   - VPN Domain: Manual 선택
   - Network Group 지정: GRP-BRANCH-NETS (10.2.0.0/16 포함)

4. IPSec VPN 탭:
   - Traditional mode 또는 Simplified mode
   - Link Selection: Main IP address 사용

5. VPN Community에 Satellite로 추가
   - Shared Secret 설정
```

### 주의사항

- Interoperable Device의 **VPN Domain 설정이 Cisco ASA ACL의 Source 네트워크와 정확히 일치**해야 합니다
- Checkpoint은 기본적으로 **인증서 기반 인증**을 선호하지만, Interoperable Device는 PSK를 사용합니다
- Checkpoint에서 Policy 설치 시, Interoperable Device에는 Policy가 Push되지 않으므로 **Cisco 측 설정은 별도로 해야** 합니다

---

## 면접관: "이기종 VPN 구성 시 자주 실수하는 부분이나 체크리스트가 있으면 정리해주세요."

### 이기종 VPN 구축 체크리스트

#### Phase 1 체크

| # | 체크 항목 | 흔한 실수 |
|---|----------|-----------|
| 1 | Encryption 알고리즘 동일 | Checkpoint: AES-256, Cisco: AES-128 (기본값 차이) |
| 2 | Hash 알고리즘 동일 | Checkpoint이 SHA-256 사용 시 Cisco에서 `sha256` 명시 필요 |
| 3 | DH Group 동일 | Checkpoint 기본 Group 2, Cisco 기본 Group 5 → 불일치 |
| 4 | Lifetime 동일 | Checkpoint: 1440분, Cisco: 86400초 → 단위 변환 확인 |
| 5 | PSK 동일 | 복사-붙여넣기 시 공백/특수문자 오류 |
| 6 | IKE 버전 동일 | 한쪽만 IKEv2 시도하면 실패 |

#### Phase 2 체크

| # | 체크 항목 | 흔한 실수 |
|---|----------|-----------|
| 7 | Transform Set 동일 | ESP-AES vs ESP-AES-256 혼동 |
| 8 | PFS 동일 | 한쪽만 PFS 활성화 → 실패 |
| 9 | **Proxy ID / Encryption Domain 정확히 Mirror** | **가장 빈번한 장애 원인** |
| 10 | Phase 2 Lifetime 동일 | 기본값이 벤더마다 다름 |

#### 네트워크 체크

| # | 체크 항목 | 흔한 실수 |
|---|----------|-----------|
| 11 | NAT Exemption 설정 | ASA에서 VPN 트래픽이 NAT 되어버림 |
| 12 | 방화벽에서 UDP 500/4500 허용 | 중간 방화벽에서 차단 |
| 13 | 라우팅 확인 | VPN 트래픽이 기본 라우트로 빠짐 |
| 14 | Anti-Spoofing | Checkpoint에서 VPN 도착 트래픽 차단 |

### 실무 팁

```
! Cisco ASA: Phase 1/2 전체 상태 한눈에 보기
show vpn-sessiondb l2l

! 특정 피어의 상세 SA 정보
show crypto ipsec sa peer 203.0.113.1

! Checkpoint Expert CLI:
cpstat vpn           # VPN 통계
vpn tu               # VPN Tunnel Utility (tunnel list/up/down)
fw ctl pstat         # 전체 터널 상태
```

이기종 VPN의 핵심은 **문서화**입니다. 양쪽 파라미터 합의 시트를 반드시 작성하고, 변경 시 양쪽 모두 동시에 반영해야 합니다. 한쪽만 변경하면 반드시 터널이 끊어집니다.
