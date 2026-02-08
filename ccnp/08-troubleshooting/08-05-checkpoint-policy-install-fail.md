# 08-05. Checkpoint 정책 설치 실패

---

### 면접관: "고객사 보안 담당자가 'Checkpoint SmartConsole에서 정책을 Install 눌렀는데 실패했다'고 합니다. 어디부터 확인하시겠어요?"

정책 설치 실패는 **Checkpoint 환경에서 가장 빈번하게 발생하는 운영 이슈** 중 하나입니다. 먼저 정책 설치 과정을 이해하고, 어느 단계에서 실패했는지 파악하는 것이 핵심입니다.

**1단계: SmartConsole의 에러 메시지 확인**

Policy Install 시 SmartConsole 하단에 표시되는 에러 메시지를 먼저 읽습니다. 대부분의 경우 실패 원인에 대한 힌트가 포함되어 있습니다.

**2단계: Management Server 로그 확인**

```bash
# Expert 모드에서
tail -f /var/log/messages
tail -f $FWDIR/log/fwd.elg
```

**3단계: Gateway와의 연결 상태 확인**

```bash
# Management Server에서 Gateway로의 SIC 통신 확인
cprid_util -server <gateway_ip> -verbose rexec -rcmd /bin/fw getifs
```

**4단계: Gateway에서 현재 정책 상태 확인**

```bash
# Gateway의 Expert 모드에서
fw stat
```

출력 예시:
```
HOST           POLICY       DATE
localhost      Standard     25Jan2026 14:30:00
```

현재 어떤 정책이 설치되어 있는지 확인합니다. 마지막 성공 시점을 알 수 있습니다.

---

### 면접관: "Policy Install 과정을 단계별로 설명해 주세요. 어느 단계에서 실패할 수 있나요?"

Checkpoint Policy Install은 크게 **4단계**로 진행됩니다.

#### Policy Install 프로세스

| 단계 | 동작 | 실패 원인 |
|------|------|----------|
| **1. Verify** | Rule Base의 문법 및 논리적 오류 검증 | Object 충돌, Rule 충돌, 잘못된 참조 |
| **2. Compile** | Rule Base를 커널이 이해할 수 있는 Inspection Script(.pf)로 변환 | 복잡한 Rule로 인한 컴파일 에러, 메모리 부족 |
| **3. Transfer** | 컴파일된 정책을 Management Server에서 Gateway로 SIC 채널을 통해 전송 | SIC 통신 실패, 네트워크 문제, 대용량 정책으로 인한 전송 타임아웃 |
| **4. Install** | Gateway가 수신한 정책을 커널에 로드 | 커널 메모리 부족, 프로세스 이상, 기존 정책과의 충돌 |

각 단계에서 실패하면 SmartConsole에 해당 단계 이름과 함께 에러가 표시됩니다.

#### Verify 단계 확인

```bash
# Management Server에서 수동으로 Verify 실행
fwm verify $FWDIR/conf/Standard.W
```

#### Compile 확인

```bash
# 정책 수동 컴파일
fwm load -p $FWDIR/conf/Standard.W
# 에러가 있으면 상세 메시지 출력
```

#### Transfer 확인

SIC(Secure Internal Communication) 채널이 정상인지가 핵심입니다.

```bash
# Management에서 Gateway로 SIC 테스트
fw getifs <gateway_ip>
```

---

### 면접관: "SIC가 끊어졌다고 합니다. SIC란 정확히 뭐고, 어떻게 복구하나요?"

**SIC(Secure Internal Communication)**는 Checkpoint 구성 요소(Management Server, Gateway, Log Server 등) 간의 **신뢰 기반 암호화 통신 채널**입니다. 내부적으로 **SSL 인증서**를 사용하여 상호 인증합니다.

#### SIC 동작 원리

1. Management Server가 **Internal CA** 역할을 합니다.
2. Gateway 초기 설정 시 **SIC Activation Key**를 사용하여 Management에 인증서를 요청합니다.
3. 인증서가 발급되면 이후 모든 통신은 이 인증서로 암호화/인증됩니다.
4. SIC는 **TCP 18211** 포트를 사용합니다.

#### SIC 상태 확인

```bash
# Gateway에서
cpconfig
# 옵션에서 "Secure Internal Communication" 선택
# 현재 SIC 상태 확인

# 또는 직접 확인
cp_conf sic state
```

출력 예시:
```
SIC Status: Trust established
! 정상

SIC Status: Trust not established / Uninitialized
! SIC 끊어진 상태
```

#### SIC 끊어지는 원인

1. **인증서 만료 또는 손상**
2. **시간 동기화 문제** (Management와 Gateway 간 시간 차이가 크면 인증서 검증 실패)
3. **네트워크 문제** (TCP 18211 차단)
4. **Gateway에서 SIC Reset을 수행한 경우**

#### SIC 복구 절차

**Gateway에서:**
```bash
cpconfig
# "Secure Internal Communication" 선택
# Reset 선택 → 새 Activation Key 입력
# cpstop; cpstart
```

**SmartConsole(Management)에서:**
1. Gateway Object 더블클릭
2. Communication 버튼 클릭
3. "Reset" 클릭
4. Gateway에서 입력한 것과 동일한 Activation Key 입력
5. "Initialize" 클릭
6. Trust가 Established되면 정책 재설치

#### SIC 통신 디버그

```bash
# Gateway에서 SIC 관련 로그 확인
cpca_client lscert -stat
# 인증서 상태 확인

# TCP 18211 연결 테스트
# Management Server에서
curl -k https://<gateway_ip>:18211
# 연결 자체가 되는지 확인 (인증서 에러는 무시)
```

---

### 면접관: "SIC는 정상인데도 정책 설치가 실패합니다. Gateway의 프로세스 상태를 어떻게 확인하나요?"

Checkpoint Gateway의 핵심 프로세스가 정상 동작하지 않으면 정책 설치가 실패합니다.

#### 프로세스 상태 확인

```bash
cpwd_admin list
```

출력 예시:
```
APP             PID     STATUS
CPD             12345   E (Waiting)
FWD             12346   E (Waiting)
FWM             0       T (Terminated)
CPCA            12348   E (Waiting)
CPSEAD          12349   E (Waiting)
CPSCEPd         12350   E (Waiting)
SQLITE_MERGE    0       D (Disabled)
```

| 상태 | 의미 |
|------|------|
| **E (Waiting)** | 정상 - 실행 중이고 이벤트를 기다리는 상태 |
| **T (Terminated)** | 비정상 - 프로세스가 종료된 상태 |
| **D (Disabled)** | 비활성화 - 의도적으로 꺼진 상태 |

위 예시에서 **FWM이 Terminated** 상태입니다. FWM은 Management 데몬으로, 이것이 죽으면 정책 컴파일과 배포가 불가능합니다.

#### 주요 프로세스 역할

| 프로세스 | 역할 |
|---------|------|
| **FWD** | Firewall Daemon - 패킷 검사, 로깅 |
| **FWM** | Firewall Management - 정책 관리 (Management Server) |
| **CPD** | Checkpoint Daemon - SIC, 라이선스, 상태 관리 |
| **CPCA** | Internal Certificate Authority |
| **FW Worker** | 커널 모듈 - 실제 패킷 처리 |

#### 프로세스 복구

```bash
# 특정 프로세스 재시작
cpwd_admin start -name FWM -path $FWDIR/bin/fwm -command "fwm"

# 전체 Checkpoint 서비스 재시작
cpstop
cpstart

# 재시작 후 확인
cpwd_admin list
cpstat os -f all
```

#### 추가 진단

```bash
# 디스크 공간 확인 (로그 파일이 디스크를 꽉 채우면 프로세스가 죽을 수 있음)
df -h
du -sh $FWDIR/log/*

# 메모리 확인
free -m
cpstat os -f memory

# 코어 덤프 확인 (프로세스 크래시 시)
ls -la $CPDIR/log/dump/
ls -la /var/log/crash/
```

---

### 면접관: "프로세스는 정상인데 Verify 단계에서 'Object conflict' 에러가 납니다. 이건 뭔가요?"

Object 충돌은 **동일한 IP 주소나 네트워크가 여러 Object에 중복 정의**되어 있을 때 발생합니다. Checkpoint의 정책 컴파일러는 이런 모호성을 허용하지 않습니다.

#### 흔한 충돌 케이스

**1. 동일 IP가 서로 다른 Object에 정의됨:**
- Object "WebServer-OLD": 10.1.1.100
- Object "WebServer-NEW": 10.1.1.100
- 두 Object가 서로 다른 Rule에서 사용되면 충돌

**2. Network Object와 Host Object의 겹침:**
- Network "Internal-Net": 10.1.0.0/16
- Host "Server-A": 10.1.1.100
- 같은 Rule에서 함께 사용 시 충돌 가능

**3. Anti-Spoofing 설정과 NAT의 충돌:**
- Gateway 인터페이스의 Topology에 정의된 네트워크와 NAT Rule의 네트워크가 모순되는 경우

#### 충돌 찾기

SmartConsole에서:
1. **Menu → Global Properties → Policy → 에러 메시지에 나온 Object 이름 검색**
2. **Where Used** 기능으로 해당 Object가 어떤 Rule에서 사용되는지 확인

CLI에서:
```bash
# Object 데이터베이스에서 검색
dbedit -local
# dbedit 내에서
print <object_name>
# Object의 상세 속성 확인
quit

# 또는 최신 버전에서
mgmt_cli show host name "WebServer-OLD" --format json
mgmt_cli show hosts filter "10.1.1.100" --format json
```

#### Rule 충돌

Rule 충돌은 **두 Rule이 모순되는 동작을 지정**할 때 발생합니다. 예를 들어:
- Rule 5: Source=Any, Dest=WebServer, Service=HTTP → Accept
- Rule 3: Source=Any, Dest=Any, Service=Any → Drop

이 경우 Rule 3이 먼저 매칭되어 HTTP 트래픽도 Drop되므로 Rule 5가 무의미합니다. Checkpoint의 Verify는 이런 경우를 경고합니다.

해결:
```
# Rule 순서 재정렬 - 더 Specific한 Rule을 위로 이동
# SmartConsole에서 드래그 앤 드롭으로 순서 변경
```

#### Shadow Rule 검사

SmartConsole의 **"Rule Base Analysis"** 또는 **"Shadowed Rules"** 기능을 사용하면, 다른 Rule에 의해 가려져서 절대 매칭되지 않는 Rule을 자동으로 찾아줍니다.

---

### 면접관: "정책 설치가 실패해서 서비스에 영향이 있습니다. 긴급하게 이전 정책으로 롤백하려면 어떻게 하나요?"

#### 방법 1: fw fetch로 현재 정책 재설치

Gateway에서 Management로부터 마지막 성공 정책을 다시 가져올 수 있습니다.

```bash
# Gateway에서
fw fetch localhost
# 로컬에 캐시된 마지막 정책을 다시 로드
```

#### 방법 2: Database Revision Control

Checkpoint의 **Database Revision Control**은 정책 변경 이력을 관리하는 기능으로, 특정 시점의 정책으로 되돌릴 수 있습니다.

SmartConsole에서:
1. **Menu → Manage Policies → Database Revision Control**
2. Revision 목록에서 정상 동작하던 시점 선택
3. "Restore" 클릭
4. 복원된 정책을 Install

CLI에서:
```bash
# Revision 목록 확인
mgmt_cli show database-revision --format json

# 특정 Revision으로 복원
mgmt_cli restore-database-revision uid "<revision-uid>"
```

#### 방법 3: 수동 롤백 (최후 수단)

Gateway의 정책 파일을 직접 교체하는 방법입니다.

```bash
# Gateway에서 이전 정책 파일 확인
ls -la $FWDIR/state/__tmp/
ls -la $FWDIR/conf/

# 이전 정책 복원 (백업이 있는 경우)
cp $FWDIR/conf/Standard.W.bak $FWDIR/conf/Standard.W
fw fetch localhost
```

> 수동 롤백은 데이터 무결성 위험이 있으므로, 가능하면 Database Revision Control을 사용하는 것이 안전합니다.

#### 긴급 상황: 모든 트래픽이 차단된 경우

정책 설치 과정에서 Gateway의 기존 정책이 제거되었는데 새 정책 설치가 실패하면, **Default Policy(모든 트래픽 차단)**가 적용될 수 있습니다.

```bash
# 긴급 조치: 정책 언로드 (모든 트래픽 허용 - 보안 위험!)
fw unloadlocal
# 이후 즉시 정상 정책 재설치 필요

# 또는 Initial Policy 로드
fw fetch localhost
```

> `fw unloadlocal`은 모든 보안 정책을 해제하므로 **절대 운영 환경에서 장시간 유지하면 안 됩니다.** 임시 조치 후 즉시 정책을 재설치해야 합니다.

---

### 면접관: "정책 설치 실패를 사전에 방지하려면 어떻게 관리해야 하나요?"

#### Database Revision Control 운영 정책

```bash
# 정책 변경 전 반드시 Revision 생성
# SmartConsole: Menu → Database Revision Control → Create Revision
# 설명에 변경 내용, 날짜, 담당자 기록
```

운영 규칙:
- **모든 정책 변경 전에 Revision을 생성**합니다. (SmartConsole에서 자동 생성 설정 가능)
- Revision에 **변경 내용과 관련 티켓 번호를 기록**합니다.
- **최소 30일치 Revision을 보관**합니다.

#### 정책 설치 전 검증 절차

1. **Verify 먼저 실행**: Install 전에 SmartConsole의 "Verify" 버튼으로 문법 오류를 사전 확인합니다.
2. **Affected Gateway 확인**: 변경한 Rule이 어떤 Gateway에 영향을 주는지 확인합니다.
3. **비업무 시간 설치**: 가능하면 서비스 영향이 적은 시간에 설치합니다.

#### SIC 상태 주기적 확인

```bash
# 스크립트로 정기 점검
cprid_util -server <gateway_ip> -verbose rexec -rcmd /bin/fw getifs
# 정상이면 인터페이스 목록 출력, SIC 문제면 에러

# Gateway에서 인증서 만료일 확인
cpca_client lscert -stat
```

#### 디스크/메모리 관리

```bash
# 로그 로테이션 설정 확인
cpstat os -f disk
# 디스크 사용률이 80% 이상이면 정리 필요

# 오래된 로그 정리
fw logswitch
# 현재 로그 파일을 닫고 새 파일 시작
# 이전 로그는 SmartLog에서 관리
```

#### 정책 최적화

| 항목 | 권장 사항 |
|------|----------|
| Rule 수 | 불필요한 Rule 정리 (Disabled Rule 삭제) |
| Object 수 | 사용하지 않는 Object 정리 (`Where Used`로 확인) |
| Rule 순서 | 매칭 빈도가 높은 Rule을 상위에 배치 |
| Cleanup Rule | 마지막에 Log+Drop Rule 배치 (트러블슈팅 용이) |
| Inline Layer | 복잡한 Rule을 계층화하여 가독성 향상 |

#### SmartConsole 팁

```
# 변경 전 현재 정책 Export (백업)
# File → Export Policy
# JSON 형태로 백업되어 복원 가능

# 변경 이력 추적
# SmartConsole → Logs & Monitor → Audit Logs
# 누가 언제 어떤 변경을 했는지 추적
```

---

### 면접관: "정책 설치 후 특정 트래픽만 안 되는 경우, 새로 추가한 Rule 때문인지 어떻게 확인하나요?"

#### 1단계: SmartLog에서 트래픽 확인

SmartConsole의 Logs & Monitor 탭에서 해당 트래픽의 Source/Destination으로 필터링합니다.

```
# Log 필터 예시
src:10.1.1.50 AND dst:10.2.1.100 AND service:443
```

로그에 **Drop**이 찍히면, **Rule 번호**를 확인합니다. 새로 추가한 Rule 번호와 일치하면 해당 Rule이 원인입니다.

#### 2단계: fw ctl zdebug로 Drop 원인 확인

```bash
# Gateway에서
fw ctl zdebug drop | grep 10.1.1.50
```

출력 예시:
```
@;sobFmjEqq;[cpu_1];[fw4_0];fw_log_drop_ex: Packet proto=6 10.1.1.50:54321 -> 10.2.1.100:443 dropped by fwpslglue_chain Reason: Access denied by rule 15;
```

**Rule 15**가 Drop한 것이 확인되면, 해당 Rule을 검토합니다.

#### 3단계: Rule Hit Count 확인

SmartConsole에서 Rule 옆의 Hit Count 열을 확인하거나:

```bash
# CLI에서
fw tab -t connections -s
# 전체 연결 수 확인

# 특정 Rule의 Hit Count
cpstat fw -f policy | grep -A2 "Rule 15"
```

#### 4단계: 임시 조치와 영구 해결

```
# 임시: 문제 Rule을 Disable하고 정책 재설치
# SmartConsole에서 Rule 우클릭 → Disable → Install Policy

# 영구: Rule의 Source/Destination/Service/Action을 수정하여 의도대로 동작하게 변경
```

> 변경 작업 시 반드시 **Revision을 먼저 생성**하고, 문제가 해결되면 **해결 내용을 Revision에 기록**합니다.

---
