# 02-06. Inter-VLAN Routing (Router-on-a-Stick)

> CCNA 200-301 라우팅 기초 면접 시나리오
> 주제: Inter-VLAN Routing 필요성, Router-on-a-Stick 구성, L3 스위치 SVI, 트렁크, 검증 및 트러블슈팅

---

### 면접관: "VLAN 간 통신이 왜 기본적으로 불가능한지 설명하고, Inter-VLAN Routing이 필요한 이유를 말씀해주세요."

VLAN은 스위치에서 논리적으로 브로드캐스트 도메인을 분리하는 기술입니다. 서로 다른 VLAN에 속한 호스트는 마치 물리적으로 다른 네트워크에 있는 것처럼 분리됩니다.

**VLAN 간 통신이 불가능한 이유:**

1. **브로드캐스트 도메인 분리**: VLAN 10의 호스트가 보내는 브로드캐스트(ARP 요청 포함)는 VLAN 10 내에서만 전달됩니다. VLAN 20의 호스트에게 도달하지 않습니다.

2. **다른 서브넷 사용**: 일반적으로 각 VLAN은 서로 다른 IP 서브넷을 사용합니다. 같은 서브넷이 아닌 목적지로 통신하려면 반드시 라우터(L3 장비)를 거쳐야 합니다.

3. **스위치는 L2 장비**: 일반 L2 스위치는 MAC 주소 기반으로 프레임을 전달하며, IP 주소를 기반으로 한 라우팅을 수행하지 못합니다.

**Inter-VLAN Routing이 필요한 실무 예시:**

```
VLAN 10 (영업팀): 192.168.10.0/24
VLAN 20 (개발팀): 192.168.20.0/24
VLAN 30 (관리팀): 192.168.30.0/24
```

영업팀 직원이 개발팀의 파일 서버에 접근하거나, 관리팀이 모든 부서의 프린터를 사용하려면 VLAN 간 라우팅이 반드시 필요합니다.

Inter-VLAN Routing을 구현하는 방법은 크게 세 가지입니다:

| 방법 | 설명 | 장단점 |
|------|------|--------|
| 라우터 물리 인터페이스 | VLAN별로 라우터 물리 포트 사용 | 포트 낭비, 비효율적 |
| Router-on-a-Stick | 하나의 트렁크로 서브인터페이스 사용 | CCNA 핵심, 소규모 적합 |
| L3 스위치 SVI | 스위치에서 직접 라우팅 | 대규모 환경 권장, 고성능 |

---

### 면접관: "Router-on-a-Stick 구성을 처음부터 끝까지 설정해주세요. 스위치 쪽과 라우터 쪽 모두 보여주시기 바랍니다."

다음 시나리오로 설정하겠습니다:

```
          Trunk (Gi0/0)
[Router R1] ============== [Switch SW1]
                              |     |     |
                           Fa0/1  Fa0/2  Fa0/3
                           VLAN10 VLAN20 VLAN30
                            PC-A   PC-B   PC-C
```

**1단계: 스위치 VLAN 설정**

```
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name DEVELOPMENT
SW1(config-vlan)# exit

SW1(config)# vlan 30
SW1(config-vlan)# name MANAGEMENT
SW1(config-vlan)# exit
```

**2단계: 스위치 액세스 포트 설정**

```
SW1(config)# interface FastEthernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit

SW1(config)# interface FastEthernet 0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit

SW1(config)# interface FastEthernet 0/3
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 30
SW1(config-if)# exit
```

**3단계: 스위치 트렁크 포트 설정 (라우터 연결)**

```
SW1(config)# interface GigabitEthernet 0/1
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20,30
SW1(config-if)# exit
```

**4단계: 라우터 서브인터페이스 설정**

```
! 물리 인터페이스 활성화 (IP 주소는 할당하지 않음)
R1(config)# interface GigabitEthernet 0/0
R1(config-if)# no shutdown
R1(config-if)# exit

! VLAN 10 서브인터페이스
R1(config)# interface GigabitEthernet 0/0.10
R1(config-subif)# description ## VLAN 10 - SALES ##
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
R1(config-subif)# exit

! VLAN 20 서브인터페이스
R1(config)# interface GigabitEthernet 0/0.20
R1(config-subif)# description ## VLAN 20 - DEVELOPMENT ##
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
R1(config-subif)# exit

! VLAN 30 서브인터페이스
R1(config)# interface GigabitEthernet 0/0.30
R1(config-subif)# description ## VLAN 30 - MANAGEMENT ##
R1(config-subif)# encapsulation dot1Q 30
R1(config-subif)# ip address 192.168.30.1 255.255.255.0
R1(config-subif)# exit
```

핵심 포인트는 다음과 같습니다:
- 물리 인터페이스(Gi0/0)에는 IP를 할당하지 않고 `no shutdown`만 합니다.
- 서브인터페이스 번호(`.10`, `.20`, `.30`)는 VLAN ID와 일치시키는 것이 관례이지만 필수는 아닙니다.
- `encapsulation dot1Q [VLAN-ID]`가 해당 서브인터페이스를 특정 VLAN에 매핑합니다.
- 서브인터페이스의 IP 주소가 각 VLAN의 기본 게이트웨이가 됩니다.

---

### 면접관: "각 PC의 게이트웨이는 어떻게 설정해야 합니까? 패킷이 VLAN 10의 PC에서 VLAN 20의 PC로 전달되는 과정을 단계별로 설명해주세요."

**PC 게이트웨이 설정:**

| PC | IP 주소 | 서브넷 마스크 | 기본 게이트웨이 |
|----|---------|-------------|---------------|
| PC-A (VLAN 10) | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC-B (VLAN 20) | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |
| PC-C (VLAN 30) | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |

각 PC의 기본 게이트웨이는 해당 VLAN의 서브인터페이스 IP 주소입니다.

**PC-A(192.168.10.10)에서 PC-B(192.168.20.10)로 ping하는 과정:**

**1단계 - PC-A의 판단:**
PC-A는 목적지 192.168.20.10이 자신의 서브넷(192.168.10.0/24)에 없음을 확인합니다. 다른 서브넷이므로 기본 게이트웨이(192.168.10.1)로 패킷을 보냅니다.

**2단계 - ARP로 게이트웨이 MAC 확인:**
PC-A는 192.168.10.1의 MAC 주소를 알기 위해 VLAN 10 내에서 ARP 요청을 브로드캐스트합니다. 라우터의 Gi0/0.10 서브인터페이스가 자신의 MAC 주소로 응답합니다.

**3단계 - 스위치로 프레임 전송:**
PC-A는 L2 프레임을 구성합니다. 목적지 MAC은 라우터(게이트웨이)의 MAC, 목적지 IP는 192.168.20.10입니다. 스위치의 Fa0/1(액세스 포트)로 태그 없는 프레임이 전송됩니다.

**4단계 - 스위치에서 트렁크로 전달:**
스위치는 이 프레임이 VLAN 10에 속함을 알고, 트렁크 포트(Gi0/1)로 전달할 때 802.1Q 태그(VLAN ID 10)를 추가합니다.

**5단계 - 라우터에서 라우팅 수행:**
라우터는 VLAN 10 태그가 붙은 프레임을 Gi0/0.10 서브인터페이스에서 수신합니다. IP 헤더의 목적지(192.168.20.10)를 확인하고 라우팅 테이블을 조회하여 Gi0/0.20 서브인터페이스로 전달할 것을 결정합니다.

**6단계 - 라우터에서 스위치로 재전송:**
라우터는 프레임에 VLAN 20 태그를 붙여 트렁크를 통해 스위치로 보냅니다. 이때 ARP로 PC-B의 MAC 주소를 먼저 확인합니다.

**7단계 - 스위치에서 PC-B로 전달:**
스위치는 VLAN 20 태그를 확인하고, PC-B가 연결된 Fa0/2(액세스 포트)로 태그를 제거한 후 프레임을 전달합니다.

---

### 면접관: "Layer 3 스위치의 SVI(Switch Virtual Interface)를 사용한 Inter-VLAN Routing은 Router-on-a-Stick과 어떻게 다릅니까?"

L3 스위치 SVI 방식은 스위치 자체에서 라우팅을 수행하므로, 외부 라우터가 필요 없습니다.

**L3 스위치 SVI 설정:**

```
! L3 라우팅 기능 활성화
L3-SW(config)# ip routing

! SVI 생성 (각 VLAN의 게이트웨이)
L3-SW(config)# interface vlan 10
L3-SW(config-if)# description ## VLAN 10 Gateway ##
L3-SW(config-if)# ip address 192.168.10.1 255.255.255.0
L3-SW(config-if)# no shutdown
L3-SW(config-if)# exit

L3-SW(config)# interface vlan 20
L3-SW(config-if)# description ## VLAN 20 Gateway ##
L3-SW(config-if)# ip address 192.168.20.1 255.255.255.0
L3-SW(config-if)# no shutdown
L3-SW(config-if)# exit

L3-SW(config)# interface vlan 30
L3-SW(config-if)# description ## VLAN 30 Gateway ##
L3-SW(config-if)# ip address 192.168.30.1 255.255.255.0
L3-SW(config-if)# no shutdown
L3-SW(config-if)# exit
```

**두 방식의 상세 비교:**

| 항목 | Router-on-a-Stick | L3 Switch SVI |
|------|-------------------|---------------|
| 라우팅 위치 | 외부 라우터 | 스위치 내부 |
| 성능 | 트렁크 대역폭에 제한됨 | 하드웨어(ASIC) 기반 와이어스피드 |
| 병목 지점 | 라우터-스위치 간 트렁크 링크 | 거의 없음 |
| 비용 | 라우터 + 스위치 필요 | L3 스위치 1대 (더 비쌈) |
| 확장성 | VLAN 수 증가 시 성능 저하 | 우수 |
| 적합한 환경 | 소규모, 트래픽 적은 환경 | 중대규모, 실무 환경 |
| 구성 복잡도 | 서브인터페이스 설정 | SVI 설정 (더 간단) |

Router-on-a-Stick의 가장 큰 단점은 모든 Inter-VLAN 트래픽이 라우터와 스위치 사이의 단일 물리 링크를 왕복해야 한다는 것입니다. VLAN 10에서 VLAN 20으로의 트래픽은 스위치에서 라우터로 올라갔다가 다시 같은 링크를 통해 스위치로 내려옵니다. 이 링크가 병목이 됩니다.

L3 스위치는 ASIC(하드웨어 칩)으로 라우팅을 처리하므로 소프트웨어 기반 라우터보다 훨씬 빠릅니다. 실무에서는 대부분 L3 스위치를 사용합니다.

---

### 면접관: "Native VLAN 불일치가 Router-on-a-Stick 환경에서 어떤 문제를 일으킬 수 있습니까?"

Native VLAN은 802.1Q 트렁크에서 태그를 붙이지 않고 전송하는 VLAN입니다. 기본값은 VLAN 1입니다.

**Native VLAN 불일치 문제:**

스위치의 트렁크 포트와 라우터의 서브인터페이스에서 Native VLAN 설정이 다르면 통신 장애가 발생합니다.

```
! 예: 스위치는 Native VLAN이 1 (기본값)
SW1(config)# interface GigabitEthernet 0/1
SW1(config-if)# switchport trunk native vlan 1

! 라우터에서 VLAN 1 서브인터페이스를 native로 설정해야 함
R1(config)# interface GigabitEthernet 0/0.1
R1(config-subif)# encapsulation dot1Q 1 native
R1(config-subif)# ip address 192.168.1.1 255.255.255.0
```

`encapsulation dot1Q 1 native`에서 `native` 키워드는 이 서브인터페이스가 태그 없는 프레임을 처리한다는 의미입니다.

**불일치가 발생하면:**

1. 스위치에서 CDP/STP 등의 메시지로 Native VLAN 불일치 경고가 나타납니다:
   ```
   %CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on
   GigabitEthernet0/1 (1), with R1 GigabitEthernet0/0 (99).
   ```

2. 태그 없이 전송된 프레임이 잘못된 VLAN에서 처리되어 통신 장애가 발생합니다.

3. 보안 위험: VLAN Hopping 공격에 취약해질 수 있습니다.

**보안 모범 사례:**

```
! Native VLAN을 사용하지 않는 VLAN으로 변경
SW1(config)# interface GigabitEthernet 0/1
SW1(config-if)# switchport trunk native vlan 999

! Native VLAN에 태그를 강제로 붙이기 (선택)
SW1(config)# vlan dot1q tag native
```

---

### 면접관: "Router-on-a-Stick 환경에서 Inter-VLAN 통신이 안 될 때, 트러블슈팅 절차를 순서대로 설명해주세요."

체계적인 트러블슈팅 절차를 단계별로 설명하겠습니다.

**1단계: 물리적 연결 및 인터페이스 상태 확인**

```
R1# show ip interface brief
! 물리 인터페이스와 모든 서브인터페이스가 up/up인지 확인

SW1# show interface trunk
! 트렁크 포트가 활성화되어 있는지 확인
! Allowed VLAN에 해당 VLAN이 포함되어 있는지 확인
```

**2단계: VLAN 및 포트 할당 확인**

```
SW1# show vlan brief
! VLAN이 생성되어 있는지, PC 포트가 올바른 VLAN에 할당되어 있는지 확인

VLAN Name                 Status    Ports
---- -------------------- --------- -------------------------
1    default              active
10   SALES                active    Fa0/1
20   DEVELOPMENT          active    Fa0/2
30   MANAGEMENT           active    Fa0/3
```

**3단계: 트렁크 설정 확인**

```
SW1# show interface trunk
Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      1

Port        Vlans allowed on trunk
Gi0/1       10,20,30

Port        Vlans allowed and active in management domain
Gi0/1       10,20,30
```

확인 사항:
- 트렁크가 `trunking` 상태인지
- 허용 VLAN 목록에 필요한 VLAN이 포함되어 있는지
- Native VLAN이 양쪽 일치하는지

**4단계: 라우터 서브인터페이스 설정 확인**

```
R1# show running-config interface GigabitEthernet 0/0.10
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

! encapsulation VLAN ID가 올바른지 확인
! IP 주소와 서브넷이 올바른지 확인
```

**5단계: PC 게이트웨이 설정 확인**

```
! PC에서 기본 게이트웨이가 올바르게 설정되어 있는지 확인
! PC-A의 게이트웨이가 192.168.10.1이어야 함

PC-A> ipconfig
IP Address: 192.168.10.10
Subnet Mask: 255.255.255.0
Default Gateway: 192.168.10.1
```

**6단계: 단계별 연결성 테스트**

```
! PC에서 게이트웨이로 ping (같은 VLAN 내 L3 연결 확인)
PC-A> ping 192.168.10.1

! 라우터에서 다른 서브인터페이스로 ping (라우터 내부 라우팅 확인)
R1# ping 192.168.20.1 source 192.168.10.1

! PC에서 다른 VLAN의 게이트웨이로 ping (Inter-VLAN 라우팅 확인)
PC-A> ping 192.168.20.1

! 최종: PC에서 다른 VLAN의 PC로 ping
PC-A> ping 192.168.20.10
```

**흔한 실수 체크리스트:**

| 증상 | 가능한 원인 | 확인 방법 |
|------|-----------|----------|
| 게이트웨이 ping 실패 | 서브인터페이스 encapsulation 잘못 설정 | show run int gi0/0.X |
| 같은 VLAN만 통신됨 | 물리 인터페이스 no shutdown 누락 | show ip int brief |
| 특정 VLAN만 실패 | 트렁크 allowed vlan에서 누락 | show int trunk |
| 간헐적 실패 | Native VLAN 불일치 | show int trunk (Native vlan 확인) |
| 모든 통신 실패 | 트렁크 미형성 (mode mismatch) | show int trunk |

가장 많이 발생하는 실수는 물리 인터페이스(Gi0/0)에 `no shutdown`을 하지 않거나, `encapsulation dot1Q` 명령어에서 VLAN ID를 잘못 입력하는 것입니다.
