# 01-05. BGP Path Attribute와 경로 제어

## 면접관: "고객사가 ISP 2곳(ISP-A, ISP-B)을 사용 중인데, '중요 업무 트래픽은 ISP-A(대역폭 큰 회선)로, 나머지 일반 트래픽은 ISP-B로 보내달라'고 요청합니다. 어떻게 설계하시겠어요?"

이것은 **Outbound 트래픽 제어** 문제입니다. BGP에서는 Path Attribute를 조작해서 특정 목적지에 대한 경로 선호도를 바꿀 수 있습니다.

**설계 전략:**
- 중요 업무 서버 대역(예: SaaS 서비스, 클라우드 업무 시스템)으로 가는 트래픽 → ISP-A 선호
- 나머지 트래픽 → ISP-B 선호
- 장애 시에는 자동으로 남은 ISP로 Failover

Outbound 제어에 가장 효과적인 Attribute는 **Local Preference**입니다. 자신의 AS 내부에서 경로 선호도를 결정하는 데 사용되며, 값이 클수록 선호합니다.

> 토폴로지 이미지 추가 예정

### 핵심 설정

```
! 중요 업무 대역 정의
ip prefix-list CRITICAL-APPS seq 10 permit 104.16.0.0/12     ! 클라우드 서비스
ip prefix-list CRITICAL-APPS seq 20 permit 13.107.0.0/16     ! M365
ip prefix-list CRITICAL-APPS seq 30 permit 52.96.0.0/14      ! Azure
!
! ISP-A Inbound Route-map (중요 트래픽 → ISP-A 선호)
route-map ISP-A-IN permit 10
 match ip address prefix-list CRITICAL-APPS
 set local-preference 300
!
route-map ISP-A-IN permit 20
 set local-preference 80
!
! ISP-B Inbound Route-map (일반 트래픽 → ISP-B 선호)
route-map ISP-B-IN permit 10
 match ip address prefix-list CRITICAL-APPS
 set local-preference 80
!
route-map ISP-B-IN permit 20
 set local-preference 200
!
router bgp 65001
 neighbor 203.0.113.1 route-map ISP-A-IN in    ! ISP-A
 neighbor 198.51.100.1 route-map ISP-B-IN in   ! ISP-B
```

**동작 결과:**
- 104.16.0.0/12 → ISP-A LP 300, ISP-B LP 80 → **ISP-A 선택**
- 기타 경로 → ISP-A LP 80, ISP-B LP 200 → **ISP-B 선택**
- ISP-A 다운 시 → ISP-B의 LP 80이라도 유일한 경로이므로 사용 (자동 Failover)

---

## 면접관: "방금 Outbound 트래픽 제어를 했잖아요. 그러면 Inbound 트래픽(인터넷에서 고객사로 들어오는 트래픽)도 제어해야 하지 않나요? BGP Path Attribute를 활용한 경로 선택 순서를 전체적으로 설명해주세요."

BGP Best Path Selection 순서는 CCNP에서 반드시 외워야 하는 핵심 중 하나입니다. Cisco 기준:

```
1. Weight (Cisco 전용, 높을수록 선호, 로컬에서만 유효)
2. Local Preference (높을수록 선호, iBGP 내 전파)
3. Locally Originated (자신이 생성한 경로 선호 - network, aggregate, redistribute)
4. AS-Path Length (짧을수록 선호)
5. Origin Type (IGP > EGP > Incomplete)
6. MED (낮을수록 선호, 동일 이웃 AS 간에만 비교)
7. eBGP > iBGP
8. IGP Metric to Next-hop (가장 가까운 Next-hop 선호)
9. Oldest Route (가장 오래된 eBGP 경로)
10. Lowest Router-ID
11. Lowest Neighbor IP
```

**각 Attribute의 실무 활용:**

| Attribute | 방향 | 범위 | 주요 사용처 |
|-----------|------|------|------------|
| **Weight** | Outbound | 로컬 라우터만 | 특정 라우터에서만 경로 조정 |
| **Local Preference** | Outbound | AS 전체 | AS 내 모든 라우터의 Outbound 제어 |
| **AS-Path Prepend** | Inbound | 인터넷 전체 | 상대 AS가 내 경로를 덜 선호하게 |
| **MED** | Inbound | 인접 AS만 | 인접 ISP에게 선호 입구 알림 |

**Inbound 트래픽 제어:**

Local Preference는 자기 AS 내부에서만 영향을 미치므로 Inbound 제어에 쓸 수 없습니다. Inbound 제어에는:

**AS-Path Prepend (가장 일반적):**
```
! ISP-B를 통한 Inbound를 줄이고 싶을 때
! ISP-B 쪽으로 광고 시 자신의 AS를 여러 번 추가
route-map TO-ISP-B-OUT permit 10
 set as-path prepend 65001 65001 65001
!
router bgp 65001
 neighbor 198.51.100.1 route-map TO-ISP-B-OUT out
```

인터넷 상의 다른 AS들이 고객사(AS 65001)로 가는 경로를 볼 때:
- ISP-A 경유: AS-Path = 64500 65001 (길이 2)
- ISP-B 경유: AS-Path = 64600 65001 65001 65001 65001 (길이 5)
→ AS-Path가 짧은 ISP-A를 선호 → Inbound 트래픽이 ISP-A로 유입

**MED:**
```
! ISP-A에게 "이 경로를 선호해달라" 요청
route-map TO-ISP-A-OUT permit 10
 set metric 100
!
route-map TO-ISP-B-OUT permit 10
 set metric 500
```
MED는 동일 AS의 여러 연결점에서만 비교하므로, ISP-A와 ISP-B가 다른 AS면 직접 비교되지 않습니다. 같은 ISP의 다른 POP에 이중 연결된 경우에 유효합니다.

---

## 면접관: "설정을 적용했는데, 고객사에서 '중요 업무 트래픽이 여전히 ISP-B로 나간다'고 합니다. 디버깅을 어떻게 하시겠어요?"

BGP 경로 제어 문제 디버깅은 **Best Path Selection 순서를 역추적**하는 것이 핵심입니다.

**1단계: 해당 목적지의 BGP 경로 확인**
```
show bgp ipv4 unicast 104.16.0.0/12
```

출력에서 `best` 마크가 어느 경로에 있는지, 그리고 각 경로의 attribute 값을 확인합니다.

```
BGP routing table entry for 104.16.0.0/12
Paths: (2 available, best #2)        ← #2가 best면 ISP-B가 선택된 것
  Path 1: 64500 13335
    203.0.113.1 from 203.0.113.1 (1.1.1.1)
    Origin IGP, localpref 300, valid, external
  Path 2: 64600 13335
    198.51.100.1 from 198.51.100.1 (2.2.2.2)
    Origin IGP, localpref 80, valid, external, best   ← 왜 이게 best?
```

**2단계: Best Path 선정 이유 확인**
```
show bgp ipv4 unicast 104.16.0.0/12 bestpath
```

또는 IOS-XE에서:
```
show bgp ipv4 unicast 104.16.0.0/12 best-path-reason
```

**3단계: Route-map이 제대로 적용되었는지 확인**
```
show route-map ISP-A-IN
show ip prefix-list CRITICAL-APPS
show bgp ipv4 unicast neighbors 203.0.113.1 policy
```

**흔한 원인들:**

1. **Route-map을 neighbor에 적용하지 않음:** `route-map ISP-A-IN in`이 빠졌을 수 있음
2. **Route-map 적용 후 soft reset을 안 함:**
```
clear ip bgp 203.0.113.1 soft in
! 또는
clear ip bgp 203.0.113.1 in
```
Route-map 변경은 기존에 받은 경로에 즉시 반영되지 않습니다. Soft reset이 필요합니다.

3. **Prefix-list 매칭 오류:** `/12`를 매칭하려 했는데 ISP에서 더 구체적인 `/13`, `/14`로 쪼개서 광고하고 있을 수 있음
```
show bgp ipv4 unicast | include 104.16
```

4. **Weight가 설정되어 있음:** Weight는 Local Preference보다 먼저 평가됩니다. 이전 설정에 Weight가 남아있으면 LP를 아무리 바꿔도 소용없음
```
show bgp ipv4 unicast 104.16.0.0/12
  ! Weight 컬럼 확인
```

---

## 면접관: "Weight와 Local Preference의 차이를 좀 더 명확하게 설명해주세요. 언제 Weight를 쓰고 언제 LP를 쓰나요?"

| 항목 | Weight | Local Preference |
|------|--------|-----------------|
| 범위 | 해당 라우터에서만 유효 | iBGP를 통해 AS 전체에 전파 |
| 기본값 | 32768 (locally originated), 0 (learned) | 100 |
| 값 범위 | 0 ~ 65535 | 0 ~ 4294967295 |
| 전파 여부 | 전파 안 됨 | iBGP Update에 포함되어 전파 |
| Cisco 전용 | Yes | No (RFC 표준) |
| 우선순위 | 1순위 (최우선) | 2순위 |

**Weight를 쓰는 경우:**
- Edge Router가 1대이고, **이 라우터에서만** 경로 선호를 바꾸고 싶을 때
- 다른 iBGP Peer에 영향을 주고 싶지 않을 때
- 빠른 임시 조치 (테스트, 긴급 우회)

```
! 특정 neighbor에서 받는 모든 경로에 Weight 부여
router bgp 65001
 neighbor 203.0.113.1 weight 500
!
! 또는 Route-map으로 선택적 Weight 부여
route-map SET-WEIGHT permit 10
 match ip address prefix-list SPECIFIC-ROUTES
 set weight 500
route-map SET-WEIGHT permit 20
 ! 나머지는 기본 weight 0
```

**Local Preference를 쓰는 경우:**
- AS 내 모든 라우터가 동일한 경로 선호를 가져야 할 때 (표준 정책)
- Edge Router가 여러 대이고, 일관된 Outbound 정책이 필요할 때

실무에서는 **LP를 주로 사용**하고, Weight는 특수한 상황에서 보조적으로 사용합니다.

---

## 면접관: "경로 제어를 Route-map으로 하고 있는데, Community를 활용하면 뭐가 좋아지나요? 실무에서 어떻게 쓰이죠?"

**BGP Community**는 경로에 태그를 붙여서, 수신측에서 해당 태그 기반으로 정책을 적용할 수 있게 하는 메커니즘입니다.

**Well-known Communities:**
| Community | 의미 |
|-----------|------|
| `no-export` | eBGP Peer에게 광고하지 않음 (AS 내부에서만 유지) |
| `no-advertise` | 어떤 Peer에게도 광고하지 않음 |
| `local-as` | Sub-AS(Confederation) 밖으로 광고하지 않음 |
| `internet` | 제한 없이 광고 (기본값) |

**실무 활용 예시:**

**시나리오: ISP-A에게 특정 경로를 ISP-A의 다른 고객에게만 전파하고, 인터넷 전체에는 퍼뜨리지 말라고 요청**

많은 ISP들이 Community 기반 정책을 제공합니다:
```
ISP-A Community 정책 예시:
  64500:100 = ISP-A의 모든 Peer에게 광고
  64500:200 = ISP-A의 국내 Peer에게만 광고
  64500:300 = ISP-A 고객에게만 광고
  64500:666 = Blackhole (DDoS 방어)
```

```
! 특정 경로에 ISP Community 태그 부착
route-map TO-ISP-A permit 10
 match ip address prefix-list INTERNAL-ONLY
 set community 64500:300        ! ISP-A 고객에게만 광고
!
route-map TO-ISP-A permit 20
 set community 64500:100        ! 나머지는 전체 광고
!
router bgp 65001
 neighbor 203.0.113.1 send-community both
 neighbor 203.0.113.1 route-map TO-ISP-A out
```

**DDoS 대응 - RTBH (Remotely Triggered Blackhole):**
```
! 공격받는 IP에 대해 ISP에 블랙홀 요청
ip route 100.64.10.50 255.255.255.255 Null0 tag 666
!
route-map BLACKHOLE permit 10
 match tag 666
 set community 64500:666
 set ip next-hop 192.0.2.1       ! ISP의 Blackhole next-hop
!
router bgp 65001
 redistribute static route-map BLACKHOLE
```

**Community 확인:**
```
show bgp ipv4 unicast 100.64.10.0/24
show bgp ipv4 unicast community 64500:300
show ip bgp community-list
!
! Community를 보려면 반드시 send-community 설정 필요
! 그리고 수신측에서 ip bgp-community new-format 권장
ip bgp-community new-format
```

---

## 면접관: "고객사가 추가로 '특정 시간대에만 ISP-A를 Primary로 쓰고, 야간에는 ISP-B를 Primary로 바꿔달라'고 요청합니다. 가능한가요?"

가능합니다. **Time-based Route-map**을 사용합니다.

```
! 시간 범위 정의
time-range BUSINESS-HOURS
 periodic weekdays 09:00 to 18:00
!
time-range NIGHT-HOURS
 periodic weekdays 18:01 to 08:59
 periodic weekend 00:00 to 23:59
!
! 시간 기반 ACL
ip access-list extended MATCH-ALL
 permit ip any any time-range BUSINESS-HOURS
!
ip access-list extended MATCH-ALL-NIGHT
 permit ip any any time-range NIGHT-HOURS
```

하지만 실제로 BGP Route-map에서 time-range를 직접 매칭하는 것은 **IOS에서 제한적**입니다. 실무에서 더 일반적인 방법은:

**방법 1: EEM (Embedded Event Manager) Script**
```
! 업무 시간: ISP-A Primary
event manager applet BGP-DAYTIME
 event timer cron cron-entry "0 9 * * 1-5"
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "route-map ISP-A-IN permit 10"
 action 4.0 cli command "set local-preference 300"
 action 5.0 cli command "route-map ISP-B-IN permit 10"
 action 6.0 cli command "set local-preference 100"
 action 7.0 cli command "end"
 action 8.0 cli command "clear ip bgp * soft"
!
! 야간: ISP-B Primary
event manager applet BGP-NIGHTTIME
 event timer cron cron-entry "0 18 * * 1-5"
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "route-map ISP-A-IN permit 10"
 action 4.0 cli command "set local-preference 100"
 action 5.0 cli command "route-map ISP-B-IN permit 10"
 action 6.0 cli command "set local-preference 300"
 action 7.0 cli command "end"
 action 8.0 cli command "clear ip bgp * soft"
```

**방법 2: 자동화 플랫폼 (Ansible/Python)**
- Cron Job으로 시간대별 BGP 정책 Push
- 변경 이력 관리 가능
- 가장 권장하는 방법

**주의사항:**
- NTP 동기화 필수 (시간 기반이므로)
- `clear ip bgp * soft` 실행 시 순간적 경로 재계산 발생 → 트래픽 영향 최소화 확인
- 전환 시점에 모니터링 필요
- 시간대 전환 실패 시 수동 Fallback 절차 마련

```
! 현재 적용된 Route-map 확인
show route-map ISP-A-IN
show bgp ipv4 unicast neighbors 203.0.113.1 received-routes
show bgp ipv4 unicast neighbors 203.0.113.1 routes
!
! EEM 스크립트 실행 이력 확인
show event manager history events
```

---

## 면접관: "마지막으로, BGP 경로 제어를 설계할 때 가장 주의해야 할 점이 뭐라고 생각하세요?"

세 가지 핵심 포인트가 있습니다.

**1. Outbound와 Inbound는 별개 제어입니다**
- Outbound (내가 나가는 트래픽): Weight, Local Preference로 제어 → 내가 100% 제어 가능
- Inbound (외부에서 들어오는 트래픽): AS-Path Prepend, MED, Community로 "요청" → 상대방이 반영할지는 보장 안 됨
- 실무에서 고객이 "모든 트래픽이 ISP-A로 오게 해주세요"라고 하면, Inbound 제어의 한계를 반드시 설명해야 합니다

**2. 비대칭 라우팅 인지**
- Outbound는 ISP-A, Inbound는 ISP-B로 들어올 수 있음
- Stateful Firewall이 있으면 비대칭 경로에서 세션이 끊어질 수 있음
- 방화벽 설정에서 비대칭 트래픽 허용 여부 확인 필수

**3. Route-map의 암묵적 Deny**
- Route-map은 마지막에 **implicit deny**가 있음
- `permit 10`에서 매칭되지 않은 경로는 모두 Drop됨
- 반드시 마지막에 빈 `permit` sequence를 넣어야 나머지 경로도 통과

```
! 잘못된 예 - CRITICAL-APPS 외 모든 경로 차단됨
route-map ISP-A-IN permit 10
 match ip address prefix-list CRITICAL-APPS
 set local-preference 300
! ← 여기서 끝나면 implicit deny로 나머지 경로 전부 Drop!

! 올바른 예
route-map ISP-A-IN permit 10
 match ip address prefix-list CRITICAL-APPS
 set local-preference 300
route-map ISP-A-IN permit 20    ← 이거 필수!
 set local-preference 80
```
