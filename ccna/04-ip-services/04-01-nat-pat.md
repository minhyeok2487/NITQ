# 04-01. NAT와 PAT 기본

> CCNA 200-301 레벨 네트워크 엔지니어 면접 시나리오
> 주제: NAT 필요성, Static/Dynamic NAT, PAT(Overload), Inside/Outside Local/Global, NAT 설정 및 트러블슈팅

---

### 면접관: "사내 네트워크에서 PC 200대가 인터넷을 사용해야 하는데, ISP로부터 공인 IP를 5개만 할당받았습니다. 어떻게 해결하시겠습니까?"

이 상황에서는 **NAT(Network Address Translation)**을 사용하여 해결할 수 있습니다. 특히 200대의 PC가 5개의 공인 IP만으로 인터넷에 접속해야 하므로, **PAT(Port Address Translation)**를 적용하는 것이 가장 효율적입니다.

PAT는 NAT Overload라고도 불리며, 하나의 공인 IP 주소에 서로 다른 포트 번호를 매핑하여 여러 사설 IP 주소가 동시에 인터넷을 사용할 수 있게 합니다. 5개의 공인 IP에 PAT를 적용하면 이론적으로 수만 개의 동시 세션을 처리할 수 있어, 200대의 PC를 충분히 수용할 수 있습니다.

NAT가 필요한 근본적인 이유는 **IPv4 주소 고갈** 문제입니다. RFC 1918에 정의된 사설 IP 대역(10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)은 인터넷에서 라우팅되지 않으므로, 외부 통신을 위해서는 반드시 공인 IP로 변환해야 합니다.

---

### 면접관: "NAT에는 여러 종류가 있는데, Static NAT, Dynamic NAT, PAT의 차이를 설명하고 각각 언제 사용하는지 말해주세요."

세 가지 NAT 유형의 차이는 다음과 같습니다.

| 구분 | Static NAT | Dynamic NAT | PAT (Overload) |
|------|-----------|-------------|-----------------|
| 매핑 방식 | 1:1 고정 매핑 | 1:1 동적 매핑 | N:1 다대일 매핑 |
| 공인 IP 수 | 사설 IP 수와 동일 | 풀(Pool)에서 할당 | 최소 1개로 가능 |
| 포트 번호 변환 | 없음 | 없음 | 있음 |
| 주요 용도 | 서버 공개 | 특정 그룹 접속 | 일반 인터넷 접속 |

**Static NAT**는 사설 IP와 공인 IP를 1:1로 고정 매핑합니다. 내부에 웹 서버나 메일 서버처럼 외부에서 항상 접근 가능해야 하는 장비가 있을 때 사용합니다.

**Dynamic NAT**는 공인 IP 풀(Pool)에서 사설 IP에 동적으로 1:1 매핑합니다. 풀에 있는 공인 IP가 모두 사용 중이면 추가 연결이 불가능합니다.

**PAT**는 하나의 공인 IP를 포트 번호로 구분하여 여러 사설 IP가 공유합니다. 실무에서 가장 많이 사용되는 방식입니다.

---

### 면접관: "그러면 Inside Local, Inside Global, Outside Local, Outside Global 개념을 설명해주시겠습니까?"

NAT에서 사용하는 네 가지 주소 용어를 정리하면 다음과 같습니다.

| 용어 | 설명 | 예시 |
|------|------|------|
| **Inside Local** | 내부 네트워크에서 사용하는 사설 IP | 192.168.1.10 |
| **Inside Global** | 내부 호스트가 외부에 보여지는 공인 IP | 203.0.113.1 |
| **Outside Local** | 내부에서 바라본 외부 호스트의 IP | 보통 Outside Global과 동일 |
| **Outside Global** | 외부 네트워크에서 실제 사용하는 공인 IP | 8.8.8.8 |

쉽게 정리하면, **Inside/Outside**는 호스트의 위치(내부/외부)를 의미하고, **Local/Global**은 주소가 NAT 변환 전(Local)인지 변환 후(Global)인지를 의미합니다.

일반적인 환경에서 Outside Local과 Outside Global은 동일한 경우가 대부분이지만, 외부 주소도 변환하는 특수한 경우에는 다를 수 있습니다.

---

### 면접관: "실제로 라우터에 PAT를 설정해본 경험이 있다면, 설정 과정을 CLI로 보여주세요."

PAT(Overload) 설정 과정을 단계별로 설명드리겠습니다.

#### 핵심 설정

```
! 1단계: NAT 대상이 되는 내부 트래픽 정의 (ACL)
Router(config)# access-list 10 permit 192.168.1.0 0.0.0.255

! 2단계: Inside/Outside 인터페이스 지정
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip address 192.168.1.1 255.255.255.0
Router(config-if)# ip nat inside
Router(config-if)# exit

Router(config)# interface GigabitEthernet0/1
Router(config-if)# ip address 203.0.113.1 255.255.255.252
Router(config-if)# ip nat outside
Router(config-if)# exit

! 3단계: PAT 설정 (외부 인터페이스 IP 사용)
Router(config)# ip nat inside source list 10 interface GigabitEthernet0/1 overload
```

핵심은 마지막 줄의 `overload` 키워드입니다. 이 키워드가 PAT를 활성화하여 하나의 공인 IP에 포트 번호를 달리하여 다수의 사설 IP를 매핑합니다.

만약 공인 IP 풀을 사용하는 PAT라면 다음과 같이 설정합니다.

```
Router(config)# ip nat pool MYPOOL 203.0.113.1 203.0.113.5 netmask 255.255.255.248
Router(config)# ip nat inside source list 10 pool MYPOOL overload
```

---

### 면접관: "Static NAT 설정은 어떻게 다릅니까? 내부 웹 서버를 외부에 공개해야 하는 상황을 가정해주세요."

Static NAT는 ACL이나 풀 없이 직접 1:1 매핑을 지정합니다.

#### 핵심 설정

```
! 내부 웹 서버 192.168.1.100을 공인 IP 203.0.113.10으로 고정 매핑
Router(config)# ip nat inside source static 192.168.1.100 203.0.113.10

! 인터페이스 방향 지정은 PAT와 동일
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip nat inside
Router(config-if)# exit

Router(config)# interface GigabitEthernet0/1
Router(config-if)# ip nat outside
Router(config-if)# exit
```

이렇게 설정하면 외부 사용자가 203.0.113.10으로 접속하면 내부의 192.168.1.100 웹 서버로 트래픽이 전달됩니다. Static NAT는 양방향으로 동작하므로, 외부에서 내부로의 접근도 가능해집니다.

---

### 면접관: "NAT 설정 후 검증과 트러블슈팅은 어떻게 진행합니까?"

NAT 상태를 확인하고 문제를 진단하는 데 사용하는 주요 명령어를 설명드리겠습니다.

#### 검증 명령어

```
! 현재 NAT 변환 테이블 확인
Router# show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
tcp  203.0.113.1:1024  192.168.1.10:3456  8.8.8.8:80         8.8.8.8:80
tcp  203.0.113.1:1025  192.168.1.11:4567  142.250.196.4:443  142.250.196.4:443

! NAT 통계 확인
Router# show ip nat statistics

! NAT 변환 과정 실시간 디버그
Router# debug ip nat
```

#### 자주 발생하는 트러블슈팅 포인트

1. **인터페이스 방향 오설정**: `ip nat inside`와 `ip nat outside`가 반대로 설정된 경우가 가장 흔합니다.
2. **ACL 미스매치**: NAT에 연결된 ACL이 실제 내부 네트워크 대역과 일치하지 않는 경우입니다.
3. **라우팅 문제**: NAT 변환은 되지만 리턴 트래픽의 라우팅 경로가 없는 경우입니다.
4. **NAT 테이블 초기화 필요**: 설정 변경 후 기존 NAT 엔트리를 지워야 할 때 `clear ip nat translation *` 명령을 사용합니다.

```
! NAT 변환 테이블 전체 초기화
Router# clear ip nat translation *
```

---

### 면접관: "PAT에서 포트 멀티플렉싱이 동작하는 원리를 좀 더 자세히 설명해주시겠습니까?"

PAT의 포트 멀티플렉싱은 TCP/UDP의 포트 번호를 활용하여 여러 내부 호스트를 구분합니다.

예를 들어, 내부 PC 두 대가 동시에 같은 웹사이트에 접속하는 경우를 보겠습니다.

```
[변환 전]
PC-A (192.168.1.10:50001) → 웹서버 (8.8.8.8:80)
PC-B (192.168.1.11:50002) → 웹서버 (8.8.8.8:80)

[PAT 변환 후]
203.0.113.1:1024 → 웹서버 (8.8.8.8:80)   ← PC-A의 트래픽
203.0.113.1:1025 → 웹서버 (8.8.8.8:80)   ← PC-B의 트래픽
```

공인 IP는 203.0.113.1로 동일하지만, **출발지 포트 번호가 1024와 1025로 다르게 할당**됩니다. 웹 서버의 응답이 돌아올 때, 라우터는 이 포트 번호를 기준으로 원래 어떤 PC의 트래픽이었는지 식별하여 올바른 내부 호스트로 전달합니다.

TCP/UDP 포트 번호는 16비트이므로 이론적으로 약 65,535개의 동시 세션을 처리할 수 있습니다. 실제로는 기존에 사용 중인 Well-known 포트 등을 제외하면 공인 IP 하나당 약 60,000개 이상의 세션을 지원합니다.

---

### 면접관: "마지막으로, NAT를 사용할 때의 장단점을 정리해주세요."

| 구분 | 내용 |
|------|------|
| **장점** | IPv4 주소 절약 (사설 IP 재사용 가능) |
| | 내부 네트워크 구조 은닉 (보안 향상) |
| | ISP 변경 시 내부 IP 변경 불필요 |
| | 네트워크 설계 유연성 확보 |
| **단점** | End-to-End 연결성 깨짐 (P2P 통신 어려움) |
| | 일부 프로토콜 호환 문제 (FTP active mode, SIP 등) |
| | 패킷 처리 지연 (IP 헤더 변환 오버헤드) |
| | 트러블슈팅 복잡도 증가 |

특히 FTP active 모드처럼 데이터 안에 IP 주소를 포함하는 프로토콜은 NAT 환경에서 문제가 발생할 수 있습니다. 이런 경우에는 라우터의 **ALG(Application Layer Gateway)** 기능이 페이로드 내의 IP 주소까지 변환해주어야 정상 동작합니다.

장기적으로는 IPv6로 전환하면 NAT 없이 모든 장비가 고유한 글로벌 주소를 가질 수 있지만, 현재까지는 NAT가 IPv4 네트워크 운영에서 필수적인 기술입니다.
