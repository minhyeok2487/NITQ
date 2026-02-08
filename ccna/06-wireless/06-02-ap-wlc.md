# 06-02. AP 모드와 WLC

> CCNA 200-301 AP 아키텍처 및 WLC 면접 시나리오

---

### 면접관: "현재 우리 회사에 AP가 50대 정도 설치되어 있는데, 각 AP에 직접 접속해서 설정을 변경하고 있습니다. 이 방식의 문제점과 더 나은 대안이 있다면 설명해 주세요."

네, 현재 사용하시는 방식은 **Autonomous AP(자율형 AP)** 모드로 운영하는 것으로 보입니다. 이 방식에서는 각 AP가 독립적으로 동작하며 개별적으로 설정을 관리해야 합니다.

**Autonomous AP의 문제점:**
- AP 50대에 **개별적으로 접속하여 설정을 변경**해야 하므로 관리 부담이 큽니다
- SSID, 보안 정책, 채널 설정 등의 **일관성 유지가 어렵습니다**
- 전체 네트워크의 무선 상태를 **한눈에 파악하기 힘듭니다**
- 채널 및 전력 **자동 최적화가 불가능**합니다
- AP 추가 시마다 수동으로 전체 설정을 해야 합니다

이에 대한 대안으로 **WLC(Wireless LAN Controller) 기반의 Lightweight AP 아키텍처**를 권장합니다.

| 구분 | Autonomous AP | Lightweight AP + WLC |
|------|--------------|---------------------|
| 설정 관리 | AP 개별 관리 | WLC에서 중앙 집중 관리 |
| 채널/파워 | 수동 설정 | RRM으로 자동 최적화 |
| 펌웨어 업데이트 | AP별 개별 작업 | WLC에서 일괄 배포 |
| 보안 정책 | AP별 개별 적용 | WLC에서 통합 적용 |
| 확장성 | 낮음 | 높음 (수천 대 관리 가능) |
| 로밍 | 기본 로밍만 가능 | 빠른 로밍 지원 |
| 비용 | AP만 구매 | WLC 추가 비용 발생 |

AP가 50대 규모라면 WLC 도입을 통해 운영 효율성을 크게 높일 수 있습니다.

---

### 면접관: "Lightweight AP와 WLC 사이의 통신은 어떤 방식으로 이루어지나요? CAPWAP이라는 용어를 들어본 적 있는데 설명해 주세요."

CAPWAP(Control And Provisioning of Wireless Access Points)은 Lightweight AP와 WLC 사이의 **터널링 프로토콜**입니다. UDP 기반으로 동작하며 두 가지 터널을 사용합니다.

**1. CAPWAP Control Tunnel (UDP 5246):**
- AP 관리 및 제어 트래픽을 전달합니다
- AP 설정, 펌웨어 관리, 채널/파워 제어 등에 사용됩니다
- **DTLS(Datagram Transport Layer Security)**로 암호화되어 있습니다
- 항상 암호화가 적용됩니다

**2. CAPWAP Data Tunnel (UDP 5247):**
- 클라이언트의 **실제 사용자 데이터**를 전달합니다
- AP에서 수신한 무선 클라이언트 트래픽을 WLC로 전달합니다
- DTLS 암호화는 **선택 사항**입니다 (기본값: 비암호화)

```
[무선 클라이언트] ---(802.11)--- [Lightweight AP] ---(CAPWAP 터널)--- [WLC]
                                                    UDP 5246: 제어
                                                    UDP 5247: 데이터
```

CAPWAP 터널은 **Layer 3 터널**이므로, AP와 WLC가 같은 서브넷에 있을 필요가 없습니다. 이는 대규모 네트워크에서 WLC를 데이터 센터에 중앙 집중 배치하고 AP를 여러 지점에 분산 배치할 수 있게 해줍니다.

참고로 CAPWAP 이전에는 Cisco 독자 프로토콜인 **LWAPP**이 사용되었으나, 현재는 표준 기반인 CAPWAP으로 전환되었습니다.

---

### 면접관: "Split-MAC 아키텍처라는 개념을 설명해 주세요. AP와 WLC가 각각 어떤 역할을 담당하나요?"

Split-MAC 아키텍처는 **무선 MAC 계층의 기능을 AP와 WLC가 나누어 처리**하는 구조입니다.

#### AP가 담당하는 기능 (실시간 처리 필요)

- **무선 프레임 송수신** (실제 RF 통신)
- **Beacon 프레임 전송** (주기적 브로드캐스트)
- **Probe Response 응답** (클라이언트 검색 응답)
- **ACK 프레임 처리** (프레임 수신 확인)
- **패킷 암호화/복호화** (데이터 보호)
- **클라이언트 신호 강도 측정** (RSSI)

#### WLC가 담당하는 기능 (중앙 관리)

- **인증 및 보안 정책 적용** (802.1X, WPA2/WPA3)
- **클라이언트 연결(Association) 관리**
- **로밍 처리** (AP 간 클라이언트 이동)
- **SSID 및 WLAN 설정 관리**
- **채널 및 전력 자동 조정** (RRM: Radio Resource Management)
- **QoS 정책 적용**
- **AP 펌웨어 관리**
- **보안 위협 탐지** (Rogue AP 등)

이렇게 역할을 나누는 이유는, **실시간으로 처리해야 하는 RF 관련 작업**은 AP에서 처리하고, **정책 결정이나 관리 기능**은 WLC에서 중앙 집중 처리하여 효율성을 높이기 위함입니다. AP는 단순히 "무선 안테나" 역할에 집중하고, "두뇌" 역할은 WLC가 담당한다고 생각하시면 됩니다.

---

### 면접관: "지사에 있는 AP가 본사 WLC와 WAN으로 연결되어 있는데, WAN 링크에 장애가 발생하면 지사 무선 네트워크가 완전히 중단될 수 있나요? 이에 대한 해결 방안이 있을까요?"

매우 중요한 포인트를 짚어 주셨습니다. 일반적인 Lightweight AP의 **Local Mode**에서는 모든 데이터 트래픽이 CAPWAP 터널을 통해 WLC로 전달되므로, WAN 장애 시 무선 서비스가 중단될 수 있습니다.

이 문제를 해결하기 위해 **FlexConnect(이전 명칭: H-REAP)** 모드를 사용합니다.

#### FlexConnect 모드 동작 방식

**WLC와 연결된 상태 (Connected Mode):**
- 제어 트래픽은 WLC로 전송합니다 (CAPWAP Control)
- 데이터 트래픽은 설정에 따라 두 가지 방식으로 처리됩니다:
  - **중앙 스위칭(Central Switching)**: 데이터를 WLC로 보냄
  - **로컬 스위칭(Local Switching)**: 데이터를 AP에서 직접 로컬 네트워크로 전달

**WLC와 연결이 끊긴 상태 (Standalone Mode):**
- AP가 **독립적으로 동작**합니다
- 클라이언트 인증을 AP가 직접 처리합니다 (사전에 캐시된 자격 증명 사용)
- 데이터 트래픽을 **로컬에서 직접 스위칭**합니다
- 기존 연결된 클라이언트는 서비스가 유지됩니다

```
[본사]                              [지사]
+-------+    WAN Link    +-----------+    무선    +----------+
|  WLC  |<----CAPWAP---->| FlexConnect|<--------->| 클라이언트 |
+-------+                |    AP      |           +----------+
                          +-----+-----+
                                |
                          로컬 스위칭 (WAN 장애 시에도 동작)
                                |
                         +------+------+
                         | 로컬 서버/  |
                         | 프린터 등   |
                         +-------------+
```

따라서 지사에는 FlexConnect 모드를 적용하고, **로컬 스위칭**을 활성화하여 WAN 장애 시에도 지사 내부의 무선 서비스가 유지되도록 구성하는 것을 권장합니다.

---

### 면접관: "새로운 AP를 네트워크에 연결하면 WLC를 자동으로 찾는다고 들었는데, AP가 WLC를 발견하고 등록하는 과정을 설명해 주세요."

네, Lightweight AP의 **WLC Discovery 및 Join 과정**은 다음 단계로 진행됩니다.

#### AP의 WLC 탐색 방법 (Discovery)

AP는 다음 방법들을 **순차적 또는 병렬적으로** 시도하여 WLC를 찾습니다:

1. **Previously known WLC**: AP가 이전에 연결했던 WLC 주소를 기억하고 있으면 먼저 시도합니다
2. **DHCP Option 43**: DHCP 서버가 AP에게 WLC의 IP 주소를 알려줍니다
3. **DNS Resolution**: AP가 `CISCO-CAPWAP-CONTROLLER.localdomain`을 DNS에 질의합니다
4. **Local Subnet Broadcast**: 같은 서브넷에 있는 WLC를 브로드캐스트로 탐색합니다
5. **Over-the-Air Provisioning**: 이미 연결된 다른 AP로부터 WLC 정보를 받습니다

#### AP Join 과정

```
AP                                    WLC
 |                                     |
 |--- Discovery Request ------------->|
 |<-- Discovery Response -------------|  (사용 가능한 WLC 목록 응답)
 |                                     |
 |--- CAPWAP Join Request ----------->|
 |<-- CAPWAP Join Response -----------|  (인증서 기반 상호 인증)
 |                                     |
 |<-- Configuration Push -------------|  (SSID, 보안, 채널 설정 등)
 |                                     |
 |<-- Firmware Check/Update ----------|  (필요 시 펌웨어 업데이트)
 |                                     |
 |--- CAPWAP Tunnel Established ----->|  (제어+데이터 터널 수립)
 |                                     |
```

#### DHCP Option 43 설정 예시

```
! DHCP 서버에서 Option 43 설정 (Cisco IOS)
ip dhcp pool AP-POOL
 network 10.10.10.0 255.255.255.0
 default-router 10.10.10.1
 option 43 hex f104.0a01.0101    # WLC IP: 10.1.1.1
```

#### DNS 기반 탐색 설정

```
! DNS 서버에 A 레코드 추가
CISCO-CAPWAP-CONTROLLER.company.com  ->  10.1.1.1
```

실무에서는 **DHCP Option 43과 DNS 방식을 함께 설정**하여 이중화하는 것이 일반적입니다. AP가 여러 WLC를 발견한 경우, **가장 적은 AP를 관리하고 있는 WLC에 우선 Join**합니다 (기본 로드 밸런싱).

---

### 면접관: "WLC를 도입하면 어떤 구체적인 이점이 있는지 정리해서 말씀해 주세요."

WLC 도입의 핵심 이점을 분류하여 정리하겠습니다.

#### 1. 중앙 집중 관리

- 수백~수천 대의 AP를 **단일 인터페이스**에서 관리합니다
- SSID, 보안 정책, QoS 설정을 **일괄 적용**할 수 있습니다
- AP 펌웨어를 **일괄 업데이트**할 수 있습니다
- 설정 변경 시 개별 AP에 접속할 필요가 없습니다

#### 2. 자동 RF 관리 (RRM)

```
! WLC에서 RRM 설정 확인
show advanced 802.11a channel
show advanced 802.11b channel

! RRM은 자동으로 다음을 수행합니다:
# - DCA (Dynamic Channel Assignment): 최적 채널 자동 할당
# - TPC (Transmit Power Control): 전력 레벨 자동 조정
# - Coverage Hole Detection: 커버리지 부족 영역 탐지
```

#### 3. 보안 강화

- **Rogue AP 탐지**: 비인가 AP를 자동으로 감지하고 알림
- **WIPS (Wireless Intrusion Prevention System)**: 무선 침입 방지
- **802.1X 인증 통합 관리**: RADIUS 서버와의 연동을 중앙에서 관리
- **Guest Access 관리**: 방문자용 웹 포털 인증을 중앙에서 제공

#### 4. 로밍 최적화

- AP 간 클라이언트 이동 시 **빠른 로밍(Fast Roaming)**을 지원합니다
- 음성 통화(VoWiFi) 등 실시간 서비스의 끊김을 최소화합니다
- 클라이언트 세션 정보를 WLC에서 중앙 관리하여 재인증 시간을 줄입니다

#### 5. 가시성 및 모니터링

- 전체 무선 네트워크의 **실시간 상태를 한눈에 파악**할 수 있습니다
- 클라이언트별 연결 상태, 데이터 사용량, 신호 품질 등을 확인할 수 있습니다
- 문제 발생 시 **원인 분석이 용이**합니다

---

### 면접관: "Cisco WLC에서 자주 사용하는 기본 명령어와 설정 방법을 알고 계신다면 보여주세요."

네, Cisco WLC에서 자주 사용하는 명령어를 카테고리별로 정리하겠습니다.

#### AP 관리 명령어

```
! AP 전체 목록 확인
show ap summary

! 특정 AP 상세 정보 확인
show ap name AP-Office1 config general

! AP의 채널 및 파워 확인
show ap name AP-Office1 config dot11 5ghz

! AP에 태그/그룹 할당 (9800 WLC 기준)
ap name AP-Office1 site-tag SITE-OFFICE
ap name AP-Office1 policy-tag POLICY-CORP
```

#### WLAN 설정 명령어

```
! WLAN 목록 확인
show wlan summary

! 새 WLAN 생성 (CLI 기준)
wlan Corp-WiFi 1 Corp-WiFi
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security dot1x
 no shutdown

! Guest WLAN 생성
wlan Guest-WiFi 2 Guest-WiFi
 security web-auth
 no shutdown
```

#### 모니터링 명령어

```
! 연결된 클라이언트 확인
show client summary

! 특정 클라이언트 상세 정보
show client detail [MAC-ADDRESS]

! Rogue AP 목록 확인
show rogue ap summary

! RF 환경 상태 확인
show advanced 802.11a summary
```

WLC는 웹 GUI도 제공하므로, 일상적인 모니터링은 GUI를 활용하고, 자동화나 스크립팅이 필요한 경우 CLI를 사용하는 것이 실무에서 일반적인 방식입니다.

---

> **핵심 정리**: CCNA 레벨에서는 Autonomous AP와 Lightweight AP의 차이, CAPWAP 터널의 역할, Split-MAC 아키텍처, FlexConnect의 필요성, AP의 WLC 탐색 과정을 이해하는 것이 중요합니다.
