# 05-03. Port Security

> CCNA 200-301 보안 면접 시나리오 - 스위치 포트 보안과 비인가 장비 차단

---

### 면접관: "Port Security가 무엇이고, 어떤 상황에서 필요한지 설명해 주세요."

Port Security는 스위치 포트에 연결할 수 있는 장비를 MAC 주소 기반으로 제한하는 Layer 2 보안 기능입니다.

Port Security가 필요한 대표적인 상황은 다음과 같습니다.

1. **비인가 장비 접속 차단**: 회사 네트워크에 개인 노트북이나 허가되지 않은 장비가 연결되는 것을 방지합니다
2. **MAC Flooding 공격 방어**: 공격자가 대량의 가짜 MAC 주소를 보내 스위치의 MAC 주소 테이블을 가득 채우는 공격을 막습니다
3. **네트워크 무결성 유지**: 각 포트에 허용된 장비만 통신할 수 있도록 보장합니다

예를 들어, 사무실의 각 책상에 있는 네트워크 포트에 Port Security를 설정하면, 등록된 PC의 MAC 주소만 통신이 가능하고, 다른 장비를 연결하면 정책에 따라 포트가 차단됩니다.

Port Security는 스위치의 **액세스 포트**에만 적용할 수 있으며, 트렁크 포트나 동적으로 설정된 포트에는 적용할 수 없습니다.

---

### 면접관: "Port Security의 세 가지 Violation Mode에 대해 설명해 주세요."

허용되지 않은 MAC 주소가 감지되었을 때 스위치가 취하는 동작을 Violation Mode라고 합니다. 세 가지 모드가 있습니다.

| Violation Mode | 트래픽 차단 | 로그 생성 | SNMP 트랩 | 포트 상태 | 카운터 증가 |
|---------------|:---------:|:--------:|:---------:|----------|:---------:|
| **Shutdown** | O | O | O | err-disabled | O |
| **Restrict** | O | O | O | 유지 (up) | O |
| **Protect** | O | X | X | 유지 (up) | X |

각 모드의 특징을 자세히 설명드리겠습니다.

**Shutdown (기본값)**:
위반 발생 시 포트를 즉시 err-disabled 상태로 변경합니다. 해당 포트의 모든 트래픽이 중단됩니다. 가장 강력한 보안이지만, 관리자가 수동으로 복구하거나 자동 복구를 설정해야 합니다.

**Restrict**:
위반 MAC 주소의 트래픽만 차단하고, 허용된 MAC 주소의 트래픽은 계속 허용합니다. 로그와 SNMP 트랩이 생성되어 관리자가 인지할 수 있습니다.

**Protect**:
Restrict와 유사하게 위반 트래픽만 차단하지만, 로그나 알림이 생성되지 않습니다. 조용히 차단만 하므로 관리자가 위반 사실을 인지하기 어렵습니다.

실무에서는 보안 정책에 따라 **Shutdown** 또는 **Restrict**를 주로 사용합니다. Protect는 문제를 감지할 수 없어 권장되지 않습니다.

---

### 면접관: "Port Security를 설정하는 전체 과정을 보여주세요."

기본적인 Port Security 설정 과정을 단계별로 보여드리겠습니다.

#### 핵심 설정

```
! 1단계: 인터페이스를 액세스 모드로 설정 (필수)
Switch(config)# interface FastEthernet0/1
Switch(config-if)# switchport mode access

! 2단계: Port Security 활성화
Switch(config-if)# switchport port-security

! 3단계: 최대 허용 MAC 주소 수 설정 (기본값: 1)
Switch(config-if)# switchport port-security maximum 2

! 4단계: Violation Mode 설정 (기본값: shutdown)
Switch(config-if)# switchport port-security violation restrict

! 5단계: 허용할 MAC 주소 지정 방법 선택

! 방법 A - 수동으로 MAC 주소 지정
Switch(config-if)# switchport port-security mac-address 0011.2233.4455

! 방법 B - Sticky Learning 사용 (자동 학습 후 저장)
Switch(config-if)# switchport port-security mac-address sticky
```

설정 시 주의사항입니다.

- `switchport mode access`를 먼저 설정해야 합니다. 동적 모드(dynamic)에서는 Port Security가 동작하지 않습니다.
- maximum 값을 1로 설정하면 하나의 MAC 주소만 허용합니다. IP Phone과 PC가 함께 연결되는 환경이라면 maximum 2 이상으로 설정해야 합니다.
- Violation Mode를 지정하지 않으면 기본값인 shutdown이 적용됩니다.

---

### 면접관: "Sticky MAC Learning이 무엇이고, 수동 설정과 비교했을 때 어떤 장단점이 있나요?"

Sticky MAC Learning은 포트에 처음 연결된 장비의 MAC 주소를 **자동으로 학습**하여 running-config에 저장하는 기능입니다.

동작 방식은 이렇습니다.

1. 포트에 장비가 연결되면 MAC 주소를 자동으로 감지합니다
2. 감지된 MAC 주소를 `switchport port-security mac-address sticky xxxx.xxxx.xxxx` 형태로 running-config에 추가합니다
3. 이후 해당 MAC 주소만 허용되고, 다른 MAC 주소가 오면 위반으로 처리합니다

| 구분 | 수동 설정 | Sticky Learning |
|------|----------|-----------------|
| 설정 방법 | 관리자가 직접 MAC 입력 | 자동 학습 |
| 관리 부담 | 높음 (포트마다 입력) | 낮음 (자동 처리) |
| 정확성 | 높음 (의도한 MAC만) | 처음 연결된 장비가 학습됨 |
| 재부팅 후 | 유지 | `copy run start` 해야 유지 |
| 적합한 환경 | 소규모, 고보안 | 중대규모, 일반 사무환경 |

```
! Sticky 학습 결과 확인
Switch# show running-config interface Fa0/1

interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 0011.2233.4455
```

주의할 점은 Sticky로 학습된 MAC 주소는 running-config에만 저장되므로, `copy running-config startup-config`을 하지 않으면 스위치 재부팅 시 학습 내용이 사라집니다.

---

### 면접관: "err-disabled 상태가 되면 어떻게 복구하나요?"

Port Security의 Shutdown 모드에서 위반이 발생하면 포트가 err-disabled 상태가 됩니다. 복구 방법은 두 가지입니다.

#### 방법 1: 수동 복구

```
! 포트 상태 확인
Switch# show interfaces FastEthernet0/1 status
Port    Name   Status       Vlan  Duplex Speed Type
Fa0/1          err-disabled 1     auto   auto  10/100BaseTX

! 수동 복구: shutdown 후 no shutdown
Switch(config)# interface FastEthernet0/1
Switch(config-if)# shutdown
Switch(config-if)# no shutdown
```

반드시 `shutdown`을 먼저 입력한 후 `no shutdown`을 해야 합니다. `no shutdown`만 입력하면 복구되지 않습니다.

#### 방법 2: 자동 복구 (errdisable recovery)

```
! Port Security 위반에 대한 자동 복구 활성화
Switch(config)# errdisable recovery cause psecure-violation

! 복구 간격 설정 (초 단위, 기본값: 300초 = 5분)
Switch(config)# errdisable recovery interval 120

! 현재 errdisable recovery 설정 확인
Switch# show errdisable recovery
```

자동 복구를 설정하면 지정된 시간이 지난 후 포트가 자동으로 다시 활성화됩니다. 하지만 위반 원인(비인가 장비)이 제거되지 않았다면 다시 err-disabled 상태가 됩니다.

실무에서는 자동 복구를 설정하되, 반복적으로 err-disabled가 발생하는 포트는 원인을 조사하는 것이 중요합니다.

---

### 면접관: "Port Security 상태를 확인하는 명령어와 출력 내용을 설명해 주세요."

주요 확인 명령어와 출력 내용을 설명드리겠습니다.

#### 핵심 명령어

```
Switch# show port-security interface FastEthernet0/1

Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Restrict
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 2
Total MAC Addresses        : 1
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1
Last Source Address:Vlan   : 0011.2233.4455:1
Security Violation Count   : 3
```

각 항목의 의미입니다.

- **Port Status**: Secure-up(정상), Secure-down(물리적 다운), Secure-shutdown(위반으로 차단)
- **Maximum MAC Addresses**: 허용된 최대 MAC 주소 수
- **Total MAC Addresses**: 현재 학습된 MAC 주소 수
- **Security Violation Count**: 위반 발생 횟수

```
! 전체 포트의 Port Security 요약
Switch# show port-security

Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
-----------  -------------  -----------  -----------------  ---------------
      Fa0/1              2            1                  3         Restrict
      Fa0/2              1            1                  0         Shutdown

! 학습된 MAC 주소 목록 확인
Switch# show port-security address

Secure Mac Address Table
-------------------------------------------------------------------
Vlan  Mac Address       Type        Ports     Remaining Age (mins)
----  -----------       ----        -----     --------------------
   1  0011.2233.4455    SecureSticky Fa0/1     -
```

---

### 면접관: "IP Phone과 PC가 함께 연결되는 환경에서 Port Security를 어떻게 설정하시겠습니까?"

IP Phone 환경에서는 하나의 스위치 포트에 IP Phone과 PC가 데이지 체인으로 연결됩니다. PC가 IP Phone의 내장 스위치 포트에 연결되는 구조입니다.

이 경우 하나의 물리 포트에서 두 개 이상의 MAC 주소가 필요합니다. IP Phone의 MAC 주소와 PC의 MAC 주소가 모두 학습되어야 하기 때문입니다.

#### 핵심 설정

```
Switch(config)# interface FastEthernet0/1
! 액세스 VLAN 설정 (데이터용)
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10

! Voice VLAN 설정
Switch(config-if)# switchport voice vlan 20

! Port Security 설정
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 3
Switch(config-if)# switchport port-security violation restrict
Switch(config-if)# switchport port-security mac-address sticky
```

maximum을 3으로 설정한 이유는 다음과 같습니다.

- IP Phone MAC 주소: 1개
- PC MAC 주소: 1개
- 여유분: 1개 (IP Phone의 CDP/LLDP 트래픽 등)

실무에서는 환경에 따라 maximum 값을 조정합니다. Voice VLAN이 설정된 포트에서는 Shutdown 모드 대신 Restrict 모드를 사용하는 것이 좋습니다. Shutdown이 되면 IP Phone과 PC 모두 통신이 끊기므로 업무에 큰 영향을 줄 수 있기 때문입니다.

---

### 면접관: "Port Security의 한계점은 무엇이고, 이를 보완하는 기술에는 어떤 것이 있나요?"

Port Security의 주요 한계점은 다음과 같습니다.

1. **MAC 주소 위조 가능**: 공격자가 허용된 MAC 주소를 알아내서 자신의 장비 MAC을 변경하면 Port Security를 우회할 수 있습니다
2. **관리 부담**: 대규모 네트워크에서 포트별로 MAC 주소를 관리하는 것은 어렵습니다
3. **사용자 인증 불가**: 장비(MAC)만 인증하고 사용자를 인증하지 못합니다
4. **액세스 포트 전용**: 트렁크 포트에는 적용할 수 없습니다

이러한 한계를 보완하는 기술들입니다.

| 보완 기술 | 설명 | Port Security 한계 보완 |
|-----------|------|----------------------|
| 802.1X | 사용자 인증 기반 포트 접근 제어 | 사용자 인증 가능 |
| DHCP Snooping | DHCP 기반 공격 방어 | Layer 3 보안 보완 |
| DAI | ARP 스푸핑 방어 | MAC 위조 공격 보완 |
| MAB (MAC Auth Bypass) | 802.1X 불가 장비용 MAC 인증 | 프린터 등 레거시 장비 지원 |

실무에서는 Port Security 단독으로 사용하기보다 802.1X, DHCP Snooping, DAI 등과 함께 다층 보안(Defense in Depth) 전략으로 구현하는 것이 일반적입니다. Port Security는 가장 기본적인 Layer 2 보안으로, 다른 보안 기능의 기반이 됩니다.
