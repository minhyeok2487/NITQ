# 08-04. NAT 통신 장애

---

### 면접관: "고객사에서 '외부에서 우리 웹서버에 접속이 안 된다'고 합니다. ASA 방화벽에서 Static NAT로 공인 IP를 매핑해 놨다는데, 어디부터 확인하시겠어요?"

외부에서 내부 서버 접속이 안 되는 경우, **NAT 변환 → ACL 허용 → 라우팅** 순서로 확인합니다.

**1단계: 현재 NAT 변환 테이블 확인**

```
! ASA
show xlate
show xlate | include 10.1.1.100
```

출력 예시:
```
3 in use, 5 most used
Flags: D - DNS, e - extended, I - identity, i - dynamic, r - portmap,
       s - static, T - twice, N - net-to-net
TCP PAT from inside:10.1.1.100/443 to outside:203.0.113.10/443
    flags s idle 0:02:30 timeout 0:00:00
```

Static NAT가 xlate 테이블에 존재하는지 확인합니다.

**2단계: NAT 설정 상세 확인**

```
show nat
show nat detail
show running-config nat
show running-config object
```

출력 예시:
```
Auto NAT Policies (Section 2)
1 (inside) to (outside) source static OBJ-WEBSERVER 203.0.113.10
    translate_hits = 0, untranslate_hits = 0
```

`translate_hits`와 `untranslate_hits`가 전부 0이면 NAT Rule이 매칭되지 않고 있다는 의미입니다.

**3단계: 외부에서의 접근성 테스트**

```
! ASA에서 packet-tracer로 시뮬레이션
packet-tracer input outside tcp 8.8.8.8 12345 203.0.113.10 443
```

이것 하나로 NAT, ACL, 라우팅, 서비스 정책 매칭을 모두 확인할 수 있습니다.

---

### 면접관: "Static NAT 설정은 되어 있는데 외부에서 접속이 안 됩니다. packet-tracer를 돌려봤더니 NAT 단계에서 'un-translate'가 안 된다고 나옵니다. 원인이 뭘까요?"

ASA의 NAT는 **Section 순서**로 처리됩니다. 상위 Section의 Rule이 먼저 매칭되므로, 의도한 Static NAT보다 위에 있는 다른 NAT Rule이 트래픽을 가로챌 수 있습니다.

#### ASA NAT Section 처리 순서

| 순서 | Section | 설명 |
|------|---------|------|
| 1 | **Section 1** | Twice NAT (Manual NAT) - `nat (if1,if2)` 명령어 순서대로 |
| 2 | **Section 2** | Auto NAT (Object NAT) - Static > Dynamic 순서 (자동 정렬) |
| 3 | **Section 3** | Twice NAT with `after-auto` 키워드 |

```
show nat
```

출력에서 Section 번호를 확인:
```
Manual NAT Policies (Section 1)
1 (inside) to (outside) source dynamic any interface
    translate_hits = 15234, untranslate_hits = 0

Auto NAT Policies (Section 2)
1 (inside) to (outside) source static OBJ-WEBSERVER 203.0.113.10
    translate_hits = 0, untranslate_hits = 0
```

위 예시에서 **Section 1의 Dynamic PAT**가 먼저 매칭됩니다. 외부에서 203.0.113.10으로 들어오는 트래픽이 Section 1의 Dynamic PAT Rule과 먼저 비교되는데, 이 Rule이 "모든 inside 트래픽"을 매칭하고 있어서, un-translate 시 충돌이 발생합니다.

**해결 방법 1: Static NAT를 Section 1으로 이동 (Twice NAT)**

```
! Object 정의
object network OBJ-WEBSERVER
 host 10.1.1.100
object network OBJ-WEBSERVER-PUBLIC
 host 203.0.113.10

! Section 1에 Static NAT 추가 (Manual NAT는 순서 중요 - 위에 배치)
nat (inside,outside) 1 source static OBJ-WEBSERVER OBJ-WEBSERVER-PUBLIC
```

**해결 방법 2: Dynamic PAT에서 웹서버 제외**

```
object-group network EXCEPT-WEBSERVER
 network-object host 10.1.1.100

nat (inside,outside) source dynamic any interface destination static any any
! 위를 수정하여 웹서버를 제외하는 ACL 사용
```

---

### 면접관: "ASA의 packet-tracer에 대해 좀 더 자세히 설명해 주세요. 실무에서 어떻게 활용하나요?"

`packet-tracer`는 ASA에서 가상의 패킷을 생성하여 **실제 처리 경로를 단계별로 추적**하는 도구입니다. 실제 트래픽 없이도 NAT, ACL, Routing, VPN, Inspection 등 모든 처리 단계를 확인할 수 있어서 트러블슈팅의 핵심 도구입니다.

#### 기본 문법

```
packet-tracer input <인터페이스> <프로토콜> <src-ip> <src-port> <dst-ip> <dst-port> [detailed]
```

#### 활용 예시

```
! 외부에서 웹서버 접속 시뮬레이션
packet-tracer input outside tcp 8.8.8.8 12345 203.0.113.10 443 detailed

! 내부에서 외부 인터넷 접속 시뮬레이션
packet-tracer input inside tcp 10.1.1.50 54321 8.8.8.8 80 detailed

! ICMP 테스트
packet-tracer input inside icmp 10.1.1.50 8 0 8.8.8.8 detailed
! (type 8 = echo request, code 0)
```

#### 출력 분석

```
Phase: 1
Type: ACCESS-LIST
Subtype:
Result: ALLOW
Config:
Implicit Rule
Additional Information:
 Forward Flow based lookup yields rule:
 in  id=0x7fff2a60d650, priority=1, domain=permit, deny=false

Phase: 2
Type: UN-NAT
Subtype: static
Result: ALLOW
Config:
nat (inside,outside) source static OBJ-WEBSERVER 203.0.113.10
Additional Information:
 NAT divert to egress interface inside
 Untranslate 203.0.113.10/443 to 10.1.1.100/443

Phase: 3
Type: ACCESS-LIST
Subtype: log
Result: ALLOW
Config:
access-group OUTSIDE-IN in interface outside
access-list OUTSIDE-IN extended permit tcp any host 203.0.113.10 eq https

...

Result:
input-interface: outside
input-status: up
input-line-status: up
output-interface: inside
output-status: up
output-line-status: up
Action: allow
```

각 Phase에서 **Result: ALLOW** 또는 **Result: DROP**을 확인합니다. DROP이 나오면 해당 Phase가 문제의 원인입니다.

> `detailed` 옵션을 붙이면 각 Phase의 Config까지 보여주어, 어떤 Rule이 매칭되었는지 정확히 파악할 수 있습니다.

---

### 면접관: "내부에서 외부 인터넷은 잘 되는데 DMZ의 서버만 접속이 안 됩니다. 뭘 봐야 하나요?"

DMZ 통신 장애는 **NAT 순서, ACL, 그리고 Security Level** 세 가지를 종합적으로 봐야 합니다.

#### ASA Security Level 기본 동작

| 방향 | 동작 |
|------|------|
| 높은 Security Level → 낮은 Security Level | 기본 허용 (Outbound) |
| 낮은 Security Level → 높은 Security Level | 기본 차단 (Inbound) - ACL 필요 |
| 같은 Security Level | 기본 차단 (`same-security-traffic permit inter-interface`로 허용 가능) |

일반적인 Security Level:
```
interface GigabitEthernet0/0
 nameif inside
 security-level 100

interface GigabitEthernet0/1
 nameif outside
 security-level 0

interface GigabitEthernet0/2
 nameif dmz
 security-level 50
```

Inside(100) → DMZ(50): 기본 허용
DMZ(50) → Inside(100): ACL 필요

#### 흔한 실수 1: Inside → DMZ NAT 누락

Inside에서 DMZ 서버에 접속할 때, NAT Rule이 없으면 ASA가 패킷을 Drop할 수 있습니다. ASA는 NAT Rule이 있어야 트래픽을 전달하는 경우가 있습니다 (특히 8.3 이전 버전과의 호환 설정).

```
! Inside → DMZ는 NAT 불필요 (Identity NAT)
nat (inside,dmz) source static any any destination static any any no-proxy-arp route-lookup
```

#### 흔한 실수 2: DMZ → Inside ACL 누락

DMZ 서버가 Inside의 DB 서버에 접근해야 하는 경우, ACL이 없으면 차단됩니다.

```
access-list DMZ-TO-INSIDE extended permit tcp host 10.2.1.100 host 10.1.1.200 eq 1521
! DMZ 웹서버 → Inside DB서버 Oracle 1521 포트
access-group DMZ-TO-INSIDE in interface dmz
```

#### 흔한 실수 3: Hairpin (U-Turn) NAT

Inside 사용자가 공인 IP로 DMZ 서버에 접속하려는 경우 (예: 내부에서 https://203.0.113.10 접속). 트래픽이 ASA의 outside로 나갔다가 다시 들어와야 하므로 특별한 설정이 필요합니다.

```
! Hairpin 허용
same-security-traffic permit intra-interface

! 또는 DNS Doctoring / Split DNS로 내부에서는 Private IP 사용하도록 유도
```

#### packet-tracer로 확인

```
packet-tracer input inside tcp 10.1.1.50 54321 10.2.1.100 443 detailed
! Inside → DMZ 통신 시뮬레이션
```

---

### 면접관: "Checkpoint 방화벽 환경에서 NAT 문제가 생기면 어떻게 디버깅하나요?"

Checkpoint에서 NAT 트러블슈팅은 `fw ctl zdebug drop`과 SmartConsole의 로그를 주로 사용합니다.

#### 1단계: Drop 원인 확인

```bash
fw ctl zdebug drop
```

출력 예시:
```
@;sobFmjEqq;[cpu_1];[fw4_0];fw_log_drop_ex: Packet proto=6 10.1.1.50:54321 -> 203.0.113.10:443 dropped by fw_handle_first_packet Reason: No matching NAT rule;
```

"No matching NAT rule"이면 NAT Rule이 없거나 매칭되지 않는 것입니다.

#### 2단계: NAT Rule Base 확인

SmartConsole에서 NAT Policy를 확인합니다. Checkpoint NAT도 **순서 기반**입니다.

| 순서 | 타입 | 설명 |
|------|------|------|
| 1 | Manual (위쪽) | 관리자가 직접 만든 Rule - 위에서 아래로 매칭 |
| 2 | Automatic | Object 속성에서 자동 생성된 Rule (Static/Hide) |
| 3 | Manual (아래쪽) | `Below automatic rules`에 배치된 Manual Rule |

```bash
# Expert 모드에서 NAT 테이블 확인
fw tab -t fwx_alloc -s
# NAT 세션 수 확인

# 특정 IP의 NAT 변환 확인
fw tab -t fwx_alloc -f | grep 10.1.1.100
```

#### 3단계: 커널 레벨 NAT 디버그

```bash
fw ctl debug 0
fw ctl debug -buf 32768
fw ctl debug -m fw + xlate
fw ctl kdebug -T -f > /var/log/nat_debug.txt
# ... 트래픽 재현 ...
fw ctl debug 0
```

#### 4단계: cpinfo로 전체 NAT 설정 덤프

```bash
# 현재 적용된 NAT Rule 확인
fwaccel stat
fwaccel conns -s | grep <IP>
# SecureXL에서의 NAT 처리 확인
```

> Checkpoint에서는 **SecureXL(가속 엔진)**이 NAT를 처리할 수 있으므로, 디버그 시 `fwaccel off`로 가속을 일시적으로 끄고 테스트하면 더 정확한 디버그가 가능합니다. 단, 운영 환경에서는 성능 저하를 유발하므로 주의합니다.

---

### 면접관: "NAT 설정에서 Proxy ARP가 누락되면 어떤 문제가 생기나요?"

Static NAT에서 매우 중요한 개념입니다.

외부에서 공인 IP 203.0.113.10으로 패킷을 보내면, 마지막 홉 라우터(보통 ISP 라우터 또는 상위 라우터)가 203.0.113.10의 **MAC 주소를 알아야** 패킷을 전달할 수 있습니다. 하지만 203.0.113.10은 실제로 ASA의 인터페이스 IP가 아닌 NAT 주소이므로, 아무도 ARP에 응답하지 않으면 패킷이 Drop됩니다.

**Proxy ARP**: ASA가 203.0.113.10에 대한 ARP 요청에 **자신의 MAC 주소로 대신 응답**하여, 해당 트래픽을 자신에게 유도하는 기능입니다.

#### ASA에서의 동작

ASA는 **Static NAT가 설정되면 기본적으로 Proxy ARP를 활성화**합니다. 단, `no-proxy-arp` 옵션을 명시적으로 넣으면 비활성화됩니다.

```
! Proxy ARP가 비활성화된 설정 (문제 발생!)
nat (inside,outside) source static OBJ-WEBSERVER 203.0.113.10 no-proxy-arp

! 정상 설정 (Proxy ARP 활성화)
nat (inside,outside) source static OBJ-WEBSERVER 203.0.113.10
```

확인:
```
show arp
! NAT 주소에 대한 ARP 엔트리가 ASA의 MAC으로 존재하는지

! 외부 라우터에서
show arp | include 203.0.113.10
! ASA의 outside 인터페이스 MAC과 일치하는지 확인
```

#### Cisco IOS에서의 Proxy ARP

IOS 라우터에서 Static NAT를 사용할 때도 동일한 문제가 발생합니다. IOS는 기본적으로 `ip proxy-arp`가 인터페이스에서 활성화되어 있지만, 누군가 `no ip proxy-arp`를 설정했다면 NAT 주소에 대한 ARP 응답이 안 됩니다.

```
show running-config interface GigabitEthernet0/0 | include proxy-arp
```

대안으로, NAT 주소와 같은 서브넷의 IP를 Secondary로 추가하거나, 상위 라우터에 Static Route를 추가하는 방법이 있습니다.

---

### 면접관: "NAT 관련 장애를 사전에 방지하려면 어떻게 관리해야 하나요?"

#### 설계 단계 체크리스트

| 항목 | 확인 내용 |
|------|----------|
| NAT Section 순서 | Static NAT가 Dynamic PAT보다 우선 처리되는지 |
| VPN + NAT 공존 | VPN 트래픽에 대한 NAT Exemption(Identity NAT) 설정 |
| Proxy ARP | Static NAT에 no-proxy-arp가 실수로 들어가지 않았는지 |
| Hairpin NAT | 내부 사용자가 공인 IP로 DMZ 접속 시 필요 여부 |
| DNS Doctoring | 내부에서 공인 IP DNS 질의 시 Private IP로 변환 여부 |
| Bidirectional | 외부 → 내부 접근 시 ACL 허용 여부 |

#### 변경 작업 시 필수 테스트

```
! 변경 전 Baseline
show xlate count
show nat detail
! hit 카운터 기록

! 변경 후 테스트
packet-tracer input outside tcp 8.8.8.8 12345 <공인IP> 443 detailed
packet-tracer input inside tcp 10.1.1.50 54321 8.8.8.8 80 detailed
packet-tracer input inside tcp 10.1.1.50 54321 <DMZ서버IP> 443 detailed
! 주요 트래픽 흐름 3가지 이상 테스트

! 변경 후 확인
show xlate | include <관련IP>
show nat detail | include translate_hits
! hit 카운터 증가 확인
```

#### 모니터링

```
! ASA Syslog (Level 6 - Informational)
logging enable
logging trap informational
logging host inside 10.1.1.200

! NAT 관련 주요 Syslog 메시지
! %ASA-3-305006: portmap translation creation failed for <proto> src <if>:<ip> dst <if>:<ip>
! → PAT 포트 풀 소진
! %ASA-3-305005: No translation group found for <proto> src <if>:<ip> dst <if>:<ip>
! → NAT Rule 매칭 실패

! PAT 포트 사용량 모니터링 (고갈 방지)
show nat pool
show xlate count
! xlate 수가 비정상적으로 많으면 PAT 포트 고갈 위험
```

#### 공인 IP 관리

Static NAT 주소가 기존 다른 장비의 IP와 충돌하지 않는지, 상위 라우터에서 해당 서브넷이 올바르게 라우팅되는지 반드시 확인합니다. 특히 ISP 변경이나 공인 IP 대역 변경 시 NAT 설정도 함께 업데이트해야 합니다.

---
