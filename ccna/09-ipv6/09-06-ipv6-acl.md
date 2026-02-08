# 09-06. IPv6 ACL

> 개요
IPv6 ACL의 특징, IPv4 ACL과의 차이점, NDP를 위한 암묵적 허용 규칙, 설정 방법, ICMPv6 고려사항, 그리고 실무 필터링 시나리오에 대한 면접 시나리오입니다.

---

### 면접관: "IPv6 네트워크에 ACL을 적용해야 하는 상황입니다. IPv6 ACL은 IPv4 ACL과 어떻게 다른가요?"

IPv6 ACL은 IPv4 ACL과 기본 개념은 동일하지만, 여러 중요한 차이점이 있습니다.

**IPv4 ACL vs IPv6 ACL 비교:**

| 항목 | IPv4 ACL | IPv6 ACL |
|------|----------|----------|
| ACL 유형 | 번호형 + 이름형 | 이름형(Named)만 지원 |
| 적용 명령어 | ip access-group | ipv6 traffic-filter |
| 와일드카드 마스크 | 사용 | 사용하지 않음 (프리픽스 길이 사용) |
| 암묵적 규칙 | deny any (마지막) | deny any + NDP 허용 (마지막) |
| 프로토콜 키워드 | ip | ipv6 |
| ICMP | icmp | icmp (ICMPv6용) |

**가장 중요한 차이점 세 가지:**

1. **Named ACL만 지원**: IPv6에서는 번호형 ACL(예: access-list 100)이 없습니다. 반드시 이름을 사용해야 합니다.

2. **NDP를 위한 암묵적 허용**: IPv6 ACL의 맨 마지막에는 `deny ipv6 any any`뿐만 아니라, NDP 관련 트래픽을 허용하는 암묵적 규칙이 포함됩니다.

3. **프리픽스 길이 사용**: IPv4에서 사용하던 와일드카드 마스크 대신 프리픽스 길이(/64 등)를 사용합니다.

```
! IPv4 ACL 예시
ip access-list extended FILTER-V4
 permit tcp 192.168.1.0 0.0.0.255 any eq 80
 deny ip any any

! IPv6 ACL 예시
ipv6 access-list FILTER-V6
 permit tcp 2001:DB8:1::/64 any eq 80
 deny ipv6 any any
```

---

### 면접관: "NDP를 위한 암묵적 허용이 매우 중요하다고 하셨는데, 구체적으로 어떤 트래픽이 암묵적으로 허용되나요? 만약 이것이 없으면 어떤 문제가 발생합니까?"

이것은 IPv6 ACL에서 가장 중요한 개념 중 하나입니다.

**IPv6 ACL의 암묵적 규칙 (맨 마지막에 자동 추가):**

```
permit icmp any any nd-na          ! Neighbor Advertisement 허용
permit icmp any any nd-ns          ! Neighbor Solicitation 허용
deny ipv6 any any                  ! 나머지 모두 거부
```

**NDP 허용이 없으면 발생하는 문제:**

NDP는 IPv6에서 ARP를 대체하는 프로토콜입니다. NDP가 차단되면 다음과 같은 심각한 문제가 발생합니다:

1. **MAC 주소 해석 불가**: Neighbor Solicitation/Advertisement가 차단되면 IPv6 주소에 대응하는 MAC 주소를 알 수 없어 통신이 완전히 중단됩니다
2. **DAD 실패**: 주소 중복 탐지가 불가능해져 주소 충돌이 발생할 수 있습니다
3. **기본 게이트웨이 발견 불가**: Router Solicitation/Advertisement가 차단되면 SLAAC가 동작하지 않습니다

따라서 Cisco IOS에서는 명시적으로 `deny ipv6 any any`를 작성하더라도 ND-NA와 ND-NS는 암묵적으로 허용됩니다.

**주의할 점:**

만약 ACL에서 명시적으로 NDP를 거부하면, 암묵적 허용보다 우선하여 NDP가 차단됩니다:

```
! 위험한 설정 - NDP가 차단됨
ipv6 access-list BAD-ACL
 deny icmp any any               ! 모든 ICMPv6 차단 (NDP 포함!)
 permit ipv6 any any
```

이 경우 ND-NS와 ND-NA도 차단되어 네트워크 통신이 불가능해집니다. ICMPv6를 필터링할 때는 반드시 NDP 관련 메시지를 허용해야 합니다.

---

### 면접관: "IPv6 ACL을 설정하고 인터페이스에 적용하는 전체 과정을 보여주시겠습니까?"

네, IPv6 ACL의 생성부터 적용까지 전체 과정을 보여드리겠습니다.

**시나리오: 웹 서버(2001:DB8:1::100)로의 HTTP/HTTPS 접근만 허용하고, 나머지 트래픽은 차단**

```
! 1단계: IPv6 ACL 생성
R1(config)# ipv6 access-list WEB-SERVER-ACL

! HTTP 허용
R1(config-ipv6-acl)# permit tcp any host 2001:DB8:1::100 eq 80

! HTTPS 허용
R1(config-ipv6-acl)# permit tcp any host 2001:DB8:1::100 eq 443

! ICMPv6 Echo (ping) 허용 (선택사항, 트러블슈팅용)
R1(config-ipv6-acl)# permit icmp any any echo-request
R1(config-ipv6-acl)# permit icmp any any echo-reply

! 나머지 거부 (명시적으로 작성하지 않아도 암묵적으로 적용)
! NDP는 암묵적으로 허용됨
R1(config-ipv6-acl)# exit

! 2단계: 인터페이스에 적용
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ipv6 traffic-filter WEB-SERVER-ACL in
```

**주요 설정 포인트:**
- `ipv6 access-list` 명령어로 ACL 생성 (Named ACL만 가능)
- `ipv6 traffic-filter` 명령어로 인터페이스에 적용 (IPv4의 `ip access-group`에 해당)
- `in` 또는 `out` 방향 지정

**확인 명령어:**

```
! ACL 내용 확인
R1# show ipv6 access-list
IPv6 access list WEB-SERVER-ACL
    permit tcp any host 2001:DB8:1::100 eq www (15 matches)
    permit tcp any host 2001:DB8:1::100 eq 443 (8 matches)
    permit icmp any any echo-request (3 matches)
    permit icmp any any echo-reply (3 matches)

! 인터페이스 적용 상태 확인
R1# show ipv6 interface GigabitEthernet0/0
  ...
  Inbound access list WEB-SERVER-ACL
  ...
```

매치 카운터를 통해 각 규칙에 얼마나 많은 패킷이 매칭되었는지 확인할 수 있어 트러블슈팅에 유용합니다.

---

### 면접관: "ICMPv6를 필터링할 때 주의할 점을 더 자세히 알려주세요. 어떤 ICMPv6 메시지는 반드시 허용해야 하나요?"

ICMPv6 필터링은 IPv6 보안에서 매우 중요한 주제입니다. IPv4에서는 ICMP를 모두 차단해도 기본 통신에는 큰 문제가 없었지만, IPv6에서는 ICMPv6를 무분별하게 차단하면 네트워크가 동작하지 않습니다.

**반드시 허용해야 하는 ICMPv6 메시지:**

| ICMPv6 타입 | 이름 | 용도 | 차단 시 영향 |
|-------------|------|------|-------------|
| 133 | Router Solicitation | 라우터 발견 | SLAAC 불가 |
| 134 | Router Advertisement | 프리픽스/게이트웨이 정보 | 자동 주소 구성 불가 |
| 135 | Neighbor Solicitation | MAC 주소 확인, DAD | 통신 불가 |
| 136 | Neighbor Advertisement | NS 응답 | 통신 불가 |
| 137 | Redirect | 최적 경로 안내 | 성능 저하 (필수 아님) |
| 2 | Packet Too Big | MTU 알림 | PMTUD 실패, 큰 패킷 전송 불가 |
| 1 | Destination Unreachable | 도달 불가 알림 | 연결 타임아웃 |

**ICMPv6를 안전하게 필터링하는 ACL 예시:**

```
ipv6 access-list SAFE-ICMPV6-FILTER
 ! NDP 관련 - 필수 허용
 permit icmp any any nd-na
 permit icmp any any nd-ns
 permit icmp any any router-solicitation
 permit icmp any any router-advertisement

 ! PMTUD - 필수 허용
 permit icmp any any packet-too-big

 ! 트러블슈팅용 - 선택 허용
 permit icmp any any echo-request
 permit icmp any any echo-reply
 permit icmp any any destination-unreachable
 permit icmp any any time-exceeded

 ! 나머지 ICMPv6 차단
 deny icmp any any

 ! 기타 트래픽 정책
 permit tcp any any established
 permit ipv6 any any
```

**핵심 원칙:** IPv6에서는 "ICMPv6를 모두 차단한다"는 정책은 적용할 수 없습니다. 최소한 NDP와 Packet Too Big 메시지는 반드시 허용해야 합니다.

---

### 면접관: "실무에서 자주 사용하는 IPv6 ACL 필터링 시나리오를 몇 가지 보여주시겠습니까?"

네, 실무에서 자주 사용하는 IPv6 ACL 시나리오를 보여드리겠습니다.

**시나리오 1: 특정 서브넷에서 서버로의 SSH만 허용**

```
ipv6 access-list SSH-ONLY
 permit tcp 2001:DB8:ADMIN::/48 host 2001:DB8:SRV::1 eq 22
 permit icmp any any nd-ns
 permit icmp any any nd-na
 deny ipv6 any any log
```

**시나리오 2: 내부 네트워크에서 외부로의 웹 트래픽 허용 (인바운드 응답 포함)**

```
ipv6 access-list OUTBOUND-WEB
 ! 외부로 나가는 웹 트래픽 허용
 permit tcp 2001:DB8:INTERNAL::/48 any eq 80
 permit tcp 2001:DB8:INTERNAL::/48 any eq 443
 ! DNS 조회 허용
 permit udp 2001:DB8:INTERNAL::/48 any eq 53
 ! 필수 ICMPv6 허용
 permit icmp any any nd-ns
 permit icmp any any nd-na
 permit icmp any any packet-too-big
 permit icmp any any echo-request
 deny ipv6 any any
```

**시나리오 3: 특정 네트워크 차단 (블랙리스트)**

```
ipv6 access-list BLACKLIST
 deny ipv6 2001:DB8:BAD::/48 any log
 deny ipv6 any 2001:DB8:BAD::/48 log
 permit ipv6 any any
```

**시나리오 4: VTY 라인 접근 제어 (라우터 관리 접근 제한)**

```
ipv6 access-list VTY-ACCESS
 permit tcp 2001:DB8:MGMT::/64 any eq 22
 deny ipv6 any any

R1(config)# line vty 0 4
R1(config-line)# ipv6 access-class VTY-ACCESS in
```

VTY 라인에 적용할 때는 `ipv6 traffic-filter`가 아닌 `ipv6 access-class` 명령어를 사용합니다.

---

### 면접관: "IPv6 ACL의 순서(sequence number)를 관리하는 방법은요? ACL 규칙을 중간에 삽입할 수 있나요?"

네, IPv6 ACL도 IPv4 Named ACL과 마찬가지로 시퀀스 번호를 사용하여 규칙의 순서를 관리할 수 있습니다.

**시퀀스 번호를 이용한 규칙 삽입:**

```
! 기존 ACL 확인
R1# show ipv6 access-list WEB-FILTER
IPv6 access list WEB-FILTER
    sequence 10 permit tcp any host 2001:DB8:1::100 eq www
    sequence 20 permit tcp any host 2001:DB8:1::100 eq 443
    sequence 30 deny ipv6 any any

! 시퀀스 15에 새 규칙 삽입
R1(config)# ipv6 access-list WEB-FILTER
R1(config-ipv6-acl)# sequence 15 permit tcp any host 2001:DB8:1::100 eq 8080

! 결과 확인
R1# show ipv6 access-list WEB-FILTER
IPv6 access list WEB-FILTER
    sequence 10 permit tcp any host 2001:DB8:1::100 eq www
    sequence 15 permit tcp any host 2001:DB8:1::100 eq 8080
    sequence 20 permit tcp any host 2001:DB8:1::100 eq 443
    sequence 30 deny ipv6 any any
```

**특정 규칙 삭제:**

```
R1(config)# ipv6 access-list WEB-FILTER
R1(config-ipv6-acl)# no sequence 15
```

**시퀀스 번호 재배열:**

```
R1(config)# ipv6 access-list WEB-FILTER
R1(config-ipv6-acl)# no sequence 15
```

ACL은 위에서 아래로 순차적으로 평가되며, 첫 번째로 매칭되는 규칙이 적용됩니다. 따라서 가장 구체적인 규칙을 위에, 일반적인 규칙을 아래에 배치하는 것이 원칙입니다.

---

### 면접관: "IPv6 ACL 관련 트러블슈팅 방법과 자주 발생하는 실수를 정리해 주세요."

IPv6 ACL 트러블슈팅 방법과 주의사항을 정리하겠습니다.

**트러블슈팅 명령어:**

```
! ACL 내용과 매치 카운터 확인
R1# show ipv6 access-list
R1# show ipv6 access-list WEB-FILTER

! 인터페이스에 적용된 ACL 확인
R1# show ipv6 interface GigabitEthernet0/0

! ACL 매치 카운터 초기화 (분석 시작점 설정)
R1# clear ipv6 access-list WEB-FILTER

! 실시간 로그 확인 (log 키워드 사용 시)
R1# show logging
```

**자주 발생하는 실수와 해결:**

| 실수 | 증상 | 해결 |
|------|------|------|
| 모든 ICMPv6 차단 | 네트워크 통신 완전 중단 | ND-NS, ND-NA 최소 허용 |
| ACL 적용 방향 오류 (in/out 혼동) | 기대와 다른 필터링 동작 | 트래픽 흐름 방향 재확인 |
| 와일드카드 마스크 사용 시도 | 설정 오류 | 프리픽스 길이 사용 (/64 등) |
| ip access-group 사용 | 적용 안 됨 | ipv6 traffic-filter 사용 |
| deny 규칙만 있고 permit 없음 | 모든 트래픽 차단 | 필요한 permit 규칙 추가 |
| Packet Too Big 차단 | 큰 파일 전송 실패 | ICMPv6 type 2 허용 |
| log 키워드 남용 | CPU 과부하 | 필요한 규칙에만 log 적용 |

**트러블슈팅 체크리스트:**

```
1. ACL이 올바른 인터페이스에 적용되어 있는가?
   → show ipv6 interface [interface]

2. ACL 방향(in/out)이 올바른가?
   → 트래픽 흐름도를 그려서 확인

3. ACL 규칙 순서가 올바른가?
   → 구체적인 규칙이 일반적인 규칙보다 먼저 오는가?

4. NDP 트래픽이 허용되어 있는가?
   → 암묵적 허용에 의존하거나 명시적으로 permit

5. ICMPv6 Packet Too Big이 허용되어 있는가?
   → PMTUD가 정상 동작하려면 필수

6. 매치 카운터가 예상대로 증가하는가?
   → show ipv6 access-list로 확인
```

실무에서는 ACL을 적용하기 전에 반드시 콘솔 접근을 확보하고, 원격 접근이 차단될 가능성에 대비해야 합니다. 또한 변경 전 현재 설정을 백업하는 것이 기본적인 안전 수칙입니다.

---

### 면접관: "마지막으로, IPv6 ACL과 IPv4 ACL을 동시에 운영하는 듀얼 스택 환경에서 ACL 관리 전략을 알려주세요."

듀얼 스택 환경에서의 ACL 관리 전략을 정리하겠습니다.

**핵심 원칙: IPv4와 IPv6에 동등한 수준의 보안 정책 적용**

많은 조직에서 IPv4 ACL은 세밀하게 관리하면서 IPv6 ACL은 소홀히 하는 경우가 있습니다. 이는 심각한 보안 위협이 됩니다.

**관리 전략:**

1. **정책 매핑**: IPv4 ACL 정책을 IPv6 ACL로 1:1 매핑

```
! IPv4 정책
ip access-list extended CORP-POLICY-V4
 permit tcp 10.1.0.0 0.0.255.255 any eq 80
 permit tcp 10.1.0.0 0.0.255.255 any eq 443
 deny ip any any log

! 동일한 IPv6 정책
ipv6 access-list CORP-POLICY-V6
 permit tcp 2001:DB8:1::/48 any eq 80
 permit tcp 2001:DB8:1::/48 any eq 443
 permit icmp any any nd-ns
 permit icmp any any nd-na
 permit icmp any any packet-too-big
 deny ipv6 any any log
```

2. **네이밍 컨벤션 통일**: ACL 이름에 V4/V6를 명시하여 구분

3. **양쪽 동시 적용**: 인터페이스에 IPv4 ACL과 IPv6 ACL을 모두 적용

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip access-group CORP-POLICY-V4 in
R1(config-if)# ipv6 traffic-filter CORP-POLICY-V6 in
```

4. **변경 관리 프로세스**: ACL 변경 시 IPv4와 IPv6를 항상 함께 검토

5. **정기 감사**: 양쪽 ACL이 동일한 보안 수준을 유지하는지 주기적으로 점검

IPv6 ACL에서 가장 기억해야 할 것은 NDP 허용의 중요성과, IPv4 ACL과 동등한 보안 정책을 유지해야 한다는 점입니다.
