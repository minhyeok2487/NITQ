# 05-04. DHCP Snooping 기본

> CCNA 200-301 보안 면접 시나리오 - DHCP 스푸핑 방어와 DHCP Snooping 구성

---

### 면접관: "DHCP Spoofing 공격이 무엇인지, 어떻게 동작하는지 설명해 주세요."

DHCP Spoofing 공격은 공격자가 네트워크에 **가짜 DHCP 서버**를 설치하여 클라이언트에게 잘못된 네트워크 정보를 배포하는 공격입니다.

공격 과정을 단계별로 설명드리겠습니다.

1. 클라이언트가 DHCP Discover 메시지를 브로드캐스트합니다
2. 정상 DHCP 서버와 공격자의 가짜 DHCP 서버가 모두 Offer를 보냅니다
3. 클라이언트는 먼저 도착한 Offer를 수락합니다 (보통 같은 네트워크에 있는 공격자의 응답이 빠름)
4. 가짜 DHCP 서버가 잘못된 정보를 제공합니다

공격자가 제공할 수 있는 잘못된 정보는 다음과 같습니다.

- **기본 게이트웨이**: 공격자의 IP를 게이트웨이로 설정하면, 모든 트래픽이 공격자를 경유합니다 (Man-in-the-Middle 공격)
- **DNS 서버**: 가짜 DNS 서버를 지정하면, 사용자를 피싱 사이트로 유도할 수 있습니다
- **잘못된 서브넷 마스크**: 네트워크 통신 장애를 유발합니다

```
정상 흐름:
[Client] → DHCP Discover → [정상 DHCP 서버]
[Client] ← DHCP Offer ← [정상 DHCP 서버]

공격 시:
[Client] → DHCP Discover → [정상 DHCP 서버]
                          → [가짜 DHCP 서버 (공격자)]
[Client] ← DHCP Offer ← [가짜 DHCP 서버] ← 더 빨리 응답!
```

이 공격은 DHCP가 인증 메커니즘 없이 동작하기 때문에 가능합니다. 클라이언트는 누가 보낸 Offer인지 검증하지 않습니다.

---

### 면접관: "이런 공격을 막기 위한 DHCP Snooping은 어떻게 동작하나요?"

DHCP Snooping은 스위치가 DHCP 메시지를 감시하여 **신뢰할 수 있는 포트와 신뢰할 수 없는 포트를 구분**하는 Layer 2 보안 기능입니다.

핵심 개념은 **Trusted Port(신뢰 포트)**와 **Untrusted Port(비신뢰 포트)**의 구분입니다.

| 구분 | Trusted Port | Untrusted Port |
|------|-------------|----------------|
| 연결 대상 | DHCP 서버, 상위 스위치/라우터 | 일반 클라이언트, 사용자 PC |
| DHCP 서버 메시지 | 허용 (Offer, Ack) | 차단 (Offer, Ack) |
| DHCP 클라이언트 메시지 | 허용 (Discover, Request) | 허용 (Discover, Request) |
| 기본 설정 | 수동 지정 필요 | 기본값 (모든 포트) |

동작 방식을 정리하면 다음과 같습니다.

1. DHCP Snooping이 활성화되면 모든 포트는 기본적으로 **Untrusted**입니다
2. 정상 DHCP 서버가 연결된 포트만 관리자가 **Trusted**로 지정합니다
3. Untrusted 포트에서 DHCP 서버 응답 메시지(Offer, Ack)가 오면 스위치가 차단합니다
4. Trusted 포트에서만 DHCP 서버 메시지가 통과할 수 있습니다
5. 정상적인 DHCP 과정에서 학습된 정보는 **DHCP Snooping Binding Table**에 기록됩니다

이렇게 하면 공격자가 가짜 DHCP 서버를 연결해도 Untrusted 포트에서 서버 응답이 차단되므로 공격이 실패합니다.

---

### 면접관: "DHCP Snooping Binding Table이 무엇이고, 왜 중요한가요?"

DHCP Snooping Binding Table은 정상적인 DHCP 과정을 통해 IP를 할당받은 장비의 정보를 기록하는 데이터베이스입니다.

기록되는 정보는 다음과 같습니다.

- **MAC 주소**: 클라이언트의 MAC 주소
- **IP 주소**: DHCP로 할당받은 IP 주소
- **VLAN**: 해당 포트가 속한 VLAN
- **포트**: 클라이언트가 연결된 스위치 포트
- **Lease Time**: IP 임대 시간

```
Switch# show ip dhcp snooping binding

MacAddress          IpAddress       Lease(sec)  Type          VLAN  Interface
------------------  --------------- ----------  ------------- ----  ----------
00:11:22:33:44:55   10.10.10.100    86400       dhcp-snooping 10    Fa0/1
00:AA:BB:CC:DD:EE   10.10.10.101    86400       dhcp-snooping 10    Fa0/2
```

이 테이블이 중요한 이유는 **다른 보안 기능의 기반**이 되기 때문입니다.

1. **DAI (Dynamic ARP Inspection)**: Binding Table의 MAC-IP 매핑 정보를 사용하여 ARP 스푸핑 공격을 탐지합니다
2. **IP Source Guard**: Binding Table을 기반으로 IP 주소 위조를 차단합니다

즉, DHCP Snooping은 단독으로도 DHCP 공격을 방어하지만, Binding Table을 통해 DAI와 IP Source Guard의 동작 기반을 제공하는 핵심 기능입니다.

---

### 면접관: "DHCP Snooping을 설정하는 과정을 보여주세요."

DHCP Snooping 설정을 단계별로 보여드리겠습니다.

#### 핵심 설정

```
! 1단계: DHCP Snooping 전역 활성화
Switch(config)# ip dhcp snooping

! 2단계: 특정 VLAN에서 DHCP Snooping 활성화
Switch(config)# ip dhcp snooping vlan 10,20

! 3단계: Trusted 포트 지정 (DHCP 서버가 연결된 포트)
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# exit

! 4단계: Untrusted 포트에 Rate Limiting 설정 (선택사항이지만 권장)
Switch(config)# interface range FastEthernet0/1 - 24
Switch(config-if-range)# ip dhcp snooping limit rate 15
Switch(config-if-range)# exit

! 5단계: Option 82 삽입 비활성화 (필요시)
Switch(config)# no ip dhcp snooping information option

! 설정 확인
Switch# show ip dhcp snooping
```

각 단계의 설명입니다.

- **VLAN 지정**: DHCP Snooping은 VLAN 단위로 활성화됩니다. 필요한 VLAN만 지정합니다.
- **Trusted 포트**: 정상 DHCP 서버 또는 상위 네트워크 장비(업링크)가 연결된 포트를 Trusted로 지정합니다.
- **Rate Limiting**: Untrusted 포트에서 초당 수신할 수 있는 DHCP 패킷 수를 제한합니다. DHCP 고갈 공격(Starvation Attack)을 방어하는 데 효과적입니다.
- **Option 82**: DHCP Relay 환경에서 사용하는 옵션입니다. 단일 서브넷 환경에서는 비활성화하는 것이 일반적입니다. 활성 상태에서 DHCP 서버가 Option 82를 지원하지 않으면 DHCP가 동작하지 않을 수 있습니다.

---

### 면접관: "DHCP Starvation 공격도 있다고 들었는데, DHCP Snooping으로 이것도 막을 수 있나요?"

DHCP Starvation 공격은 공격자가 대량의 DHCP Discover 메시지를 보내 DHCP 서버의 IP 주소 풀을 모두 소진시키는 공격입니다. 정상 사용자가 IP를 할당받지 못하게 하는 서비스 거부(DoS) 공격의 일종입니다.

DHCP Snooping만으로는 이 공격을 완전히 차단하기 어렵지만, 두 가지 방법으로 완화할 수 있습니다.

**1. Rate Limiting**

```
Switch(config)# interface FastEthernet0/1
Switch(config-if)# ip dhcp snooping limit rate 10
```

초당 DHCP 패킷 수를 제한하여 대량의 Discover 메시지를 차단합니다. Rate Limit을 초과하면 해당 포트가 err-disabled 상태가 됩니다.

**2. Port Security와 결합**

```
Switch(config)# interface FastEthernet0/1
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 1
```

Port Security로 포트당 허용 MAC 주소를 제한하면, 공격자가 다양한 MAC 주소로 DHCP Discover를 보내는 것을 차단할 수 있습니다.

| 공격 유형 | DHCP Snooping 대응 | 추가 필요 기능 |
|-----------|-------------------|---------------|
| DHCP Spoofing (가짜 서버) | Trusted/Untrusted로 완전 차단 | - |
| DHCP Starvation (풀 고갈) | Rate Limiting으로 완화 | Port Security 병행 |

따라서 실무에서는 DHCP Snooping과 Port Security를 함께 설정하는 것이 가장 효과적입니다.

---

### 면접관: "DAI(Dynamic ARP Inspection)는 무엇이고, DHCP Snooping과 어떤 관계가 있나요?"

DAI는 ARP 스푸핑 공격을 방어하는 보안 기능입니다. ARP 스푸핑은 공격자가 가짜 ARP 응답을 보내 다른 장비의 ARP 테이블을 오염시키는 공격입니다.

DAI는 DHCP Snooping Binding Table을 참조하여 ARP 패킷의 정당성을 검증합니다.

```
검증 과정:
1. ARP 패킷이 Untrusted 포트로 들어옴
2. DAI가 ARP 패킷의 출발지 MAC과 IP를 추출
3. DHCP Snooping Binding Table과 비교
4. 일치하면 허용, 불일치하면 차단
```

#### 핵심 설정

```
! DHCP Snooping이 먼저 설정되어 있어야 함

! DAI 활성화 (VLAN 단위)
Switch(config)# ip arp inspection vlan 10,20

! Trusted 포트 지정 (보통 DHCP Snooping의 Trusted 포트와 동일)
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# ip arp inspection trust

! DAI Rate Limiting (선택사항)
Switch(config)# interface range FastEthernet0/1 - 24
Switch(config-if-range)# ip arp inspection limit rate 15

! 확인 명령어
Switch# show ip arp inspection
Switch# show ip arp inspection vlan 10
```

DHCP Snooping과 DAI의 관계를 정리하면 다음과 같습니다.

| 기능 | 방어 대상 | 의존성 |
|------|----------|--------|
| DHCP Snooping | DHCP Spoofing | 독립적으로 동작 |
| DAI | ARP Spoofing | DHCP Snooping Binding Table 필요 |
| IP Source Guard | IP Spoofing | DHCP Snooping Binding Table 필요 |

DAI는 DHCP Snooping 없이는 제대로 동작할 수 없습니다. Binding Table이 없으면 ARP 패킷을 검증할 기준이 없기 때문입니다. 따라서 DAI를 구현하려면 반드시 DHCP Snooping을 먼저 설정해야 합니다.

---

### 면접관: "DHCP Snooping 트러블슈팅 시 확인해야 할 사항은 무엇인가요?"

DHCP Snooping 관련 문제가 발생했을 때 체계적으로 점검해야 할 항목을 정리하겠습니다.

#### 핵심 명령어

```
! DHCP Snooping 전체 상태 확인
Switch# show ip dhcp snooping

! Binding Table 확인
Switch# show ip dhcp snooping binding

! 통계 확인 (차단된 패킷 수 등)
Switch# show ip dhcp snooping statistics

! 특정 인터페이스의 Trust 상태 확인
Switch# show ip dhcp snooping | include Interface
```

자주 발생하는 문제와 해결 방법입니다.

**1. DHCP가 아예 동작하지 않는 경우**
- Trusted 포트 설정을 확인합니다. DHCP 서버 방향의 포트가 Trusted인지 점검합니다.
- 업링크 포트도 Trusted로 설정해야 합니다. DHCP 서버가 다른 스위치에 있다면 중간 경로의 모든 업링크가 Trusted여야 합니다.

**2. Option 82 관련 문제**
- `no ip dhcp snooping information option`으로 Option 82를 비활성화하면 해결되는 경우가 많습니다.

**3. Rate Limit 초과로 포트 차단**
- 정상 환경에서 Rate Limit이 너무 낮게 설정된 경우입니다. 값을 적절히 높이고, errdisable recovery를 설정합니다.

```
Switch(config)# errdisable recovery cause dhcp-rate-limit
Switch(config)# errdisable recovery interval 120
```

**4. Binding Table에 항목이 없는 경우**
- DHCP Snooping 활성화 전에 이미 IP를 받은 장비는 Binding Table에 없습니다. 해당 장비가 IP를 갱신해야 테이블에 추가됩니다. 수동으로 항목을 추가할 수도 있습니다.

정리하면, DHCP Snooping은 Layer 2 보안의 핵심 기둥입니다. 단독으로도 DHCP 공격을 방어하며, DAI와 IP Source Guard의 기반이 되어 ARP 스푸핑과 IP 위조 공격까지 방어 범위를 확장할 수 있습니다.
