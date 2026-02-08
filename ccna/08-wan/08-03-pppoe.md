# 08-03. PPPoE 기본

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 시나리오

---

### 면접관: "회사에서 ISP 인터넷 회선을 연결하는데 PPPoE 방식을 사용한다고 합니다. PPPoE가 무엇이고 왜 필요한지 설명해 주세요."

PPPoE(Point-to-Point Protocol over Ethernet)는 이더넷 네트워크 위에서 PPP 프레임을 캡슐화하여 전달하는 프로토콜입니다.

PPPoE가 필요한 이유는 PPP의 장점과 이더넷의 장점을 결합하기 위해서입니다.

**PPP가 제공하는 기능:**
- **인증(Authentication)**: 가입자를 ID/Password로 인증할 수 있습니다.
- **IP 주소 할당**: ISP가 가입자에게 동적으로 공인 IP를 할당합니다.
- **과금(Accounting)**: 세션 시작/종료를 추적하여 사용량 기반 과금이 가능합니다.

**이더넷이 제공하는 장점:**
- 저렴한 이더넷 인프라를 그대로 활용할 수 있습니다.
- 기존 DSL(ADSL, VDSL) 모뎀이나 광랜 장비와 호환됩니다.

한국에서는 KT, SK브로드밴드, LG U+ 등 ISP가 가정이나 기업에 인터넷 서비스를 제공할 때 PPPoE를 많이 사용합니다. 특히 **동적 IP 할당**과 **가입자 인증**이 필요한 환경에서 표준적인 방식입니다.

---

### 면접관: "PPPoE의 동작 과정을 단계별로 설명해 주세요. 연결이 수립되기까지 어떤 과정을 거칩니까?"

PPPoE 연결은 크게 **두 단계**로 나뉩니다.

#### 1단계: Discovery (탐색 단계)

이 단계에서 클라이언트가 PPPoE 서버(ISP의 Access Concentrator)를 찾고 세션을 설정합니다.

| 순서 | 패킷 | 방향 | 설명 |
|------|------|------|------|
| 1 | **PADI** (Initiation) | 클라이언트 -> 브로드캐스트 | PPPoE 서버를 찾기 위한 브로드캐스트 |
| 2 | **PADO** (Offer) | 서버 -> 클라이언트 | 서버가 자신을 알리는 응답 |
| 3 | **PADR** (Request) | 클라이언트 -> 서버 | 클라이언트가 특정 서버를 선택 |
| 4 | **PADS** (Session-Confirm) | 서버 -> 클라이언트 | 세션 ID 할당, Discovery 완료 |

#### 2단계: PPP Session (세션 단계)

Discovery가 완료되면 PPP 세션이 시작됩니다.

1. **LCP (Link Control Protocol)**: 링크 파라미터 협상 (MTU, 인증 방식 등)
2. **Authentication**: PAP 또는 CHAP 인증 수행
3. **NCP/IPCP (Network Control Protocol)**: IP 주소 할당 및 네트워크 계층 설정

이 모든 과정이 완료되면 클라이언트는 ISP로부터 공인 IP 주소를 받고 인터넷에 접속할 수 있습니다.

---

### 면접관: "PAP과 CHAP 인증의 차이를 설명하고, 어떤 것을 사용하는 것이 좋습니까?"

PAP과 CHAP은 PPP에서 사용하는 두 가지 인증 방식입니다.

#### PAP (Password Authentication Protocol)

- ID와 Password를 **평문(cleartext)**으로 전송합니다.
- **2-Way Handshake** 방식입니다.
  1. 클라이언트가 ID/Password를 전송
  2. 서버가 Accept 또는 Reject 응답

#### CHAP (Challenge Handshake Authentication Protocol)

- Password를 **해시(MD5)**로 변환하여 전송하므로 평문 노출이 없습니다.
- **3-Way Handshake** 방식입니다.
  1. 서버가 Challenge(랜덤 값)를 전송
  2. 클라이언트가 Challenge + Password의 MD5 해시를 전송
  3. 서버가 Success 또는 Failure 응답
- 세션 유지 중에도 **주기적으로 재인증**을 수행하여 보안성이 높습니다.

| 비교 항목 | PAP | CHAP |
|-----------|-----|------|
| **비밀번호 전송** | 평문 | MD5 해시 |
| **핸드셰이크** | 2-Way | 3-Way |
| **재인증** | 없음 | 주기적 수행 |
| **보안성** | 낮음 | 높음 |
| **사용 권장** | 비권장 | 권장 |

실무에서는 반드시 **CHAP**을 사용해야 합니다. PAP은 패킷 캡처 시 비밀번호가 그대로 노출되기 때문에 보안 위험이 큽니다.

---

### 면접관: "Cisco 라우터에서 PPPoE 클라이언트를 설정하는 방법을 보여 주세요. ISP에서 ID는 'company@isp.co.kr', 비밀번호는 'pass1234'를 받았습니다."

PPPoE 클라이언트 설정은 Dialer 인터페이스를 사용합니다.

#### PPPoE 클라이언트 설정

```
! 1. CHAP 인증 정보 설정
Router(config)# username company@isp.co.kr password pass1234

! 2. Dialer 인터페이스 생성
Router(config)# interface dialer 1
Router(config-if)# ip address negotiated
Router(config-if)# ip mtu 1492
Router(config-if)# encapsulation ppp
Router(config-if)# ppp authentication chap callin
Router(config-if)# ppp chap hostname company@isp.co.kr
Router(config-if)# ppp chap password pass1234
Router(config-if)# dialer pool 1
Router(config-if)# no shutdown
Router(config-if)# exit

! 3. 물리 인터페이스에 PPPoE 클라이언트 연결
Router(config)# interface GigabitEthernet 0/0
Router(config-if)# pppoe enable
Router(config-if)# pppoe-client dial-pool-number 1
Router(config-if)# no ip address
Router(config-if)# no shutdown
Router(config-if)# exit

! 4. 기본 라우팅 설정
Router(config)# ip route 0.0.0.0 0.0.0.0 dialer 1
```

#### 핵심 설정 항목 설명

| 명령어 | 설명 |
|--------|------|
| `ip address negotiated` | ISP로부터 동적으로 IP 할당 |
| `ip mtu 1492` | PPPoE 오버헤드(8바이트)를 고려한 MTU |
| `encapsulation ppp` | PPP 캡슐화 지정 |
| `dialer pool 1` | 물리 인터페이스와 연결하는 Pool 번호 |
| `pppoe enable` | 물리 인터페이스에서 PPPoE 활성화 |
| `pppoe-client dial-pool-number 1` | Dialer Pool 연결 |

---

### 면접관: "MTU를 1492로 설정한 이유를 좀 더 자세히 설명해 주세요. 이 값이 왜 중요합니까?"

PPPoE를 사용하면 이더넷 프레임 안에 PPPoE 헤더(6바이트)와 PPP 헤더(2바이트), 총 **8바이트의 오버헤드**가 추가됩니다.

```
[이더넷 헤더: 14B] [PPPoE 헤더: 6B] [PPP 헤더: 2B] [IP 패킷: 최대 1492B] [FCS: 4B]
```

이더넷의 최대 페이로드(MTU)가 1500바이트이므로, PPPoE 오버헤드 8바이트를 빼면 IP 패킷의 최대 크기는 **1492바이트**가 됩니다.

이 값을 올바르게 설정하지 않으면 다음과 같은 문제가 발생합니다.

1. **대형 패킷 전송 실패**: 1492바이트를 초과하는 IP 패킷은 단편화되거나 드롭됩니다.
2. **특정 웹사이트 접속 불가**: HTTPS 사이트에서 DF 비트가 설정된 큰 패킷이 드롭되어 접속이 안 되는 증상이 나타납니다.
3. **Path MTU Discovery 실패**: 일부 방화벽이 ICMP "Fragmentation Needed" 메시지를 차단하면 PMTUD가 동작하지 않습니다.

#### 해결 방법

```
! Dialer 인터페이스에 MTU 설정
Router(config-if)# ip mtu 1492

! TCP MSS 클램핑 (가장 효과적)
Router(config-if)# ip tcp adjust-mss 1452
```

TCP MSS를 1452로 설정하는 이유는 1492(PPPoE MTU) - 40(TCP 20바이트 + IP 20바이트) = 1452이기 때문입니다. 이렇게 하면 TCP 세션 수립 시 MSS 값을 자동으로 조정하여 단편화를 예방합니다.

---

### 면접관: "PPPoE 세션이 정상적으로 연결되었는지 확인하는 방법을 알려 주세요."

PPPoE 세션 확인에 사용하는 주요 명령어와 확인 포인트입니다.

#### PPPoE 세션 상태 확인

```
Router# show pppoe session

     1 session  in LOCALLY_TERMINATED (PTA) State
     1 session  total

Uniq ID  PPPoE  RemMAC          Port        VT  VA      State
           SID  LocMAC                              VA-st
    1      1    aabb.cc00.0100  Gi0/0        1   Vi1     PTA
                aabb.cc00.0200                      UP
```

#### Dialer 인터페이스 상태 확인

```
Router# show interface dialer 1
Dialer1 is up, line protocol is up (spoofing)
  Internet address is 211.xxx.xxx.xxx/32
  MTU 1492 bytes, BW 56 Kbit/sec
  Encapsulation PPP
  DTR is pulsed, Idle timer is 120 seconds
```

#### PPP 인증 상태 확인

```
Router# show ppp all
Interface/ID  OPEN+ Nego* Fail-  Stage    Peer Address  Peer Name
Vi1           LCP+  CHAP+ IPCP+  LocalT   211.x.x.x    ISP-AC
```

#### IP 주소 할당 확인

```
Router# show ip interface brief
Interface          IP-Address      OK? Method Status Protocol
GigabitEthernet0/0 unassigned      YES unset  up     up
Dialer1            211.xxx.xxx.xxx YES IPCP   up     up
Virtual-Access1    unassigned      YES unset  up     up
```

핵심 확인 포인트를 정리하면 다음과 같습니다.

| 확인 항목 | 정상 상태 | 문제 시 조치 |
|-----------|-----------|-------------|
| PPPoE Session State | PTA / UP | PADI 재시도, ISP 확인 |
| Dialer 인터페이스 | up/up | 인증 정보 확인 |
| IP 주소 할당 | IPCP로 할당됨 | ISP IP Pool 확인 |
| PPP 인증 | CHAP+ (성공) | ID/Password 확인 |

---

### 면접관: "PPPoE 연결이 안 될 때 트러블슈팅 순서를 설명해 주세요."

PPPoE 연결 장애는 단계별로 접근합니다.

**1단계 - 물리 계층 확인**

```
Router# show interface GigabitEthernet 0/0
! 인터페이스가 up/up 인지 확인
! 케이블 연결, 모뎀/ONT 장비 상태 확인
```

**2단계 - PPPoE Discovery 확인**

```
Router# debug pppoe events
! PADI가 전송되는지 확인
! PADO를 수신하는지 확인
! PADO가 오지 않으면 ISP 장비 문제 또는 VLAN 설정 오류
```

**3단계 - PPP 인증 확인**

```
Router# debug ppp authentication
! CHAP Challenge/Response 과정 확인
! 인증 실패 시 ID/Password 재확인
! ISP에서 계정이 활성화되어 있는지 확인
```

**4단계 - IPCP(IP 할당) 확인**

```
Router# debug ppp negotiation
! IPCP 협상에서 IP 주소가 정상 할당되는지 확인
! IP 할당 실패 시 ISP의 IP Pool 문제 가능성
```

**5단계 - 라우팅 확인**

```
Router# show ip route
! Default Route가 Dialer 인터페이스를 가리키는지 확인
Router# ping 8.8.8.8 source dialer 1
! 외부 통신 테스트
```

트러블슈팅 시 가장 흔한 원인은 **인증 정보 오류**입니다. 특히 hostname에 도메인(@isp.co.kr)을 빠뜨리거나, password의 대소문자가 틀린 경우가 많습니다.

---

### 면접관: "마지막으로, PPPoE가 아닌 IPoE라는 것도 있다고 들었습니다. 차이점을 간단히 설명해 주세요."

IPoE(IP over Ethernet)는 PPPoE와 달리 PPP 세션 없이 **DHCP**를 통해 직접 IP 주소를 할당받는 방식입니다.

| 비교 항목 | PPPoE | IPoE |
|-----------|-------|------|
| **인증 방식** | PAP/CHAP (ID/PW) | DHCP Option 82, 포트 기반 |
| **IP 할당** | IPCP | DHCP |
| **세션 관리** | PPP 세션 유지 | 세션 개념 없음 |
| **오버헤드** | 8바이트 | 없음 |
| **MTU** | 1492 | 1500 |
| **설정 복잡도** | Dialer 인터페이스 필요 | DHCP 클라이언트만 필요 |

최근에는 FTTH(광가입자망) 환경에서 IPoE 방식이 점차 증가하고 있습니다. 오버헤드가 없어 MTU 문제가 발생하지 않고, 설정이 간단하다는 장점이 있습니다. 하지만 PPPoE의 세션 기반 관리와 인증 기능이 여전히 필요한 ISP도 많아, 두 방식이 공존하고 있습니다.

한국에서는 ISP마다 다르지만, 기업용 회선에서는 PPPoE가, 가정용에서는 IPoE(DHCP)가 많이 사용되는 추세입니다.
