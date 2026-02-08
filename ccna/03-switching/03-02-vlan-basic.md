# 03-02. VLAN 기본 구성

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 - VLAN 개념과 구성

---

### 면접관: "회사에 스위치 한 대가 있고 여기에 영업부, 개발부, 인사부 PC가 모두 연결되어 있습니다. 부서별로 네트워크를 분리해 달라는 요청을 받았는데, 어떻게 하시겠습니까?"

네, 이 경우 **VLAN(Virtual Local Area Network)**을 사용하여 하나의 물리적 스위치를 논리적으로 분리하겠습니다.

VLAN이 필요한 이유를 먼저 설명드리면:

1. **브로드캐스트 도메인 분리**: VLAN 없이는 스위치에 연결된 모든 장비가 하나의 브로드캐스트 도메인에 속합니다. 영업부에서 보낸 브로드캐스트가 개발부, 인사부까지 전달되어 불필요한 트래픽이 발생합니다.

2. **보안 강화**: 인사부의 민감한 급여 정보가 다른 부서에서 접근 가능하면 안 됩니다. VLAN으로 분리하면 서로 다른 VLAN 간 통신은 라우터나 L3 스위치를 거쳐야 하므로 ACL 등으로 접근 제어가 가능합니다.

3. **관리 효율성**: 부서 이동 시 물리적 케이블 변경 없이 VLAN 설정만 변경하면 됩니다.

구성 계획은 다음과 같습니다:

| VLAN ID | VLAN 이름 | 서브넷 | 포트 할당 |
|---------|----------|--------|----------|
| VLAN 10 | SALES | 192.168.10.0/24 | Fa0/1 ~ Fa0/8 |
| VLAN 20 | DEVELOP | 192.168.20.0/24 | Fa0/9 ~ Fa0/16 |
| VLAN 30 | HR | 192.168.30.0/24 | Fa0/17 ~ Fa0/22 |
| VLAN 99 | MANAGEMENT | 10.10.99.0/24 | SVI만 사용 |

---

### 면접관: "그럼 실제로 VLAN을 생성하고 포트에 할당하는 과정을 CLI로 보여주세요."

네, VLAN 생성부터 포트 할당까지 단계별로 보여드리겠습니다.

#### 1단계: VLAN 생성

```
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name DEVELOP
SW1(config-vlan)# exit

SW1(config)# vlan 30
SW1(config-vlan)# name HR
SW1(config-vlan)# exit

SW1(config)# vlan 99
SW1(config-vlan)# name MANAGEMENT
SW1(config-vlan)# exit
```

#### 2단계: 포트에 VLAN 할당 (Access 포트 설정)

```
! 영업부 포트 설정 (Fa0/1 ~ Fa0/8)
SW1(config)# interface range FastEthernet 0/1 - 8
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 10
SW1(config-if-range)# exit

! 개발부 포트 설정 (Fa0/9 ~ Fa0/16)
SW1(config)# interface range FastEthernet 0/9 - 16
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 20
SW1(config-if-range)# exit

! 인사부 포트 설정 (Fa0/17 ~ Fa0/22)
SW1(config)# interface range FastEthernet 0/17 - 22
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 30
SW1(config-if-range)# exit
```

#### 3단계: 설정 확인

```
SW1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- ----------------------------
1    default                          active    Fa0/23, Fa0/24, Gi0/1, Gi0/2
10   SALES                            active    Fa0/1, Fa0/2, Fa0/3, Fa0/4
                                                Fa0/5, Fa0/6, Fa0/7, Fa0/8
20   DEVELOP                          active    Fa0/9, Fa0/10, Fa0/11, Fa0/12
                                                Fa0/13, Fa0/14, Fa0/15, Fa0/16
30   HR                               active    Fa0/17, Fa0/18, Fa0/19, Fa0/20
                                                Fa0/21, Fa0/22
99   MANAGEMENT                       active
```

`show vlan brief`로 각 VLAN에 어떤 포트가 할당되었는지 한눈에 확인할 수 있습니다. 여기서 trunk 포트는 이 출력에 표시되지 않는다는 점을 참고해야 합니다.

---

### 면접관: "VLAN 1에 대해 좀 더 설명해 주세요. 기본 VLAN이라고 하는데, 특별한 점이 있습니까?"

VLAN 1은 Cisco 스위치에서 **기본 VLAN(Default VLAN)**으로, 몇 가지 특별한 특성이 있습니다:

1. **모든 포트의 초기 VLAN**: 스위치의 모든 포트는 출하 시 VLAN 1에 속합니다
2. **삭제 불가**: `no vlan 1` 명령어로 삭제할 수 없습니다
3. **Native VLAN 기본값**: 802.1Q 트렁크에서 기본 Native VLAN이 VLAN 1입니다
4. **관리 트래픽**: CDP, VTP, DTP 같은 제어 프로토콜이 VLAN 1을 통해 전송됩니다

보안 관점에서 VLAN 1의 문제점은:

- 모든 포트가 기본으로 VLAN 1에 속하므로, 설정되지 않은 포트를 통해 VLAN 1 트래픽에 접근 가능합니다
- VLAN Hopping 공격의 대상이 될 수 있습니다

따라서 보안 모범 사례(Best Practice)로는:

```
! 사용하지 않는 포트를 별도 VLAN에 할당하고 shutdown
SW1(config)# vlan 999
SW1(config-vlan)# name BLACKHOLE
SW1(config-vlan)# exit

SW1(config)# interface range FastEthernet 0/23 - 24
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 999
SW1(config-if-range)# shutdown
SW1(config-if-range)# exit
```

사용하지 않는 포트를 VLAN 999 같은 "블랙홀 VLAN"에 넣고 shutdown 하는 것이 권장됩니다.

---

### 면접관: "같은 VLAN에 속한 PC끼리는 통신이 되고, 다른 VLAN의 PC와는 통신이 안 되는 이유를 브로드캐스트 도메인 관점에서 설명해 주세요."

VLAN의 핵심 개념이 바로 **브로드캐스트 도메인 분리**입니다.

스위치는 프레임을 처리할 때 VLAN 정보를 함께 관리합니다. 구체적으로 설명하면:

**같은 VLAN 내 통신 (VLAN 10 내부)**:
1. PC-A(Fa0/1, VLAN 10)가 PC-B(Fa0/3, VLAN 10)에게 통신하려 합니다
2. PC-A가 ARP Request(브로드캐스트)를 보냅니다
3. 스위치는 이 브로드캐스트를 **VLAN 10에 속한 포트에만** 전달합니다 (Fa0/1 ~ Fa0/8)
4. PC-B가 ARP Reply를 보내고, 이후 유니캐스트 통신이 가능합니다

**다른 VLAN 간 통신 (VLAN 10 -> VLAN 20)**:
1. PC-A(VLAN 10)가 VLAN 20의 PC에게 통신하려 합니다
2. ARP Request를 보내도 VLAN 10 포트에서만 전달되므로, VLAN 20의 장비는 이 브로드캐스트를 받을 수 없습니다
3. 따라서 직접 통신이 **불가능**합니다

다른 VLAN 간 통신이 필요하면 **Inter-VLAN Routing**이 필요합니다. 이는 다음 방법으로 구현합니다:

- **라우터 + 서브인터페이스** (Router-on-a-Stick)
- **L3 스위치의 SVI** (Switch Virtual Interface)

즉, VLAN은 물리적으로 하나의 스위치이지만 논리적으로 별도의 스위치처럼 동작하여 브로드캐스트 도메인을 분리하는 기술입니다.

---

### 면접관: "VLAN을 잘못 설정해서 장애가 난 경험이 있습니까? 아니면, VLAN 설정 시 흔히 발생하는 실수에 대해 말씀해 주세요."

VLAN 설정 시 자주 발생하는 실수와 트러블슈팅 포인트를 정리하겠습니다.

#### 실수 1: VLAN 생성 없이 포트에 할당

```
! VLAN 50을 생성하지 않고 포트에 할당하면
SW1(config)# interface Fa0/5
SW1(config-if)# switchport access vlan 50

! 자동으로 VLAN이 생성되기는 하지만, 이름이 없어서 관리가 어려움
! show vlan brief에서 VLAN0050으로 표시됨
```

#### 실수 2: 관리 VLAN 변경 후 접속 불가

원격으로 접속 중에 관리 VLAN IP를 변경하거나, 자신이 접속 중인 포트의 VLAN을 변경하면 접속이 끊깁니다. 반드시 콘솔 접속 상태에서 변경해야 합니다.

#### 실수 3: Trunk 포트에서 VLAN 확인

```
! show vlan brief에는 trunk 포트가 표시되지 않음
! trunk 포트의 VLAN 정보는 다른 명령어로 확인해야 함
SW1# show interfaces trunk

! 포트가 어떤 VLAN에 속해 있는지 확인
SW1# show interfaces FastEthernet 0/1 switchport
```

#### 실수 4: VLAN 데이터베이스 저장 위치 혼동

VLAN 정보는 **vlan.dat** 파일에 별도로 저장됩니다. `erase startup-config` 후 재부팅해도 VLAN 정보는 남아 있습니다.

```
! VLAN 정보까지 완전히 초기화하려면
SW1# delete flash:vlan.dat
SW1# erase startup-config
SW1# reload
```

이 점을 모르면 스위치를 초기화했는데 이전 VLAN 설정이 그대로 남아 있어서 혼란이 생길 수 있습니다.

---

### 면접관: "VLAN의 범위에 대해 아시는 대로 설명해 주세요. VLAN ID는 어디서 어디까지 사용 가능합니까?"

VLAN ID의 범위는 다음과 같이 구분됩니다:

| 범위 | VLAN ID | 설명 |
|------|---------|------|
| 예약 | 0 | 시스템 예약 (802.1Q 우선순위 태깅) |
| 기본 | 1 | Default VLAN, 삭제 불가 |
| Normal Range | 2 ~ 1001 | 일반적으로 사용하는 VLAN |
| Extended Range | 1002 ~ 4094 | 확장 VLAN (VTP Transparent 모드에서 사용) |
| 예약 | 1002 ~ 1005 | FDDI, Token Ring 용으로 예약 |
| 예약 | 4095 | 시스템 예약 |

CCNA 레벨에서는 **Normal Range (2 ~ 1001)**를 주로 사용합니다.

Normal Range VLAN은 **vlan.dat** 파일에 저장되고, Extended Range VLAN은 **running-config**에 저장됩니다 (VTP 모드가 Transparent일 때).

실무에서는 보통 VLAN 번호를 체계적으로 관리합니다. 예를 들어:

- VLAN 10~19: 1층 부서
- VLAN 20~29: 2층 부서
- VLAN 99: 관리용
- VLAN 100: 서버팜
- VLAN 200: VoIP
- VLAN 999: 미사용 포트 (블랙홀)

이렇게 체계적으로 번호를 부여하면 네트워크 규모가 커져도 관리가 수월합니다.

---

### 면접관: "VLAN 설정 후 정상적으로 동작하는지 확인하는 검증 절차를 말씀해 주세요."

VLAN 구성 후 검증은 다음 단계로 진행합니다:

#### 1단계: VLAN 생성 확인

```
SW1# show vlan brief
```

모든 계획된 VLAN이 active 상태인지, 올바른 포트가 할당되었는지 확인합니다.

#### 2단계: 개별 포트 설정 확인

```
SW1# show interfaces FastEthernet 0/1 switchport
Name: Fa0/1
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Administrative Trunking Encapsulation: negotiate
Operational Trunking Encapsulation: native
Negotiation of Trunking: Off
Access Mode VLAN: 10 (SALES)
Trunking Native Mode VLAN: 1 (default)
```

`Access Mode VLAN`이 의도한 VLAN과 일치하는지 확인합니다.

#### 3단계: 같은 VLAN 내 통신 테스트

```
! PC-A (VLAN 10, 192.168.10.10)에서
C:\> ping 192.168.10.20    ← 같은 VLAN의 PC-B
Reply from 192.168.10.20: bytes=32 time<1ms TTL=128  ← 성공해야 함

C:\> ping 192.168.20.10    ← 다른 VLAN의 PC
Request timed out.          ← 라우팅 없이는 실패해야 정상
```

#### 4단계: MAC 주소 테이블에서 VLAN별 확인

```
SW1# show mac address-table vlan 10
```

해당 VLAN에 연결된 장비의 MAC 주소가 올바르게 학습되고 있는지 확인합니다.

이 네 단계를 거치면 VLAN 구성이 정상적으로 동작하는지 충분히 검증할 수 있습니다.

---

> **핵심 정리**: VLAN은 하나의 물리적 스위치를 논리적으로 분리하여 브로드캐스트 도메인을 나누는 기술입니다. 보안, 성능, 관리 편의성을 위해 사용하며, VLAN 간 통신에는 라우팅이 필요합니다. VLAN 1의 보안 이슈와 vlan.dat 파일의 존재를 기억해야 합니다.
