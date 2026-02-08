# 08-06. 패킷 캡처와 분석 (tcpdump, fw monitor)

---

## 면접관: "고객사에서 '특정 서버 간 통신이 간헐적으로 실패한다'고 합니다. 로그상으로는 뚜렷한 원인이 안 보여요. 어떻게 접근하시겠어요?"

간헐적 통신 장애는 로그만으로 원인 파악이 어려운 경우가 많습니다. 이럴 때는 **패킷 캡처를 통한 실제 트래픽 분석**이 가장 확실한 방법입니다.

접근 순서:

**1단계: 장애 현상 정확히 정의**
- 어떤 서버(Source)에서 어떤 서버(Destination)로?
- 어떤 프로토콜/포트?
- 간헐적이라면 패턴이 있는지? (특정 시간대, 특정 작업 시)
- 완전 실패인지, 느린 것인지, 타임아웃인지?

**2단계: 캡처 포인트 결정**

```
[Server A] --- [Switch] --- [Firewall] --- [Switch] --- [Server B]
     (1)          (2)          (3)           (4)          (5)
```

양 끝단(1, 5)과 방화벽 전후(2, 3, 4)에서 동시에 캡처하면, **어느 구간에서 패킷이 사라지는지** 정확히 파악할 수 있습니다.

**3단계: 캡처 도구 선택**

| 위치 | 도구 |
|------|------|
| 서버(Linux) | `tcpdump` |
| 서버(Windows) | Wireshark, `netsh trace` |
| Cisco ASA | `capture` 명령어 |
| Checkpoint Gateway | `fw monitor`, `tcpdump` |
| 네트워크 스위치 | SPAN/Mirror 포트 + 외부 캡처 장비 |

---

## 면접관: "Cisco ASA에서 패킷 캡처를 하려면 어떻게 하나요? 운영 장비에서 안전하게 할 수 있나요?"

ASA의 `capture` 명령어는 **운영 중에도 안전하게 사용 가능**합니다. 성능 영향이 최소화되도록 설계되어 있으며, 필터를 적용하면 특정 트래픽만 캡처할 수 있습니다.

### 기본 캡처 설정

```
! 캡처용 ACL 생성 (캡처할 트래픽 정의)
access-list CAP-FILTER extended permit tcp host 10.1.1.100 host 10.2.1.200 eq 443
access-list CAP-FILTER extended permit tcp host 10.2.1.200 eq 443 host 10.1.1.100

! Inside 인터페이스에서 캡처 시작
capture CAP-INSIDE interface inside access-list CAP-FILTER buffer 1048576

! Outside 인터페이스에서도 캡처 (NAT 전후 비교용)
capture CAP-OUTSIDE interface outside access-list CAP-FILTER-OUTSIDE buffer 1048576
```

> **buffer 크기를 지정**합니다. 기본 512KB는 금방 차므로, 1MB(1048576) 이상 권장합니다.

### 캡처 확인

```
! 캡처된 패킷 목록 보기
show capture CAP-INSIDE

! 상세 보기 (각 패킷의 헤더 정보)
show capture CAP-INSIDE detail

! 패킷 수와 바이트 수만 확인
show capture CAP-INSIDE | include packets
```

출력 예시:
```
12 packets captured

   1: 14:30:01.123456       10.1.1.100.54321 > 10.2.1.200.443: S 1234567890:1234567890(0) win 65535 <mss 1460>
   2: 14:30:01.123789       10.2.1.200.443 > 10.1.1.100.54321: S 9876543210:9876543210(0) ack 1234567891 win 65535 <mss 1460>
   3: 14:30:01.124012       10.1.1.100.54321 > 10.2.1.200.443: . ack 9876543211 win 65535
```

### pcap 파일로 다운로드 (Wireshark 분석용)

```
! 웹 브라우저에서 다운로드
! https://<ASA_IP>/admin/capture/CAP-INSIDE/pcap

! 또는 TFTP로 전송
copy /pcap capture:CAP-INSIDE tftp://10.1.1.50/cap-inside.pcap
```

### 캡처 정리

```
! 캡처 중지 및 삭제
no capture CAP-INSIDE
no capture CAP-OUTSIDE
no access-list CAP-FILTER

! 또는 버퍼만 비우고 캡처 유지
clear capture CAP-INSIDE
```

> 캡처를 끝낸 후 **반드시 정리**합니다. 캡처가 계속 동작하면 불필요한 리소스를 소모합니다.

---

## 면접관: "방화벽의 Inside에서는 패킷이 보이는데 Outside에서는 안 보입니다. 방화벽이 Drop하고 있다는 건데, 정확히 왜 Drop하는지 어떻게 알 수 있나요?"

ASA에서 패킷이 Drop되는 정확한 원인을 알려면 **asp-drop 캡처**를 사용합니다.

### asp-drop 캡처

ASA의 **Accelerated Security Path(ASP)**는 패킷 처리 엔진입니다. 패킷이 Drop될 때 ASP가 이유를 기록하는데, 이것을 캡처할 수 있습니다.

```
! 특정 트래픽의 Drop만 캡처
capture ASP-DROP type asp-drop all access-list CAP-FILTER

! 모든 Drop 캡처 (필터 없이 - 대량 출력 주의)
capture ASP-DROP type asp-drop all
```

확인:
```
show capture ASP-DROP

! Drop 이유별 통계
show asp drop
```

`show asp drop` 출력 예시:
```
Frame drop:
  No route to host (no-route)                                        1523
  Flow is denied by configured rule (acl-drop)                       8901
  First TCP packet not SYN (tcp-not-syn)                              234
  TCP failed 3 way handshake (tcp-3whs-failed)                        56
  Reverse-path verify failed (rpf-violated)                           12
```

각 Drop 이유의 의미:

| Drop 이유 | 설명 | 해결 방향 |
|-----------|------|----------|
| **acl-drop** | ACL에 의해 차단됨 | 해당 트래픽을 허용하는 ACL Rule 추가 |
| **no-route** | 목적지로의 라우팅 경로 없음 | 라우팅 테이블 확인, Static Route 추가 |
| **tcp-not-syn** | 기존 세션 없이 SYN 아닌 TCP 패킷 수신 | 비대칭 라우팅 또는 세션 타임아웃 이후 재전송 |
| **tcp-3whs-failed** | TCP 3-way Handshake 미완료 | 서버 측 문제 또는 중간 장비 차단 |
| **rpf-violated** | Reverse Path Forwarding 검증 실패 | 비대칭 라우팅, ip verify reverse-path 확인 |
| **inspect-dns-pak-too-long** | DNS 패킷이 Inspection 허용 크기 초과 | DNS Inspection 설정 조정 |
| **conn-limit** | 연결 수 제한 초과 | `set connection conn-max` 설정 확인 |

### 특정 Drop에 대한 상세 캡처

```
! acl-drop만 캡처
capture ACP-ACL type asp-drop acl-drop access-list CAP-FILTER
show capture ASP-ACL detail
```

### packet-tracer와 함께 사용

```
! 실시간 캡처 중에 packet-tracer로 시뮬레이션
packet-tracer input inside tcp 10.1.1.100 54321 10.2.1.200 443 detailed
! 어느 Phase에서 DROP이 되는지 확인
```

asp-drop 캡처와 packet-tracer를 **함께 사용**하면, 실제 트래픽의 Drop 원인과 정책 매칭 과정을 동시에 파악할 수 있습니다.

---

## 면접관: "Checkpoint 환경이면 fw monitor를 사용하죠? fw monitor의 캡처 포지션에 대해 설명해 주세요."

`fw monitor`는 Checkpoint Gateway에서 **커널 레벨**의 패킷 캡처를 수행하는 도구입니다. 가장 큰 특징은 **패킷이 방화벽 커널의 어느 지점을 통과하는지** 위치별로 캡처할 수 있다는 것입니다.

### fw monitor의 4가지 캡처 포지션

```
                          Checkpoint Kernel
                    ┌──────────────────────────┐
Inbound             │                          │           Outbound
Interface  ──(i)──> │   Firewall Inspection    │ ──(I)──>  Interface
(pre-inbound)       │       Engine             │  (post-inbound)
                    │                          │
Outbound            │                          │           Inbound
Interface  <──(o)── │                          │ <──(O)──  Interface
(pre-outbound)      │                          │  (post-outbound)
                    └──────────────────────────┘
```

| 포지션 | 이름 | 위치 | 설명 |
|--------|------|------|------|
| **i** | Pre-Inbound | 인바운드 인터페이스 진입 직후, 검사 전 | 방화벽이 패킷을 받았지만 아직 검사하지 않은 상태 |
| **I** | Post-Inbound | 검사 완료 후, 아웃바운드 인터페이스로 나가기 전 | 방화벽 검사를 통과한 패킷 (Accept된 패킷) |
| **o** | Pre-Outbound | 아웃바운드 인터페이스에서 나가기 직전 | 실제로 인터페이스로 전달되기 직전 |
| **O** | Post-Outbound | 돌아오는 패킷이 아웃바운드 인터페이스에서 들어온 직후 | 리턴 트래픽의 진입 지점 |

### 핵심 해석법

- **i에서 보이는데 I에서 안 보이면**: 방화벽이 Drop한 것 (Rule에 의한 차단)
- **I에서 보이는데 o에서 안 보이면**: 라우팅 또는 NAT 문제
- **i에서도 안 보이면**: 패킷이 방화벽까지 도달하지 못한 것 (L2/L3 문제)
- **i, I, o 모두 보이는데 상대방에 도달 안 하면**: 방화벽 이후 구간 문제

### fw monitor 사용법

```bash
# 기본 캡처 (모든 포지션, 모든 트래픽)
fw monitor -e "accept;"

# 특정 호스트만 캡처
fw monitor -e "accept src=10.1.1.100 or dst=10.1.1.100;"

# 특정 포트만 캡처
fw monitor -e "accept src=10.1.1.100 and dport=443;"

# 특정 포지션만 캡처 (i와 I만)
fw monitor -e "accept src=10.1.1.100;" -m iI

# pcap 파일로 저장
fw monitor -e "accept src=10.1.1.100;" -o /var/log/capture.pcap
```

출력 예시:
```
eth0:i[60]: 10.1.1.100 -> 10.2.1.200 (TCP) len=60 id=12345
  TCP: 54321 -> 443 .S.... seq=1234567890 ack=0 win=65535
eth1:I[60]: 10.1.1.100 -> 10.2.1.200 (TCP) len=60 id=12345
  TCP: 54321 -> 443 .S.... seq=1234567890 ack=0 win=65535
eth1:o[60]: 10.1.1.100 -> 10.2.1.200 (TCP) len=60 id=12345
  TCP: 54321 -> 443 .S.... seq=1234567890 ack=0 win=65535
```

위 출력에서:
- `eth0:i` - eth0 인터페이스에서 들어온 패킷 (검사 전)
- `eth1:I` - 검사를 통과하여 eth1으로 나갈 준비가 된 패킷
- `eth1:o` - eth1에서 실제로 나가는 패킷

세 포지션 모두 보이므로 **방화벽은 정상 통과**한 것입니다.

---

## 면접관: "fw monitor에서 i는 보이는데 I가 안 보입니다. 방화벽이 Drop하는 건 확실한데, 어떤 Rule이 Drop하는지는 어떻게 알 수 있나요?"

### 방법 1: fw ctl zdebug drop

```bash
fw ctl zdebug drop
```

이 명령어는 **Drop된 패킷의 원인을 실시간으로 출력**합니다.

출력 예시:
```
@;sobFmjEqq;[cpu_1];[fw4_0];fw_log_drop_ex: Packet proto=6 10.1.1.100:54321 -> 10.2.1.200:443 dropped by fwpslglue_chain Reason: Access denied by rule 23;
```

- **Reason: Access denied by rule 23** - Rule 23이 차단한 것
- **Reason: No matching NAT rule** - NAT Rule 누락
- **Reason: Anti-Spoofing** - Anti-Spoofing 위반

### 방법 2: SmartConsole Log

SmartConsole의 Logs & Monitor에서 해당 시간대의 Drop 로그를 필터링합니다:

```
src:10.1.1.100 AND dst:10.2.1.200 AND action:Drop
```

로그의 "Rule Number"와 "Blade"(어떤 보안 기능이 차단했는지) 정보를 확인합니다.

### 방법 3: kernel debug (심화)

일반적인 방법으로 원인을 찾기 어려우면 커널 디버그를 사용합니다.

```bash
# 디버그 버퍼 설정
fw ctl debug 0
fw ctl debug -buf 32768

# 특정 모듈 디버그 활성화
fw ctl debug -m fw + drop
fw ctl debug -m fw + conn

# 디버그 출력
fw ctl kdebug -T -f > /var/log/fw_debug.txt

# ... 트래픽 재현 ...

# 디버그 종료
fw ctl debug 0
```

> kernel debug는 **대량의 출력을 생성**하므로, 반드시 특정 트래픽으로 필터링하거나 짧은 시간만 실행합니다. 운영 장비에서 장시간 실행하면 성능에 영향을 줄 수 있습니다.

### SecureXL 고려사항

Checkpoint의 SecureXL(하드웨어 가속)이 활성화되어 있으면, **가속된 트래픽은 fw monitor에 보이지 않을 수 있습니다.**

```bash
# SecureXL 상태 확인
fwaccel stat

# SecureXL에서의 Drop 확인
fwaccel conns -s | grep <IP>

# 디버그를 위해 일시적으로 SecureXL 비활성화
fwaccel off
# ... fw monitor로 캡처 ...
fwaccel on
```

> `fwaccel off`는 **모든 트래픽이 커널을 통과**하게 하므로 성능이 크게 저하됩니다. 운영 환경에서는 매우 짧은 시간만 사용하거나, 특정 연결만 가속에서 제외하는 것이 안전합니다.

```bash
# 특정 트래픽만 SecureXL에서 제외 (더 안전한 방법)
fwaccel dbg resetall
fwaccel dbg -m general + drop
```

---

## 면접관: "서버 측에서 tcpdump로 캡처한다면 어떤 필터를 사용하나요? 실무에서 자주 쓰는 필터 조합을 알려주세요."

### tcpdump 기본 문법과 주요 필터

```bash
# 기본 사용법
tcpdump -i <인터페이스> [필터] -w <파일명>
```

### 실무 필터 조합

**1. 특정 호스트 간 통신 전체 캡처**
```bash
tcpdump -i eth0 host 10.1.1.100 and host 10.2.1.200 -w /tmp/capture.pcap -c 1000
# -c 1000: 1000개 패킷 캡처 후 자동 종료 (디스크 보호)
```

**2. 특정 포트만 캡처**
```bash
tcpdump -i eth0 host 10.1.1.100 and port 443 -w /tmp/https.pcap
```

**3. SYN 패킷만 캡처 (연결 시도만 보기)**
```bash
tcpdump -i eth0 "tcp[tcpflags] & (tcp-syn) != 0" and host 10.1.1.100 -w /tmp/syn.pcap
```

**4. SYN + RST 캡처 (연결 실패 분석)**
```bash
tcpdump -i eth0 "tcp[tcpflags] & (tcp-syn|tcp-rst) != 0" and host 10.1.1.100 -w /tmp/syn-rst.pcap
```

**5. ICMP만 캡처 (Ping/Traceroute 분석)**
```bash
tcpdump -i eth0 icmp and host 10.1.1.100 -w /tmp/icmp.pcap
```

**6. 특정 서브넷 간 통신**
```bash
tcpdump -i eth0 net 10.1.0.0/16 and net 10.2.0.0/16 -w /tmp/inter-site.pcap
```

**7. DNS 쿼리만 캡처**
```bash
tcpdump -i eth0 port 53 and host 10.1.1.100 -w /tmp/dns.pcap
```

**8. 대용량 패킷만 캡처 (MTU 문제 분석)**
```bash
tcpdump -i eth0 greater 1400 and host 10.1.1.100 -w /tmp/large-packets.pcap
```

### 실무 옵션

```bash
# -nn: IP와 포트를 숫자로 표시 (DNS 역질의 하지 않아 빠름)
# -v / -vv / -vvv: 상세 출력 레벨
# -s 0: 패킷 전체 캡처 (기본값은 잘릴 수 있음)
# -G 60: 60초마다 파일 로테이션
# -W 10: 최대 10개 파일 보관 (순환)
# -Z root: 파일 권한 문제 방지

# 실무 권장 조합
tcpdump -i eth0 -nn -s 0 host 10.1.1.100 and port 443 \
  -w /tmp/capture_%Y%m%d_%H%M%S.pcap -G 300 -W 12 -c 100000
# 5분마다 파일 교체, 최대 12개 파일, 10만 패킷 제한
```

### 실시간 분석 (파일 저장 없이)

```bash
# TCP Handshake 성공/실패 실시간 확인
tcpdump -i eth0 -nn "tcp[tcpflags] & (tcp-syn|tcp-rst|tcp-fin) != 0" and host 10.1.1.100

# HTTP 요청 URL 확인 (비암호화 트래픽)
tcpdump -i eth0 -nn -A port 80 and host 10.1.1.100 | grep "GET\|POST\|Host:"
```

---

## 면접관: "방화벽이나 서버에서 직접 캡처하기 어려운 경우에는 어떻게 하나요? 네트워크 구간에서 캡처하는 방법은?"

네트워크 구간에서 패킷을 캡처하려면 **SPAN(Switch Port Analyzer)** 또는 **TAP(Test Access Point)**을 사용합니다.

### SPAN (Port Mirroring)

스위치에서 특정 포트의 트래픽을 다른 모니터링 포트로 복사하는 기능입니다.

```
! Cisco 스위치에서 SPAN 설정
monitor session 1 source interface GigabitEthernet0/1 both
! source: 모니터링 대상 포트, both: 양방향(in+out)

monitor session 1 destination interface GigabitEthernet0/24
! destination: 캡처 장비가 연결된 포트

! 특정 VLAN의 트래픽만 미러링
monitor session 1 source vlan 10 both
monitor session 1 destination interface GigabitEthernet0/24

! SPAN 세션 확인
show monitor session 1
```

출력 예시:
```
Session 1
---------
Type              : Local Session
Source Ports       :
    Both          : Gi0/1
Destination Ports  : Gi0/24
    Encapsulation : Native
          Ingress : Disabled
```

### RSPAN (Remote SPAN)

모니터링 대상과 캡처 장비가 **다른 스위치**에 있을 때 사용합니다.

```
! Source 스위치
vlan 999
 name RSPAN
 remote-span

monitor session 1 source interface GigabitEthernet0/1 both
monitor session 1 destination remote vlan 999

! Destination 스위치
vlan 999
 name RSPAN
 remote-span

monitor session 1 source remote vlan 999
monitor session 1 destination interface GigabitEthernet0/24
```

### ERSPAN (Encapsulated Remote SPAN)

L3 네트워크를 넘어서 미러링이 필요할 때 사용합니다. GRE 터널을 통해 캡처 트래픽을 전달합니다.

```
! Source 장비
monitor session 1 type erspan-source
 source interface GigabitEthernet0/1 both
 destination
  erspan-id 100
  ip address 10.1.1.200
  origin ip address 10.1.1.1
 no shutdown
```

### TAP (Test Access Point)

물리적으로 네트워크 케이블 중간에 설치하는 장비입니다. SPAN과 달리 **스위치 리소스를 사용하지 않고, 패킷 Drop이 없습니다.**

장점:
- 스위치 CPU/메모리에 영향 없음
- 100% 패킷 캡처 보장 (SPAN은 과부하 시 Drop 가능)
- Full-duplex 트래픽 완벽 캡처

단점:
- 별도 하드웨어 필요
- 설치 시 잠깐의 링크 단절 발생

### SPAN 사용 시 주의사항

| 주의 항목 | 설명 |
|----------|------|
| **과부하** | Source 포트 대역폭 > Destination 포트 대역폭이면 패킷 Drop 발생 |
| **CPU 영향** | 일부 스위치에서는 SPAN이 CPU 부하를 유발 |
| **SPAN 제한** | 스위치당 동시 SPAN 세션 수 제한 (보통 2~4개) |
| **보안** | Destination 포트에 연결된 장비가 네트워크의 모든 트래픽을 볼 수 있음 |
| **정리** | 캡처 완료 후 반드시 SPAN 세션 제거 |

```
! SPAN 세션 제거
no monitor session 1
```

---

## 면접관: "캡처한 pcap 파일을 분석할 때 어떤 점을 중점적으로 보나요? 간헐적 통신 실패의 패턴을 어떻게 찾나요?"

### Wireshark 분석 핵심 포인트

**1. TCP Handshake 분석 - 연결 자체가 되는지**

```
# Wireshark 필터
tcp.flags.syn == 1
```

- SYN만 보이고 SYN-ACK가 없으면: 서버 또는 중간 장비가 차단
- SYN-ACK 후 ACK가 없으면: 클라이언트 측 문제
- RST가 돌아오면: 서버가 연결을 거부 (포트가 닫혀있거나 서비스 문제)

**2. Retransmission 분석 - 간헐적 패킷 유실**

```
# Wireshark 필터
tcp.analysis.retransmission
tcp.analysis.fast_retransmission
tcp.analysis.duplicate_ack
```

Retransmission이 많으면 **네트워크 구간에서 패킷 유실**이 발생하고 있는 것입니다. 유실 위치를 찾으려면 양쪽에서 동시 캡처 후 비교합니다.

**3. TCP Window 분석 - 수신 측 처리 지연**

```
# Wireshark 필터
tcp.analysis.window_full
tcp.analysis.zero_window
```

Zero Window가 보이면 **수신 측 애플리케이션이 데이터를 제때 읽지 못해** TCP 버퍼가 꽉 찬 것입니다. 서버 애플리케이션 성능 문제를 의심합니다.

**4. RST 분석 - 비정상 연결 종료**

```
# Wireshark 필터
tcp.flags.reset == 1
```

RST의 출처와 타이밍을 분석합니다:
- 서버에서 RST: 애플리케이션 에러, 포트 미오픈
- 방화벽에서 RST: 세션 타임아웃 후 재전송, ACL 차단
- 클라이언트에서 RST: 타임아웃으로 연결 포기

**5. 시간 기반 분석 - 간헐적 장애 패턴**

```
# Wireshark: Statistics → I/O Graphs
# X축: 시간, Y축: 패킷 수 또는 Retransmission 수
# 특정 시간대에 Retransmission이 급증하면 그 시간에 네트워크 문제 발생
```

```
# Wireshark: Statistics → TCP Stream Graphs → Round Trip Time
# RTT가 갑자기 급증하는 구간이 있으면 해당 시간대에 지연 발생
```

### 간헐적 장애 분석 전략

1. **장시간 캡처** 후 문제 발생 시점을 로그와 대조합니다.
2. Wireshark의 **Expert Information** (Analyze → Expert Information)에서 Warning/Error를 먼저 확인합니다.
3. **양쪽 캡처 비교**: 클라이언트에서 보낸 패킷이 서버 쪽 캡처에 없으면, 그 사이 구간에서 Drop된 것입니다.
4. **TCP Stream 추적**: 실패한 연결의 TCP Stream을 Follow(우클릭 → Follow → TCP Stream)하여 전체 대화 흐름을 봅니다.

---

## 면접관: "패킷 캡처를 통한 트러블슈팅을 체계적으로 하려면 어떤 프로세스를 따라야 하나요?"

### 체계적인 캡처 프로세스

**Phase 1: 계획**

| 항목 | 내용 |
|------|------|
| 목적 | 무엇을 확인하려고 캡처하는지 명확히 정의 |
| 캡처 포인트 | 어디에서 캡처할지 (서버, 방화벽, 스위치 SPAN) |
| 필터 | 어떤 트래픽을 캡처할지 (Source/Destination/Port) |
| 시간 | 얼마나 오래 캡처할지 (디스크 용량 고려) |
| 재현 방법 | 장애를 어떻게 재현할지 |
| 비교 기준 | 정상 트래픽도 함께 캡처하여 비교 |

**Phase 2: 캡처**

```bash
# 양쪽 동시 캡처 시작 (시간 동기화 중요!)
# 서버 A
tcpdump -i eth0 -nn -s 0 host 10.2.1.200 -w /tmp/serverA.pcap &

# 방화벽 (ASA)
capture CAP-IN interface inside access-list CAP-FILTER
capture CAP-OUT interface outside access-list CAP-FILTER

# 서버 B
tcpdump -i eth0 -nn -s 0 host 10.1.1.100 -w /tmp/serverB.pcap &

# 트래픽 재현 (장애 발생 유도)
# 이 시점의 시각을 기록

# 캡처 종료
```

> **NTP로 시간 동기화**가 되어 있어야 양쪽 캡처의 타임스탬프를 비교할 수 있습니다.

**Phase 3: 분석**

1. 각 캡처 파일을 Wireshark로 열기
2. 장애 발생 시점으로 이동
3. Expert Information으로 이상 징후 확인
4. 구간별 비교 (Source에서 보낸 패킷이 Destination에 도착했는지)
5. Drop이 발생한 구간 특정
6. Drop 원인 분석 (ASP-drop, fw ctl zdebug drop 등)

**Phase 4: 문서화**

| 항목 | 기록 내용 |
|------|----------|
| 캡처 일시 | 시작/종료 시간 |
| 캡처 위치 | 어느 장비에서 캡처했는지 |
| 발견 사항 | 어떤 패킷이 Drop되었는지, 원인은 무엇인지 |
| 근거 | 캡처 파일명, 패킷 번호 |
| 조치 사항 | 어떤 설정을 변경했는지 |
| 결과 | 조치 후 정상 통신 확인 |

### 사전 준비 권장사항

1. **캡처 도구를 미리 설치/확인**: 장애 발생 후 tcpdump 설치부터 해야 하면 시간 낭비
2. **캡처 스크립트 준비**: 자주 사용하는 캡처 명령어를 스크립트로 만들어 두기
3. **디스크 공간 확보**: /tmp에 최소 1GB 이상 여유 공간 확인
4. **NTP 동기화 확인**: 양쪽 장비의 시간이 맞는지 사전 확인
5. **Wireshark Profile**: 프로토콜별 분석 Profile을 미리 만들어 두면 분석 속도가 빨라짐

```bash
# 캡처 전 디스크 확인
df -h /tmp

# NTP 동기화 확인
ntpq -p
# 또는
chronyc tracking
```

---
