# 03-01. 스위치 기본 구성과 포트 설정

> CCNA 200-301 레벨 네트워크 엔지니어 면접 대비 - 스위칭 기초

---

### 면접관: "새로 입사해서 랙에 방금 올린 신규 스위치를 초기 설정해야 합니다. 스위치 부팅 과정부터 설명하고, 가장 먼저 어떤 설정들을 하시겠습니까?"

네, 먼저 Cisco 스위치의 부팅 과정을 설명드리겠습니다.

스위치 전원을 켜면 다음 순서로 부팅이 진행됩니다:

1. **POST (Power-On Self-Test)**: 하드웨어 자체 진단을 수행합니다
2. **Boot Loader 실행**: Flash 메모리에서 IOS 이미지를 찾아 로드합니다
3. **IOS 로딩**: 운영체제가 RAM에 적재됩니다
4. **Startup-config 로딩**: NVRAM에 저장된 설정 파일을 적용합니다

만약 startup-config가 없으면 **Setup Mode**로 진입하게 됩니다. 저는 Setup Mode를 건너뛰고 CLI에서 직접 설정하는 것을 선호합니다.

초기 설정 순서는 다음과 같습니다:

```
Switch> enable
Switch# configure terminal

! 1) 호스트네임 설정
Switch(config)# hostname SW-FLOOR1

! 2) 콘솔 패스워드 설정
SW-FLOOR1(config)# line console 0
SW-FLOOR1(config-line)# password cisco123
SW-FLOOR1(config-line)# login
SW-FLOOR1(config-line)# exit

! 3) Enable 패스워드 설정 (암호화)
SW-FLOOR1(config)# enable secret class123

! 4) VTY 라인 설정 (원격 접속용)
SW-FLOOR1(config)# line vty 0 15
SW-FLOOR1(config-line)# password vty123
SW-FLOOR1(config-line)# login
SW-FLOOR1(config-line)# transport input ssh
SW-FLOOR1(config-line)# exit

! 5) 관리용 VLAN IP 설정
SW-FLOOR1(config)# interface vlan 1
SW-FLOOR1(config-if)# ip address 192.168.1.10 255.255.255.0
SW-FLOOR1(config-if)# no shutdown
SW-FLOOR1(config-if)# exit

! 6) 기본 게이트웨이 설정
SW-FLOOR1(config)# ip default-gateway 192.168.1.1

! 7) 패스워드 암호화
SW-FLOOR1(config)# service password-encryption

! 8) 배너 설정
SW-FLOOR1(config)# banner motd #Unauthorized access prohibited#

! 9) 설정 저장
SW-FLOOR1(config)# end
SW-FLOOR1# copy running-config startup-config
```

호스트네임, 패스워드, 관리 IP가 기본 중의 기본이고, 보안을 위해 `service password-encryption`과 SSH 설정도 반드시 함께 합니다.

---

### 면접관: "관리용 IP를 VLAN 1에 설정하셨는데, 실무에서는 보통 어떻게 합니까?"

좋은 지적입니다. 실무에서는 보안상의 이유로 **VLAN 1을 관리용으로 사용하지 않는 것**이 권장됩니다.

VLAN 1은 모든 포트의 기본 VLAN이기 때문에, 공격자가 쉽게 접근할 수 있습니다. 따라서 별도의 관리 VLAN을 생성해서 사용합니다.

```
SW-FLOOR1(config)# vlan 99
SW-FLOOR1(config-vlan)# name MANAGEMENT
SW-FLOOR1(config-vlan)# exit

SW-FLOOR1(config)# interface vlan 99
SW-FLOOR1(config-if)# ip address 10.10.99.10 255.255.255.0
SW-FLOOR1(config-if)# no shutdown
SW-FLOOR1(config-if)# exit

SW-FLOOR1(config)# ip default-gateway 10.10.99.1
```

이렇게 하면 관리 트래픽이 일반 사용자 트래픽과 분리되어 보안이 강화됩니다.

---

### 면접관: "스위치 포트의 Speed와 Duplex 설정에 대해 설명해 주세요. Auto-negotiation이 실패하면 어떤 문제가 생길 수 있습니까?"

스위치 포트의 Speed와 Duplex는 기본적으로 **auto**로 설정되어 있어서, 연결된 상대 장비와 자동 협상(Auto-negotiation)을 통해 결정됩니다.

```
SW-FLOOR1(config)# interface FastEthernet 0/1
SW-FLOOR1(config-if)# speed auto
SW-FLOOR1(config-if)# duplex auto
```

수동으로 고정하려면 다음과 같이 설정합니다:

```
SW-FLOOR1(config-if)# speed 100
SW-FLOOR1(config-if)# duplex full
```

Auto-negotiation이 실패하는 경우가 문제입니다. 예를 들어, 한쪽은 수동으로 100Mbps/Full-duplex로 설정하고, 상대방은 auto로 두면:

| 상황 | Speed 결과 | Duplex 결과 |
|------|-----------|-------------|
| 양쪽 모두 auto | 협상된 최고 속도 | 협상된 최적 모드 |
| 한쪽 수동, 한쪽 auto | auto 쪽이 감지된 속도 사용 | auto 쪽이 **half-duplex로 폴백** |
| 양쪽 모두 수동 (일치) | 설정된 속도 | 설정된 모드 |
| 양쪽 수동 (불일치) | 링크 다운 가능 | Duplex mismatch 발생 |

가장 흔한 문제는 **Duplex Mismatch**입니다. 한쪽이 full, 한쪽이 half가 되면 late collision, FCS error가 증가하고 성능이 크게 저하됩니다. `show interfaces` 명령어에서 이런 에러 카운터를 확인할 수 있습니다.

실무에서는 서버나 중요한 장비 연결 포트는 **양쪽 모두 수동으로 동일하게 설정**하는 것이 안정적입니다.

---

### 면접관: "포트 상태를 확인하는 방법과 err-disabled 상태가 무엇인지 설명해 주세요."

포트 상태 확인에는 주로 다음 명령어들을 사용합니다:

```
! 모든 포트 상태 한눈에 보기
SW-FLOOR1# show interfaces status

Port      Name     Status       Vlan  Duplex  Speed  Type
Fa0/1     SERVER1  connected    1     a-full  a-100  10/100BaseTX
Fa0/2              notconnect   1     auto    auto   10/100BaseTX
Fa0/3     PRINTER  err-disabled 1     auto    auto   10/100BaseTX

! 특정 포트 상세 정보
SW-FLOOR1# show interfaces FastEthernet 0/1

! 포트 에러 카운터만 확인
SW-FLOOR1# show interfaces FastEthernet 0/1 counters errors
```

포트 상태(Status)는 크게 세 가지입니다:

- **connected**: 정상적으로 연결되어 동작 중
- **notconnect**: 케이블이 연결되지 않았거나 상대 장비가 꺼져 있음
- **err-disabled**: 보안 위반 등의 이유로 스위치가 포트를 강제로 비활성화한 상태

**err-disabled**가 되는 주요 원인은:

1. **Port Security violation**: MAC 주소 제한 초과
2. **BPDU Guard**: Access 포트에서 STP BPDU가 수신됨
3. **Duplex mismatch**: Duplex 불일치 감지
4. **ARP inspection 위반**

err-disabled 상태를 복구하는 방법은 두 가지입니다:

```
! 방법 1: 수동 복구 (포트를 shutdown 후 다시 no shutdown)
SW-FLOOR1(config)# interface FastEthernet 0/3
SW-FLOOR1(config-if)# shutdown
SW-FLOOR1(config-if)# no shutdown

! 방법 2: 자동 복구 설정 (타이머 기반)
SW-FLOOR1(config)# errdisable recovery cause psecure-violation
SW-FLOOR1(config)# errdisable recovery cause bpduguard
SW-FLOOR1(config)# errdisable recovery interval 300
```

자동 복구를 설정하면 원인이 해결된 경우 지정된 시간(기본 300초) 후 자동으로 포트가 활성화됩니다. 다만 원인을 먼저 해결하지 않으면 다시 err-disabled로 빠질 수 있으므로, 근본 원인 파악이 우선입니다.

---

### 면접관: "show mac address-table 명령어를 실행했더니 여러 MAC 주소가 보입니다. MAC 주소 테이블이 어떻게 만들어지고 관리되는지 설명해 주세요."

MAC 주소 테이블은 스위치가 프레임을 전달할 때 사용하는 핵심 테이블입니다. **CAM(Content Addressable Memory) 테이블**이라고도 부릅니다.

MAC 주소 테이블 학습 과정은 다음과 같습니다:

1. PC-A가 프레임을 전송하면, 스위치는 **출발지 MAC 주소**와 **수신 포트 번호**를 테이블에 기록합니다
2. 목적지 MAC 주소가 테이블에 있으면 해당 포트로만 전송합니다 (**Known Unicast**)
3. 목적지 MAC 주소가 테이블에 없으면 수신 포트를 제외한 모든 포트로 전송합니다 (**Unknown Unicast Flooding**)

```
SW-FLOOR1# show mac address-table
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0011.2233.4455    DYNAMIC     Fa0/1
   1    0066.7788.9900    DYNAMIC     Fa0/2
   1    00aa.bbcc.ddee    STATIC      Fa0/10
Total Mac Addresses for this criterion: 3

! 특정 포트의 MAC 주소만 확인
SW-FLOOR1# show mac address-table interface FastEthernet 0/1

! 특정 VLAN의 MAC 주소만 확인
SW-FLOOR1# show mac address-table vlan 10

! MAC 주소 테이블 aging time 확인
SW-FLOOR1# show mac address-table aging-time
```

주요 특징은 다음과 같습니다:

| 항목 | 설명 |
|------|------|
| DYNAMIC | 프레임 수신 시 자동 학습 |
| STATIC | 관리자가 수동으로 등록 |
| Aging Time | 기본 300초 (5분), 이 시간 동안 해당 MAC에서 트래픽이 없으면 삭제 |
| 테이블 크기 | 스위치 모델에 따라 다름 (수천~수만 개) |

테이블이 가득 차면 새로운 MAC 주소를 학습할 수 없고, 모든 트래픽이 flooding되어 성능 저하가 발생합니다. 이를 악용한 공격이 **MAC Flooding Attack**이며, 이를 방지하기 위해 Port Security를 사용합니다.

---

### 면접관: "마지막으로, 스위치 설정을 저장하는 것과 관련해서 running-config와 startup-config의 차이를 정리해 주세요."

| 구분 | running-config | startup-config |
|------|---------------|----------------|
| 저장 위치 | RAM | NVRAM |
| 역할 | 현재 동작 중인 설정 | 부팅 시 로드되는 설정 |
| 전원 OFF 시 | 소멸됨 | 유지됨 |
| 수정 방법 | CLI에서 명령어 입력 즉시 반영 | copy run start 명령으로 저장 |

설정 저장 관련 주요 명령어:

```
! 현재 설정을 NVRAM에 저장
SW-FLOOR1# copy running-config startup-config
! 또는 줄여서
SW-FLOOR1# write memory

! 현재 설정 확인
SW-FLOOR1# show running-config

! 저장된 설정 확인
SW-FLOOR1# show startup-config

! 저장된 설정 삭제 (공장 초기화 시)
SW-FLOOR1# erase startup-config
SW-FLOOR1# reload
```

실무에서 가장 흔한 실수가 설정 변경 후 `copy running-config startup-config`를 하지 않아서, 스위치 재부팅 후 설정이 날아가는 것입니다. 저는 중요한 설정 변경 후에는 반드시 저장하는 습관을 가지고 있습니다.

또한, 설정이 잘못되었을 때를 대비해서 변경 전에 현재 설정을 TFTP 서버에 백업해 두는 것도 좋은 습관입니다:

```
SW-FLOOR1# copy running-config tftp:
Address or name of remote host []? 10.10.10.100
Destination filename [sw-floor1-confg]? sw-floor1-backup-20240115
```

---

> **핵심 정리**: 스위치 초기 설정은 호스트네임, 패스워드, 관리 IP 설정이 기본이며, 포트의 speed/duplex 불일치 방지, err-disabled 복구 방법, MAC 주소 테이블 동작 원리를 이해하는 것이 중요합니다.
