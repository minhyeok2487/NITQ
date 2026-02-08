# 04-05. Checkpoint Logging과 SmartView

---

### 면접관: "고객사에서 보안 감사(Audit)를 앞두고 있어서 방화벽 로그를 체계적으로 분석해야 합니다. Check Point에서 로그를 어떻게 확인하고 분석하시겠어요?"

Check Point 로그 분석은 **SmartConsole의 Logs & Monitor** 탭에서 수행합니다. 과거에는 SmartView Tracker라는 별도 애플리케이션을 사용했지만, R80 이후 SmartConsole에 통합되었습니다.

#### 로그 확인 기본 절차

```
# SmartConsole → Logs & Monitor 탭

# 1. 기본 로그 뷰:
#    - 시간순 정렬된 전체 로그 확인
#    - 각 로그 엔트리: 시간, Source, Destination, Service, Action, Rule, Blade 등

# 2. 필터링 (보안 감사용):
#    - 기간 설정: "Last 24 hours", "Last 7 days", 또는 Custom Range
#    - 필터 쿼리 예시:
#      action:Drop                          → 차단된 트래픽만
#      blade:"Firewall"                     → 방화벽 Blade 로그만
#      src:10.10.10.0/24                    → 특정 네트워크 발신
#      service:https AND action:Accept      → HTTPS 허용 트래픽
#      rule_name:"Cleanup Rule"             → 특정 룰 매칭 로그

# 3. 로그 상세 보기:
#    각 로그 엔트리 클릭 시:
#    - Rule Number / Rule Name
#    - NAT 전후 IP
#    - Application (APCL 활성 시)
#    - Threat 정보 (IPS/AV 활성 시)
#    - Session 정보
```

#### 보안 감사 대비 주요 로그 분석 항목

| 감사 항목 | SmartConsole 쿼리 | 목적 |
|-----------|-------------------|------|
| 외부→내부 접근 시도 | `direction:incoming AND action:Drop` | 침입 시도 파악 |
| 관리자 접속 기록 | `blade:"Management" OR service:ssh` | 관리 접근 추적 |
| 정책 변경 이력 | Audit 로그 탭 | 누가 언제 정책 변경했는지 |
| Drop된 트래픽 통계 | `action:Drop` + Statistics 뷰 | 비정상 트래픽 패턴 |
| 특정 서버 접근 기록 | `dst:172.16.1.10` | 주요 자산 접근 추적 |

```
# Audit 로그 확인 (정책 변경 이력):
# SmartConsole → Logs & Monitor → Audit 탭
# → 관리자별 작업 내역, 정책 Publish/Install 이력 확인

# 로그 Export:
# SmartConsole → Logs & Monitor → 필터 적용 후
# → File → Export Logs → CSV 또는 기타 형식
```

> 토폴로지 이미지 추가 예정

---

### 면접관: "Rule Base에서 Track 옵션이 여러 개 있던데, 각각 차이가 뭡니까?"

Security Policy의 각 Rule에서 **Track** 컬럼은 해당 룰에 매칭된 트래픽의 **로깅 방식**을 결정합니다.

#### Track 옵션 종류

| Track 옵션 | 설명 | 사용 시나리오 |
|------------|------|-------------|
| **None** | 로그 기록 안 함 | Noise Rule (NetBIOS, Broadcast 등 불필요한 로그 억제) |
| **Log** | 로그 기록 (기본) | 대부분의 룰에 사용. 세션 시작 시 1회 기록 |
| **Detailed Log** | 상세 로그 기록 | 세션 시작 + 종료 시 기록, 바이트/패킷 수 포함 |
| **Extended Log** | 확장 로그 | Detailed + 추가 데이터 (HTTPS 내 URL 등, Blade별) |
| **Full Log** | 전체 로그 (패킷 캡처 포함) | 트러블슈팅 시 일시적 사용. 저장 공간 급증 주의 |
| **Alert** | 로그 + 경고 발생 | 심각한 보안 이벤트 (관리자에게 알림) |
| **Mail** | 로그 + 이메일 알림 | 특정 이벤트 발생 시 메일 발송 |
| **SNMP Trap** | 로그 + SNMP Trap 전송 | NMS 연동 시 |
| **User Defined** | 사용자 정의 스크립트 실행 | 커스텀 대응 자동화 |

#### 실무 권장 설정

```
# Rule별 Track 설정 가이드:

# Admin Access Rule       → Log (관리자 접근은 반드시 기록)
# Stealth Rule            → Log + Alert (방화벽 공격 시도 알림)
# 업무 허용 Rule          → Log (기본)
# 중요 서버 접근 Rule     → Detailed Log (세션 정보 상세 기록)
# Noise Rule              → None (불필요한 로그 억제로 성능/저장공간 절약)
# Cleanup Rule            → Log (차단된 모든 트래픽 기록 - 트러블슈팅 필수)

# SmartConsole에서 Track 설정:
# Security Policy → Rule → Track 컬럼 클릭 → 옵션 선택

# Accounting (바이트/패킷 통계) 활성화:
# Track → Log → 우클릭 → "Accounting" 체크
# → 해당 세션의 전송량(bytes, packets) 정보가 로그에 포함됨
```

#### Log vs Detailed Log 차이

```
# Log:
#   세션 시작 시점에 1개 로그 엔트리 생성
#   → 누가, 어디로, 어떤 서비스로 접근했는지 기록
#   → 저장 공간 효율적

# Detailed Log:
#   세션 시작 + 종료 시 각각 로그 엔트리 생성
#   → 세션 지속 시간, 전송 바이트/패킷 수 포함
#   → 보안 감사에서 "얼마나 오래, 얼마나 많이" 확인 가능
#   → 로그 양이 2배 이상 증가하므로 저장 공간 고려 필요
```

---

### 면접관: "로그가 안 쌓입니다. Gateway에서 트래픽은 잘 처리되는데 SmartConsole에서 로그가 보이질 않아요. 어디부터 확인하시겠어요?"

로그가 안 쌓이는 문제는 **Gateway → Management(Log Server) 간 로그 전송 경로**를 따라 점검합니다.

#### 1단계: Gateway에서 로그 생성 확인

```
# Gateway Expert Mode
cd $FWDIR/log/
ls -la fw.log*
# → 로컬에 로그 파일이 있는지, 크기가 증가하는지 확인

# 실시간 로그 확인 (Gateway 로컬)
fw log -f -t
# → 트래픽 처리 시 로그가 실시간 출력되면 Gateway 로그 생성은 정상

# Track 설정 확인
# → 문제 되는 룰의 Track이 "None"이 아닌지 SmartConsole에서 확인
```

#### 2단계: Log Server 설정 확인

```
# Gateway가 로그를 보내는 대상 확인
# SmartConsole → Gateway Object → Logs 탭
# → "Send logs to" → Management Server 또는 별도 Log Server가 지정되어 있는지

# Gateway Expert Mode에서 확인
cpstat fw -f log_servers
# → 연결된 Log Server IP, 상태 표시

# Log Server 연결 테스트
fw logswitch           # 로그 파일 스위치 (새 로그 파일로 전환)
fw fetchlogs           # 로그 전송 시도
```

#### 3단계: SIC 및 네트워크 확인

```
# Gateway → Management/Log Server 간 SIC 통신 확인
cpstat fw -f log_servers
# → Status: "connected" / "disconnected"

# 포트 확인 (TCP 257 - 로그 전송 포트)
# Gateway에서:
telnet <Management_IP> 257

# SIC가 깨진 경우 → 04-04 SIC 문서 참조하여 재수립

# Management Server에서 fwd 데몬 상태 확인
cpwd_admin list | grep FWD
# FWD가 E(Enabled) 상태인지 확인
```

#### 4단계: 디스크 공간 및 로그 로테이션 확인

```
# Management/Log Server 디스크 공간 확인
df -h
# /var/log 파티션이 100%이면 로그 수신 불가!

# Expert Mode
du -sh $FWDIR/log/
# 로그 디렉토리 크기 확인

# 로그 파일 현황
ls -la $FWDIR/log/fw.log*
ls -la $FWDIR/log/*.log

# 로그 로테이션 설정 확인
# SmartConsole → Manage & Settings → Logs
# → Log rotation 설정: 파일 크기 또는 시간 기준
# → "When disk space is below X%" → 오래된 로그 자동 삭제
```

#### 5단계: 데몬 및 프로세스 확인

```
# Management Server Expert Mode
cpwd_admin list
# FWM, FWD, CPD, CPMAD, LogServer 관련 프로세스 확인

# 로그 데몬 재시작
fwm logexport          # 로그 내보내기 상태 확인

# 전체 CP 서비스 재시작 (최후 수단)
cpstop
cpstart
cpwd_admin list        # 재시작 후 모든 프로세스 확인

# Log indexing 상태 확인
fwm logexport -d $FWDIR/log/fw.log
```

---

### 면접관: "Check Point의 Log Server 아키텍처를 설명해주세요. 대규모 환경에서는 어떻게 구성하나요?"

Check Point의 로그 아키텍처는 환경 규모에 따라 세 가지 구성이 가능합니다.

#### 구성 1: 기본 구성 (Management = Log Server)

```
# 가장 일반적인 소규모 구성
# Management Server가 Log Server 역할도 겸함

[Gateway] ---(로그)--→ [Management Server + Log Server]
                              ↑
                      [SmartConsole로 조회]

# 장점: 단순, 비용 절감
# 단점: 로그 양이 많으면 Management 성능 저하
```

#### 구성 2: 별도 Log Server 분리

```
# 중규모 환경 권장
# Log Server를 별도 장비로 분리

[Gateway A] ---(로그)--→ [Log Server (Dedicated)]
[Gateway B] ---(로그)--↗         ↑
                          [SmartConsole로 조회]
                                 ↑
[SmartConsole] ←---(관리)--- [Management Server]

# 장점: Management와 로그 부하 분리, 성능 향상
# 설치: Log Server도 GAiA OS 설치 후 FTW에서 "Log Server" 역할 선택
# SIC: Management → Log Server 간 SIC 수립 필요
```

#### 구성 3: Smart-1 Cloud / Log Exporter 연동

```
# 대규모 / 클라우드 환경

[Gateway] --→ [Log Server] --→ [Smart-1 Cloud (SaaS)]
                    |
                    └──→ [SIEM (Splunk/QRadar 등)]
                          (Log Exporter / syslog)

# Check Point Smart-1 Cloud:
#   SaaS 형태의 로그 관리/분석 서비스
#   온프레미스 로그 서버의 저장 공간 걱정 없음
```

#### Log Server 설정

```
# 1. Log Server 전용 장비 FTW:
#    Products → "Log Server" 선택

# 2. Management에서 Log Server 오브젝트 등록:
#    SmartConsole → New → Check Point Host → Log Server
#    → SIC 초기화

# 3. Gateway에서 로그 전송 대상 변경:
#    SmartConsole → Gateway Object → Logs 탭
#    → "Send logs to" → 새 Log Server 선택

# 4. Policy Install

# Log Server 상태 확인 (Log Server Expert Mode):
cpstat os
cpstat mg               # Log Server Management 연결 상태
cpwd_admin list          # 데몬 상태
df -h                    # 디스크 공간
```

---

### 면접관: "고객사에서 Check Point 로그를 SIEM(Splunk)으로 보내달라고 합니다. 어떻게 연동하시겠어요?"

Check Point 로그를 SIEM으로 전송하는 방법은 크게 **Log Exporter**, **LEA(Log Export API)**, **syslog** 세 가지가 있습니다. 현재 가장 권장되는 방법은 **Log Exporter**입니다.

#### 방법 1: Log Exporter (R80.20+, 권장)

Log Exporter는 Check Point 로그를 **syslog/CEF/LEEF/JSON** 형식으로 SIEM에 실시간 전송합니다.

```
# Management Server 또는 Log Server에서 설정 (Expert Mode)

# 1. Log Exporter 설치 확인
cp_log_export --help
# 명령어가 없으면 Jumbo Hotfix 업데이트 필요

# 2. Log Exporter 설정 생성
cp_log_export add name splunk_export \
    target-server <Splunk_IP> \
    target-port 514 \
    protocol tcp \
    format splunk \
    read-mode semi-unified

# format 옵션:
#   splunk      → Splunk용 최적화 형식
#   cef         → ArcSight 등 CEF 호환 SIEM
#   leef        → QRadar용 LEEF 형식
#   json        → 범용 JSON 형식
#   generic     → 일반 syslog

# 3. Log Exporter 시작
cp_log_export restart name splunk_export

# 4. 상태 확인
cp_log_export status name splunk_export

# 5. 설정 목록 확인
cp_log_export show name splunk_export
```

#### 방법 2: syslog 직접 전송 (간단한 연동)

```
# Gateway에서 직접 syslog 전송 설정
# SmartConsole → Gateway Object → Logs 탭
# → "Additional Logging" → "Forward log files to the syslog server"
# → Syslog Server IP 입력

# 또는 Expert Mode에서 수동 설정
vi $FWDIR/conf/log_export.conf
# SIEM 서버 IP, 포트, 형식 설정

# syslog 데몬 설정 (rsyslog)
vi /etc/rsyslog.conf
# *.* @@<SIEM_IP>:514    (TCP)
# *.* @<SIEM_IP>:514     (UDP)
systemctl restart rsyslog
```

#### 방법 3: OPSEC LEA (Legacy)

```
# 구버전 연동 방식. OPSEC 인증 애플리케이션이 Management에 접속하여 로그를 Pull
# → 현재는 Log Exporter가 더 효율적이므로 신규 구축에서는 비권장

# LEA 설정 시 필요:
# - OPSEC Application Object 생성 (SmartConsole)
# - SIC 수립 (OPSEC Application ↔ Management)
# - SIEM 측에서 LEA Client 설정
```

#### Splunk 연동 상세

```
# Splunk 측 설정:

# 1. Check Point App for Splunk 설치 (Splunkbase에서 다운로드)
#    → 대시보드, 파서, 필드 추출 자동 구성

# 2. Data Input 설정:
#    Settings → Data Inputs → TCP/UDP → New
#    Port: 514 (또는 지정 포트)
#    Source type: cp_log 또는 syslog

# Check Point 측 확인:
# cp_log_export로 전송 시작 후 Splunk에서 검색:
# index=main sourcetype=cp_log
# → 로그가 수신되는지 확인

# 트러블슈팅:
# 1. 네트워크 연결 확인
telnet <Splunk_IP> 514

# 2. Log Exporter 로그 확인
tail -f /var/log/log_exporter/splunk_export.log

# 3. 방화벽 룰 확인 (Log Server → SIEM 간 514 포트 허용)
```

---

### 면접관: "로그 보관 기간은 어떻게 관리합니까? 고객사에서 법적으로 1년 이상 보관해야 한다고 하는데요."

로그 보관 기간 관리는 **디스크 용량**, **로그 로테이션**, **외부 저장소 연동** 세 가지를 종합적으로 설계해야 합니다.

#### 로그 로테이션 및 보관 설정

```
# SmartConsole → Manage & Settings → Blades → Logging & Monitoring

# Log Rotation 설정:
# - Schedule: Daily / Weekly / By file size (예: 2GB)
# - "Rotate when file size exceeds" → 2048 MB (2GB) 등

# Log Cleanup 설정:
# - "Delete logs older than" → 365 days (1년)
# - "When disk space is below X%" → 15% (남은 공간이 15% 미만이면 오래된 로그 삭제)

# 수동 로그 로테이션 (Expert Mode)
fw logswitch
# → 현재 로그 파일을 닫고 새 파일 시작
# → 이전 파일: fw.log.YYYYMMDD-HHMMSS.log

# 스케줄 설정
fw logswitch -h 24    # 24시간마다 자동 로테이션
```

#### 장기 보관 전략

```
# 방법 1: Log Server 전용 디스크 확장
# → RAID 구성, 대용량 스토리지 할당
# → 1년치 로그 용량 사전 산출 필요:
#   일일 로그 양 × 365일 × 1.2 (여유 계수)

# 방법 2: 외부 스토리지로 로그 아카이빙
# Expert Mode에서 cron job으로 자동 복사
crontab -e
# 매주 일요일 02:00에 오래된 로그를 NFS/SAN으로 이동:
# 0 2 * * 0 /bin/bash /opt/scripts/log_archive.sh

# 아카이브 스크립트 예시:
# find $FWDIR/log/ -name "fw.log.*" -mtime +30 -exec mv {} /mnt/archive/ \;
# → 30일 이전 로그를 아카이브 스토리지로 이동

# 방법 3: SIEM으로 장기 보관 위임
# Log Exporter로 SIEM에 실시간 전송
# SIEM 측에서 1년 이상 보관 정책 적용 (Cold Storage, S3 등)
# → Check Point 로컬에는 최근 30~90일만 유지
```

#### SmartEvent 활용 (로그 기반 리포팅)

```
# SmartEvent Blade 활성화:
# SmartConsole → Manage & Settings → Blades → SmartEvent → Enable

# SmartEvent가 활성화되면:
# - 로그 기반 보안 이벤트 자동 상관 분석(Correlation)
# - 사전 정의된 보안 리포트 생성
# - Timeline 뷰로 시간축 기반 분석
# - 보안 감사용 PDF 리포트 자동 생성 가능

# SmartConsole → Logs & Monitor → SmartEvent 탭
# → Reports → 기간 설정 → Generate Report
# → PDF / CSV Export

# 보안 감사 시 필수 리포트:
# - Top Dropped Connections (차단 상위 트래픽)
# - Policy Change Audit (정책 변경 이력)
# - Admin Login/Logout Report (관리자 접속 기록)
# - Top Sources / Destinations (통신량 상위)
# - Security Incident Timeline (보안 이벤트 타임라인)
```

---

### 면접관: "SmartConsole에서 보는 로그와 Expert Mode에서 보는 로그가 다를 수 있습니까? 어떤 상황에서 Expert Mode로 직접 로그를 봐야 합니까?"

SmartConsole 로그는 **Log Database에 인덱싱된 로그**를 보여주고, Expert Mode에서는 **원본 로그 파일과 시스템 로그**에 직접 접근할 수 있습니다.

#### SmartConsole vs Expert Mode 로그

| 항목 | SmartConsole (Logs & Monitor) | Expert Mode (CLI) |
|------|-------------------------------|-------------------|
| **데이터 소스** | Log Server의 인덱싱된 DB | 원본 파일 직접 접근 |
| **검색 성능** | 빠름 (인덱싱) | 느림 (파일 파싱) |
| **실시간성** | 약간의 지연 가능 | 즉시 확인 |
| **시스템 로그** | 일부만 표시 | 전체 OS/CP 로그 접근 |
| **커널 디버그** | 불가 | 가능 (fw ctl zdebug 등) |

#### Expert Mode에서 직접 로그를 봐야 하는 상황

```
# 1. SmartConsole 접속 불가 시
#    → Management Server에 직접 SSH 접속하여 로그 확인
fw log -f -t                # 실시간 로그 출력

# 2. 커널 레벨 디버그 필요 시
fw ctl zdebug drop          # Drop된 패킷의 커널 디버그
# → SmartConsole에 나오지 않는 커널 레벨 Drop 원인 확인

# 3. 시스템/OS 레벨 로그 확인
tail -f /var/log/messages       # GAiA OS 시스템 로그
tail -f /var/log/routed.log     # 라우팅 데몬 로그
tail -f $CPDIR/log/cpd.elg      # CPD 데몬 로그
tail -f $FWDIR/log/fwm.elg      # FWM 데몬 로그 (Management)
tail -f $FWDIR/log/fwd.elg      # FWD 데몬 로그

# 4. Policy Install 실패 로그
tail -f $FWDIR/log/install_policy.elg

# 5. 로그 파일 직접 분석 (인덱싱 안 된 로그)
fw log -l $FWDIR/log/fw.log.20260101-120000.log
# → 과거 로테이션된 로그 파일 직접 열기

# 6. 로그 통계 빠르게 확인
fw log -c                   # 현재 로그 파일 엔트리 수
ls -la $FWDIR/log/fw.log    # 현재 로그 파일 크기
```

**실무 팁:**
- 일상 모니터링은 SmartConsole로 충분
- 장애 대응/심층 분석 시에만 Expert Mode 사용
- 커널 디버그(`fw ctl zdebug`)는 성능 영향이 있으므로 **짧게 실행하고 반드시 중지**
- 로그 파일 직접 조작 시 원본 손상 주의 (읽기 전용으로)
