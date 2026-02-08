# 05-01. IPSec Site-to-Site VPN (Cisco-Cisco)

---

### 면접관: "고객사 본사와 지사 간에 보안 통신이 필요한데, 전용선 예산이 없다고 합니다. 인터넷 회선만 있는 상황인데 어떻게 구성하시겠어요?"

본사와 지사 모두 인터넷 회선이 있고 고정 공인 IP를 보유하고 있다면, **IPSec Site-to-Site VPN**으로 구성하겠습니다.

전용선 대비 비용이 거의 들지 않으면서도, IPSec이 제공하는 **기밀성(Encryption)**, **무결성(Integrity)**, **인증(Authentication)**을 통해 공용 인터넷 위에서도 안전한 통신이 가능합니다.

양쪽 라우터(또는 ASA)에 IPSec VPN을 구성하면 내부 대역 간 트래픽이 자동으로 암호화되어 전달됩니다.

> 토폴로지 이미지 추가 예정

#### 전체 설계 개요

| 항목 | 본사 (HQ) | 지사 (Branch) |
|------|-----------|---------------|
| 장비 | Cisco ISR 4331 | Cisco ISR 4221 |
| WAN IP | 203.0.113.1 | 198.51.100.1 |
| 내부 대역 | 10.1.0.0/16 | 10.2.0.0/16 |
| IKE Version | IKEv1 (Phase 1 + Phase 2) | IKEv1 (Phase 1 + Phase 2) |

#### 핵심 설정 (HQ Router)

```
! ── IKE Phase 1 Policy (ISAKMP) ──
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

! ── Pre-Shared Key ──
crypto isakmp key C0mpl3x!Key address 198.51.100.1

! ── IKE Phase 2 (IPSec Transform Set) ──
crypto ipsec transform-set TSET-AES256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Interesting Traffic 정의 ──
access-list 100 permit ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255

! ── Crypto Map ──
crypto map CMAP-BRANCH 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set TSET-AES256
 match address 100

! ── 인터페이스 적용 ──
interface GigabitEthernet0/0/0
 crypto map CMAP-BRANCH
```

#### 핵심 설정 (Branch Router)

```
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key C0mpl3x!Key address 203.0.113.1

crypto ipsec transform-set TSET-AES256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── ACL은 HQ의 mirror (출발지/목적지 반대) ──
access-list 100 permit ip 10.2.0.0 0.0.255.255 10.1.0.0 0.0.255.255

crypto map CMAP-HQ 10 ipsec-isakmp
 set peer 203.0.113.1
 set transform-set TSET-AES256
 match address 100

interface GigabitEthernet0/0/0
 crypto map CMAP-HQ
```

---

### 면접관: "IPSec에서 IKE Phase 1과 Phase 2가 정확히 뭔지 설명해주세요. 왜 두 단계로 나눠져 있죠?"

IPSec VPN이 두 단계로 나뉜 이유는 **보안 협상의 분리**와 **효율성** 때문입니다.

#### IKE Phase 1 (ISAKMP SA)

- **목적**: 양쪽 피어 간 **안전한 관리 채널(ISAKMP SA)**을 만드는 단계
- **하는 일**:
  - 암호화 알고리즘, 해시, 인증 방식, DH 그룹, Lifetime 협상
  - Pre-Shared Key 또는 인증서로 상대방 신원 확인
  - Diffie-Hellman 키 교환으로 공유 비밀키 생성
- **결과물**: ISAKMP SA (양방향, 1개)
- **Mode**: Main Mode (6 packets) 또는 Aggressive Mode (3 packets)

#### IKE Phase 2 (IPSec SA)

- **목적**: Phase 1에서 만든 보안 채널 위에서 **실제 데이터 암호화 정책(IPSec SA)**을 협상
- **하는 일**:
  - Transform Set(암호화 + 무결성 알고리즘) 협상
  - Interesting Traffic(어떤 트래픽을 암호화할지) 확인
  - 선택적으로 PFS(Perfect Forward Secrecy)용 추가 DH 교환
- **결과물**: IPSec SA (단방향, 2개 - 인바운드/아웃바운드 각 1개)
- **Mode**: Quick Mode (3 packets)

**두 단계로 나눈 이유**는, Phase 1 SA를 한 번 만들어 놓으면 그 위에서 Phase 2 SA를 여러 개 빠르게 만들 수 있기 때문입니다. 예를 들어 본사-지사 간 서브넷 조합이 여러 개면 Phase 2 SA만 추가하면 됩니다.

```
! Phase 1 SA 확인
show crypto isakmp sa

! Phase 2 SA 확인
show crypto ipsec sa
```

---

### 면접관: "구성을 마쳤는데, VPN 터널이 맺어졌다가 끊어졌다가 반복합니다. 어디부터 확인하시겠어요?"

IPSec 터널의 Flapping은 실무에서 매우 흔한 문제입니다. 다음 순서로 확인하겠습니다.

#### 1단계: SA 상태 확인

```
show crypto isakmp sa
```

출력에서 `QM_IDLE` 상태면 Phase 1은 정상입니다. `MM_NO_STATE`나 `AG_NO_STATE`가 반복되면 Phase 1 자체가 불안정한 것입니다.

#### 2단계: 디버그로 실패 지점 특정

```
debug crypto isakmp
debug crypto ipsec
```

#### 3단계: 주요 원인 체크리스트

| 증상 | 가능한 원인 | 확인 방법 |
|------|------------|-----------|
| Phase 1이 주기적으로 끊김 | SA Lifetime 불일치 | 양쪽 `lifetime` 값 비교 |
| Phase 2만 주기적으로 끊김 | IPSec SA Lifetime 불일치 | `show crypto ipsec sa` 에서 remaining lifetime 확인 |
| 트래픽 없을 때 끊김 | DPD(Dead Peer Detection) 설정 | DPD keepalive 확인 |
| 간헐적 끊김 | ISP 구간 패킷 드롭 | `ping` + `traceroute` + ISP 확인 |
| NAT 장비 뒤에서 끊김 | NAT-T 미설정 또는 NAT 테이블 타임아웃 | UDP 4500 확인 |

#### 4단계: 가장 흔한 원인 - Lifetime 불일치

양쪽의 Lifetime이 다르면, 한쪽이 SA를 갱신하려 할 때 상대방은 아직 유효하다고 판단해서 rekey 과정에서 불일치가 발생합니다.

```
! 양쪽 동일하게 맞춰야 함
! Phase 1
crypto isakmp policy 10
 lifetime 86400

! Phase 2
crypto ipsec security-association lifetime seconds 3600
```

#### 5단계: DPD 설정 확인

DPD가 설정되어 있지 않으면 상대방이 죽었는지 모르고 Stale SA가 남아서, 새로운 SA를 만들 때 충돌합니다.

```
! DPD 설정 (10초 간격, 5번 실패 시 터널 재시작)
crypto isakmp keepalive 10 5 periodic
```

---

### 면접관: "SA Lifetime과 DPD에 대해 좀 더 자세히 설명해주세요. Lifetime은 왜 Phase 1이 Phase 2보다 길죠?"

#### SA Lifetime

SA Lifetime은 **해당 SA가 유효한 시간**입니다. 만료되면 새로운 키로 SA를 재협상(rekey)합니다.

| 구분 | 기본값 | 권장값 | 이유 |
|------|--------|--------|------|
| Phase 1 (ISAKMP) | 86400초 (24시간) | 86400초 | 관리 채널이므로 상대적으로 길게 |
| Phase 2 (IPSec) | 3600초 (1시간) | 3600초 또는 그 이하 | 실제 데이터 암호화 키이므로 자주 교체 |

Phase 1이 Phase 2보다 긴 이유는 **Phase 2 rekey 시 Phase 1 SA를 사용**하기 때문입니다. Phase 1이 먼저 만료되면 Phase 2를 갱신할 보안 채널이 없어져서 처음부터 다시 해야 합니다.

**핵심 공식**: Phase 1 Lifetime > Phase 2 Lifetime (최소 2배 이상 권장)

#### DPD (Dead Peer Detection) - RFC 3706

DPD는 상대 피어가 살아있는지 확인하는 메커니즘입니다.

- **On-demand 방식**: 보낼 트래픽이 있을 때만 상대방 생존 확인 (Cisco 기본)
- **Periodic 방식**: 주기적으로 R-U-THERE 메시지를 보내서 확인

```
! On-demand (트래픽 있을 때만 확인)
crypto isakmp keepalive 10 5

! Periodic (주기적으로 확인 - 권장)
crypto isakmp keepalive 10 5 periodic
```

DPD가 없으면 **Stale SA 문제**가 발생합니다. 한쪽 라우터가 리부팅되면 상대방은 아직 이전 SA가 유효하다고 판단해서 새로운 Phase 1 협상을 거부합니다. 이 경우 수동으로 `clear crypto isakmp` 해야 하는데, DPD가 있으면 자동으로 감지하고 재협상합니다.

---

### 면접관: "IKEv1과 IKEv2의 차이점은 뭐가 있나요? 고객사에 IKEv2를 권장할 만한 이유가 있습니까?"

네, 신규 구축이라면 **IKEv2를 강력히 권장**합니다.

#### IKEv1 vs IKEv2 비교

| 항목 | IKEv1 | IKEv2 |
|------|-------|-------|
| RFC | RFC 2409 | RFC 7296 |
| 협상 패킷 수 | Phase 1: 6~9개 + Phase 2: 3개 | 총 4개 (IKE_SA_INIT 2 + IKE_AUTH 2) |
| NAT-T | 별도 설정 필요 | 기본 내장 |
| DPD | 별도 설정 | 기본 내장 (Liveness Check) |
| EAP 인증 | 미지원 | 지원 (Remote Access에 유리) |
| 안정성 | Aggressive Mode 보안 취약점 | Mode 개념 없음, 보안 강화 |
| Anti-DoS | 없음 | Cookie Challenge 메커니즘 |
| SA 관리 | Phase 1 SA + Phase 2 SA 별도 | IKE SA + Child SA 통합 관리 |
| Rekey | 복잡한 절차 | CREATE_CHILD_SA로 단순화 |

#### IKEv2 핵심 장점

1. **협상 속도**: 4개 메시지로 완료되므로 터널 수립이 빠름
2. **NAT 환경 대응**: NAT-T가 자동이므로 추가 설정 불요
3. **안정성**: Rekey 과정이 깔끔해서 터널 flapping 감소
4. **Mobile 지원**: EAP 인증으로 Remote Access VPN에도 적합

#### IKEv2 설정 예시

```
! ── IKEv2 Proposal ──
crypto ikev2 proposal PROP-AES256
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── IKEv2 Policy ──
crypto ikev2 policy POL-SITE
 proposal PROP-AES256

! ── IKEv2 Keyring ──
crypto ikev2 keyring KR-BRANCH
 peer BRANCH
  address 198.51.100.1
  pre-shared-key C0mpl3x!Key

! ── IKEv2 Profile ──
crypto ikev2 profile PROF-BRANCH
 match identity remote address 198.51.100.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-BRANCH

! ── IPSec Profile (Phase 2는 동일 개념) ──
crypto ipsec transform-set TSET-AES256 esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto map CMAP-BRANCH 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set TSET-AES256
 set ikev2-profile PROF-BRANCH
 match address 100
```

---

### 면접관: "고객사가 보안 감사에서 PFS(Perfect Forward Secrecy) 적용을 요구받았습니다. 이게 뭔지 설명하고, 적용해주세요."

#### PFS란?

PFS는 **Phase 2에서 독립적인 Diffie-Hellman 키 교환을 추가로 수행**하는 것입니다.

일반적으로 Phase 2의 세션 키는 Phase 1에서 만든 공유 비밀키로부터 파생됩니다. 만약 Phase 1의 키가 미래에 유출되면, 이전에 캡처해둔 Phase 2 트래픽도 모두 복호화될 위험이 있습니다.

PFS를 적용하면 **Phase 2마다 별도의 DH 교환**을 수행하므로, Phase 1 키가 유출되더라도 각 Phase 2 세션의 데이터는 안전합니다. 즉, 하나의 키 손상이 다른 세션에 영향을 주지 않습니다.

**트레이드오프**: DH 연산이 추가되므로 rekey 시 CPU 부하가 약간 증가합니다. 하지만 보안 감사 기준에서는 거의 필수 항목입니다.

#### PFS 적용 설정

```
! ── IKEv1 환경 ──
crypto map CMAP-BRANCH 10 ipsec-isakmp
 set pfs group14

! ── IKEv2 환경 (IPSec Profile에서 설정) ──
crypto ipsec profile IPSEC-PROF
 set pfs group14

! ── 적용 확인 ──
show crypto ipsec sa | include PFS
show crypto map
```

#### DH Group 선택 가이드

| DH Group | 비트 수 | 보안 강도 | 사용 권장 |
|----------|---------|-----------|-----------|
| Group 2 | 1024-bit | 취약 (deprecated) | 사용 금지 |
| Group 5 | 1536-bit | 최소한 | 레거시 호환 시에만 |
| Group 14 | 2048-bit | 적정 | **일반 권장** |
| Group 19 | 256-bit ECP | 높음 | ECDH, 성능 우수 |
| Group 20 | 384-bit ECP | 매우 높음 | 고보안 환경 |

**주의사항**: PFS에 사용하는 DH Group은 **양쪽이 반드시 동일**해야 합니다. 불일치 시 Phase 2 협상이 실패하며, 디버그에서 `INVALID_KEY_INFORMATION` 또는 `NO_PROPOSAL_CHOSEN` 에러가 나타납니다.

```
! 장애 발생 시 확인
debug crypto ipsec
! "no acceptable transform" → PFS group 불일치 의심
```

---

### 면접관: "마지막으로 정리하면, 이 구성의 한계점과 보완책은 뭐가 있을까요?"

#### Policy-based IPSec VPN의 한계

1. **정적 피어(Static Peer)**: 양쪽 모두 고정 IP 필요. 한쪽이 유동 IP면 추가 설정(Dynamic Crypto Map) 필요
2. **확장성 부족**: 지사가 늘어날 때마다 ACL + Crypto Map 수동 추가. 지사 10개 이상이면 관리가 어려움
3. **동적 라우팅 불가**: Crypto Map 기반 VPN은 GRE 같은 터널 없이는 OSPF/EIGRP 불가
4. **Interesting Traffic 관리**: ACL이 복잡해지면 실수 가능성 증가

#### 보완책

| 한계 | 보완책 | 참고 |
|------|--------|------|
| 확장성 | **DMVPN** 또는 **FlexVPN** | 지사 50개 이상 시 필수 |
| 동적 라우팅 | **GRE over IPSec** | 터널 인터페이스로 라우팅 프로토콜 구동 |
| 유동 IP | **Dynamic Crypto Map** 또는 **IKEv2 FlexVPN** | Responder 측에서 any 피어 허용 |
| 이중화 | **Dual Hub** + **IP SLA** | 회선 장애 시 자동 절체 |

현재 고객사가 본사-지사 1:1 구성이고 고정 IP가 있다면 이 구성이 충분하지만, 향후 지사 확장 가능성이 있다면 처음부터 **DMVPN**이나 **VTI(Virtual Tunnel Interface)** 기반으로 설계하는 것을 제안드리겠습니다.
