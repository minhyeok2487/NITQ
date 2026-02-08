# 03-03. ASA ACL과 MPF (Modular Policy Framework)

---

## 면접관: "고객사 DMZ에 웹서버가 있는데, 외부에서 HTTP(80)와 HTTPS(443) 포트로만 접근을 허용하고 나머지는 전부 차단해야 합니다. ACL을 설계해주세요."

Outside → DMZ 방향은 Security Level이 낮→높이므로 기본 차단입니다. 허용할 트래픽만 명시적으로 permit하는 ACL을 Outside 인터페이스 인바운드 방향에 적용하겠습니다.

> 토폴로지 이미지 추가 예정

### 핵심 설정

```
! 서버 오브젝트 정의
object network OBJ_DMZ_WEB
 host 172.16.1.10

object network OBJ_DMZ_MAIL
 host 172.16.1.20

! 서비스 오브젝트 그룹
object-group service SVC_WEB tcp
 port-object eq www
 port-object eq https

object-group service SVC_MAIL tcp
 port-object eq smtp
 port-object eq 587

! ACL 정의
access-list outside_access_in extended permit tcp any object OBJ_DMZ_WEB object-group SVC_WEB
access-list outside_access_in extended permit tcp any object OBJ_DMZ_MAIL object-group SVC_MAIL
access-list outside_access_in extended deny ip any any log

! ACL 적용
access-group outside_access_in in interface outside
```

여기서 몇 가지 중요한 포인트가 있습니다.

- **object-group** 사용: 개별 포트를 하나씩 나열하는 것보다 관리가 편하고, 정책 변경 시 ACL을 수정하지 않고 그룹만 수정하면 됩니다.
- **마지막 deny any any log**: ASA는 ACL 끝에 implicit deny가 있지만, 명시적으로 `log`를 붙여 차단된 트래픽을 로깅하면 트러블슈팅에 큰 도움이 됩니다.
- **목적지 IP는 NAT 후 주소(공인 IP)가 아닌 실제 서버 IP(사설 IP)를 사용**: ASA 8.3 이후 버전에서는 ACL에 **Real IP (변환 전 사설 IP)**를 사용합니다.

---

## 면접관: "ASA ACL이 라우터 ACL과 다른 점이 있나요? 특히 ACL 방향이랑 NAT와의 관계를 설명해주세요."

ASA ACL과 라우터 ACL의 핵심적인 차이점이 몇 가지 있습니다.

### 1. ACL 적용 방향

**라우터**: 인터페이스에 inbound/outbound 모두 적용 가능
**ASA**: 기본적으로 **인바운드(in) 방향만** 적용. 트래픽이 ASA에 **들어오는 인터페이스**에서 검사합니다.

```
! ASA: 항상 inbound 방향으로 적용
access-group outside_access_in in interface outside
access-group inside_access_in in interface inside
access-group dmz_access_in in interface dmz
```

ASA 9.x부터 `access-group ... out interface` (아웃바운드)도 가능하지만, 일반적으로 인바운드 ACL로 충분하고 권장됩니다. 아웃바운드 ACL은 성능에 영향을 줄 수 있습니다.

### 2. ACL과 NAT의 처리 순서 (ASA 8.3+ / 9.x)

```
패킷 도착 → [NAT Untranslate (Destination)] → [ACL 검사] → [Routing] → [NAT Translate (Source)] → 패킷 전송
```

ASA 8.3 이후, ACL은 **Real IP (NAT 변환 전 사설 IP)**를 기준으로 작성합니다.

```
! ASA 8.3+: Real IP 사용 (올바른 방법)
access-list outside_access_in permit tcp any host 172.16.1.10 eq www

! ASA 8.2 이전: Mapped IP 사용 (레거시)
! access-list outside_access_in permit tcp any host 203.0.113.10 eq www
```

### 3. Stateful 처리

라우터 ACL은 Stateless이므로 리턴 트래픽에 대해 별도 ACL이 필요하지만, ASA는 Stateful이므로 **허용된 세션의 리턴 트래픽은 ACL 없이도 자동 허용**됩니다.

```
! ASA: Inside → Outside ACL이 없어도 기본 허용 (높→낮)
! Outside에서 돌아오는 리턴 트래픽도 conn table로 자동 허용

! 라우터: 양방향 모두 ACL 필요
! ip access-list extended OUTBOUND
!  permit tcp 192.168.10.0 0.0.0.255 any eq 80
! ip access-list extended INBOUND
!  permit tcp any eq 80 192.168.10.0 0.0.0.255 established
```

---

## 면접관: "고객사에서 DMZ 웹서버로 들어오는 HTTP 트래픽에 대해 Deep Inspection을 하고 싶다고 합니다. 단순히 포트만 열어주는 것이 아니라 HTTP 프로토콜 레벨에서 검사하고 싶다는 건데요. ASA에서 어떻게 구현하나요?"

ASA의 **MPF (Modular Policy Framework)**를 사용하면 됩니다. MPF는 Cisco IOS의 MQC(Modular QoS CLI)와 유사한 구조로, **트래픽 분류 → 정책 정의 → 정책 적용**의 3단계로 동작합니다.

### MPF의 3요소 (C3PL과 유사)

```
+-------------------+     +-------------------+     +-------------------+
|  1. Class Map     | --> |  2. Policy Map    | --> |  3. Service Policy|
|  (트래픽 분류)     |     |  (정책 정의)       |     |  (정책 적용)       |
|  "어떤 트래픽?"    |     |  "무엇을 할 것인가?"|     |  "어디에 적용?"    |
+-------------------+     +-------------------+     +-------------------+
```

### HTTP Inspection 설정

```
! 1단계: Class Map - DMZ 웹서버로 향하는 HTTP 트래픽 분류
class-map CM_HTTP_INSPECT
 match port tcp eq www

! 2단계: Policy Map - HTTP inspection 적용
! 먼저 HTTP inspection 세부 정책 정의
policy-map type inspect http HTTP_INSPECT_POLICY
 parameters
  protocol-violation action drop-connection log
 match request header host length gt 256
  drop-connection log
 match request method connect
  drop-connection log
 match request uri length gt 2048
  drop-connection log

! 메인 Policy Map에 inspection 연결
policy-map PM_OUTSIDE
 class CM_HTTP_INSPECT
  inspect http HTTP_INSPECT_POLICY

! 3단계: Service Policy - Outside 인터페이스에 적용
service-policy PM_OUTSIDE interface outside
```

**HTTP Inspection이 수행하는 검사 항목 예시**:
- HTTP 프로토콜 위반 탐지 (RFC 비준수 요청)
- 비정상적으로 긴 URI, 헤더 차단
- CONNECT 메서드 차단 (터널링 방지)
- 특정 Content-Type 필터링
- ActiveX, Java applet 차단 가능

```
! 참고: Global Service Policy (기본값)
! ASA는 기본적으로 global_policy가 적용되어 있음
show running-config service-policy
! service-policy global_policy global

! 기본 global_policy 확인
show running-config policy-map global_policy
```

**중요**: ASA는 인터페이스당 하나의 service-policy만 적용 가능합니다. 그리고 global service-policy는 하나만 존재할 수 있습니다. 인터페이스별 service-policy와 global service-policy가 동시에 있으면, 해당 인터페이스에서는 인터페이스별 정책이 우선합니다.

---

## 면접관: "그런데 DMZ 웹서버에 연결이 갑자기 느려지면서, 일부 사용자는 접속이 아예 안 된다는 보고가 들어왔습니다. Connection limit 문제일 수 있을까요? 어디부터 확인하시겠어요?"

Connection limit 초과, DDoS 공격, 또는 리소스 고갈 등 여러 원인이 가능합니다. 순서대로 확인하겠습니다.

### 1단계: 현재 연결 상태 확인

```
! DMZ 웹서버 관련 연결 수 확인
show conn address 172.16.1.10
show conn count

! 특정 호스트의 연결 수만 카운트
show conn address 172.16.1.10 | include in use

! embryonic connection (반쪽 연결) 확인 - SYN Flood 의심
show conn address 172.16.1.10 detail | include embryonic
```

### 2단계: ASA 리소스 확인

```
show cpu usage
show memory
show blocks
show resource usage
```

### 3단계: MPF로 connection limit 설정 확인 및 조정

```
! 현재 connection limit 확인
show running-config policy-map
show service-policy

! Connection limit이 설정되어 있다면:
policy-map PM_DMZ_PROTECT
 class CM_DMZ_WEB
  set connection conn-max 5000
  set connection embryonic-conn-max 500
  set connection per-client-max 100
  set connection per-client-embryonic-max 20
```

각 파라미터의 의미:
- **conn-max**: 해당 class에 매칭되는 전체 최대 동시 연결 수
- **embryonic-conn-max**: 미완성(half-open) 연결 최대 수 (SYN Flood 방어)
- **per-client-max**: 클라이언트(출발지 IP)별 최대 연결 수
- **per-client-embryonic-max**: 클라이언트별 미완성 연결 최대 수

### 4단계: 실시간 모니터링

```
! 실시간 연결 통계
show service-policy interface outside

! 출력 예시:
! Class-map: CM_DMZ_WEB
!   set connection conn-max 5000, current 4987  ← 거의 한계!
!   set connection embryonic-conn-max 500, current 498  ← 거의 한계!
!   drop count: 1523  ← 차단된 연결 수

! TCP intercept 통계 (embryonic limit 초과 시 활성화)
show threat-detection statistics
```

만약 embryonic connection이 비정상적으로 많다면 **SYN Flood 공격**을 의심해야 하고, ASA의 **TCP intercept** 기능이 자동으로 동작하여 SYN proxy를 수행합니다.

---

## 면접관: "지금 말씀하신 TCP intercept 외에, ASA에서 제공하는 threat-detection 기능에 대해 좀 더 자세히 설명해주시겠어요?"

ASA의 **threat-detection**은 크게 3가지 레벨로 나뉩니다.

### 1. Basic Threat Detection (기본 활성화)

```
! 기본으로 활성화되어 있음
threat-detection basic-threat

! 통계 확인
show threat-detection rate

! 출력 예시:
!                             Average(eps)    Current(eps)   Trigger
! Scanning threat              0               0              0
! SYN attack                   15              234            1   ← 경고!
! ACL drop                     45              67             0
```

감지 항목: ACL drop, bad packet, connection limit, DoS, scanning, SYN attack 등

### 2. Advanced Threat Detection (통계 기반)

```
threat-detection statistics access-list
threat-detection statistics tcp-intercept rate-interval 30 burst-rate 400 average-rate 200

! ACL별 히트 통계
show threat-detection statistics access-list

! 상위 공격자/대상 확인
show threat-detection statistics top access-list
```

### 3. Scanning Threat Detection

```
threat-detection scanning-threat
threat-detection scanning-threat shun duration 3600

! 포트 스캔 감지 시 자동으로 공격자 IP를 shun (차단)
show threat-detection scanning-threat
show shun
```

**shun** 기능은 감지된 공격자 IP를 지정된 시간 동안 완전히 차단합니다. 단, 오탐(false positive) 가능성이 있으므로 운영 환경에서는 신중하게 설정해야 합니다.

```
! 수동으로 특정 IP 차단/해제
shun 198.51.100.50
no shun 198.51.100.50
```

### MPF와 threat-detection 조합

```
! 종합적인 DMZ 보호 정책
class-map CM_DMZ_SERVERS
 match access-list ACL_DMZ_TRAFFIC

policy-map PM_GLOBAL
 class CM_DMZ_SERVERS
  set connection conn-max 10000
  set connection embryonic-conn-max 1000
  set connection per-client-max 200
  set connection per-client-embryonic-max 50
  set connection timeout embryonic 0:00:15
  set connection timeout idle 1:00:00
  set connection timeout half-closed 0:05:00

service-policy PM_GLOBAL global
```

---

## 면접관: "고객사가 추가로 특정 국가에서 오는 트래픽을 전부 차단하고 싶다고 합니다. ASA에서 가능한가요?"

ASA 자체에는 국가 기반 IP 차단 기능(GeoIP)이 내장되어 있지 않습니다. 이를 구현하는 방법은 몇 가지가 있습니다.

### 방법 1: 수동 ACL로 IP 대역 차단

국가별 IP 대역 목록(예: RIPE, APNIC 등 RIR 데이터)을 가져와서 ACL로 작성하는 방법입니다.

```
! 예시: 특정 대역 차단
object-group network BLOCKED_COUNTRY_IPS
 network-object 45.0.0.0 255.0.0.0
 network-object 46.161.0.0 255.255.0.0
 ! ... (수백~수천 개의 엔트리)

access-list outside_access_in extended deny ip object-group BLOCKED_COUNTRY_IPS any log
```

단점: IP 대역은 지속적으로 변경되므로 주기적인 업데이트가 필요하고, ACL 엔트리가 매우 많아져 성능에 영향을 줄 수 있습니다.

### 방법 2: ASA with FirePOWER Services (SFR 모듈)

ASA에 FirePOWER Services 모듈(SFR)을 추가하면 **GeoIP 기반 차단**이 가능합니다.

```
! ASA에서 트래픽을 SFR 모듈로 리다이렉트
class-map CM_SFR
 match any

policy-map PM_SFR_REDIRECT
 class CM_SFR
  sfr fail-open

service-policy PM_SFR_REDIRECT interface outside
```

FirePOWER 모듈(FMC 관리)에서 GeoIP 정책을 설정하면 자동으로 국가별 차단이 가능하고, IP 데이터베이스도 자동 업데이트됩니다.

### 방법 3: FTD로 마이그레이션 (권장)

Firepower Threat Defense(FTD)로 전환하면 **Snort 엔진 + GeoIP + URL 필터링 + IPS/IDS**가 모두 통합되어 있어 가장 효율적입니다.

```
! FTD/FMC에서의 GeoIP 차단은 GUI에서 설정
! Access Control Policy → Rule → Source Networks → Geolocation
```

고객사의 보안 요구사항이 점점 높아지고 있다면, ASA에 기능을 추가하기보다 **FTD로의 마이그레이션을 중장기적으로 검토**하시는 것을 권장드리겠습니다.

---

## 면접관: "ACL을 관리하다 보면 규칙이 계속 늘어나는데, ACL 최적화 방법이 있을까요?"

실무에서 ACL 비대화는 매우 흔한 문제입니다. 최적화 방법을 정리하겠습니다.

**1. Object-group 적극 활용**

```
! 비효율적: 개별 규칙 나열
access-list OUT_IN permit tcp any host 172.16.1.10 eq 80
access-list OUT_IN permit tcp any host 172.16.1.10 eq 443
access-list OUT_IN permit tcp any host 172.16.1.20 eq 80
access-list OUT_IN permit tcp any host 172.16.1.20 eq 443

! 효율적: Object-group으로 통합
object-group network DG_DMZ_WEBSERVERS
 network-object host 172.16.1.10
 network-object host 172.16.1.20

object-group service SG_WEB tcp
 port-object eq www
 port-object eq https

access-list OUT_IN permit tcp any object-group DG_DMZ_WEBSERVERS object-group SG_WEB
```

4개의 ACE가 1개로 줄었습니다.

**2. 히트 카운트 기반 정리**

```
show access-list outside_access_in

! 출력에서 hitcnt=0인 규칙 식별
! access-list outside_access_in line 5 ... (hitcnt=0) 0x12345678
```

hitcnt=0인 규칙은 사용되지 않는 규칙일 가능성이 높으므로, 해당 서비스가 실제로 종료된 것인지 확인 후 제거합니다. 다만 hitcnt는 ASA 재부팅 시 초기화되므로, 충분한 기간 동안의 데이터를 기반으로 판단해야 합니다.

```
! 히트 카운트 초기화 (모니터링 시작점)
clear access-list outside_access_in counters
```

**3. 자주 매칭되는 규칙을 상위에 배치**

ASA는 ACL을 **top-down(위에서 아래)** 순서로 처리하므로, 트래픽 양이 많은 규칙을 상위에 놓으면 전체적인 처리 효율이 높아집니다. 단, ASA 내부적으로 hash table 기반 lookup을 사용하는 경우가 있어 실제 성능 차이는 크지 않을 수 있지만, 논리적 가독성 측면에서도 상위 배치가 유리합니다.

**4. Remark로 문서화**

```
access-list outside_access_in remark === DMZ Web Server Access ===
access-list outside_access_in remark Ticket#: CHG-2024-0156, Approved: 2024-03-15
access-list outside_access_in permit tcp any object OBJ_DMZ_WEB object-group SVC_WEB
```

ACL 규칙마다 변경 이력(티켓 번호, 승인 날짜)을 remark로 남기면, 나중에 불필요한 규칙을 식별하고 제거할 때 큰 도움이 됩니다.
