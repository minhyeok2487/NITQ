# 05-02. SSH 설정과 Telnet 차단

> CCNA 200-301 보안 면접 시나리오 - 원격 접속 보안과 SSH 구성

---

### 면접관: "네트워크 장비에 원격으로 접속할 때 Telnet 대신 SSH를 사용해야 하는 이유를 설명해 주세요."

Telnet과 SSH의 가장 큰 차이는 **암호화 여부**입니다.

Telnet은 모든 데이터를 평문(plaintext)으로 전송합니다. 즉, 사용자 이름, 비밀번호, 입력하는 모든 명령어가 네트워크 상에서 그대로 노출됩니다. Wireshark 같은 패킷 캡처 도구로 쉽게 내용을 확인할 수 있습니다.

반면 SSH(Secure Shell)는 공개키 암호화를 사용하여 모든 통신 내용을 암호화합니다. 패킷을 캡처하더라도 내용을 해독할 수 없습니다.

| 구분 | Telnet | SSH |
|------|--------|-----|
| 포트 번호 | TCP 23 | TCP 22 |
| 암호화 | 없음 (평문 전송) | 있음 (공개키 암호화) |
| 인증 방식 | 비밀번호만 | 비밀번호 + 공개키 인증 |
| 보안 수준 | 매우 취약 | 강력 |
| 실무 사용 | 테스트 환경에서만 제한적 | 표준 원격 접속 방식 |
| 버전 | - | SSHv1 (취약), SSHv2 (권장) |

보안 감사나 컴플라이언스 규정에서도 Telnet 사용은 금지되는 경우가 대부분입니다. 실무에서는 반드시 SSH를 사용해야 합니다.

---

### 면접관: "그렇다면 Cisco 라우터에서 SSH를 설정하는 전체 과정을 단계별로 설명해 주세요."

SSH 설정은 총 6단계로 진행됩니다. 순서가 중요합니다.

#### 핵심 설정

```
! 1단계: 호스트네임 설정 (기본 이름이면 SSH 키 생성 불가)
Router(config)# hostname R1

! 2단계: 도메인 이름 설정 (RSA 키 생성에 필요)
R1(config)# ip domain-name company.local

! 3단계: RSA 키 쌍 생성 (SSH 암호화에 사용)
R1(config)# crypto key generate rsa modulus 2048

! 4단계: SSH 버전 2 지정 (보안 강화)
R1(config)# ip ssh version 2

! 5단계: 로컬 사용자 계정 생성
R1(config)# username admin privilege 15 secret Str0ngP@ss!

! 6단계: VTY 라인 설정
R1(config)# line vty 0 4
R1(config-line)# transport input ssh
R1(config-line)# login local
R1(config-line)# exec-timeout 5 0
```

각 단계의 의미를 설명드리면, 1-2단계의 호스트네임과 도메인 이름은 RSA 키를 생성할 때 FQDN(Fully Qualified Domain Name)으로 조합됩니다. 기본 호스트네임인 "Router"로는 키 생성이 되지 않습니다.

3단계에서 RSA 키의 modulus 값은 최소 1024비트 이상이어야 SSHv2를 사용할 수 있고, 보안을 위해 2048비트를 권장합니다.

6단계에서 `transport input ssh`는 VTY 라인에서 SSH만 허용하고 Telnet을 차단하는 핵심 명령어입니다. `login local`은 로컬 데이터베이스의 사용자 계정으로 인증하라는 의미입니다.

---

### 면접관: "VTY 라인이 정확히 무엇이고, line vty 0 4에서 0과 4는 무슨 의미인가요?"

VTY는 Virtual TeletYpe의 약자로, 네트워크를 통한 원격 접속을 위한 가상 터미널 라인입니다. 물리적인 콘솔 포트와 달리, 네트워크 연결을 통해 장비에 접근할 때 사용됩니다.

`line vty 0 4`에서 0과 4는 VTY 라인의 범위를 의미합니다. 0번부터 4번까지 총 5개의 VTY 라인이 있으며, 이는 동시에 5명까지 원격 접속이 가능하다는 뜻입니다.

Cisco 장비에는 일반적으로 VTY 라인이 0-4 (5개) 또는 0-15 (16개)까지 존재합니다. 보안을 위해 모든 VTY 라인에 동일한 설정을 적용해야 합니다.

```
! 모든 VTY 라인에 동일한 보안 설정 적용
R1(config)# line vty 0 15
R1(config-line)# transport input ssh
R1(config-line)# login local
R1(config-line)# exec-timeout 5 0
R1(config-line)# logging synchronous
```

만약 line vty 0 4에만 SSH를 설정하고 line vty 5 15에는 설정하지 않으면, 5번 이후 라인으로 Telnet 접속이 가능한 보안 구멍이 생길 수 있습니다.

---

### 면접관: "SSH 접속 관련 추가 보안 설정에는 어떤 것들이 있나요?"

SSH 기본 설정 외에 추가할 수 있는 보안 강화 설정은 다음과 같습니다.

#### 핵심 설정

```
! SSH 타임아웃 설정 (인증 대기 시간 60초)
R1(config)# ip ssh time-out 60

! SSH 인증 재시도 횟수 제한 (3회)
R1(config)# ip ssh authentication-retries 3

! VTY 접근 제어 (특정 IP에서만 SSH 허용)
R1(config)# access-list 10 permit 10.10.10.0 0.0.0.255
R1(config)# line vty 0 15
R1(config-line)# access-class 10 in

! 비밀번호 최소 길이 설정
R1(config)# security passwords min-length 8

! 로그인 실패 시 차단 (60초 내 3회 실패 시 120초 차단)
R1(config)# login block-for 120 attempts 3 within 60

! 실행 타임아웃 (5분 동안 입력 없으면 자동 로그아웃)
R1(config-line)# exec-timeout 5 0
```

특히 `access-class` 명령어는 매우 중요합니다. ACL과 결합하여 특정 관리자 네트워크에서만 SSH 접속을 허용함으로써, 외부에서의 무단 접속 시도를 원천 차단할 수 있습니다.

`login block-for` 명령어는 무차별 대입 공격(brute-force attack)을 방어하는 데 효과적입니다. 일정 횟수 이상 로그인에 실패하면 해당 소스를 일정 시간 차단합니다.

---

### 면접관: "enable password와 enable secret의 차이는 무엇이고, 왜 구분이 필요한가요?"

두 명령어 모두 특권 모드(privileged EXEC mode) 진입 시 비밀번호를 설정하지만, 보안 수준이 다릅니다.

| 구분 | enable password | enable secret |
|------|----------------|---------------|
| 암호화 | 평문 저장 (type 0) | MD5 해시 저장 (type 5) 또는 SHA-256 (type 9) |
| 보안 수준 | 매우 취약 | 강력 |
| show running-config | 비밀번호가 그대로 보임 | 해시값만 보임 |
| 우선순위 | 낮음 | 높음 (둘 다 설정 시 secret 사용) |

```
! 나쁜 예시 - 평문 비밀번호
R1(config)# enable password cisco123

! show running-config 결과:
! enable password cisco123    ← 비밀번호 노출!

! 좋은 예시 - 암호화된 비밀번호
R1(config)# enable secret Str0ngP@ss!

! show running-config 결과:
! enable secret 5 $1$mERr$hx5rVt7rPNoS4wqbXKX7m0    ← 해시값만 표시
```

추가로 `service password-encryption` 명령어를 사용하면 설정 파일에 있는 평문 비밀번호를 type 7으로 암호화합니다. 하지만 type 7 암호화는 쉽게 복호화할 수 있어 보안이 취약합니다. 따라서 가능하면 항상 `secret` 키워드를 사용하는 것이 좋습니다.

```
R1(config)# service password-encryption
```

---

### 면접관: "현재 SSH 접속 상태와 설정을 확인하는 명령어들을 알려주세요."

SSH 관련 주요 확인 명령어는 다음과 같습니다.

#### 핵심 명령어

```
R1# show ssh
  → 현재 활성 SSH 세션 목록 확인
  → 연결된 사용자, 버전, 암호화 방식 등 표시

R1# show ip ssh
  → SSH 서버 설정 상태 확인
  → SSH 버전, 타임아웃, 인증 재시도 횟수 등 표시

  출력 예시:
  SSH Enabled - version 2.0
  Authentication timeout: 60 secs; Authentication retries: 3

R1# show users
  → 현재 장비에 접속한 모든 사용자 확인
  → 콘솔, VTY 라인별 접속 사용자 표시

R1# show line vty 0 4
  → VTY 라인 상태 확인

R1# show running-config | section line vty
  → VTY 라인 관련 설정만 필터링하여 확인

R1# show crypto key mypubkey rsa
  → 생성된 RSA 키 정보 확인
```

트러블슈팅 시 자주 확인하는 사항으로는 RSA 키가 정상 생성되었는지, SSH 버전이 2로 설정되었는지, VTY 라인에 transport input ssh가 적용되었는지, login local이 설정되었는지, 그리고 사용자 계정이 생성되었는지 등이 있습니다.

---

### 면접관: "실무에서 기존 Telnet을 SSH로 전환할 때 주의해야 할 점은 무엇인가요?"

Telnet에서 SSH로 전환할 때 가장 주의해야 할 점은 **자기 자신의 접속이 끊기지 않도록** 하는 것입니다.

단계별 주의사항을 설명드리겠습니다.

1. **콘솔 접속으로 작업**: 원격 Telnet 세션이 아닌 물리적 콘솔 포트로 접속하여 설정을 변경해야 합니다. SSH 설정 중 오류가 발생하면 원격 접속이 모두 차단될 수 있기 때문입니다.

2. **SSH 설정 완료 후 Telnet 차단**: SSH 접속을 먼저 테스트하고, 정상 동작을 확인한 후에 Telnet을 차단해야 합니다.

3. **기존 사용자에게 사전 공지**: SSH 클라이언트 프로그램(PuTTY 등)을 준비하도록 안내합니다.

4. **설정 저장**: 모든 변경 후 반드시 `copy running-config startup-config`으로 저장합니다.

```
! 전환 작업 순서 예시
! 1. 콘솔로 접속
! 2. SSH 전체 설정 완료
! 3. transport input ssh telnet (임시로 둘 다 허용)
! 4. SSH 접속 테스트 성공 확인
! 5. transport input ssh (Telnet 완전 차단)
! 6. copy running-config startup-config
```

만약 원격에서 작업할 수밖에 없는 상황이라면, 일시적으로 `transport input ssh telnet`으로 설정하여 SSH와 Telnet을 모두 허용한 상태에서 SSH 테스트를 진행하고, 확인 후에 Telnet을 차단하는 방법이 안전합니다.

또한 `reload in 10` 명령어로 10분 후 자동 재부팅을 예약해 놓으면, 설정 오류로 접속이 끊겨도 장비가 재부팅되면서 이전 설정으로 돌아갈 수 있는 안전장치가 됩니다.
