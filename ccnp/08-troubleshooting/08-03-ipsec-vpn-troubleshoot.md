# 08-03. IPSec VPN 터널 장애

---

### 면접관: "고객사에서 '어제부터 지사 ERP에 접속이 안 된다'고 연락이 왔습니다. 본사-지사 간 IPSec Site-to-Site VPN으로 연결되어 있습니다. 어디부터 확인하시겠어요?"

IPSec VPN 장애는 **Phase 1(IKE SA) → Phase 2(IPSec SA) → 트래픽 흐름** 순서로 확인합니다. 위에서부터 정상인지 보고, 문제가 되는 단계에 집중합니다.

**1단계: Phase 1 (IKE/ISAKMP SA) 상태 확인**

```
! Cisco IOS/ASA
show crypto isakmp sa
! 또는
show crypto ikev2 sa
```

출력 예시:
```
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
203.0.113.1     198.51.100.1    MM_NO_STATE       0    ACTIVE
```

- **QM_IDLE**: Phase 1 정상 완료 (IKEv1)
- **MM_NO_STATE / MM_KEY_EXCH**: Phase 1 협상 중 실패
- **아예 출력이 없음**: Phase 1 시도조차 안 함

**2단계: Phase 2 (IPSec SA) 상태 확인**

```
show crypto ipsec sa
```

출력에서 확인할 핵심:
```
interface: GigabitEthernet0/0
    Crypto map tag: VPN-MAP, local addr 198.51.100.1

   local  ident (addr/mask/prot/port): (10.1.0.0/255.255.0.0/0/0)
   remote ident (addr/mask/prot/port): (10.2.0.0/255.255.0.0/0/0)

   #pkts encaps: 0, #pkts encrypt: 0, #pkts digest: 0
   #pkts decaps: 0, #pkts decrypt: 0, #pkts verify: 0
   #pkts compressed: 0, #pkts decompressed: 0
   #pkts not compressed: 0, #pkts compr. failed: 0
   #pkts not decompressed: 0, #pkts decompress failed: 0
   #send errors 0, #recv errors 0
```

- **encaps/decaps 카운터가 증가하면**: 터널로 트래픽이 흐르는 중
- **전부 0이면**: 터널은 있지만 트래픽이 안 흐르거나, Phase 2 자체가 안 된 상태

**3단계: 기본 연결성 확인**

```
ping 203.0.113.1
! Peer IP로 ICMP 확인 (UDP 500, ESP 허용 여부도 같이 점검)
```

---

### 면접관: "show crypto isakmp sa에서 MM_NO_STATE로 나옵니다. Phase 1이 실패하는 건데, 원인이 뭘까요?"

Phase 1(Main Mode 기준) 실패의 주요 원인은 다음과 같습니다.

#### 1. ISAKMP Policy 불일치

Phase 1에서는 양쪽이 다음 5가지 파라미터에 합의해야 합니다:

| 파라미터 | 설명 |
|---------|------|
| Encryption | AES-256, AES-128, 3DES 등 |
| Hash | SHA-256, SHA-1, MD5 등 |
| Authentication | Pre-Shared Key, RSA, Certificate |
| DH Group | Group 14, 19, 20 등 |
| Lifetime | SA 유효 시간 (초) |

이 중 **하나라도 양쪽이 일치하는 Policy가 없으면** Phase 1이 실패합니다.

```
show crypto isakmp policy
```

출력 예시:
```
Global IKE policy
 Protection suite priority 10
  encryption algorithm:   AES-CBC-256
  hash algorithm:         SHA256
  authentication method:  Pre-Shared Key
  Diffie-Hellman group:   #14 (2048 bit)
  lifetime:               86400
```

양쪽 장비의 출력을 비교하여 **하나라도 다른 파라미터**가 있는지 확인합니다.

#### 2. Pre-Shared Key 불일치

```
show running-config | include crypto isakmp key
! 또는 ASA에서
show running-config | include ikev1 pre-shared-key
```

> PSK는 대소문자를 구분합니다. 복사/붙여넣기 시 공백이나 특수문자 차이를 주의합니다.

#### 디버그로 정확한 실패 지점 확인

```
debug crypto isakmp
debug crypto ipsec
```

로그 예시 (Policy 불일치):
```
ISAKMP: no offers accepted!
ISAKMP: SA negotiation failed
```

로그 예시 (PSK 불일치):
```
ISAKMP: hash comparison failed
```

---

### 면접관: "Phase 1은 해결했는데 이번엔 Phase 2가 안 됩니다. Quick Mode에서 실패해요."

Phase 2(Quick Mode) 실패는 **Transform Set 불일치** 또는 **Proxy ID(Interesting Traffic ACL) 불일치**가 원인인 경우가 대부분입니다.

#### Transform Set 불일치

Phase 2에서는 실제 데이터를 암호화할 때 사용할 파라미터를 협상합니다.

```
show running-config | section crypto ipsec
```

양쪽이 일치해야 하는 항목:
- **ESP Encryption**: AES-256, AES-128 등
- **ESP Integrity(Hash)**: SHA-256, SHA-1 등
- **Mode**: Tunnel Mode / Transport Mode
- **PFS (Perfect Forward Secrecy)**: DH Group (사용 여부 및 Group 번호)

```
! 본사
crypto ipsec transform-set MY-TS esp-aes 256 esp-sha256-hmac
 mode tunnel

! 지사 (불일치 - hash가 다름)
crypto ipsec transform-set MY-TS esp-aes 256 esp-sha-hmac
 mode tunnel
```

#### Proxy ID (Interesting Traffic ACL) 불일치 - Mirror 문제

IPSec은 "어떤 트래픽을 터널로 보낼 것인가"를 ACL로 정의합니다. 이 ACL이 양쪽에서 **Mirror(거울상)**가 되어야 합니다.

```
! 본사 (Source: 본사, Destination: 지사)
access-list 100 permit ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255

! 지사 (Source: 지사, Destination: 본사) - Mirror
access-list 100 permit ip 10.2.0.0 0.0.255.255 10.1.0.0 0.0.255.255
```

만약 본사가 10.1.0.0/16을 정의했는데, 지사가 10.1.1.0/24만 정의했다면 **Proxy ID Mismatch**로 Phase 2가 실패합니다.

디버그 로그:
```
IPSec(validate_proposal_request): proxy identities not supported
```

ASA에서는 더 직관적으로 확인 가능:
```
show vpn-sessiondb l2l
show running-config crypto map
show access-list VPN-TRAFFIC
```

---

### 면접관: "터널은 올라왔는데 트래픽이 안 흐릅니다. encaps/decaps가 전부 0이에요. 이건 뭔가요?"

Phase 1, Phase 2 모두 정상인데 트래픽이 안 흐르는 경우는 **Interesting Traffic이 터널로 유도되지 않는 것**입니다. 주요 원인:

#### 1. 라우팅 문제 - 트래픽이 터널 대신 다른 경로로 나감

```
show ip route 10.2.0.0
! 지사 네트워크로 가는 경로가 Crypto Map이 적용된 인터페이스를 통하는지 확인
```

만약 Default Route나 더 Specific한 Static Route가 있어서 트래픽이 인터넷 쪽으로 나가버리면, Crypto Map ACL에 매칭되지 않아 암호화 없이 나갑니다.

#### 2. NAT와 VPN ACL 순서 문제 (중요!)

NAT가 VPN보다 먼저 처리되면, Source IP가 변환된 후 VPN ACL과 매칭을 시도하므로 실패합니다.

**IOS에서 NAT Exemption:**
```
! VPN 트래픽은 NAT에서 제외
access-list 110 deny   ip 10.1.0.0 0.0.255.255 10.2.0.0 0.0.255.255
access-list 110 permit ip 10.1.0.0 0.0.255.255 any

ip nat inside source list 110 interface GigabitEthernet0/0 overload
```

**ASA에서 NAT Exemption (Identity NAT):**
```
nat (inside,outside) source static OBJ-LOCAL OBJ-LOCAL destination static OBJ-REMOTE OBJ-REMOTE no-proxy-arp route-lookup
```

#### 3. Crypto Map이 인터페이스에 적용되지 않음

```
show running-config interface GigabitEthernet0/0
! "crypto map VPN-MAP" 라인이 있는지 확인
```

의외로 설정은 다 했는데 마지막에 인터페이스 적용을 빼먹는 경우가 있습니다.

#### 4. Reverse Route 없음 (ASA)

```
! ASA에서 VPN을 통한 리턴 트래픽 라우팅
crypto map VPN-MAP 10 set reverse-route
! 또는 수동 Static Route
route outside 10.2.0.0 255.255.0.0 203.0.113.1 tunneled
```

---

### 면접관: "고객이 'VPN은 되는데 특정 애플리케이션만 안 된다'고 합니다. 디버깅은 어떻게 하나요?"

특정 애플리케이션만 실패하면 **PMTUD(Path MTU Discovery) 문제**를 가장 먼저 의심합니다. IPSec 오버헤드(ESP Header 등)로 인해 터널의 실효 MTU가 줄어드는데, 큰 패킷을 보내는 애플리케이션(ERP, 파일 공유 등)이 영향을 받습니다.

```
! 터널에서의 MTU 확인
show crypto ipsec sa | include mtu

! 특정 크기의 패킷 테스트
ping 10.2.1.100 size 1400 df-bit
ping 10.2.1.100 size 1500 df-bit
! df-bit 설정으로 단편화 금지 - PMTUD 동작 시뮬레이션
```

IPSec 오버헤드 계산:
- ESP Header: 8 bytes
- ESP Trailer: 2 bytes + Padding
- ESP Auth (SHA-256): 16 bytes
- Outer IP Header (Tunnel Mode): 20 bytes
- **총 약 56~73 bytes** (암호화 알고리즘에 따라 다름)

해결:
```
! TCP MSS 조정 (VPN 인터페이스 또는 전역)
ip tcp adjust-mss 1360
! 또는 Crypto Map에서
crypto ipsec fragmentation before-encryption

! ASA에서
sysopt connection tcpmss 1360
```

IOS/ASA에서 실시간 디버그:
```
! ASA packet-tracer로 특정 트래픽 시뮬레이션
packet-tracer input inside tcp 10.1.1.100 12345 10.2.1.100 443
! VPN 매칭, NAT, ACL 순서를 한번에 확인 가능
```

---

### 면접관: "ISP 구간에서 NAT가 걸려 있어서 VPN이 안 되는 케이스도 있죠? NAT-T에 대해 설명해 주세요."

네, IPSec과 NAT는 기본적으로 호환되지 않습니다. 이유는 두 가지입니다:

**1. ESP 프로토콜에는 포트 번호가 없습니다.** NAT(특히 PAT)는 포트 번호를 기반으로 세션을 추적하는데, ESP는 IP Protocol 50으로 포트 개념이 없어서 NAT 장비가 리턴 트래픽을 올바른 내부 호스트로 매핑할 수 없습니다.

**2. IKE에서 IP를 검증합니다.** IKE 패킷의 Payload에 포함된 IP 주소가 NAT 이후 변경되면 무결성 검증에 실패합니다.

#### NAT-Traversal (NAT-T) 동작

NAT-T는 **ESP 패킷을 UDP 4500으로 캡슐화**하여 NAT를 통과할 수 있게 합니다.

동작 과정:
1. IKE Phase 1의 첫 두 메시지에서 양쪽이 **NAT-T 지원 여부**를 확인 (Vendor ID Payload)
2. **NAT Detection Payload**를 교환하여 경로상에 NAT가 있는지 탐지
3. NAT가 감지되면 **IKE 포트를 UDP 500 → UDP 4500으로 전환**
4. Phase 2 이후 **ESP 패킷도 UDP 4500으로 캡슐화**하여 전송

```
! NAT-T 활성화 (대부분 기본 활성화)
crypto isakmp nat-traversal 20
! 20초마다 Keepalive 전송 (NAT 세션 유지)

! ASA에서
crypto isakmp nat-traversal 30
```

NAT-T가 활성화된 상태의 확인:
```
show crypto isakmp sa detail
! 포트가 4500으로 표시되면 NAT-T 동작 중

show crypto ipsec sa
! UDP encap이 표시됨
```

방화벽에서 열어야 할 포트:
| 프로토콜/포트 | 용도 |
|-------------|------|
| UDP 500 | IKE (초기 협상) |
| UDP 4500 | IKE + ESP over NAT-T |
| IP Protocol 50 | ESP (NAT 없을 때) |

> 중간에 NAT가 있으면 IP Protocol 50은 필요 없고, UDP 4500만 열면 됩니다.

---

### 면접관: "IPSec VPN 장애를 사전에 방지하려면 어떤 점을 관리해야 하나요?"

#### 설정 표준화 템플릿

양쪽 장비의 설정을 대조할 수 있는 체크리스트:

| 항목 | 본사 | 지사 | 일치 여부 |
|------|------|------|----------|
| IKE Version | IKEv2 | IKEv2 | O |
| Encryption (Phase 1) | AES-256 | AES-256 | O |
| Hash (Phase 1) | SHA-256 | SHA-256 | O |
| DH Group | 14 | 14 | O |
| Authentication | PSK | PSK | O |
| PSK 값 | *** | *** | **확인 필요** |
| Encryption (Phase 2) | AES-256 | AES-256 | O |
| Hash (Phase 2) | SHA-256 | SHA-256 | O |
| PFS | Group 14 | Group 14 | O |
| Proxy ACL | Mirror 확인 | Mirror 확인 | **확인 필요** |
| NAT Exemption | 설정됨 | 설정됨 | O |

#### 모니터링

```
! SNMP Trap 설정
snmp-server enable traps ipsec

! Syslog 모니터링 키워드
! %CRYPTO-4-RECVD_PKT_NOT_IPSEC
! %CRYPTO-4-IKMP_NO_SA
! %CRYPTO-4-PKT_REPLAY_ERR

! Phase 2 Lifetime 만료 전 Rekey 확인
show crypto ipsec sa | include remaining
```

#### IKEv2로 마이그레이션 권장

IKEv1 대비 IKEv2의 장점:
- **메시지 수 감소**: Main Mode 6개 → IKEv2 4개 (빠른 협상)
- **NAT-T 내장**: 별도 설정 불필요
- **EAP 지원**: 다양한 인증 방법
- **Dead Peer Detection (DPD) 내장**
- **CHILD_SA Rekey가 더 안정적**

```
! IKEv2 기본 설정 예시
crypto ikev2 proposal PROP-1
 encryption aes-cbc-256
 integrity sha256
 group 14

crypto ikev2 policy POL-1
 proposal PROP-1

crypto ikev2 keyring KR-1
 peer BRANCH
  address 203.0.113.1
  pre-shared-key MySecureKey123!

crypto ikev2 profile PROF-1
 match identity remote address 203.0.113.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KR-1
```

#### DPD (Dead Peer Detection) 설정

```
! 상대방이 죽었는지 감지하여 SA를 정리
crypto isakmp keepalive 10 retry 3
! 10초 간격, 3회 실패 시 Peer Dead 판정

! IKEv2에서는 Profile에서 설정
crypto ikev2 profile PROF-1
 dpd 10 3 on-demand
 ! on-demand: 트래픽이 없을 때만 DPD 전송 (효율적)
 ! periodic: 항상 주기적으로 전송
```

---
