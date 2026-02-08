# 07-04. Cisco DNA Center

> 네트워크 엔지니어 면접 시나리오

> CCNA 200-301 수준의 Cisco DNA Center 관련 면접 질문과 답변

---

### 면접관: "Cisco DNA Center가 무엇이고, 전통적인 네트워크 관리 방식과 어떻게 다른지 설명해 주세요."

Cisco DNA Center는 **Cisco의 엔터프라이즈 네트워크용 SDN 컨트롤러**입니다. Intent-Based Networking(의도 기반 네트워킹)을 구현하는 핵심 플랫폼으로, 네트워크의 설계, 배포, 운영, 모니터링을 하나의 대시보드에서 수행할 수 있습니다.

#### 핵심 비교: 전통적 관리 vs DNA Center

| 구분 | 전통적 관리 | DNA Center |
|------|-----------|------------|
| 장비 설정 | CLI로 장비별 개별 접속 | GUI/API에서 중앙 일괄 관리 |
| 모니터링 | SNMP 기반, 별도 NMS 도구 | 내장 Assurance 기능 |
| 문제 해결 | 장비별 로그 수동 확인 | AI/ML 기반 자동 문제 감지 |
| 정책 적용 | ACL을 장비별 수동 설정 | 정책 한 번 정의, 전체 적용 |
| 소프트웨어 관리 | 장비별 수동 업그레이드 | 중앙에서 일괄 업그레이드 |
| 토폴로지 파악 | 수동 다이어그램 작성 | 자동 네트워크 토폴로지 생성 |

DNA Center의 핵심 가치는 **네트워크 관리자의 의도(Intent)를 정의하면, 그에 맞는 설정을 자동으로 생성하고 배포**하는 것입니다. 예를 들어 "영업팀은 인터넷과 CRM 시스템에만 접근 가능하다"라는 정책을 정의하면, DNA Center가 필요한 ACL, QoS, VLAN 설정을 각 장비에 자동으로 적용합니다.

---

### 면접관: "Intent-Based Networking(IBN)이란 무엇인가요? 구체적인 예를 들어 설명해 주세요."

Intent-Based Networking은 **관리자가 비즈니스 의도(Intent)를 정의하면, 네트워크가 자동으로 그에 맞게 구성되고 지속적으로 검증하는 네트워킹 방식**입니다.

**IBN의 동작 흐름:**

```
1. 의도 정의 (Translate)
   관리자: "게스트 사용자는 인터넷만 접근 가능하고,
            내부 서버에는 접근 불가"

          ↓

2. 자동 설정 (Activate)
   DNA Center가 자동으로:
   - 게스트 VLAN 생성
   - ACL 설정 (내부 서버 차단)
   - QoS 정책 적용
   - 모든 관련 스위치에 배포

          ↓

3. 지속적 검증 (Assurance)
   - 설정이 의도대로 동작하는지 실시간 모니터링
   - 정책 위반 감지 시 관리자에게 알림
   - 자동 문제 해결 제안
```

#### 핵심 개념: IBN의 3단계

| 단계 | 설명 | DNA Center 기능 |
|------|------|----------------|
| Translate | 비즈니스 의도를 네트워크 정책으로 변환 | Policy 설정, SSID 설정 |
| Activate | 정책을 네트워크 장비에 자동 배포 | Template 배포, Provisioning |
| Assurance | 네트워크가 의도대로 동작하는지 검증 | Assurance, Health Score |

전통적 방식에서는 관리자가 "게스트 인터넷만 허용"이라는 의도를 직접 ACL, VLAN, 라우팅 설정으로 번역해야 했습니다. IBN에서는 이 번역 과정을 DNA Center가 자동으로 처리합니다.

---

### 면접관: "DNA Center의 Assurance 기능에 대해 설명해 주세요."

Assurance는 DNA Center의 **네트워크 상태 모니터링 및 문제 해결 기능**입니다. 전통적인 모니터링과 달리 **AI/ML(인공지능/머신러닝) 기반**으로 문제를 사전에 감지하고 원인을 분석합니다.

**Assurance의 주요 기능:**

**1. Health Score (건강 점수)**
- 네트워크 장비, 클라이언트, 애플리케이션에 **0~10점의 건강 점수**를 부여합니다
- 점수가 낮은 항목을 우선적으로 확인할 수 있습니다

```
네트워크 건강 대시보드 예시:
┌──────────────┬──────────┐
│ 카테고리      │ 점수      │
├──────────────┼──────────┤
│ 전체 네트워크  │ 8/10     │
│ 유선 클라이언트 │ 9/10     │
│ 무선 클라이언트 │ 7/10     │
│ 핵심 스위치    │ 10/10    │
│ 액세스 스위치  │ 6/10 ⚠️  │
└──────────────┴──────────┘
```

**2. Issue Detection (문제 감지)**
- 네트워크 문제를 자동으로 감지하고 분류합니다
- 예: "AP-03에서 클라이언트 접속 실패율이 평소보다 300% 증가"

**3. Path Trace**
- 출발지에서 목적지까지의 **경로를 시각적으로 추적**합니다
- 어느 구간에서 문제가 발생하는지 빠르게 파악할 수 있습니다

**4. Intelligent Capture**
- 무선 클라이언트의 **패킷을 원격으로 캡처**하여 분석합니다
- 현장 방문 없이 무선 문제를 진단할 수 있습니다

Assurance가 없던 시절에는 문제가 발생하면 여러 장비에 로그인해서 로그를 확인하고, Wireshark로 패킷을 캡처해야 했습니다. Assurance는 이 과정을 자동화하고 시각화하여 **MTTR(Mean Time To Resolve, 평균 문제 해결 시간)**을 크게 줄여줍니다.

---

### 면접관: "SD-Access가 무엇인지 간단히 설명해 주세요."

SD-Access(Software-Defined Access)는 Cisco DNA Center 위에서 동작하는 **캠퍼스 네트워크의 자동화 솔루션**입니다. 네트워크 접근 제어와 세그멘테이션을 자동화합니다.

#### 핵심 개념

**1. Fabric(패브릭)**
- SD-Access에서 관리하는 네트워크 영역을 Fabric이라고 합니다
- VXLAN 기반의 오버레이 네트워크를 자동으로 구성합니다

**2. 주요 구성 요소:**

| 구성 요소 | 역할 |
|----------|------|
| Control Plane Node | LISP를 사용하여 호스트 위치 정보 관리 |
| Fabric Edge Node | 엔드포인트가 연결되는 액세스 스위치 |
| Fabric Border Node | 외부 네트워크와 연결되는 경계 장비 |
| ISE (Identity Services Engine) | 사용자 인증 및 정책 적용 |

**3. 핵심 장점: 매크로/마이크로 세그멘테이션**
- **매크로 세그멘테이션**: VN(Virtual Network)으로 네트워크를 논리적으로 분리
- **마이크로 세그멘테이션**: SGT(Scalable Group Tag)로 같은 네트워크 내에서도 세밀한 접근 제어

```
예시: 같은 사무실 내 사용자 분류
VN-Corporate:
  ├─ SGT: 영업팀 → CRM 접근 허용
  ├─ SGT: 개발팀 → 개발 서버 접근 허용
  └─ SGT: 인사팀 → HR 시스템 접근 허용

VN-Guest:
  └─ SGT: 방문자 → 인터넷만 허용
```

CCNA 수준에서는 SD-Access의 기본 개념과 DNA Center가 SD-Access의 컨트롤러 역할을 한다는 점을 이해하면 충분합니다.

---

### 면접관: "DNA Center의 API를 통해 어떤 작업을 할 수 있나요?"

DNA Center는 **REST API를 제공**하여 프로그래밍 방식으로 네트워크를 관리할 수 있습니다.

**주요 API 카테고리:**

| API 카테고리 | 설명 | 예시 |
|------------|------|------|
| Authentication | 토큰 발급 | POST /dna/system/api/v1/auth/token |
| Network Device | 장비 관리 | GET /dna/intent/api/v1/network-device |
| Site | 사이트 관리 | GET /dna/intent/api/v1/site |
| Client | 클라이언트 정보 | GET /dna/intent/api/v1/client-health |
| Command Runner | CLI 명령 실행 | POST /dna/intent/api/v1/network-device-poller/cli/read-request |
| Template | 템플릿 관리 | GET /dna/intent/api/v1/template-programmer/template |

```json
// 장비 목록 조회 API 응답 예시
// GET /dna/intent/api/v1/network-device
{
  "response": [
    {
      "hostname": "Core-SW-01",
      "managementIpAddress": "192.168.1.1",
      "platformId": "C9300-48U",
      "softwareVersion": "17.6.3",
      "role": "DISTRIBUTION",
      "upTime": "45 days, 12:30:00",
      "reachabilityStatus": "Reachable",
      "serialNumber": "FCW2345L0AB"
    },
    {
      "hostname": "Access-SW-01",
      "managementIpAddress": "192.168.2.1",
      "platformId": "C9200L-24P",
      "softwareVersion": "17.6.3",
      "role": "ACCESS",
      "upTime": "30 days, 08:15:00",
      "reachabilityStatus": "Reachable",
      "serialNumber": "FCW2345L0CD"
    }
  ],
  "version": "1.0"
}
```

API를 활용하면 정기적으로 장비 상태를 확인하는 스크립트를 작성하거나, 장애 발생 시 자동으로 진단하는 프로그램을 만들 수 있습니다.

---

### 면접관: "DNA Center에서 장비를 어떻게 발견(Discovery)하고 관리에 추가하나요?"

DNA Center에서 네트워크 장비를 발견하는 방법은 크게 세 가지입니다.

**1. IP Range Discovery**
- IP 범위를 지정하여 해당 범위 내 장비를 자동으로 찾습니다
- 예: 192.168.1.1 ~ 192.168.1.254

**2. CDP Discovery**
- CDP(Cisco Discovery Protocol)를 사용하여 이웃 장비를 자동으로 발견합니다
- 시작 장비(Seed Device)를 지정하면 연결된 모든 장비를 찾아갑니다

**3. LLDP Discovery**
- LLDP(Link Layer Discovery Protocol) 기반으로 장비를 발견합니다
- 비Cisco 장비도 발견 가능합니다

#### 핵심 설정: Discovery에 필요한 정보

| 필요 정보 | 설명 |
|----------|------|
| SNMP Credentials | SNMP 커뮤니티 문자열 또는 v3 자격 증명 |
| CLI Credentials | SSH 접속용 사용자 이름/비밀번호 |
| NETCONF 설정 | NETCONF를 사용하는 장비의 포트 정보 |
| IP Range | 스캔할 IP 주소 범위 |

**Discovery 후 관리 과정:**

```
1. Discovery 실행
   → DNA Center가 지정된 범위의 장비를 스캔

2. 장비 발견
   → 발견된 장비의 기본 정보 수집 (호스트명, OS, 시리얼)

3. Inventory에 추가
   → 장비가 DNA Center의 관리 목록에 등록

4. Site 할당
   → 장비를 적절한 사이트(본사, 지점 등)에 할당

5. Provisioning
   → 사이트 정책에 맞는 설정을 장비에 자동 배포
```

이 과정이 완료되면 DNA Center에서 해당 장비를 중앙 관리할 수 있게 됩니다.

---

### 면접관: "DNA Center의 Template 기능은 어떤 용도로 사용하나요?"

Template은 **네트워크 장비에 배포할 설정을 표준화된 템플릿으로 정의**하는 기능입니다.

**Template의 장점:**
- 반복적인 설정을 템플릿화하여 일관성 유지
- 변수를 사용하여 장비별 다른 값 적용 가능
- 설정 오류 감소, 배포 시간 단축

```
# DNA Center Template 예시 (Jinja2 기반)

hostname {{ hostname }}

! NTP 설정
ntp server {{ ntp_server }}

! SNMP 설정
snmp-server community {{ snmp_community }} RO
snmp-server location {{ site_location }}

! 인터페이스 설정
interface {{ mgmt_interface }}
 ip address {{ mgmt_ip }} {{ mgmt_mask }}
 no shutdown

! Banner
banner motd ^
==============================================
  Authorized Access Only - {{ hostname }}
  Managed by Cisco DNA Center
==============================================
^
```

이 Template을 사용하면 장비별로 `hostname`, `ntp_server`, `mgmt_ip` 등의 값만 변경하여 동일한 구조의 설정을 수백 대의 장비에 일관성 있게 배포할 수 있습니다.

#### 핵심 비교: Template 사용 전후

| 구분 | Template 미사용 | Template 사용 |
|------|----------------|--------------|
| 설정 생성 | 장비별 수동 작성 | 변수만 입력하면 자동 생성 |
| 일관성 | 장비마다 미세한 차이 발생 | 모든 장비 동일한 표준 설정 |
| 변경 관리 | 개별 장비 접속하여 수정 | 템플릿 수정 후 일괄 재배포 |
| 감사 | 어떤 설정이 적용되었는지 추적 어려움 | 템플릿 버전으로 변경 이력 관리 |

---

### 면접관: "DNA Center를 도입할 때 고려해야 할 사항은 무엇인가요?"

DNA Center 도입 시 고려해야 할 사항들입니다.

**1. 지원 장비 확인**
- 모든 Cisco 장비가 DNA Center를 지원하는 것은 아닙니다
- **Catalyst 9000 시리즈**가 가장 완벽하게 지원되며, 구형 장비는 기능이 제한될 수 있습니다
- 비Cisco 장비는 기본적으로 지원하지 않습니다

**2. 라이선스 모델**
- DNA Center는 **구독 기반 라이선스** 모델을 사용합니다
- Essentials, Advantage, Premier 등 등급에 따라 사용 가능한 기능이 다릅니다
- Assurance 기능은 Advantage 이상에서 사용 가능합니다

**3. 인프라 요구사항**
- DNA Center 어플라이언스(물리 서버)가 필요합니다
- 소규모(최소 1대)부터 대규모(3대 클러스터)까지 규모에 맞게 구성합니다
- 충분한 네트워크 대역폭과 장비 접근성 확보가 필요합니다

**4. 운영 팀 역량**
- CLI 중심에서 GUI/API 중심으로 운영 방식이 변화합니다
- 팀원들의 교육과 역량 강화 계획이 필요합니다
- 기존 운영 프로세스의 변경이 수반됩니다

**5. 단계적 도입**
- 전체 네트워크를 한 번에 전환하기보다 파일럿 사이트부터 시작하는 것을 권장합니다
- 기존 관리 방식과 병행하면서 점진적으로 전환합니다

DNA Center는 강력한 도구이지만, 도입 자체가 목적이 되어서는 안 됩니다. 현재 네트워크의 규모, 복잡도, 운영 과제를 분석하여 **실질적인 효과가 있는지 먼저 판단**하는 것이 중요합니다.
