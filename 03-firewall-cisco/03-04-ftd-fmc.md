# 03-04. FTD와 FMC 기본 구성

---

## 면접관: "고객사에서 기존 ASA 방화벽을 FTD(Firepower Threat Defense)로 마이그레이션하고 싶다고 합니다. ASA와 FTD의 근본적인 차이가 뭔가요? 고객한테 어떻게 설명하시겠어요?"

ASA와 FTD는 같은 Cisco 하드웨어(Firepower 1000/2100/4100/9300 시리즈 등)에서 동작할 수 있지만, 소프트웨어 아키텍처가 근본적으로 다릅니다.

### ASA vs FTD 핵심 차이

| 구분 | ASA | FTD |
|---|---|---|
| **아키텍처** | 독립 방화벽 엔진 | ASA 엔진(Lina) + **Snort IPS 엔진** 통합 |
| **관리 방식** | CLI / ASDM (GUI) | **FMC** (중앙 관리) 또는 FDM (로컬 관리) |
| **IPS/IDS** | 별도 모듈(SFR) 필요 | **내장** (Snort 엔진) |
| **Application Visibility** | 제한적 (NBAR 없음) | **완전한 L7 Application 식별** |
| **URL Filtering** | 제한적 | **카테고리 기반 URL 필터링** 내장 |
| **Malware 방어** | 없음 (AMP 모듈 별도) | **AMP for Networks** 내장 |
| **SSL Decryption** | 제한적 | **완전한 SSL/TLS Inspection** |
| **ACL 체계** | 전통적 ACL (L3/L4) | **ACP** (Application 기반 L7 포함) |
| **NAT** | CLI 기반 | FMC GUI 또는 FlexConfig |
| **설정 배포** | 즉시 반영 (CLI) | **FMC에서 정책 배포** (Deploy 필요) |

> 토폴로지 이미지 추가 예정

고객에게 설명할 때 핵심은: "ASA가 문(door)이라면, FTD는 **X-ray가 달린 보안 게이트**입니다. ASA는 포트 번호로만 허용/차단했지만, FTD는 패킷 내부의 애플리케이션까지 식별하고 악성코드, 침입 시도까지 탐지합니다."

---

## 면접관: "FMC와 FDM 중 어떤 걸 추천하시겠어요? 각각의 장단점을 설명해주세요."

고객사의 규모와 운영 환경에 따라 달라집니다.

### FMC (Firepower Management Center)

```
+------------------+
|       FMC        |  ← 중앙 관리 서버 (별도 VM 또는 어플라이언스)
|  (정책 관리/배포)  |
+--------+---------+
         |
    +----+----+----+
    |         |    |
  FTD-1    FTD-2  FTD-3   ← 여러 FTD 장비를 하나의 FMC로 관리
```

- **장점**: 다수의 FTD 중앙 관리, 상관 분석(Event Correlation), 대시보드, 리포팅, 자동 업데이트
- **단점**: 별도 서버 필요 (VM: 28GB RAM 이상 권장), 라이선스 비용, 초기 구축 복잡
- **적합 환경**: 중대규모 기업, 다수 사이트, SOC 운영

### FDM (Firepower Device Manager)

- **장점**: 별도 서버 불필요 (FTD 내장 웹 GUI), 간단한 설정, 소규모에 적합
- **단점**: FTD 1대만 관리 가능, 고급 기능 제한 (Custom IPS 정책 제한, 이벤트 분석 제한)
- **적합 환경**: 소규모 사무실, 단일 사이트, 예산 제한

### CDO (Cisco Defense Orchestrator)

최근에는 **CDO (클라우드 기반 관리)**도 옵션으로 있습니다. FMC의 기능을 클라우드에서 제공하여 온프레미스 서버 없이 다수의 FTD를 관리할 수 있습니다.

이 고객사가 본사 + 지사 3곳에 FTD를 운영한다면 **FMC를 권장**합니다. 단일 사이트에 FTD 1대라면 **FDM으로 시작**하고, 규모가 커지면 FMC로 전환하는 것도 방법입니다.

---

## 면접관: "FTD에서 Access Control Policy(ACP) Rule의 처리 순서를 설명해주세요. ASA ACL과는 어떻게 다른가요?"

FTD의 트래픽 처리 순서는 ASA보다 복잡합니다. 여러 단계의 정책이 순서대로 적용됩니다.

### FTD 트래픽 처리 순서 (Packet Flow)

```
패킷 도착
    │
    ▼
[1. Prefilter Policy]  ← 가장 먼저 처리 (Fastpath / Block / Analyze)
    │
    ▼
[2. SSL Policy]  ← SSL/TLS 복호화 여부 결정
    │
    ▼
[3. Identity Policy]  ← 사용자 인증 (AD 연동)
    │
    ▼
[4. Security Intelligence]  ← IP/URL/DNS 블랙리스트
    │
    ▼
[5. Access Control Policy (ACP)]  ← 메인 정책 (L3~L7)
    │   ├─ ACP Rule 1 (Trust / Allow / Block / Monitor)
    │   ├─ ACP Rule 2
    │   ├─ ...
    │   └─ Default Action
    │
    ▼
[6. Intrusion Policy (IPS)]  ← Allow된 트래픽에 대해 IPS 검사
    │
    ▼
[7. File / Malware Policy]  ← 파일 검사 (AMP)
    │
    ▼
[8. NAT]
    │
    ▼
패킷 전송
```

### ACP Rule 내부 처리

ACP Rule은 **위에서 아래로 (top-down)** 순서대로 매칭됩니다. 첫 번째 매칭되는 규칙이 적용됩니다.

각 Rule의 **Action** 유형:

| Action | 동작 | IPS 검사 | 파일 검사 |
|---|---|---|---|
| **Trust** | 허용 (추가 검사 없이 Fastpath) | X | X |
| **Allow** | 허용 (추가 검사 수행) | O | O |
| **Monitor** | 로그만 기록 (다음 규칙으로 계속) | - | - |
| **Block** | 즉시 차단 | X | X |
| **Block with reset** | 차단 + TCP RST 전송 | X | X |
| **Interactive Block** | 사용자에게 경고 페이지 표시 | X | X |

**ASA ACL과의 핵심 차이**:

1. ASA는 L3/L4(IP, Port)만 기준이지만, ACP Rule은 **Application, URL Category, User, GeoIP** 등 L7 조건까지 사용 가능
2. ASA ACL은 permit/deny 2가지이지만, ACP는 **Trust/Allow/Monitor/Block** 등 다양한 Action 제공
3. Allow와 Trust의 차이가 중요: **Trust는 Snort 엔진을 우회**하므로 성능이 좋지만 IPS 검사를 하지 않음

---

## 면접관: "FMC에서 정책을 배포했는데, 특정 트래픽이 예상과 다르게 차단됩니다. 어떻게 트러블슈팅하시겠어요?"

FTD에서의 트래픽 차단 원인을 찾기 위해 체계적으로 접근하겠습니다.

### 1단계: FMC의 Connection Events 확인

FMC GUI에서 **Analysis → Connections → Events**를 확인합니다. 여기서 해당 트래픽의:
- **Action**: Allow / Block / Trust
- **Reason**: 어떤 정책에 의해 차단되었는지
- **Rule**: 매칭된 ACP Rule 이름
- **Initiator/Responder IP**

```
! FTD CLI에서도 직접 확인 가능 (expert mode)
> system support firewall-engine-debug
```

### 2단계: FTD CLI에서 상세 확인

```
! FTD CLI 접속 (FMC에서 SSH 또는 콘솔)
> show access-control-config

! 현재 적용된 ACP 규칙 확인
> show access-list

! 연결 상태 확인 (ASA의 show conn과 동일)
> show conn address 10.1.1.100

! NAT 확인
> show nat
> show xlate

! packet-tracer (ASA와 동일하게 사용 가능)
> packet-tracer input inside tcp 192.168.10.100 12345 8.8.8.8 443 detailed
```

### 3단계: Snort 엔진 레벨 확인

FTD의 차단 원인이 ACP Rule이 아닌 **Snort IPS**에 의한 것일 수 있습니다.

```
! Snort 통계 확인
> show snort statistics

! Snort 인스턴스 상태
> show snort instances

! Intrusion Events 확인은 FMC GUI에서:
! Analysis → Intrusions → Events
```

### 4단계: Capture 활용

```
! FTD에서 패킷 캡처 (ASA와 동일 문법)
> capture CAP_TEST interface inside match tcp host 192.168.10.100 host 8.8.8.8 eq 443
> show capture CAP_TEST

! 캡처 파일 다운로드
> copy /pcap capture:CAP_TEST disk0:cap_test.pcap
```

### 흔한 차단 원인들

1. **ACP Default Action이 Block으로 설정**: 매칭되는 규칙이 없으면 기본 차단
2. **Security Intelligence에 의한 차단**: IP/URL이 Talos 블랙리스트에 포함
3. **Intrusion Policy 오탐**: Snort 규칙이 정상 트래픽을 공격으로 오인
4. **SSL Policy에 의한 차단**: 인증서 문제로 SSL 세션 차단
5. **Prefilter Policy**: Fastpath로 빠져야 할 트래픽이 Block에 매칭

---

## 면접관: "Snort 엔진이 트래픽을 두 번 처리한다는 이야기를 들었는데, 이게 무슨 뜻인가요? 성능에 문제가 되지 않나요?"

이것은 FTD의 **Dual-Engine Architecture**에 대한 질문입니다. FTD는 내부적으로 두 개의 엔진이 동작합니다.

### FTD 내부 아키텍처

```
                      FTD
+--------------------------------------------------+
|                                                    |
|  +-------------+          +-------------------+   |
|  | Lina Engine |  -----→  |   Snort Engine    |   |
|  | (ASA 코드)   |          |   (IPS/App ID)    |   |
|  |             |  ←-----  |                   |   |
|  | - L3/L4 ACL |          | - Application ID  |   |
|  | - NAT       |          | - IPS/IDS         |   |
|  | - Routing   |          | - URL Filtering   |   |
|  | - VPN       |          | - File/Malware    |   |
|  | - Failover  |          | - SSL Inspection  |   |
|  +-------------+          +-------------------+   |
|                                                    |
+--------------------------------------------------+
```

**패킷 처리 흐름**:

1. 패킷이 FTD에 도착 → **Lina 엔진**이 먼저 처리 (인터페이스, NAT, 기본 ACL)
2. Lina가 Snort로 패킷 전달 → **Snort 엔진**이 L7 검사 (Application ID, IPS, URL, Malware)
3. Snort의 판정(permit/deny) 결과가 Lina로 반환
4. Lina가 최종 전달 또는 차단

이 구조 때문에 **패킷이 Lina → Snort → Lina로 왕복**하면서 지연(latency)이 추가됩니다.

### 성능 최적화 방법

**1. Prefilter Policy 활용**

Snort 검사가 불필요한 트래픽(예: 백업 트래픽, 신뢰할 수 있는 내부 트래픽)은 **Prefilter Policy에서 Fastpath**로 설정하면 Snort를 완전히 우회합니다.

```
! Prefilter Policy의 Fastpath 규칙 (FMC GUI에서 설정)
! Policies → Access Control → Prefilter
! Rule: Backup_Traffic
!   Source: Backup_Server_Group
!   Destination: DR_Site
!   Action: Fastpath   ← Snort 우회, Lina에서만 처리
```

**2. ACP Rule에서 Trust Action 사용**

```
! ACP Rule Action을 Trust로 설정하면:
! - 첫 몇 개의 패킷만 Snort로 보내어 Application 식별
! - 식별 완료 후 나머지 패킷은 Snort 우회 (Fastpath)
! - IPS/Malware 검사 미수행
```

**Trust vs Fastpath 차이**:
- **Fastpath (Prefilter)**: 처음부터 Snort 완전 우회. Application도 식별 안 함
- **Trust (ACP)**: 초기 패킷으로 Application 식별 후 Snort 우회. 로그에 Application 정보 포함

**3. Snort 3.0 (FTD 7.0+)**

Snort 3.0은 멀티스레딩을 지원하여 Snort 2.0 대비 성능이 크게 향상되었습니다.

```
! FTD에서 Snort 버전 확인
> show snort3 status
```

---

## 면접관: "고객사가 FTD로 마이그레이션할 때, 기존 ASA 설정을 어떻게 이전하나요? 처음부터 다시 설정해야 하나요?"

전부 수동으로 재설정할 필요는 없습니다. Cisco에서 공식 마이그레이션 도구를 제공합니다.

### Firepower Migration Tool (FMT)

```
+------------------+        +---------+        +------------------+
|    기존 ASA       | ----→  |   FMT   | ----→  |    FMC / FTD     |
| (running-config) |        | (변환)   |        | (정책 import)     |
+------------------+        +---------+        +------------------+
```

**마이그레이션 가능한 항목**:
- ACL → ACP Rules로 변환
- NAT Rules
- Object/Object-Group
- Interface 설정
- Route
- Site-to-Site VPN (일부)

**수동 마이그레이션이 필요한 항목**:
- MPF (Service Policy) → ACP + Prefilter로 재설계 필요
- ASDM 기반 설정 → FMC GUI로 재구성
- Remote Access VPN (AnyConnect) → FTD RA-VPN으로 재설정
- Failover → FTD HA로 재구성

### 마이그레이션 단계

```
1. 기존 ASA 설정 백업
   > show running-config | redirect disk0:asa_backup.cfg

2. FMT (Firepower Migration Tool) 실행
   - ASA running-config 업로드
   - 변환 결과 검토 (매핑 리포트)
   - FMC로 정책 push

3. FTD 장비에 새 이미지 설치
   - ASA 이미지 제거 → FTD 이미지 설치
   - FMC에 FTD 등록

4. 정책 배포 및 검증
   - FMC에서 변환된 정책을 FTD에 배포
   - packet-tracer, 캡처로 트래픽 검증

5. 커팅오버
   - 유지보수 시간에 기존 ASA → FTD로 전환
   - 모니터링
```

### 주의사항

```
! FTD에서는 ASA CLI를 직접 사용할 수 없음 (일부 show 명령만 가능)
! FTD의 설정 변경은 반드시 FMC/FDM을 통해야 함
! 단, FlexConfig로 FMC GUI에서 지원하지 않는 ASA 명령을 직접 입력 가능

! FTD CLI에서 ASA Lina 엔진에 접근
> system support diagnostic-cli
firepower# show running-config   ← Lina 설정 확인 가능 (읽기 전용)
```

---

## 면접관: "마지막으로, FTD 운영 중 FMC와 FTD 간 통신이 끊어지면 어떻게 되나요? FTD가 동작을 멈추나요?"

아닙니다. **FMC와의 연결이 끊어져도 FTD는 마지막으로 배포된 정책대로 정상 동작합니다.**

### FMC-FTD 통신 구조

```
! FMC ↔ FTD 통신 포트
- TCP 8305: sftunnel (관리 터널, 정책 배포/이벤트 전송)
- 기타: HTTPS(443), SSH(22) 등 관리 접속

! FTD에서 FMC 연결 상태 확인
> show managers
! 출력: Host : 10.0.0.100
!        Registration : Completed
!        Connection : Established    ← 또는 Not Connected
```

### FMC 연결 끊김 시 영향

| 기능 | FMC 연결 끊김 시 |
|---|---|
| **트래픽 처리 (ACP, NAT 등)** | 정상 동작 (마지막 배포 정책) |
| **IPS/IDS** | 정상 동작 |
| **정책 변경/배포** | 불가 |
| **이벤트/로그 전송** | 로컬 버퍼에 저장 (재연결 시 전송) |
| **VDB/SRU 업데이트** | 불가 |
| **라이선스** | 90일 Grace Period |

### 비상 시 FTD 직접 관리

FMC가 완전히 사용 불가능한 상황에서 긴급하게 설정을 변경해야 할 때:

```
! FTD CLI에서 FDM(로컬 관리) 모드로 전환 가능 (FMC 등록 해제 필요)
> configure manager local

! 이후 FTD의 HTTPS(웹 GUI)로 FDM 접속하여 관리
! 단, FMC에서 관리하던 정책은 유실될 수 있으므로 최후의 수단
```

따라서 **FMC 이중화(HA)**를 구성하는 것이 중요합니다.

```
! FMC HA 구성
! - Active FMC: 정책 관리/배포 담당
! - Standby FMC: 실시간 동기화, Active 장애 시 자동 전환
! - 별도 HA 링크 필요 (전용 인터페이스 또는 관리 네트워크)
```

FMC HA는 FMC 웹 GUI에서 **System → Integration → High Availability**에서 설정합니다. Active FMC의 모든 설정, 정책, 이벤트 데이터가 Standby로 동기화됩니다.
