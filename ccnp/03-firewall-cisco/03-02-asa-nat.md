# 03-02. ASA NAT (Auto NAT / Manual NAT)

---

### 면접관: "고객사에서 두 가지를 요청했습니다. 첫째, 내부 사용자들이 인터넷에 접속할 수 있어야 하고, 둘째, DMZ에 있는 웹서버(172.16.1.10)를 외부에 공개해야 합니다. NAT 설계를 어떻게 하시겠어요?"

두 가지 NAT를 조합하여 설계하겠습니다.

**1. 내부 사용자 → 인터넷: PAT (Port Address Translation)**

내부 사용자 전체가 하나의 공인 IP(또는 Outside 인터페이스 IP)를 공유하여 인터넷에 접속하도록 PAT를 구성합니다. 수백~수천 명의 사용자가 동시에 접속해도 포트 번호로 세션을 구분할 수 있습니다.

**2. DMZ 웹서버 → 외부 공개: Static NAT**

DMZ 웹서버의 사설 IP(172.16.1.10)를 공인 IP(203.0.113.10)에 1:1로 매핑하여 외부에서 접근 가능하게 합니다.

> 토폴로지 이미지 추가 예정

#### 핵심 설정

```
! 1. 내부 사용자 PAT (Interface PAT)
object network OBJ_INSIDE_NET
 subnet 192.168.10.0 255.255.255.0
 nat (inside,outside) dynamic interface

! 2. DMZ 웹서버 Static NAT
object network OBJ_DMZ_WEB
 host 172.16.1.10
 nat (dmz,outside) static 203.0.113.10

! 3. 외부에서 웹서버 접근을 위한 ACL (NAT만으로는 접근 불가)
access-list outside_access_in extended permit tcp any host 203.0.113.10 eq www
access-list outside_access_in extended permit tcp any host 203.0.113.10 eq https
access-group outside_access_in in interface outside
```

여기서 중요한 포인트는, **Static NAT를 설정했다고 외부에서 자동으로 접근 가능한 것이 아닙니다**. Outside → DMZ는 Security Level이 낮→높이므로 ACL도 반드시 함께 설정해야 합니다.

---

### 면접관: "지금 설정하신 것이 Auto NAT인가요? Auto NAT과 Manual NAT의 차이를 설명해주세요."

네, 위에서 설정한 방식이 **Auto NAT (Object NAT)**입니다. ASA 8.3 이후 도입된 방식으로, **object network 안에서 nat 명령을 선언**하는 것이 특징입니다.

#### Auto NAT vs Manual NAT 비교

| 구분 | Auto NAT (Object NAT) | Manual NAT (Twice NAT) |
|---|---|---|
| 설정 위치 | `object network` 내부 | 글로벌 설정 (object 외부) |
| NAT 대상 | **하나의 object**만 기준 | **source + destination** 동시 지정 가능 |
| 유연성 | 단순한 1:1, PAT에 적합 | 복잡한 조건부 NAT에 적합 |
| 순서 제어 | 불가 (ASA가 자동 정렬) | 가능 (설정 순서대로 처리) |

**Auto NAT 예시** (위에서 설정한 것):
```
object network OBJ_INSIDE_NET
 subnet 192.168.10.0 255.255.255.0
 nat (inside,outside) dynamic interface
```

**Manual NAT 예시** (동일 기능을 Manual NAT로 작성):
```
nat (inside,outside) source dynamic OBJ_INSIDE_NET interface
```

Manual NAT가 필요한 대표적인 상황:

1. **목적지에 따라 다른 NAT 적용** - 같은 출발지라도 인터넷으로 갈 때는 PAT, VPN으로 갈 때는 NAT 제외
2. **Policy NAT** - 출발지 + 목적지 조합에 따라 NAT 결정
3. **NAT Exemption** - 특정 트래픽을 NAT에서 제외

```
! Manual NAT 예시: 목적지가 VPN 대역이면 NAT 제외
nat (inside,outside) source static OBJ_INSIDE_NET OBJ_INSIDE_NET destination static OBJ_VPN_REMOTE OBJ_VPN_REMOTE no-proxy-arp route-lookup
```

---

### 면접관: "그러면 Auto NAT과 Manual NAT가 동시에 존재할 때, ASA는 어떤 순서로 NAT 규칙을 처리하나요?"

이것은 ASA NAT에서 매우 중요한 개념입니다. ASA는 NAT 규칙을 **Section** 단위로 나누어 처리합니다.

#### NAT Processing Order (Section 1 → 2 → 3)

```
+------------------------------------------+
|  Section 1: Manual NAT (before auto)     |  ← nat (inside,outside) ...
|  - 설정 순서대로 처리 (first match)       |    (line 번호 없거나 before-auto)
+------------------------------------------+
|  Section 2: Auto NAT                     |  ← object network 내부의 nat
|  - ASA가 자동 정렬:                      |
|    1) Static NAT (구체적 → 광범위)       |
|    2) Dynamic NAT/PAT (구체적 → 광범위)  |
+------------------------------------------+
|  Section 3: Manual NAT (after auto)      |  ← nat (inside,outside) after-auto ...
|  - 설정 순서대로 처리 (first match)       |
+------------------------------------------+
```

**Section 2 (Auto NAT) 내부 자동 정렬 규칙**:

1. Static NAT가 Dynamic NAT보다 우선
2. 같은 타입이면 **더 구체적인(서브넷이 작은) 규칙**이 우선
   - `host 172.16.1.10`(32비트) > `172.16.1.0/24`(24비트) > `172.16.0.0/16`(16비트)

이 순서를 이해하면 NAT 관련 트러블슈팅이 훨씬 수월해집니다.

```
! NAT 규칙 순서 확인
show nat

! 출력 예시:
! Manual NAT Policies (Section 1)
! 1 (inside) to (outside) source static OBJ_INSIDE OBJ_INSIDE dest static OBJ_VPN OBJ_VPN
!   translate_hits = 245, untranslate_hits = 312
!
! Auto NAT Policies (Section 2)
! 1 (dmz) to (outside) source static OBJ_DMZ_WEB 203.0.113.10
!   translate_hits = 1024, untranslate_hits = 890
! 2 (inside) to (outside) source dynamic OBJ_INSIDE_NET interface
!   translate_hits = 56789, untranslate_hits = 0
```

`translate_hits`과 `untranslate_hits` 카운터로 각 NAT 규칙이 실제로 매칭되고 있는지 확인할 수 있습니다.

---

### 면접관: "그런데 문제가 생겼습니다. 내부 사용자가 DMZ 웹서버에 공인 IP(203.0.113.10)로 접속하려고 하는데 안 된다고 합니다. 원인이 뭘까요?"

이것은 ASA에서 매우 흔한 문제로, **Hairpin NAT (U-Turn NAT)** 이슈입니다.

Inside 사용자(192.168.10.100) → 공인 IP(203.0.113.10) → 실제로는 DMZ 서버(172.16.1.10)

이 트래픽의 흐름을 분석해 보면:

1. 사용자가 203.0.113.10으로 요청을 보냄
2. 패킷이 ASA의 Inside 인터페이스로 들어옴
3. ASA가 목적지 203.0.113.10을 172.16.1.10으로 un-translate 해야 함
4. 하지만 **기존 Static NAT는 (dmz,outside) 방향**으로만 설정됨
5. Inside에서 온 트래픽은 이 NAT 규칙에 매칭되지 않음

#### 해결 방법

```
! 방법 1: DNS Doctoring (권장 - 내부 DNS가 있는 경우)
object network OBJ_DMZ_WEB
 host 172.16.1.10
 nat (dmz,outside) static 203.0.113.10 dns

! 방법 2: 별도 Manual NAT 규칙 추가 (Hairpin NAT)
! Inside에서 DMZ 웹서버 공인 IP로의 트래픽을 처리
object network OBJ_DMZ_WEB_PUB
 host 203.0.113.10

nat (inside,dmz) source dynamic OBJ_INSIDE_NET interface destination static OBJ_DMZ_WEB_PUB OBJ_DMZ_WEB
```

**방법 1 (DNS Doctoring)**: ASA가 DNS 응답 패킷을 inspect하여 공인 IP를 사설 IP로 변환해줍니다. 내부 사용자가 DNS 쿼리를 하면 172.16.1.10으로 응답을 받게 되어 직접 DMZ로 통신합니다.

**방법 2 (Hairpin NAT)**: DNS를 수정할 수 없는 경우, Inside → DMZ 방향으로 별도 NAT 규칙을 만듭니다. 이 경우 `same-security-traffic permit intra-interface`도 필요할 수 있습니다.

```
! 트러블슈팅: packet-tracer로 확인
packet-tracer input inside tcp 192.168.10.100 12345 203.0.113.10 80 detailed
```

---

### 면접관: "Section 1, 2, 3 순서를 이해했는데, 실무에서 Section 순서 때문에 NAT가 의도대로 동작하지 않았던 경우를 설명해주실 수 있나요?"

대표적인 사례가 **VPN과 인터넷 NAT가 충돌하는 경우**입니다.

**상황**: Inside 네트워크(192.168.10.0/24)에 대해 인터넷용 PAT가 Auto NAT로 설정되어 있고, 이후에 Site-to-Site VPN을 추가하면서 VPN 트래픽은 NAT를 제외해야 하는 상황.

```
! 기존 Auto NAT (Section 2에 위치)
object network OBJ_INSIDE_NET
 subnet 192.168.10.0 255.255.255.0
 nat (inside,outside) dynamic interface
```

여기서 VPN용 NAT Exemption을 Auto NAT로는 구현할 수 없습니다. Auto NAT는 source만 기준으로 하기 때문에 "목적지가 원격 VPN 대역이면 NAT 제외"라는 조건을 걸 수 없습니다.

따라서 **Manual NAT를 Section 1에 배치**하여 Auto NAT보다 먼저 매칭되도록 해야 합니다.

```
! VPN 관련 오브젝트
object network OBJ_LOCAL_LAN
 subnet 192.168.10.0 255.255.255.0
object network OBJ_REMOTE_LAN
 subnet 10.1.1.0 255.255.255.0

! Section 1: Manual NAT - VPN 트래픽 NAT 제외 (Identity NAT)
nat (inside,outside) source static OBJ_LOCAL_LAN OBJ_LOCAL_LAN destination static OBJ_REMOTE_LAN OBJ_REMOTE_LAN no-proxy-arp route-lookup
```

**처리 순서**:
1. Section 1 (Manual NAT): 목적지가 10.1.1.0/24이면 NAT 제외 → **매칭되면 여기서 끝**
2. Section 2 (Auto NAT): 그 외 모든 Inside 트래픽은 PAT → **인터넷 트래픽 처리**

만약 이 Manual NAT를 `after-auto` 키워드로 Section 3에 넣었다면, VPN 트래픽이 Section 2의 PAT에 먼저 매칭되어 **NAT 변환된 상태로 VPN 터널에 들어가게 되고**, VPN interesting traffic에 매칭되지 않아 터널이 형성되지 않는 문제가 발생합니다.

```
! NAT 순서 확인 및 트러블슈팅
show nat
show nat detail
show xlate | include 192.168.10
```

---

### 면접관: "고객사가 추가로 Site-to-Site VPN을 구성해달라고 합니다. 원격지 네트워크는 10.1.1.0/24이고, VPN 트래픽에 대해서는 NAT를 적용하지 않아야 합니다. 전체 NAT 구성을 정리해주시겠어요?"

전체 NAT 정책을 Section별로 정리하여 구성하겠습니다.

#### 전체 NAT 설계

```
!=== 오브젝트 정의 ===
object network OBJ_INSIDE_NET
 subnet 192.168.10.0 255.255.255.0

object network OBJ_REMOTE_VPN
 subnet 10.1.1.0 255.255.255.0

object network OBJ_DMZ_WEB
 host 172.16.1.10

object network OBJ_DMZ_NET
 subnet 172.16.1.0 255.255.255.0

!=== Section 1: Manual NAT (VPN NAT Exemption) ===
! VPN 트래픽은 NAT 하지 않음 (Identity NAT)
nat (inside,outside) 1 source static OBJ_INSIDE_NET OBJ_INSIDE_NET destination static OBJ_REMOTE_VPN OBJ_REMOTE_VPN no-proxy-arp route-lookup

!=== Section 2: Auto NAT ===
! DMZ 웹서버 Static NAT (외부 공개)
object network OBJ_DMZ_WEB
 nat (dmz,outside) static 203.0.113.10

! Inside 전체 PAT (인터넷 접속)
object network OBJ_INSIDE_NET
 nat (inside,outside) dynamic interface

! DMZ 서버 인터넷 접속용 PAT
object network OBJ_DMZ_NET
 nat (dmz,outside) dynamic interface

!=== 최종 확인 ===
```

```
! NAT 규칙 전체 확인
show nat

! 기대 출력:
! Manual NAT Policies (Section 1)
! 1 (inside) to (outside) source static OBJ_INSIDE_NET OBJ_INSIDE_NET
!                         destination static OBJ_REMOTE_VPN OBJ_REMOTE_VPN
!     no-proxy-arp route-lookup
!     translate_hits = 0, untranslate_hits = 0
!
! Auto NAT Policies (Section 2)
! 1 (dmz) to (outside) source static OBJ_DMZ_WEB 203.0.113.10
!     translate_hits = 0, untranslate_hits = 0
! 2 (inside) to (outside) source dynamic OBJ_INSIDE_NET interface
!     translate_hits = 0, untranslate_hits = 0
! 3 (dmz) to (outside) source dynamic OBJ_DMZ_NET interface
!     translate_hits = 0, untranslate_hits = 0
```

```
! VPN 트래픽이 NAT 없이 처리되는지 확인
packet-tracer input inside tcp 192.168.10.100 12345 10.1.1.100 80

! 인터넷 트래픽이 PAT 되는지 확인
packet-tracer input inside tcp 192.168.10.100 12345 8.8.8.8 80

! DMZ 웹서버 외부 접근 확인
packet-tracer input outside tcp 1.1.1.1 12345 203.0.113.10 80
```

---

### 면접관: "NAT 규칙이 많아지면 관리가 복잡해질 것 같은데, 실무에서 NAT 설정 시 주의할 점이나 베스트 프랙티스가 있을까요?"

실무에서 ASA NAT 운영 시 주의할 점을 정리하겠습니다.

**1. Object 네이밍 컨벤션 통일**

```
! 좋은 예 - 용도와 대상이 명확
object network NET_INSIDE_USERS
object network HOST_DMZ_WEBSERVER
object network NET_VPN_REMOTE_SEOUL

! 나쁜 예 - 의미 불명확
object network obj1
object network test_nat
```

**2. NAT 규칙 변경 시 xlate clear 필요 여부 확인**

NAT 규칙을 변경한 후에도 기존 xlate(translation) 엔트리가 남아 있으면 새 규칙이 적용되지 않을 수 있습니다.

```
! xlate 테이블 확인
show xlate

! 특정 호스트의 xlate 삭제
clear xlate interface inside
! 또는 전체 삭제 (주의: 모든 기존 연결 끊김)
clear xlate
```

`clear xlate`는 **모든 기존 NAT 세션을 끊으므로** 업무 시간에는 주의해야 합니다. 특정 대상만 삭제하는 것을 권장합니다.

**3. NAT 규칙 히트 카운터 활용**

```
show nat detail
```

`translate_hits = 0`인 규칙은 사용되지 않는 규칙일 수 있으므로, 정기적으로 정리하여 NAT 테이블을 간결하게 유지합니다.

**4. packet-tracer 습관화**

NAT 규칙 추가/변경 후 반드시 `packet-tracer`로 의도대로 동작하는지 검증합니다. 실제 트래픽을 발생시키지 않고도 NAT 변환 결과를 확인할 수 있습니다.

**5. Manual NAT 순서 관리**

Manual NAT는 설정 순서가 곧 처리 순서이므로, 규칙 추가 시 `line` 번호를 지정하여 원하는 위치에 삽입합니다.

```
! 기존 규칙 앞에 새 규칙 삽입
nat (inside,outside) 1 source static OBJ_NEW OBJ_NEW destination static OBJ_DEST OBJ_DEST
```
