# 02-01. 라우터 기본 구성과 인터페이스

> CCNA 200-301 라우팅 기초 면접 시나리오
> 주제: 라우터 부팅 과정, 기본 설정, 인터페이스 구성, show 명령어, 설정 저장

---

### 면접관: "네트워크 엔지니어로서 새 라우터를 처음 받았다고 가정해봅시다. 라우터의 부팅 과정을 단계별로 설명해주시겠습니까?"

라우터의 부팅 과정은 크게 4단계로 구분됩니다.

**1단계 - POST (Power-On Self-Test)**
라우터 전원이 켜지면 ROM에 저장된 POST가 실행되어 CPU, 메모리, 인터페이스 등 하드웨어를 점검합니다.

**2단계 - 부트스트랩 프로그램 로드**
POST가 정상 완료되면 ROM에서 부트스트랩 프로그램이 로드됩니다. 이 프로그램은 IOS 이미지를 찾아 로드하는 역할을 합니다.

**3단계 - IOS 이미지 로드**
부트스트랩은 NVRAM의 설정 레지스터(config-register)와 startup-config의 boot system 명령어를 확인하여 IOS 이미지를 찾습니다. 일반적으로 Flash 메모리에서 IOS를 로드합니다.

**4단계 - 설정 파일 로드**
IOS가 로드된 후, NVRAM에서 startup-config를 찾아 running-config로 로드합니다. startup-config가 없으면 Setup Mode로 진입합니다.

| 메모리 종류 | 저장 내용 | 특징 |
|-----------|---------|------|
| ROM | POST, 부트스트랩, ROMMON | 읽기 전용, 전원 꺼져도 유지 |
| Flash | IOS 이미지 | 전원 꺼져도 유지 |
| NVRAM | startup-config | 전원 꺼져도 유지 |
| RAM | running-config | 전원 꺼지면 소실 |

---

### 면접관: "좋습니다. 그러면 라우터를 처음 설정할 때 반드시 해야 하는 기본 보안 설정들은 무엇이 있습니까?"

라우터 초기 보안 설정으로 반드시 수행해야 하는 항목들은 다음과 같습니다.

```
Router> enable
Router# configure terminal

! 1. 호스트네임 설정
Router(config)# hostname HQ-RTR

! 2. enable secret 설정 (MD5 해시로 암호화됨)
HQ-RTR(config)# enable secret Cisco@2024

! 3. 콘솔 라인 패스워드 설정
HQ-RTR(config)# line console 0
HQ-RTR(config-line)# password ConsolePass
HQ-RTR(config-line)# login
HQ-RTR(config-line)# logging synchronous
HQ-RTR(config-line)# exec-timeout 5 0
HQ-RTR(config-line)# exit

! 4. VTY 라인 설정 (원격 접속용)
HQ-RTR(config)# line vty 0 4
HQ-RTR(config-line)# password VtyPass
HQ-RTR(config-line)# login
HQ-RTR(config-line)# transport input ssh
HQ-RTR(config-line)# exit

! 5. 배너 설정 (법적 경고 메시지)
HQ-RTR(config)# banner motd #Unauthorized access is prohibited!#

! 6. 평문 패스워드 암호화
HQ-RTR(config)# service password-encryption
```

특히 `enable secret`과 `enable password`의 차이를 알아야 합니다. `enable secret`은 MD5로 해시 처리되어 보안이 강하고, `enable password`는 평문 저장되어 `service password-encryption`을 적용해도 Type 7(가역적 암호화)만 적용됩니다. 두 개가 동시에 설정되면 `enable secret`이 우선합니다.

---

### 면접관: "인터페이스에 IP 주소를 할당하고 활성화하는 과정을 보여주세요. 실무에서 어떤 정보를 함께 설정합니까?"

인터페이스 설정은 다음과 같이 진행합니다.

```
HQ-RTR(config)# interface GigabitEthernet 0/0
HQ-RTR(config-if)# description ## Link to LAN Switch - Floor 1 ##
HQ-RTR(config-if)# ip address 192.168.10.1 255.255.255.0
HQ-RTR(config-if)# no shutdown
HQ-RTR(config-if)# exit

HQ-RTR(config)# interface GigabitEthernet 0/1
HQ-RTR(config-if)# description ## WAN Link to ISP ##
HQ-RTR(config-if)# ip address 203.0.113.1 255.255.255.252
HQ-RTR(config-if)# no shutdown
HQ-RTR(config-if)# exit
```

실무에서 중요한 포인트는 다음과 같습니다:

- **description**: 반드시 설정합니다. 어떤 장비와 연결되어 있는지 기록해두면 장애 처리 시 매우 유용합니다.
- **no shutdown**: Cisco 라우터의 인터페이스는 기본적으로 administratively down 상태이므로 반드시 활성화해야 합니다. 스위치는 기본적으로 up 상태인 것과 차이가 있습니다.
- **IP 주소 확인**: 서브넷 마스크를 잘못 입력하면 통신 장애가 발생하므로 네트워크 설계 문서와 대조하여 확인합니다.

---

### 면접관: "show ip interface brief 명령어의 출력을 읽는 방법을 설명해주세요. Status와 Protocol 컬럼이 각각 의미하는 것은 무엇입니까?"

`show ip interface brief`는 인터페이스 상태를 빠르게 확인할 수 있는 가장 유용한 명령어 중 하나입니다.

```
HQ-RTR# show ip interface brief
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     192.168.10.1    YES manual up                    up
GigabitEthernet0/1     203.0.113.1     YES manual up                    up
GigabitEthernet0/2     unassigned      YES unset  administratively down down
Serial0/0/0            10.0.0.1        YES manual up                    down
```

**Status 컬럼 (Layer 1 - 물리 계층)**:
- `up`: 물리적으로 연결되어 정상 동작
- `down`: 케이블이 연결되지 않았거나 상대측 인터페이스가 down
- `administratively down`: `shutdown` 명령어로 관리자가 비활성화한 상태

**Protocol 컬럼 (Layer 2 - 데이터링크 계층)**:
- `up`: L2 프로토콜이 정상 동작 (keepalive 수신 중)
- `down`: L2 프로토콜 문제 (encapsulation 불일치, 클럭 미설정 등)

| Status | Protocol | 의미 |
|--------|----------|------|
| up | up | 정상 동작 |
| up | down | L2 문제 (encapsulation, clock rate 등) |
| down | down | 물리적 연결 문제 |
| administratively down | down | shutdown 상태 |

Serial0/0/0 인터페이스처럼 Status는 up인데 Protocol이 down인 경우는 물리적 케이블은 연결되어 있지만 상대측과 L2 협상에 실패한 상태로, encapsulation 불일치나 clock rate 미설정 등이 원인일 수 있습니다.

---

### 면접관: "running-config와 startup-config의 차이를 설명하고, 설정을 저장하는 방법을 알려주세요. 저장하지 않으면 어떻게 됩니까?"

**running-config**는 RAM에 저장된 현재 동작 중인 설정입니다. 명령어를 입력하면 즉시 running-config에 반영되어 바로 적용됩니다.

**startup-config**는 NVRAM에 저장된 설정으로, 라우터가 부팅될 때 읽어오는 파일입니다.

설정을 저장하지 않고 라우터를 재부팅하면 RAM의 running-config는 사라지고, NVRAM의 startup-config가 다시 로드되므로 변경한 설정이 모두 사라집니다.

```
! 설정 저장 방법 1 (권장)
HQ-RTR# copy running-config startup-config
Destination filename [startup-config]? [Enter]

! 설정 저장 방법 2 (축약)
HQ-RTR# write memory

! 설정 저장 방법 3 (더 축약)
HQ-RTR# wr
```

반대로 잘못된 설정을 했을 때 이전 상태로 되돌리고 싶다면:

```
! startup-config를 running-config로 복원
HQ-RTR# copy startup-config running-config

! 또는 재부팅하여 startup-config로 복원
HQ-RTR# reload
```

실무에서는 설정 변경 전에 반드시 `show running-config`를 백업해두고, 변경 후 충분히 테스트한 뒤 `copy run start`로 저장하는 것이 좋은 습관입니다.

---

### 면접관: "콘솔 접속과 VTY 접속의 차이를 설명해주세요. SSH와 Telnet 중 어떤 것을 사용해야 합니까?"

**콘솔 접속 (line console 0)**:
- 라우터에 콘솔 케이블(롤오버 케이블)을 물리적으로 직접 연결하여 접속합니다.
- 네트워크가 동작하지 않아도 접속할 수 있어 초기 설정이나 장애 복구 시 사용합니다.
- 물리적 접근이 필요하므로 보안 측면에서 상대적으로 안전합니다.

**VTY 접속 (line vty 0 4)**:
- 네트워크를 통해 원격으로 접속합니다. Telnet 또는 SSH를 사용합니다.
- `line vty 0 4`는 5개의 동시 원격 접속을 허용한다는 의미입니다.

**SSH vs Telnet 비교**:

| 항목 | Telnet | SSH |
|------|--------|-----|
| 포트 | TCP 23 | TCP 22 |
| 암호화 | 없음 (평문 전송) | 있음 (암호화 전송) |
| 보안 | 취약 | 안전 |
| 실무 사용 | 사용하지 않음 | 필수 사용 |

반드시 SSH를 사용해야 합니다. Telnet은 패킷 캡처 시 ID와 비밀번호가 평문으로 노출됩니다.

```
! SSH 설정 절차
HQ-RTR(config)# hostname HQ-RTR
HQ-RTR(config)# ip domain-name company.com
HQ-RTR(config)# crypto key generate rsa modulus 2048
HQ-RTR(config)# ip ssh version 2
HQ-RTR(config)# username admin privilege 15 secret AdminPass

HQ-RTR(config)# line vty 0 4
HQ-RTR(config-line)# transport input ssh
HQ-RTR(config-line)# login local
```

`transport input ssh`로 설정하면 Telnet 접속은 차단되고 SSH만 허용됩니다. `login local`은 로컬 사용자 데이터베이스를 인증에 사용하겠다는 의미입니다.

---

### 면접관: "마지막으로, 라우터 상태를 점검할 때 주로 사용하는 show 명령어 5개를 우선순위로 알려주세요."

장애 상황에서 가장 먼저 확인하는 show 명령어 5가지를 우선순위 순으로 말씀드리겠습니다.

```
! 1. 인터페이스 상태 요약 확인
HQ-RTR# show ip interface brief

! 2. 라우팅 테이블 확인
HQ-RTR# show ip route

! 3. 현재 동작 중인 전체 설정 확인
HQ-RTR# show running-config

! 4. 특정 인터페이스 상세 정보 확인
HQ-RTR# show interface GigabitEthernet 0/0

! 5. 로그 메시지 확인
HQ-RTR# show logging
```

각 명령어의 활용 목적은 다음과 같습니다:

- **show ip interface brief**: 모든 인터페이스의 IP와 up/down 상태를 한눈에 파악합니다. 장애 시 가장 먼저 입력하는 명령어입니다.
- **show ip route**: 라우팅 테이블을 확인하여 목적지까지 경로가 존재하는지 확인합니다.
- **show running-config**: 현재 설정 전체를 확인합니다. 특정 섹션만 보려면 `show running-config | section interface` 같은 파이프 필터를 사용할 수 있습니다.
- **show interface**: 특정 인터페이스의 에러 카운터, CRC 에러, 패킷 드롭 등 상세 통계를 확인합니다.
- **show logging**: 인터페이스 up/down 이벤트, 에러 메시지 등 시스템 로그를 확인하여 장애 원인의 실마리를 찾습니다.

추가로 `show version`은 IOS 버전, 가동 시간(uptime), 하드웨어 정보를 확인할 때 유용합니다.
