# 05-05. AAA 기본 개념

> CCNA 200-301 보안 면접 시나리오 - 인증/인가/계정관리 프레임워크

---

### 면접관: "AAA가 무엇의 약자이고, 각각 어떤 역할을 하는지 설명해 주세요."

AAA는 **Authentication(인증)**, **Authorization(인가)**, **Accounting(계정관리)**의 약자로, 네트워크 접근 제어를 위한 보안 프레임워크입니다.

각 요소의 역할을 설명드리겠습니다.

**Authentication (인증) - "당신은 누구인가?"**
사용자의 신원을 확인하는 과정입니다. 사용자 이름과 비밀번호, 인증서, 토큰 등을 통해 접근을 시도하는 사용자가 실제로 주장하는 사람인지 검증합니다.

**Authorization (인가) - "무엇을 할 수 있는가?"**
인증된 사용자가 어떤 자원에 접근할 수 있고, 어떤 작업을 수행할 수 있는지 결정합니다. 예를 들어 일반 직원은 show 명령어만 사용 가능하고, 네트워크 관리자는 configure 명령어까지 사용할 수 있도록 권한을 구분합니다.

**Accounting (계정관리) - "무엇을 했는가?"**
인증된 사용자가 실제로 수행한 작업을 기록합니다. 누가 언제 접속했는지, 어떤 명령어를 실행했는지, 얼마나 오래 접속했는지 등을 로깅합니다. 보안 감사와 문제 추적에 필수적입니다.

쉬운 비유로 설명하면, 회사 건물 출입 시스템과 같습니다.

| AAA 요소 | 비유 | 네트워크 예시 |
|----------|------|-------------|
| Authentication | 사원증으로 본인 확인 | 라우터 로그인 시 ID/PW 확인 |
| Authorization | 출입 가능 구역 확인 | 사용 가능한 명령어 수준 결정 |
| Accounting | 출입 기록 저장 | 접속 시간, 실행 명령어 로깅 |

---

### 면접관: "AAA를 구현하는 방법에는 로컬 방식과 서버 방식이 있다고 들었습니다. 차이를 설명해 주세요."

AAA 구현 방법은 크게 **Local AAA**와 **Server-Based AAA** 두 가지로 나뉩니다.

**Local AAA (로컬 방식)**

각 네트워크 장비에 사용자 계정 정보를 직접 저장하는 방식입니다.

```
Router(config)# username admin privilege 15 secret AdminP@ss
Router(config)# username operator privilege 7 secret OperP@ss
Router(config)# username viewer privilege 1 secret ViewP@ss
```

**Server-Based AAA (서버 기반 방식)**

중앙 인증 서버(RADIUS 또는 TACACS+)에 사용자 정보를 집중 관리하는 방식입니다. 모든 네트워크 장비가 인증 요청을 중앙 서버로 전달합니다.

| 구분 | Local AAA | Server-Based AAA |
|------|-----------|-----------------|
| 사용자 DB 위치 | 각 장비에 분산 저장 | 중앙 서버에 집중 관리 |
| 확장성 | 낮음 (장비마다 설정) | 높음 (서버 한 곳에서 관리) |
| 관리 편의성 | 장비 수가 많으면 매우 번거로움 | 일괄 관리 가능 |
| 비밀번호 변경 | 모든 장비에서 개별 변경 | 서버에서 한 번만 변경 |
| 장애 시 | 해당 장비만 영향 | 서버 장애 시 전체 영향 |
| 적합한 환경 | 소규모 (장비 5대 이하) | 중대규모 (장비 다수) |
| Accounting 기능 | 제한적 | 상세한 로깅 가능 |

실무에서는 서버 기반 AAA를 주로 사용하면서, 서버 장애에 대비하여 로컬 계정을 백업용으로 함께 설정합니다. 서버가 응답하지 않을 때 로컬 계정으로 로그인할 수 있도록 하는 것입니다.

---

### 면접관: "RADIUS와 TACACS+를 비교해 주세요. 실무에서 언제 어떤 것을 사용하나요?"

RADIUS와 TACACS+는 모두 AAA 서버 프로토콜이지만 중요한 차이점이 있습니다.

| 구분 | RADIUS | TACACS+ |
|------|--------|---------|
| 개발 | IETF 개방형 표준 | Cisco 독자 개발 |
| 전송 프로토콜 | UDP (1812/1813) | TCP (49) |
| 암호화 범위 | 비밀번호만 암호화 | 전체 패킷 암호화 |
| AAA 결합 | 인증+인가 결합 | 인증/인가/계정관리 분리 |
| 명령어 인가 | 지원하지 않음 | 명령어 단위 인가 가능 |
| 주요 사용처 | 네트워크 접근 (802.1X, VPN) | 장비 관리 접근 |
| 멀티벤더 지원 | 우수 (표준 프로토콜) | 제한적 (주로 Cisco) |

사용 시나리오별로 정리하면 다음과 같습니다.

**RADIUS를 사용하는 경우:**
- 무선 네트워크 802.1X 인증
- VPN 사용자 인증
- ISP 환경에서 인터넷 사용자 인증
- 다양한 벤더의 장비가 혼재된 환경

**TACACS+를 사용하는 경우:**
- Cisco 장비 관리자 접근 제어
- 명령어 단위의 세밀한 인가가 필요한 경우 (예: 주니어 엔지니어는 show만, 시니어는 configure까지)
- 상세한 관리 활동 로깅이 필요한 경우

TACACS+에서 인증과 인가가 분리되어 있다는 것은 큰 장점입니다. 예를 들어 인증은 Active Directory로, 인가는 TACACS+ 서버의 정책으로, 계정관리는 별도 로그 서버로 각각 다르게 구성할 수 있습니다.

---

### 면접관: "Cisco 장비에서 AAA를 설정하는 기본 과정을 보여주세요."

Server-Based AAA 기본 설정을 단계별로 보여드리겠습니다. TACACS+ 서버를 예로 들겠습니다.

#### 핵심 설정

```
! 1단계: AAA 활성화 (이 명령어 실행 후 이전 인증 방식이 변경됨 - 주의!)
Router(config)# aaa new-model

! 2단계: TACACS+ 서버 지정
Router(config)# tacacs server MAIN-SERVER
Router(config-server-tacacs)# address ipv4 10.10.10.100
Router(config-server-tacacs)# key SecretKey123
Router(config-server-tacacs)# exit

! 3단계: 서버 그룹 설정 (선택사항이지만 권장)
Router(config)# aaa group server tacacs+ TACACS-GROUP
Router(config-sg-tacacs+)# server name MAIN-SERVER
Router(config-sg-tacacs+)# exit

! 4단계: 인증 방법 목록 설정
Router(config)# aaa authentication login default group TACACS-GROUP local

! 5단계: 인가 설정
Router(config)# aaa authorization exec default group TACACS-GROUP local

! 6단계: 계정관리 설정
Router(config)# aaa accounting exec default start-stop group TACACS-GROUP

! 백업용 로컬 계정 생성 (서버 장애 대비)
Router(config)# username admin privilege 15 secret BackupP@ss!
```

4단계의 `aaa authentication login default group TACACS-GROUP local` 명령어를 해석하면 다음과 같습니다.

- `default`: 기본 인증 방법 목록 (모든 로그인에 적용)
- `group TACACS-GROUP`: 첫 번째로 TACACS+ 서버에 인증 시도
- `local`: TACACS+ 서버가 응답하지 않을 때 로컬 데이터베이스로 인증

여기서 중요한 점은 `local`이 "서버가 거부했을 때"가 아니라 **"서버에 도달할 수 없을 때"**의 대체 수단이라는 것입니다. 서버가 인증을 거부하면 로컬로 넘어가지 않고 접근이 차단됩니다.

---

### 면접관: "aaa new-model 명령어를 실행할 때 특별히 주의해야 할 점이 있나요?"

`aaa new-model`은 AAA 설정의 첫 단계이지만, 매우 주의해야 하는 명령어입니다.

이 명령어를 실행하면 **기존의 모든 인증 방식이 AAA 기반으로 전환**됩니다. 만약 AAA 인증 방법을 제대로 설정하지 않은 상태에서 이 명령어를 실행하면, 장비에 로그인할 수 없게 될 위험이 있습니다.

주의사항을 정리하면 다음과 같습니다.

1. **콘솔로 작업**: 원격 접속이 아닌 물리적 콘솔 포트로 접속하여 설정해야 합니다
2. **로컬 계정 먼저 생성**: `aaa new-model` 전에 로컬 사용자 계정을 먼저 만들어 놓습니다
3. **단계별 테스트**: 한 번에 모든 설정을 하지 말고, 각 단계마다 접속을 테스트합니다
4. **설정 저장 시점 주의**: 정상 동작이 확인되기 전까지는 `copy run start`를 하지 않습니다

```
! 안전한 설정 순서

! 1. 먼저 로컬 계정 생성
Router(config)# username admin privilege 15 secret AdminP@ss

! 2. aaa new-model 활성화
Router(config)# aaa new-model

! 3. 즉시 로컬 인증 방법 설정
Router(config)# aaa authentication login default local

! 4. 콘솔에서 로그아웃 후 다시 로그인하여 테스트

! 5. 서버 기반 인증 추가
Router(config)# aaa authentication login default group tacacs+ local

! 6. 모든 테스트 완료 후 저장
Router# copy running-config startup-config
```

만약 실수로 잠겨버렸다면, 장비를 password recovery 모드로 재부팅하여 설정을 초기화해야 합니다. 이는 시간이 많이 걸리고 서비스 중단이 발생하므로 사전 계획이 매우 중요합니다.

---

### 면접관: "중앙 집중식 인증이 왜 중요한지, 실무 관점에서 설명해 주세요."

중앙 집중식 AAA가 중요한 이유를 실무 시나리오로 설명드리겠습니다.

**시나리오: 직원 퇴사 시 계정 처리**

로컬 AAA 환경(장비 50대)에서 직원이 퇴사하면, 관리자가 50대 장비에 일일이 접속하여 해당 계정을 삭제해야 합니다. 하나라도 빠뜨리면 보안 구멍이 됩니다.

서버 기반 AAA 환경에서는 중앙 서버에서 해당 계정을 비활성화하면 모든 장비에 즉시 반영됩니다.

```
로컬 방식:
  퇴사자 계정 삭제 → Router1에서 삭제
                   → Router2에서 삭제
                   → Router3에서 삭제
                   → ... (50대 반복)
                   → 빠뜨린 장비가 보안 위협!

서버 방식:
  퇴사자 계정 삭제 → AAA 서버에서 한 번만 삭제
                   → 모든 장비에 즉시 적용 완료!
```

중앙 집중식 AAA의 실무적 이점을 정리하면 다음과 같습니다.

| 항목 | 장점 |
|------|------|
| 계정 관리 | 한 곳에서 생성/삭제/변경 |
| 비밀번호 정책 | 복잡성, 만료 기간 등 일괄 적용 |
| 감사 로그 | 모든 장비의 접속 기록이 한 곳에 집중 |
| 권한 관리 | 역할별 권한 중앙 정의 (주니어/시니어/매니저) |
| 컴플라이언스 | 보안 감사 시 일관된 증적 제공 |
| Active Directory 연동 | 기존 사내 계정 시스템과 통합 가능 |

---

### 면접관: "AAA에서 인증 방법 목록(method list)의 동작 방식을 설명해 주세요."

인증 방법 목록은 사용자 인증 시 **어떤 순서로 어떤 인증 소스를 사용할지** 정의하는 것입니다.

```
Router(config)# aaa authentication login default group tacacs+ group radius local
```

위 설정은 다음과 같이 동작합니다.

```
사용자 로그인 시도
    │
    ▼
1순위: TACACS+ 서버에 인증 요청
    │
    ├─ 서버 응답: 인증 성공 → 로그인 허용
    ├─ 서버 응답: 인증 거부 → 로그인 차단 (다음 방법으로 넘어가지 않음!)
    └─ 서버 무응답 (도달 불가) → 2순위로 이동
         │
         ▼
    2순위: RADIUS 서버에 인증 요청
         │
         ├─ 서버 응답: 인증 성공 → 로그인 허용
         ├─ 서버 응답: 인증 거부 → 로그인 차단
         └─ 서버 무응답 → 3순위로 이동
              │
              ▼
         3순위: 로컬 데이터베이스 인증
              │
              ├─ 인증 성공 → 로그인 허용
              └─ 인증 실패 → 로그인 차단
```

핵심 포인트는 **"서버가 거부"하는 것과 "서버에 도달할 수 없는" 것은 다르게 처리**된다는 것입니다.

- **서버가 인증 거부**: 다음 방법으로 넘어가지 않고, 즉시 접근 차단
- **서버에 도달 불가**: 다음 방법으로 넘어감 (fallback)

이 차이를 이해하지 못하면, "서버에서 거부당한 사용자가 로컬 계정으로 로그인하는 보안 취약점"이 있다고 잘못 걱정하는 경우가 있는데, 실제로는 그런 일이 발생하지 않습니다.

또한 `default` 외에 이름을 지정하여 **Named Method List**를 만들 수 있습니다. 이를 사용하면 VTY 라인과 콘솔 라인에 서로 다른 인증 방법을 적용할 수 있습니다.

```
! VTY용 인증 방법 (서버 기반)
Router(config)# aaa authentication login VTY-AUTH group tacacs+ local

! 콘솔용 인증 방법 (로컬만)
Router(config)# aaa authentication login CONSOLE-AUTH local

! 각 라인에 적용
Router(config)# line vty 0 15
Router(config-line)# login authentication VTY-AUTH

Router(config)# line console 0
Router(config-line)# login authentication CONSOLE-AUTH
```

이렇게 하면 콘솔은 항상 로컬 인증을 사용하므로, AAA 서버 장애 시에도 물리적 콘솔로 장비에 접근할 수 있는 안전장치가 됩니다.
