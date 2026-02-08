# 07-01. AAA (RADIUS / TACACS+)

---

### 면접관: "고객사에서 네트워크 장비가 50대 정도 되는데, 지금은 각 장비마다 로컬 계정으로 관리하고 있어요. 관리자 입퇴사 때마다 50대 전부 일일이 계정 수정하는 게 너무 비효율적이라 통합 인증 체계를 구축해달라고 합니다. 어떻게 설계하시겠어요?"

AAA 서버를 중앙에 구축해서 TACACS+ 기반의 인증/인가/어카운팅 체계를 설계하겠습니다.

전체 구조는 다음과 같습니다.

- **AAA 서버**: Cisco ISE 또는 ACS를 이중화로 구성 (Primary / Secondary)
- **프로토콜**: 네트워크 장비 관리 접근이므로 TACACS+ 선택
- **인증 소스**: Active Directory와 연동하여 기존 사내 계정 체계를 그대로 활용
- **권한 분리**: NOC 운영자(show 명령어만), 시니어 엔지니어(config 가능), 관리자(전체 권한) 3단계 구성
- **Fallback**: AAA 서버 장애 시 로컬 계정으로 로그인 가능하도록 method list에 local 추가

> 토폴로지 이미지 추가 예정

#### 핵심 설정

```
! ---- AAA 서버 정의 ----
aaa new-model
!
tacacs server ISE-PRIMARY
 address ipv4 10.10.100.10
 key 0 T@c@csK3y!2025
 timeout 5
!
tacacs server ISE-SECONDARY
 address ipv4 10.10.100.11
 key 0 T@c@csK3y!2025
 timeout 5
!
aaa group server tacacs+ TACACS-GROUP
 server name ISE-PRIMARY
 server name ISE-SECONDARY
!
! ---- 인증 (Authentication) ----
aaa authentication login AUTH-LIST group TACACS-GROUP local
aaa authentication enable default group TACACS-GROUP enable
!
! ---- 인가 (Authorization) ----
aaa authorization exec AUTHOR-LIST group TACACS-GROUP local
aaa authorization commands 15 AUTHOR-CMD group TACACS-GROUP local
!
! ---- 어카운팅 (Accounting) ----
aaa accounting exec default start-stop group TACACS-GROUP
aaa accounting commands 15 default start-stop group TACACS-GROUP
!
! ---- VTY 적용 ----
line vty 0 15
 login authentication AUTH-LIST
 authorization exec AUTHOR-LIST
 authorization commands 15 AUTHOR-CMD
 transport input ssh
!
! ---- 로컬 Fallback 계정 (반드시 생성) ----
username admin privilege 15 secret F@llb@ckP@ss!
```

50대 장비에 대한 배포는 Ansible이나 Python netmiko 스크립트를 활용해서 일괄 적용하고, 사전에 소수 장비에서 테스트를 반드시 진행합니다. 잘못 적용하면 전 장비 로그인 불가 사태가 발생할 수 있어서 local fallback 계정을 먼저 생성한 후 AAA 설정을 적용하는 순서가 절대적으로 중요합니다.

---

### 면접관: "TACACS+를 선택하셨는데, RADIUS도 있잖아요. 왜 TACACS+를 선택했고, 둘의 차이는 뭔가요?"

네트워크 장비 관리 접근 제어 목적이기 때문에 TACACS+를 선택했습니다. 핵심 차이는 다음과 같습니다.

| 항목 | RADIUS | TACACS+ |
|------|--------|---------|
| **개발** | Livingston (현재 IETF 표준) | Cisco 독자 개발 |
| **전송 프로토콜** | UDP 1812/1813 | TCP 49 |
| **암호화** | 패스워드만 암호화 | 전체 Payload 암호화 |
| **AAA 분리** | 인증+인가 결합 | 인증/인가/어카운팅 완전 분리 |
| **명령어 인가** | 불가 | 명령어 단위 인가 가능 |
| **주 용도** | 네트워크 접속 (802.1X, VPN) | 장비 관리 접근 제어 |
| **멀티프로토콜** | 지원 (PPP, SLIP 등) | 미지원 |

TACACS+의 결정적 장점은 **명령어 레벨 인가(Command Authorization)** 입니다. 예를 들어 주니어 엔지니어에게는 `show` 명령어만 허용하고, `configure terminal`이나 `reload` 같은 위험한 명령어는 차단할 수 있습니다. RADIUS는 이런 세밀한 인가가 불가능합니다.

또한 TCP 기반이므로 서버와의 연결 상태를 확인할 수 있고, 전체 패킷이 암호화되기 때문에 장비 관리 트래픽의 보안성이 훨씬 높습니다.

반대로 802.1X나 VPN 사용자 인증 같은 대규모 사용자 인증에는 RADIUS가 적합합니다. UDP 기반이라 오버헤드가 적고, IETF 표준이므로 멀티벤더 환경에서 호환성이 좋습니다.

---

### 면접관: "그런데 운영 중에 AAA 서버가 둘 다 장애가 났습니다. 엔지니어가 장비에 로그인을 전혀 못 한다고 연락이 왔어요. 어디부터 확인하시겠어요?"

이 상황은 AAA 설계에서 가장 치명적인 시나리오입니다. 단계별로 대응하겠습니다.

**1단계: 로컬 Fallback 동작 여부 확인**

method list에 `local`이 포함되어 있다면, TACACS+ 서버가 응답하지 않을 때 자동으로 local DB를 참조합니다. 하지만 여기서 중요한 구분이 있습니다.

- **서버 Unreachable (timeout)** → method list의 다음 method로 fallback **됨**
- **서버가 Reject 응답** → fallback **안 됨** (인증 실패로 확정)

즉, 서버가 완전히 죽었으면 local fallback이 동작하지만, 서버가 살아있는데 계정 정보가 틀려서 Reject를 보내면 local로 넘어가지 않습니다.

**2단계: 콘솔 접근으로 긴급 복구**

VTY 접속이 불가하므로 콘솔 포트로 직접 접근합니다.

```
! 콘솔에서 로컬 계정으로 로그인 시도
! 콘솔 라인에 별도 method list가 없다면 default가 적용됨

! 긴급 복구: 콘솔 라인을 로컬 인증으로 변경
line console 0
 login authentication LOCAL-ONLY
!
aaa authentication login LOCAL-ONLY local
```

**3단계: AAA 서버 연결 상태 확인**

```
show tacacs
show aaa servers
show aaa method-list
!
! 서버 연결 테스트
ping 10.10.100.10
ping 10.10.100.11
!
! AAA 디버그 (주의: CPU 부하 발생)
debug aaa authentication
debug aaa authorization
debug tacacs
```

**4단계: 근본 원인 분석**

`show aaa servers` 출력에서 각 서버별 Transaction 성공/실패 카운터를 확인하고, 서버 측 로그와 대조합니다. 일반적인 원인으로는 Key mismatch, 서버 프로세스 다운, 방화벽에서 TCP 49 차단, 서버 IP 변경 등이 있습니다.

**재발 방지**: 콘솔 라인에는 반드시 로컬 전용 인증을 적용하고, AAA 서버 모니터링 알람을 설정합니다.

---

### 면접관: "method list 얘기가 나왔는데, method list가 정확히 뭔가요? default랑 named method list의 차이도 설명해주세요."

method list는 AAA 인증/인가/어카운팅 시 어떤 방법(method)을 어떤 순서로 시도할지를 정의하는 리스트입니다.

**default method list**는 이름 그대로 모든 라인(VTY, Console, AUX)에 자동 적용됩니다. 별도로 라인에서 지정하지 않아도 동작합니다.

```
! default method list: 모든 라인에 자동 적용
aaa authentication login default group TACACS-GROUP local
```

**named method list**는 특정 라인이나 인터페이스에 명시적으로 적용해야 동작합니다.

```
! named method list: 지정한 라인에만 적용
aaa authentication login VTY-AUTH group TACACS-GROUP local
aaa authentication login CONSOLE-AUTH local
!
line vty 0 15
 login authentication VTY-AUTH        ! VTY는 TACACS+ → local
!
line console 0
 login authentication CONSOLE-AUTH    ! 콘솔은 local만
```

실무 Best Practice는 다음과 같습니다.

1. **default method list에는 가장 보수적인 설정** 적용 (예: local만)
2. **VTY에는 named method list**로 TACACS+ + local fallback 적용
3. **Console에는 반드시 local 전용** named method list 적용

이렇게 하면 AAA 서버가 완전히 장애 나도 콘솔로는 반드시 로그인할 수 있습니다. 가장 흔한 실수가 `aaa authentication login default group tacacs+ none`으로 설정하는 건데, `none`을 쓰면 서버 장애 시 인증 없이 누구나 접속 가능해지므로 보안 위험이 큽니다.

method list의 동작 순서를 정리하면:

```
aaa authentication login AUTH-LIST group TACACS-GROUP local
                                    [1순위]           [2순위]

1순위 시도 → 서버 응답 없음(timeout) → 2순위로 fallback
1순위 시도 → 서버가 Reject 응답    → 인증 실패 (fallback 안 함)
1순위 시도 → 서버가 Accept 응답    → 인증 성공
```

---

### 면접관: "고객사에서 추가로 VPN 사용자 인증도 AAA로 통합해달라고 합니다. 그리고 이미 무선 네트워크에서 ISE로 802.1X를 쓰고 있는데, VPN 인증도 같은 ISE를 쓰고 싶다고요. 어떻게 하시겠어요?"

VPN 사용자 인증은 RADIUS 프로토콜이 적합합니다. 기존 ISE를 RADIUS 서버로 활용하면 됩니다.

한 장비에서 TACACS+(장비 관리)와 RADIUS(VPN 사용자)를 동시에 사용하는 구성입니다.

```
! ---- 기존 TACACS+ (장비 관리용) 유지 ----
aaa group server tacacs+ TACACS-GROUP
 server name ISE-PRIMARY
 server name ISE-SECONDARY
!
! ---- RADIUS 추가 (VPN 사용자 인증용) ----
radius server ISE-RAD-PRIMARY
 address ipv4 10.10.100.10 auth-port 1812 acct-port 1813
 key 0 R@d1usK3y!2025
 timeout 5
!
radius server ISE-RAD-SECONDARY
 address ipv4 10.10.100.11 auth-port 1812 acct-port 1813
 key 0 R@d1usK3y!2025
 timeout 5
!
aaa group server radius RADIUS-GROUP
 server name ISE-RAD-PRIMARY
 server name ISE-RAD-SECONDARY
!
! ---- VPN 인증용 method list (RADIUS) ----
aaa authentication login VPN-AUTH group RADIUS-GROUP local
aaa authorization network VPN-AUTHOR group RADIUS-GROUP
aaa accounting network VPN-ACCT start-stop group RADIUS-GROUP
!
! ---- 장비 관리용 method list (TACACS+) 유지 ----
aaa authentication login DEVICE-AUTH group TACACS-GROUP local
!
! ---- 적용 ----
line vty 0 15
 login authentication DEVICE-AUTH
!
crypto map VPN-MAP client authentication list VPN-AUTH
crypto map VPN-MAP isakmp authorization list VPN-AUTHOR
```

ISE 측에서는 장비 관리 요청(TACACS+, TCP 49)과 VPN 인증 요청(RADIUS, UDP 1812/1813)을 별도 Policy Set으로 분리합니다. 같은 ISE지만 프로토콜이 다르므로 자연스럽게 구분됩니다.

---

### 면접관: "최근에 CoA(Change of Authorization)를 도입하고 싶다는 요구도 있습니다. CoA가 뭔지, 어떤 경우에 쓰는 건지 설명해주세요."

CoA(Change of Authorization, RFC 5176)는 **RADIUS 서버가 NAS(Network Access Server)에게 먼저 요청을 보내서** 이미 인증된 세션의 권한을 실시간으로 변경하는 기능입니다.

일반적인 RADIUS 흐름은 클라이언트(NAS) → 서버 방향이지만, CoA는 **서버 → 클라이언트 방향**으로 동작합니다. 이것이 핵심 차이입니다.

**주요 사용 시나리오:**

1. **Posture 평가 결과 반영**: 802.1X 인증 후 단말이 접속 → ISE가 Posture 체크(백신 설치 여부 등) → 미준수 시 CoA로 해당 세션을 격리 VLAN으로 즉시 변경
2. **게스트 승인**: 게스트가 자가등록 후 스폰서가 승인 → CoA로 제한 VLAN에서 인터넷 접속 VLAN으로 실시간 변경
3. **침해 대응**: SIEM에서 악성 행위 탐지 → ISE 연동 → CoA로 해당 단말 세션 즉시 종료(Disconnect)

```
! ---- NAS(스위치) 측 CoA 설정 ----
aaa server radius dynamic-author
 client 10.10.100.10 server-key R@d1usK3y!2025
 client 10.10.100.11 server-key R@d1usK3y!2025
 port 1700
 auth-type any
!
! CoA 동작 확인
show aaa server radius dynamic-author
debug radius dynamic-authorization
```

CoA 메시지 타입은 크게 세 가지입니다.

| 메시지 | 동작 | 용도 |
|--------|------|------|
| **CoA-Request** | 세션 속성 변경 (VLAN, ACL 등) | Posture 결과 반영, 권한 변경 |
| **Disconnect-Request** | 세션 강제 종료 | 침해 단말 격리, 세션 정리 |
| **CoA-Reauthenticate** | 재인증 트리거 | 정책 변경 후 즉시 반영 |

CoA가 동작하려면 NAS 측에서 UDP 1700(또는 3799) 포트로 ISE의 요청을 수신할 수 있어야 하므로, 중간 방화벽에서 해당 포트가 열려 있는지도 반드시 확인해야 합니다.

---

### 면접관: "마지막으로, AAA 설정을 50대 장비에 배포할 때 실수로 잠기는 걸 방지하는 실무 팁이 있다면?"

AAA 배포 시 Lock-out 방지는 실무에서 가장 중요한 부분입니다. 순서를 반드시 지켜야 합니다.

**배포 순서 (절대 규칙)**

```
! ---- Step 1: 로컬 Fallback 계정 먼저 생성 ----
username emergency privilege 15 secret Em3rg3ncy!
!
! ---- Step 2: 콘솔 라인 로컬 전용 설정 ----
aaa authentication login CONSOLE-LOCAL local
line console 0
 login authentication CONSOLE-LOCAL
!
! ---- Step 3: aaa new-model 적용 ----
aaa new-model
!
! ---- Step 4: TACACS+ 서버 및 method list 설정 ----
! (이 시점에서 문제 발생해도 콘솔로 접근 가능)
!
! ---- Step 5: 반드시 새 SSH 세션으로 테스트 ----
! (기존 세션은 유지한 채로 새 세션 열어서 확인)
!
! ---- Step 6: 문제 없으면 write memory ----
write memory
```

**추가 안전장치:**

```
! Resilient Login: AAA 서버 전체 장애 시 로컬 접근 보장
aaa new-model
aaa local authentication attempts max-fail 5
!
! Archive + Rollback: 설정 실수 시 자동 복구
archive
 path flash:rollback-config
!
configure terminal revert timer 10
! → 10분 내 confirm 없으면 자동으로 이전 설정 복구
! → confirm하려면:
configure confirm
```

`configure terminal revert timer`는 AAA 배포에서 가장 강력한 안전장치입니다. 설정 변경 후 10분 내에 `configure confirm`을 입력하지 않으면 자동으로 원래 설정으로 롤백됩니다. 만약 AAA 설정 실수로 세션이 끊겨도, 10분 후 자동 복구되므로 50대 장비를 일일이 콘솔로 복구하는 최악의 상황을 방지할 수 있습니다.
