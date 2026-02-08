# 07-03. IPS / IDS 구성과 탐지

---

## 면접관: "고객사에서 최근에 내부 서버가 랜섬웨어에 감염되는 침해사고가 발생했습니다. 방화벽은 있는데 탐지나 차단이 안 됐다고 하네요. 네트워크 레벨에서 악성 트래픽을 탐지하고 차단할 수 있는 솔루션을 설계해달라고 합니다. 어떻게 접근하시겠어요?"

기존 방화벽은 L3/L4 기반 ACL 위주로 동작하기 때문에, 정상 포트(HTTP 80, HTTPS 443)를 통한 악성 트래픽은 탐지하지 못합니다. 이를 보완하기 위해 **IPS(Intrusion Prevention System)를 Inline 모드로 배치**하겠습니다.

전체 설계 방향은 다음과 같습니다.

- **배치 위치**: 인터넷 경계(방화벽 뒤) + 내부 주요 세그먼트 사이 (East-West 트래픽)
- **모드**: Inline (차단 가능) - 탐지만 하는 IDS로는 이미 발생한 사고를 반복할 수 있음
- **솔루션**: Cisco Firepower(Snort 기반), Palo Alto Threat Prevention, 또는 전용 IPS 어플라이언스
- **Signature 업데이트**: 자동 업데이트 + 정기 튜닝
- **고가용성**: HA 구성 또는 Bypass Module 장착 (장애 시 트래픽 통과)

> 토폴로지 이미지 추가 예정

### 핵심 배치 구성

```
[Internet]
    |
[Router] ──── (미러링) ──→ [IDS Sensor] (모니터링 전용)
    |
[Firewall]
    |
[IPS - Inline] ← 핵심 차단 포인트
    |
[Core Switch]
   / \
  /   \
[Server Farm]  [User Segment]
```

### Cisco Firepower IPS (Snort 기반) 설정 개념

```
! ---- Firepower Threat Defense (FTD) - FMC에서 관리 ----
!
! 1. Inline Interface Pair 구성
!    Interface: GigabitEthernet1/1 (Outside) ←→ GigabitEthernet1/2 (Inside)
!    Mode: Inline Pair (투명하게 트래픽 통과)
!
! 2. Intrusion Policy 생성
!    Base Policy: "Balanced Security and Connectivity"
!    → 보안과 성능의 균형 (초기 권장)
!
! 3. Access Control Policy에 IPS Policy 연결
!    Rule: Allow + Intrusion Policy 적용
!    → 허용된 트래픽에 대해서도 IPS 검사 수행
!
! ---- Snort Rule 예시 (커스텀 시그니처) ----
! alert tcp $EXTERNAL_NET any -> $HOME_NET 445 (
!   msg:"Ransomware SMB Exploit Attempt";
!   flow:to_server,established;
!   content:"|FF|SMB";
!   content:"|00 00 00 00|";
!   sid:1000001; rev:1;
! )
!
! ---- CLI에서 IPS 상태 확인 (FTD SSH) ----
> show asp inspect-dp snort
> show snort statistics
> system support diagnostic-cli
# show service-policy
```

---

## 면접관: "IDS와 IPS가 각각 뭔지, 차이점을 명확히 설명해주세요. 그리고 왜 IPS를 선택했나요?"

IDS와 IPS는 모두 네트워크 침입을 탐지하지만, **대응 방식**에서 근본적인 차이가 있습니다.

| 항목 | IDS (Intrusion Detection System) | IPS (Intrusion Prevention System) |
|------|----------------------------------|-----------------------------------|
| **배치 모드** | Out-of-Band (미러링/SPAN) | Inline (트래픽 경로 상) |
| **동작** | 탐지 + 알림 (Passive) | 탐지 + 차단 (Active) |
| **트래픽 영향** | 없음 (복사본 분석) | 있음 (지연 발생 가능) |
| **장애 영향** | 없음 (탐지만 중단) | 있음 (트래픽 단절 가능) |
| **대응 속도** | 느림 (알림 후 수동 대응) | 즉각 (실시간 차단) |
| **오탐 위험** | 낮음 (차단 안 하므로) | 높음 (정상 트래픽 차단 가능) |

```
[IDS 배치 - SPAN/Mirror]
                                    ┌──── [IDS Sensor]
                                    │     (분석만)
[Router] ── [Switch] ──── [Server]
              │ SPAN Port
              └──── (복사 트래픽)


[IPS 배치 - Inline]
[Router] ── [IPS] ── [Switch] ── [Server]
            (통과하며 검사/차단)
```

IPS를 선택한 이유는 고객사가 이미 침해사고를 경험했기 때문입니다. IDS는 탐지만 하고 관리자에게 알림을 보내는데, 대응이 늦으면 피해가 확산됩니다. 특히 랜섬웨어처럼 전파 속도가 빠른 위협은 실시간 차단이 필수입니다.

다만 IPS의 리스크도 관리해야 합니다.
- **성능**: Inline이므로 처리 지연(Latency)이 발생할 수 있음 → 트래픽 볼륨에 맞는 성능의 어플라이언스 선정
- **가용성**: IPS 장비 장애 시 트래픽 단절 → Fail-Open 모드 또는 HA 구성
- **오탐**: False Positive로 정상 서비스 차단 가능 → 초기에는 탐지 모드(IDS 모드)로 운영 후 튜닝을 거쳐 차단 모드 전환

---

## 면접관: "그런데 IPS를 배치한 지 일주일 됐는데, False Positive가 너무 많이 발생합니다. 정상적인 업무 트래픽이 자꾸 차단돼서 사용자 불만이 쏟아지고 있어요. 어떻게 하시겠어요?"

False Positive 대량 발생은 IPS 초기 도입에서 가장 흔한 문제입니다. 체계적인 **Signature 튜닝** 프로세스가 필요합니다.

**1단계: 즉각 대응 - 차단 모드에서 탐지 모드로 전환**

가장 먼저 영향받는 정책을 차단(Drop)에서 탐지(Alert)로 변경하여 업무 영향을 제거합니다.

```
! Firepower: FMC에서 Intrusion Policy 수정
! 오탐 발생 시그니처의 Action을 Drop → Alert로 변경
! 또는 전체 Policy를 "Detection" 모드로 전환
!
! Snort Rule Action 변경 예시
! drop → alert (차단 → 탐지만)
! alert tcp $EXTERNAL_NET any -> $HOME_NET 80 (
!   msg:"HTTP SQL Injection Attempt";
!   ...
!   sid:31423; rev:5;
! )
```

**2단계: 오탐 원인 분석**

```
! IPS 이벤트 로그 분석 (FMC Dashboard)
! 1. 가장 많이 발생하는 시그니처 Top 10 확인
! 2. 해당 시그니처의 Source/Destination 패턴 확인
! 3. 실제 트래픽 캡처(PCAP)와 시그니처 매칭 내용 대조
!
! FTD CLI에서 캡처
> capture CAPIN interface inside match ip host 10.10.1.100 any
> show capture CAPIN
```

**일반적인 오탐 원인:**

- **시그니처가 너무 광범위**: 정상 HTTP 트래픽의 특정 패턴이 SQL Injection 시그니처에 매칭
- **내부 스캐너/모니터링 도구**: 취약점 스캐너(Nessus, Qualys)가 IPS에 탐지됨
- **업무 애플리케이션 특성**: 커스텀 앱의 특이한 프로토콜 동작이 Anomaly로 탐지

**3단계: 시그니처 튜닝**

```
! 방법 1: 특정 시그니처 비활성화
! FMC > Intrusion Policy > Rules > SID 31423 > Disable

! 방법 2: 시그니처를 유지하되 특정 IP/네트워크 제외
! Variable Set에서 제외 대상 정의
! $HOME_NET에서 스캐너 IP 제외, 또는 Suppression 설정
! FMC > Intrusion Policy > Rules > SID 31423 > Suppression
!   Source: 10.10.50.10 (취약점 스캐너)

! 방법 3: Threshold 설정 (N회 이상일 때만 이벤트 생성)
! FMC > Intrusion Policy > Rules > SID 31423 > Threshold
!   Type: Limit, Count: 10, Seconds: 60
!   → 60초 내 10회 이상 발생할 때만 이벤트 생성
```

**4단계: 단계적 차단 전환**

2~4주간 탐지 모드로 운영하면서 시그니처별 오탐률을 분석하고, 오탐이 없는 시그니처부터 단계적으로 차단 모드로 전환합니다. 이 과정을 **"Burn-in Period"** 라고 합니다.

---

## 면접관: "IPS 장비 자체가 장애가 나면 어떻게 되나요? Fail-Open과 Fail-Close의 차이를 설명해주세요."

IPS는 Inline 배치이므로 장비 장애가 곧 트래픽 단절로 이어질 수 있습니다. 이때 두 가지 동작 모드가 있습니다.

**Fail-Open (Bypass)**
- IPS 장애 시 트래픽을 **검사 없이 그대로 통과**시킴
- 네트워크 가용성 우선
- **보안은 일시적으로 해제**되지만 서비스는 유지
- 대부분의 기업 환경에서 선택

**Fail-Close (Block)**
- IPS 장애 시 트래픽을 **차단**함
- 보안 우선
- 장애 시 네트워크 전체 다운
- 금융기관, 군사 네트워크 등 극도의 보안이 필요한 환경

```
! ---- Hardware Bypass Module 설정 (Firepower) ----
! FTD에서 Hardware Bypass 지원 인터페이스의 경우:
!
! FMC > Devices > Interface > Inline Set
!   - Bypass Mode: Enabled (Fail-Open)
!   - Tap Mode: Disabled
!
! CLI에서 Bypass 상태 확인
> show inline-set
!
! 출력 예시:
! Inline-set 1:
!   Bypass Mode: Armed (Bypass 준비 상태)
!   Bypass State: Normal (정상 동작 중)
!   Hardware Bypass: Supported
!
! 장애 시:
!   Bypass State: Bypass (검사 없이 통과 중)
```

**실무 권장 구성:**

```
[Internet]
     |
[Firewall]
     |
[IPS-1] ──── [IPS-2]    ← Active/Standby HA
     |
[Core Switch]

! HA 구성이면 IPS-1 장애 시 IPS-2가 인계
! 두 대 모두 장애 시에만 Fail-Open 동작
! → 이중 안전장치
```

실무에서는 **HA + Fail-Open**을 조합합니다. 1차로 HA Failover, 2차로 Hardware Bypass. 이렇게 하면 두 대가 동시에 죽는 극단적 상황에서도 서비스는 유지됩니다. 물론 이 경우 보안 모니터링은 일시적으로 중단되므로, SNMP/Syslog로 IPS 상태를 실시간 모니터링해서 빠른 복구가 가능하도록 체계를 갖춰야 합니다.

---

## 면접관: "탐지 방식에 대해 좀 더 깊이 들어가볼게요. Signature 기반 탐지와 Anomaly 기반 탐지의 차이는 뭔가요?"

두 가지는 IPS/IDS의 핵심 탐지 엔진입니다.

**1. Signature-Based Detection (시그니처 기반)**

알려진 공격 패턴을 데이터베이스로 보유하고, 트래픽과 패턴 매칭하여 탐지합니다.

```
! Snort 시그니처 예시
alert tcp $EXTERNAL_NET any -> $HOME_NET 445 (
  msg:"ET EXPLOIT EternalBlue SMB Remote Code Execution";
  flow:to_server,established;
  content:"|FF|SMB|73|";               ! SMB Session Setup
  content:"|00 00 00 00|";
  byte_test:4,>,65535,0,relative;      ! Buffer overflow 조건
  reference:cve,2017-0144;
  classtype:attempted-admin;
  sid:2024218; rev:3;
)
```

- **장점**: 알려진 공격에 대해 정확한 탐지, 낮은 오탐률
- **단점**: Zero-day 공격 탐지 불가, 시그니처 DB를 지속 업데이트해야 함
- **비유**: 지명수배 사진 대조 → 사진에 없는 범죄자는 못 잡음

**2. Anomaly-Based Detection (이상 행위 기반)**

정상 트래픽의 Baseline(기준선)을 학습하고, 이를 벗어나는 행위를 탐지합니다.

- **Protocol Anomaly**: RFC 규격에 어긋나는 패킷 (비정상 TCP 플래그 조합 등)
- **Statistical Anomaly**: 평소 대비 트래픽 급증, 비정상 포트 사용량 증가
- **Behavioral Anomaly**: 특정 호스트가 갑자기 다수 IP에 스캔 시도

- **장점**: Zero-day 공격, 새로운 공격 패턴 탐지 가능
- **단점**: 높은 오탐률, Baseline 학습 기간 필요 (보통 2~4주)
- **비유**: 평소와 다른 행동을 하면 의심 → 출장 간 직원도 잡힐 수 있음

**실무에서는 두 가지를 조합(Hybrid)**해서 사용합니다. Cisco Firepower는 Snort(Signature) + Network Analysis Policy(Anomaly)를 함께 적용할 수 있고, Palo Alto는 Threat Prevention(Signature) + WildFire(Sandboxing, Unknown Malware)를 조합합니다.

---

## 면접관: "고객사가 현재 Check Point 방화벽을 사용하고 있는데, Check Point에도 IPS 기능(IPS Blade)이 있다고 합니다. 별도 IPS 어플라이언스를 두는 것과 방화벽 통합 IPS를 쓰는 것, 각각의 장단점은 뭔가요?"

현실적으로 많이 비교하게 되는 구성입니다.

| 항목 | 전용 IPS 어플라이언스 | 방화벽 통합 IPS (NGFW) |
|------|---------------------|----------------------|
| **성능** | IPS 전용 하드웨어, 높은 처리량 | 방화벽과 리소스 공유, IPS 활성화 시 성능 30~50% 저하 |
| **배치 유연성** | 원하는 구간에 자유 배치 | 방화벽 위치에 종속 |
| **관리 포인트** | 추가 장비 관리 부담 | 단일 콘솔(SmartConsole)에서 통합 관리 |
| **비용** | 높음 (추가 장비 구매) | 낮음 (기존 장비 라이선스 활성화) |
| **East-West 트래픽** | 내부 구간에도 배치 가능 | 보통 North-South에만 적용 |
| **시그니처 품질** | 전문 IPS 벤더의 방대한 DB | NGFW 벤더의 통합 DB (전문성 차이 가능) |

**Check Point IPS Blade 활성화:**

```
# Check Point SmartConsole에서:
# 1. Gateway Properties > Network Security > IPS Blade: Enable
# 2. IPS Policy:
#    - Profile: Recommended (기본 권장 시그니처 세트)
#    - Activation Mode: Prevent (차단) 또는 Detect (탐지)
# 3. Threat Prevention Policy에서 IPS Profile 적용
#
# CLI (Expert Mode):
[Expert@CP-GW]# cpstat fw -f inspect
[Expert@CP-GW]# fw ctl zdebug drop | grep -i ips
[Expert@CP-GW]# cpstat threat-prevention
```

**실무 추천:**

- **중소기업 (트래픽 1Gbps 이하)**: Check Point IPS Blade 같은 NGFW 통합 IPS로 충분합니다. 관리 포인트를 줄이는 것이 운영 효율에 더 도움이 됩니다.
- **대기업/데이터센터 (트래픽 10Gbps 이상)**: 전용 IPS 어플라이언스를 별도 배치합니다. NGFW에 IPS까지 활성화하면 성능 저하가 심각하고, 내부 세그먼트 간 East-West 트래픽 검사도 필요합니다.
- **하이브리드**: 인터넷 경계는 NGFW 통합 IPS, 내부 핵심 구간(서버팜 앞)에는 전용 IPS 배치. 비용과 보안의 균형을 맞출 수 있습니다.

이번 고객사는 이미 침해사고를 경험했으므로, 최소한 서버팜 앞에는 전용 IPS를 배치하고, 인터넷 경계는 기존 Check Point IPS Blade를 활성화하는 하이브리드 구성을 추천하겠습니다.

---

## 면접관: "마지막으로, IPS 로그를 어떻게 관리하고 활용하시겠어요?"

IPS 로그는 보안 사고 분석과 컴플라이언스의 핵심 데이터입니다.

**로그 아키텍처:**

```
[IPS/IDS]
    |
    ├── Syslog ──→ [SIEM] (Splunk, QRadar, Elastic SIEM)
    |               ├── 실시간 상관 분석 (Correlation)
    |               ├── 대시보드 / 알림
    |               └── 장기 보관 (1년 이상, 컴플라이언스)
    |
    ├── SNMP ──→ [NMS] (Nagios, Zabbix)
    |             └── IPS 장비 Health 모니터링 (CPU, Memory, Bypass 상태)
    |
    └── PCAP ──→ [Packet Broker / Storage]
                  └── 풀 패킷 캡처 (포렌식 분석용)
```

**SIEM 연동 시 핵심 활용:**

1. **상관 분석(Correlation)**: IPS 탐지 이벤트 + 방화벽 로그 + 서버 로그를 결합하여 공격 체인 파악
   - 예: IPS에서 Exploit 탐지 → 방화벽에서 해당 IP의 C2 통신 확인 → 서버에서 비정상 프로세스 생성 로그 → 침해 확정
2. **Threat Intelligence 연동**: IOC(Indicators of Compromise) 피드와 IPS 로그 대조
3. **보고서 자동화**: 주간/월간 IPS 탐지 현황, 시그니처별 통계, 오탐률 분석

로그 보관 기간은 ISMS-P 기준 최소 1년, PCI-DSS 기준 최소 1년이며, 실무적으로는 6개월 Hot(즉시 검색) + 6개월 Cold(아카이브) 구성이 일반적입니다.
