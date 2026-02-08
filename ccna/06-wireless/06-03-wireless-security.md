# 06-03. 무선 보안 (WPA2/WPA3)

> CCNA 200-301 무선 보안 면접 시나리오

---

### 면접관: "우리 회사의 무선 네트워크 보안을 강화하려고 합니다. 무선 보안 프로토콜이 WEP에서 WPA3까지 어떻게 발전해 왔는지 설명해 주세요."

네, 무선 보안 프로토콜의 발전 과정을 시대순으로 설명드리겠습니다.

**1. WEP (Wired Equivalent Privacy) - 1999년:**
- 최초의 무선 보안 표준이었습니다
- RC4 암호화 알고리즘을 사용했습니다
- **심각한 보안 취약점**이 발견되어 현재는 사용이 금지됩니다
- 24비트 IV(Initialization Vector)가 너무 짧아 **몇 분 내에 키가 크래킹**될 수 있습니다
- 정적 키를 사용하며 키 관리가 어렵습니다

**2. WPA (Wi-Fi Protected Access) - 2003년:**
- WEP의 긴급 대체로 도입되었습니다
- **TKIP(Temporal Key Integrity Protocol)**을 사용하여 RC4의 취약점을 보완했습니다
- 패킷마다 다른 키를 생성하여 보안을 향상시켰습니다
- 그러나 근본적으로 RC4 기반이라 여전히 취약점이 존재합니다

**3. WPA2 - 2004년:**
- 현재 가장 널리 사용되는 표준입니다
- **AES-CCMP** 암호화를 사용합니다 (RC4 완전 대체)
- 802.11i 표준을 완전히 구현했습니다
- 2018년 KRACK 공격이 발견되었지만, 패치로 대응 가능합니다

**4. WPA3 - 2018년:**
- 최신 보안 표준입니다
- **SAE(Simultaneous Authentication of Equals)**를 도입했습니다
- 오프라인 사전 공격에 대한 방어력이 향상되었습니다
- 개별 데이터 암호화(Forward Secrecy)를 제공합니다

| 표준 | 암호화 | 인증 | 보안 수준 | 현재 상태 |
|------|--------|------|----------|----------|
| WEP | RC4 (40/104비트) | Open/Shared | 매우 취약 | 사용 금지 |
| WPA | TKIP (RC4 기반) | PSK/Enterprise | 취약 | 비권장 |
| WPA2 | AES-CCMP | PSK/Enterprise | 양호 | 현재 주력 |
| WPA3 | AES-GCMP (256비트) | SAE/Enterprise | 우수 | 점진적 도입 중 |

현재 **WPA2를 최소 기준**으로 사용하고, 신규 장비에서는 **WPA3로 전환**하는 것을 권장합니다.

---

### 면접관: "WPA2에서 PSK 모드와 Enterprise 모드의 차이점을 설명해 주시고, 각각 어떤 환경에 적합한지 말씀해 주세요."

WPA2는 인증 방식에 따라 **두 가지 모드**로 나뉩니다.

#### WPA2-PSK (Personal)

- **PSK(Pre-Shared Key)**, 즉 사전 공유 키를 사용합니다
- 모든 사용자가 **동일한 비밀번호**를 사용합니다
- 설정이 간단하고 별도의 인증 서버가 필요 없습니다
- 4-Way Handshake로 세션 키를 생성합니다

```
! WLC에서 WPA2-PSK 설정 예시
wlan Home-WiFi 1 Home-WiFi
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security wpa psk set-key ascii MySecretP@ss123
 no shutdown
```

**PSK의 한계:**
- 직원이 퇴사해도 **비밀번호를 변경하지 않으면 여전히 접속 가능**합니다
- 비밀번호 변경 시 **모든 사용자에게 새 비밀번호를 알려야** 합니다
- **개별 사용자를 구분할 수 없습니다** (모두 같은 키 사용)

#### WPA2-Enterprise (802.1X)

- **RADIUS 서버**를 통한 개별 사용자 인증을 수행합니다
- 사용자마다 **고유한 자격 증명**(ID/비밀번호 또는 인증서)을 사용합니다
- **EAP(Extensible Authentication Protocol)**를 사용합니다

```
! WLC에서 WPA2-Enterprise 설정 예시
wlan Corp-WiFi 2 Corp-WiFi
 security wpa wpa2
 security wpa wpa2 ciphers aes
 security dot1x
 aaa-override
 no shutdown

! RADIUS 서버 연동 설정
radius server RADIUS-SRV
 address ipv4 10.1.1.100 auth-port 1812 acct-port 1813
 key Radius@Secret
```

**인증 과정:**
```
[클라이언트] <--802.11--> [AP] <--CAPWAP--> [WLC] <--RADIUS--> [AAA 서버]
  (Supplicant)        (Authenticator)              (Auth Server)

1. 클라이언트가 연결 요청
2. WLC가 EAP 인증 시작
3. 클라이언트 자격 증명을 RADIUS 서버로 전달
4. RADIUS 서버가 인증 성공/실패 응답
5. 인증 성공 시 동적 키 생성 및 연결 완료
```

| 구분 | WPA2-PSK | WPA2-Enterprise |
|------|----------|-----------------|
| 인증 방식 | 공유 비밀번호 | 개별 사용자 인증 |
| 인증 서버 | 불필요 | RADIUS 서버 필요 |
| 사용자 구분 | 불가 | 가능 |
| 설정 난이도 | 쉬움 | 복잡 |
| 적합 환경 | 가정, 소규모 사무실 | 기업 환경 |
| 보안 수준 | 중간 | 높음 |

기업 환경에서는 **WPA2-Enterprise**를 사용하여 사용자별 접근 제어와 감사(Audit)가 가능하도록 구성하는 것이 권장됩니다.

---

### 면접관: "TKIP와 AES-CCMP의 차이를 기술적으로 좀 더 자세히 설명해 주시겠어요?"

네, 두 암호화 방식의 기술적 차이를 설명드리겠습니다.

**TKIP (Temporal Key Integrity Protocol):**
- WEP의 RC4 암호화를 **개선한 래퍼(wrapper)** 방식입니다
- WEP 하드웨어에서도 소프트웨어 업그레이드로 사용 가능했습니다 (과도기 솔루션)
- 패킷마다 다른 **임시 키를 생성**합니다 (Per-Packet Key Mixing)
- **MIC(Message Integrity Check)**를 추가하여 패킷 변조를 감지합니다
- 48비트 IV를 사용하여 WEP의 24비트 IV 문제를 해결했습니다
- 하지만 근본적으로 RC4 기반이므로 **현재는 비권장**입니다

**AES-CCMP (Counter Mode with CBC-MAC Protocol):**
- **AES(Advanced Encryption Standard)** 블록 암호를 사용합니다
- 128비트 키 길이로 강력한 암호화를 제공합니다
- **CCM** 모드는 두 가지 기능을 결합합니다:
  - **Counter Mode**: 데이터 암호화 (기밀성)
  - **CBC-MAC**: 메시지 인증 코드 (무결성)
- WPA2의 **필수 암호화 방식**입니다
- 하드웨어 가속이 가능하여 성능이 우수합니다

| 구분 | TKIP | AES-CCMP |
|------|------|----------|
| 기반 알고리즘 | RC4 (스트림 암호) | AES (블록 암호) |
| 키 길이 | 128비트 | 128비트 |
| 암호화 방식 | 스트림 암호화 | 블록 암호화 (Counter Mode) |
| 무결성 검증 | MIC (Michael) | CBC-MAC |
| 보안 수준 | 낮음 (취약점 존재) | 높음 |
| 성능 | 소프트웨어 처리 | 하드웨어 가속 가능 |
| WPA 버전 | WPA | WPA2/WPA3 |
| 현재 권장 여부 | 비권장 | 권장 |

WPA3에서는 AES-CCMP를 더 강화한 **AES-GCMP(Galois/Counter Mode Protocol)**를 사용하며, 256비트 키를 지원합니다.

---

### 면접관: "WPA3에서 도입된 SAE가 기존 PSK 방식보다 어떻게 보안이 강화되었는지 설명해 주세요."

WPA3-SAE(Simultaneous Authentication of Equals)는 기존 WPA2-PSK의 핵심 취약점을 해결합니다.

**WPA2-PSK의 취약점:**
- 4-Way Handshake 패킷을 **캡처**할 수 있습니다
- 캡처한 패킷에 대해 **오프라인 사전 공격(Dictionary Attack)**이 가능합니다
- 강력한 GPU를 사용하면 짧은 비밀번호는 **무차별 대입(Brute Force)**으로 크래킹될 수 있습니다
- 비밀번호가 노출되면 이전에 캡처한 **과거 트래픽도 복호화** 가능합니다

**WPA3-SAE의 개선사항:**

**1. Dragonfly Handshake:**
- SAE는 **Dragonfly Key Exchange** 프로토콜을 사용합니다
- 비밀번호를 직접 전송하지 않고, **수학적 증명(Zero-Knowledge Proof)**을 통해 인증합니다
- 핸드셰이크 패킷을 캡처하더라도 **오프라인 사전 공격이 불가능**합니다

**2. Forward Secrecy (전방향 비밀성):**
- 각 세션마다 **고유한 세션 키**를 생성합니다
- 비밀번호가 나중에 노출되더라도 **이전에 캡처한 트래픽을 복호화할 수 없습니다**
- WPA2-PSK에서는 이 기능이 없었습니다

**3. 오프라인 공격 방지:**
- 인증 시도는 반드시 **AP와의 실시간 상호작용**이 필요합니다
- 공격자가 트래픽을 캡처해도 **오프라인에서 비밀번호를 추측할 수 없습니다**
- AP에서 인증 시도 횟수를 제한하여 **온라인 무차별 대입 공격**도 방어합니다

```
WPA2-PSK 취약점:
캡처한 4-Way Handshake + 사전(Dictionary) → 오프라인 크래킹 가능

WPA3-SAE 방어:
Dragonfly Handshake 캡처 → 오프라인 크래킹 불가능
                           (실시간 AP 상호작용 필수)
```

WPA3는 또한 **WPA3-Enterprise 192-bit Mode**를 제공하여, 정부기관이나 금융기관 등 높은 보안 수준이 필요한 환경에서 192비트 수준의 암호화를 사용할 수 있습니다.

---

### 면접관: "보안 담당자가 'SSID를 숨기고 MAC 필터링을 적용하면 무선 보안이 충분하다'고 주장하는데, 이에 대한 의견을 말씀해 주세요."

솔직히 말씀드리면, SSID 숨기기와 MAC 필터링은 **보안 대책으로는 매우 부족합니다**. 그 이유를 설명드리겠습니다.

#### SSID 숨기기 (SSID Broadcasting 비활성화)의 한계

- AP가 Beacon 프레임에 SSID를 포함하지 않을 뿐입니다
- 클라이언트가 연결할 때 보내는 **Probe Request에 SSID가 평문으로 포함**됩니다
- **Wireshark, Kismet** 등의 도구로 쉽게 SSID를 확인할 수 있습니다
- 오히려 클라이언트가 숨겨진 SSID를 찾기 위해 **Probe Request를 계속 브로드캐스트**하므로, 이동 중에 해당 SSID 정보가 더 많이 노출될 수 있습니다
- **"Security through obscurity(모호함에 의한 보안)"** 방식으로, 실제 보안 효과가 거의 없습니다

#### MAC 필터링의 한계

- MAC 주소는 무선 프레임 헤더에 **암호화되지 않은 상태로 전송**됩니다
- 공격자가 허용된 MAC 주소를 **스니핑**할 수 있습니다
- MAC 주소는 소프트웨어적으로 쉽게 **변조(Spoofing)**할 수 있습니다

```
! MAC 주소 변조 예시 (공격자 관점)
# Linux에서 MAC 주소 변경
ifconfig wlan0 down
ifconfig wlan0 hw ether AA:BB:CC:DD:EE:FF
ifconfig wlan0 up

# 위와 같이 간단하게 MAC 주소를 변조할 수 있습니다
```

- AP가 많아지면 **MAC 주소 목록 관리가 비현실적**입니다
- 새 장비 등록 시마다 관리자가 수동으로 추가해야 합니다

#### 올바른 보안 계층 구조

SSID 숨기기와 MAC 필터링을 완전히 무시하라는 것은 아니지만, 이것만으로는 부족합니다. **진정한 무선 보안**은 다음과 같이 구성해야 합니다:

| 우선순위 | 보안 대책 | 효과 |
|---------|----------|------|
| 필수 | WPA2/WPA3 암호화 | 데이터 기밀성과 무결성 보호 |
| 필수 | 802.1X 인증 (Enterprise) | 개별 사용자 인증 및 접근 제어 |
| 권장 | 강력한 비밀번호 정책 | 사전 공격 방어 |
| 보조 | SSID 숨기기 | 일반 사용자 노출 감소 (보조적) |
| 보조 | MAC 필터링 | 추가적인 접근 제어 계층 (보조적) |
| 권장 | Rogue AP 탐지 | 비인가 AP 차단 |
| 권장 | 무선 IPS | 공격 탐지 및 방어 |

**핵심은 WPA2/WPA3 암호화와 802.1X 인증이며**, SSID 숨기기와 MAC 필터링은 보조적인 수단일 뿐입니다.

---

### 면접관: "기업 환경에서 무선 보안을 구축할 때의 베스트 프랙티스를 정리해서 말씀해 주세요."

기업 무선 보안 베스트 프랙티스를 카테고리별로 정리하겠습니다.

#### 1. 암호화 및 인증

- **최소 WPA2-Enterprise(802.1X)**를 사용합니다
- 가능하면 **WPA3로 전환**을 계획합니다
- TKIP는 절대 사용하지 않고, **AES-CCMP만 허용**합니다
- EAP 방식은 **EAP-TLS(인증서 기반)**가 가장 안전합니다
- PSK를 사용해야 하는 경우, **최소 20자 이상의 복잡한 비밀번호**를 설정합니다

#### 2. 네트워크 분리

```
! SSID별 VLAN 분리 예시
SSID: Corp-WiFi    → VLAN 10 (업무용, 802.1X 인증)
SSID: Guest-WiFi   → VLAN 99 (방문자용, 웹 포털 인증)
SSID: IoT-WiFi     → VLAN 50 (IoT 장비, PSK + ACL 제한)
```

- **업무용, 게스트용, IoT용 SSID를 분리**합니다
- 각 SSID에 서로 다른 VLAN을 할당합니다
- 게스트 네트워크는 **인터넷만 접근 가능**하도록 ACL로 제한합니다
- IoT VLAN은 필요한 서비스만 접근 가능하도록 **최소 권한 원칙**을 적용합니다

#### 3. AP 및 WLC 보안

- WLC 관리 인터페이스에 **강력한 인증**을 적용합니다
- 불필요한 관리 프로토콜(Telnet, HTTP)을 비활성화하고 **SSH, HTTPS만 허용**합니다
- AP와 WLC의 **펌웨어를 최신 버전으로 유지**합니다
- 관리 VLAN을 별도로 분리합니다

#### 4. 모니터링 및 감지

- **Rogue AP 탐지** 기능을 활성화합니다
- **WIPS(Wireless Intrusion Prevention System)**를 구성합니다
- 무선 클라이언트 연결 로그를 **중앙 집중 수집(Syslog/SIEM)**합니다
- 비정상적인 패턴(대량 인증 실패 등)에 대한 알림을 설정합니다

#### 5. 물리적 보안

- AP를 **물리적으로 접근이 어려운 위치(천장 등)**에 설치합니다
- AP의 콘솔 포트 접근을 제한합니다
- 건물 외부로의 무선 신호 유출을 최소화하도록 **AP 전력을 조정**합니다

```
! WLC에서 보안 관련 확인 명령어
show wlan summary                    # WLAN 보안 설정 확인
show rogue ap summary                # Rogue AP 목록 확인
show client summary                  # 연결 클라이언트 확인
show radius summary                  # RADIUS 서버 상태 확인
show wlan [WLAN-ID] | include Security  # 특정 WLAN 보안 설정
```

---

### 면접관: "마지막으로, WEP를 아직 사용하고 있는 레거시 장비가 있다면 어떻게 대응하시겠습니까?"

WEP를 사용하는 레거시 장비에 대한 대응 전략을 말씀드리겠습니다.

**원칙: WEP는 어떤 상황에서도 기업 네트워크에서 허용되어서는 안 됩니다.**

그러나 현실적으로 WEP만 지원하는 레거시 장비(구형 의료기기, 산업용 장비 등)가 있을 수 있습니다. 이런 경우의 대응 방법은 다음과 같습니다:

**1. 장비 교체 (최우선):**
- WPA2 이상을 지원하는 장비로 교체하는 것이 가장 좋습니다
- 교체 일정과 예산을 수립합니다

**2. 네트워크 격리 (교체 전 임시 조치):**
```
! WEP 전용 SSID를 별도 VLAN에 격리
SSID: Legacy-Device → VLAN 200 (격리된 VLAN)

! VLAN 200에 엄격한 ACL 적용
access-list 100 permit ip 10.200.0.0 0.0.0.255 host 10.1.1.50  # 특정 서버만 허용
access-list 100 deny ip 10.200.0.0 0.0.0.255 any               # 나머지 차단

! 해당 VLAN에 방화벽 규칙 적용
# - 필요한 최소한의 통신만 허용
# - 인터넷 접근 차단
# - 다른 내부 네트워크 접근 차단
```

**3. 추가 보안 조치:**
- 해당 SSID의 **전력을 최소화**하여 커버리지 범위를 줄입니다
- **MAC 필터링을 적용**합니다 (WEP 환경에서는 추가 보안 계층으로 의미 있음)
- 해당 네트워크 트래픽을 **지속적으로 모니터링**합니다
- 레거시 장비에서 발생하는 트래픽을 **IDS/IPS로 검사**합니다

핵심은 WEP 장비를 **가능한 빨리 교체**하되, 교체 전까지는 **철저한 네트워크 격리**를 통해 위험을 최소화하는 것입니다.

---

> **핵심 정리**: 무선 보안의 핵심은 WPA2/WPA3 암호화, 802.1X 인증, 네트워크 분리이며, SSID 숨기기와 MAC 필터링은 보조적 수단일 뿐입니다. WEP와 TKIP는 사용하지 않아야 하며, 기업 환경에서는 반드시 Enterprise 모드를 사용해야 합니다.
