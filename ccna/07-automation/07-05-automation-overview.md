# 07-05. 네트워크 자동화 개요

> 네트워크 엔지니어 면접 시나리오

> CCNA 200-301 수준의 네트워크 자동화 전반에 대한 면접 질문과 답변

---

### 면접관: "네트워크 자동화가 왜 필요한지, 실무적인 관점에서 설명해 주세요."

네트워크 자동화가 필요한 이유는 크게 네 가지입니다.

**1. 규모의 문제 (Scale)**
- 현대 네트워크는 수백~수천 대의 장비로 구성됩니다
- 새로운 VLAN을 추가하거나 NTP 서버를 변경할 때, 수백 대의 장비에 CLI로 접속하는 것은 비현실적입니다

**2. 일관성 문제 (Consistency)**
- 사람이 수동으로 작업하면 **설정 불일치(Configuration Drift)**가 발생합니다
- 예: 100대 스위치 중 3대만 NTP 설정이 다른 경우
- 자동화 도구는 모든 장비에 동일한 설정을 보장합니다

**3. 속도 문제 (Speed)**
- 장애 발생 시 빠른 대응이 필요합니다
- 자동화된 스크립트는 수동 작업 대비 **수십 배 빠르게** 작업을 완료합니다

**4. 인적 오류 감소 (Human Error Reduction)**
- 네트워크 장애의 상당수가 **설정 실수**에서 비롯됩니다
- 코드로 검증된 설정은 타이핑 실수를 원천적으로 차단합니다

#### 핵심 비교: 수동 작업 vs 자동화

| 시나리오 | 수동 (CLI) | 자동화 |
|---------|-----------|--------|
| 50대 스위치 VLAN 추가 | 약 4시간 (장비당 5분) | 약 10분 |
| 전체 장비 설정 백업 | 하루 종일 | 5분 (스크립트 1회 실행) |
| 설정 표준 준수 여부 확인 | 장비별 수동 점검 | 자동 비교 리포트 생성 |
| NTP 서버 전체 변경 | 수 시간 | 수 분 |
| 신규 사이트 배포 | 수일 | 수 시간 |

---

### 면접관: "Ansible, Puppet, Chef의 차이점을 비교해 주세요. 네트워크 자동화에는 어떤 것이 적합한가요?"

세 가지 모두 **Configuration Management(설정 관리) 도구**이지만, 접근 방식이 다릅니다.

#### 핵심 비교

| 구분 | Ansible | Puppet | Chef |
|------|---------|--------|------|
| 개발사 | Red Hat | Puppet Inc. | Progress (구 Chef) |
| 아키텍처 | **Agentless** (SSH) | Agent 기반 (Master-Agent) | Agent 기반 (Server-Client) |
| 설정 언어 | **YAML** (Playbook) | Puppet DSL (Manifest) | **Ruby** (Recipe) |
| 학습 난이도 | **쉬움** | 보통 | 어려움 |
| Push/Pull | **Push** (컨트롤 노드에서 전송) | Pull (에이전트가 주기적 확인) | Pull (클라이언트가 주기적 확인) |
| 네트워크 지원 | **매우 우수** | 보통 | 제한적 |
| 멱등성 | 모듈에 따라 지원 | 기본 지원 | 기본 지원 |

**네트워크 자동화에 Ansible이 가장 적합한 이유:**

1. **Agentless**: 네트워크 장비(스위치, 라우터)에는 Agent를 설치할 수 없는 경우가 많습니다. Ansible은 SSH만 있으면 동작하므로 네트워크 장비에 이상적입니다

2. **YAML 기반**: 네트워크 엔지니어가 프로그래밍 언어를 몰라도 쉽게 시작할 수 있습니다

3. **풍부한 네트워크 모듈**: Cisco, Juniper, Arista 등 주요 벤더의 모듈이 잘 갖춰져 있습니다

Puppet과 Chef는 주로 **서버/인프라 관리**에 강점이 있고, 네트워크 자동화에는 Ansible이 업계 표준에 가깝습니다.

---

### 면접관: "Git을 네트워크 관리에 어떻게 활용할 수 있나요?"

Git은 **버전 관리 시스템(Version Control System)**으로, 네트워크 관리에서 매우 유용하게 활용됩니다.

**1. 네트워크 설정 파일 관리**

```
network-configs/
├── core-switches/
│   ├── core-sw-01.cfg
│   └── core-sw-02.cfg
├── access-switches/
│   ├── acc-sw-01.cfg
│   ├── acc-sw-02.cfg
│   └── acc-sw-03.cfg
├── routers/
│   ├── rt-01.cfg
│   └── rt-02.cfg
└── templates/
    ├── base-switch.j2
    └── base-router.j2
```

**2. Ansible Playbook 관리**
- Playbook을 Git으로 관리하면 변경 이력이 자동으로 기록됩니다
- 누가, 언제, 왜 변경했는지 추적 가능합니다

**3. Git의 네트워크 관리 활용 이점:**

| 활용 방식 | 설명 |
|----------|------|
| 변경 이력 추적 | `git log`로 모든 설정 변경 기록 확인 |
| 롤백 | `git revert`로 이전 설정으로 복원 |
| 코드 리뷰 | Pull Request를 통한 설정 변경 검토 |
| 브랜치 | 새 설정을 별도 브랜치에서 테스트 후 병합 |
| 협업 | 여러 엔지니어가 동시에 작업 가능 |

```bash
# Git 기본 워크플로 예시
git checkout -b feature/add-vlan-500     # 새 브랜치 생성
# VLAN 500 관련 Playbook 수정
git add vlan_config.yml
git commit -m "Add VLAN 500 for Marketing team"
git push origin feature/add-vlan-500     # 원격 저장소에 푸시
# Pull Request 생성 → 동료 리뷰 → 승인 후 병합
```

Git을 사용하면 "어제 이 스위치 설정 누가 바꿨지?"라는 질문에 즉시 답할 수 있고, 문제가 발생하면 이전 상태로 바로 롤백할 수 있습니다.

---

### 면접관: "CI/CD를 네트워크에 적용한다는 것은 어떤 의미인가요?"

CI/CD는 소프트웨어 개발에서 온 개념으로, 네트워크에 적용하면 **설정 변경을 자동으로 테스트하고 배포하는 파이프라인**을 구축하는 것입니다.

**CI (Continuous Integration) - 지속적 통합:**
- 네트워크 설정 변경을 Git에 커밋할 때마다 자동으로 **검증**합니다
- 문법 검사, 정책 준수 여부 확인, 시뮬레이션 테스트 등

**CD (Continuous Deployment) - 지속적 배포:**
- 검증을 통과한 설정을 **자동으로 네트워크에 배포**합니다
- 단계적으로 (테스트 환경 → 스테이징 → 운영) 배포합니다

```
네트워크 CI/CD 파이프라인 예시:

1. 엔지니어가 Playbook 수정 후 Git Push
         ↓
2. CI 서버(Jenkins/GitLab CI)가 자동으로:
   - YAML 문법 검사 (ansible-lint)
   - Playbook 문법 검증 (ansible-playbook --check)
   - 테스트 환경에서 실행
         ↓
3. 테스트 통과 시:
   - 코드 리뷰 요청 (Pull Request)
   - 승인 후 자동 병합
         ↓
4. 운영 환경에 자동 배포:
   - 먼저 파일럿 장비 1대에 적용
   - 검증 통과 시 전체 배포
```

CCNA 수준에서는 CI/CD의 개념과 네트워크에 어떻게 적용되는지 이해하면 됩니다. 실제 파이프라인 구축은 보통 DevOps/NetDevOps 역할에서 담당합니다.

---

### 면접관: "Python을 네트워크 자동화에 어떻게 활용할 수 있나요? 주요 라이브러리도 소개해 주세요."

Python은 네트워크 자동화에서 가장 널리 사용되는 프로그래밍 언어입니다. 문법이 간결하고 네트워크 관련 라이브러리가 풍부하기 때문입니다.

#### 핵심 비교: 주요 네트워크 Python 라이브러리

| 라이브러리 | 용도 | 특징 |
|-----------|------|------|
| **Netmiko** | SSH 접속 및 CLI 명령어 실행 | Paramiko 기반, 다양한 벤더 지원 |
| **NAPALM** | 멀티벤더 네트워크 관리 | 벤더 독립적 API, 설정 비교/교체 |
| **Nornir** | 자동화 프레임워크 | Ansible의 Python 버전, 병렬 처리 |
| **requests** | REST API 호출 | HTTP 요청 처리 |
| **paramiko** | SSH 저수준 라이브러리 | Netmiko의 기반 |
| **TextFSM** | CLI 출력 파싱 | 비구조화 텍스트를 구조화 데이터로 변환 |

```python
# Netmiko 기본 사용 예시
from netmiko import ConnectHandler

# 장비 접속 정보
device = {
    "device_type": "cisco_ios",
    "host": "192.168.1.1",
    "username": "admin",
    "password": "cisco123"
}

# SSH 접속 및 명령어 실행
connection = ConnectHandler(**device)
output = connection.send_command("show ip interface brief")
print(output)

# 설정 변경
config_commands = [
    "interface GigabitEthernet1/0/1",
    "description Connected-to-Server",
    "switchport mode access",
    "switchport access vlan 100"
]
connection.send_config_set(config_commands)
connection.save_config()
connection.disconnect()
```

```python
# NAPALM 기본 사용 예시
from napalm import get_network_driver

driver = get_network_driver("ios")
device = driver("192.168.1.1", "admin", "cisco123")
device.open()

# 장비 기본 정보 조회 (벤더 독립적)
facts = device.get_facts()
print(f"호스트명: {facts['hostname']}")
print(f"OS 버전: {facts['os_version']}")
print(f"가동 시간: {facts['uptime']}초")

# 인터페이스 정보 조회
interfaces = device.get_interfaces()
for name, info in interfaces.items():
    print(f"{name}: {'UP' if info['is_up'] else 'DOWN'}")

device.close()
```

CCNA 수준에서는 Python 코딩을 깊이 알 필요는 없지만, 이러한 라이브러리들이 존재하며 네트워크 자동화에 활용된다는 점을 이해하면 좋습니다.

---

### 면접관: "Infrastructure as Code(IaC)라는 개념을 네트워크 관점에서 설명해 주세요."

Infrastructure as Code는 **인프라의 구성과 관리를 코드로 정의하고 관리하는 방식**입니다. 네트워크에 적용하면 "Network as Code"라고도 부릅니다.

**기존 방식:**
```
엔지니어가 머릿속으로 설계
  → CLI로 수동 설정
    → 작업 완료 후 엑셀에 문서화 (종종 빠뜨림)
      → 시간이 지나면 문서와 실제 설정이 불일치
```

**IaC 방식:**
```
코드로 네트워크 구성 정의 (YAML/JSON)
  → Git에 저장하여 버전 관리
    → 자동화 도구(Ansible)로 배포
      → 코드 = 문서 = 실제 설정 (항상 일치)
```

#### 핵심 개념: IaC의 원칙

| 원칙 | 설명 |
|------|------|
| 선언적 정의 | "이렇게 되어야 한다"를 코드로 정의 (절차가 아닌 상태) |
| 버전 관리 | 모든 변경 이력이 Git에 기록됨 |
| 재현성 | 같은 코드로 동일한 환경을 반복 생성 가능 |
| 자동화 | 사람의 수동 개입 최소화 |
| 테스트 가능 | 배포 전 코드 검증 가능 |

```yaml
# IaC 방식의 네트워크 정의 예시
# site-config.yml

site:
  name: Seoul-Branch
  location: "서울특별시 강남구"

  vlans:
    - id: 10
      name: MANAGEMENT
      subnet: 10.10.10.0/24
    - id: 20
      name: EMPLOYEES
      subnet: 10.10.20.0/24
    - id: 30
      name: GUEST
      subnet: 10.10.30.0/24

  switches:
    - hostname: SEL-ACC-SW-01
      model: C9200L-24P
      interfaces:
        - name: Gi1/0/1-12
          vlan: 20
          mode: access
        - name: Gi1/0/13-24
          vlan: 30
          mode: access

  routing:
    protocol: OSPF
    area: 0
    networks:
      - 10.10.0.0/16
```

이렇게 네트워크를 코드로 정의하면, 새로운 지점을 열 때 기존 코드를 복사하여 변수만 수정하면 됩니다. 동일한 표준 구성이 자동으로 배포되어 일관성이 보장됩니다.

---

### 면접관: "네트워크 엔지니어가 자동화를 시작하려면 어떤 순서로 학습하는 것이 좋을까요?"

네트워크 엔지니어가 자동화를 시작하는 현실적인 로드맵을 제안드립니다.

**1단계: 기본 도구 익히기 (1~2개월)**
- **Linux 기초**: 명령어, 파일 시스템, SSH
- **Git 기초**: add, commit, push, pull, branch
- **YAML/JSON**: 데이터 형식 읽기/쓰기

**2단계: Ansible 시작 (2~3개월)**
- Ansible 설치 및 기본 구성
- 간단한 Playbook 작성 (show 명령어 실행)
- 점진적으로 설정 변경 Playbook으로 확장

**3단계: Python 기초 (2~3개월)**
- Python 기본 문법 (변수, 조건문, 반복문, 함수)
- Netmiko로 SSH 접속 및 명령어 실행
- REST API 호출 (requests 라이브러리)

**4단계: 실무 적용 (지속)**
- 일상적인 반복 작업부터 자동화 시작
- 설정 백업 자동화, 상태 점검 자동화
- 점진적으로 설정 변경 자동화로 확장

#### 핵심 개념: 자동화 시작 시 권장 사항

| 권장 사항 | 설명 |
|----------|------|
| 작게 시작하기 | 전체 네트워크가 아닌 테스트 장비 1~2대부터 |
| 읽기부터 시작 | show 명령어 자동화 → 설정 변경 자동화 순서로 |
| 백업 먼저 | 변경 전 항상 설정 백업하는 습관 |
| 문서화 | 작성한 코드에 주석 달기, README 작성 |
| 테스트 환경 활용 | GNS3, CML(Cisco Modeling Lab) 등으로 안전하게 연습 |

```
자동화 성숙도 모델:

Level 0: 완전 수동 (CLI만 사용)
    ↓
Level 1: 스크립트 지원 (간단한 Shell/Python 스크립트)
    ↓
Level 2: 도구 기반 (Ansible Playbook 활용)
    ↓
Level 3: CI/CD 통합 (자동 테스트 + 배포)
    ↓
Level 4: 완전 자동화 (Self-Healing Network)
```

가장 중요한 것은 **완벽한 자동화를 처음부터 목표로 하지 않는 것**입니다. Level 0에서 Level 1로 가는 것만으로도 업무 효율이 크게 향상됩니다. "매일 30분씩 반복하는 작업"을 찾아서 자동화하는 것부터 시작하면 됩니다.

---

### 면접관: "마지막으로, 네트워크 자동화의 위험 요소와 이를 어떻게 관리할 수 있는지 말씀해 주세요."

네트워크 자동화는 큰 이점이 있지만 **잘못 사용하면 대규모 장애를 유발**할 수도 있습니다.

**주요 위험 요소:**

**1. 대량 오류 (Blast Radius)**
- 수동 작업은 1대씩 실수하지만, 자동화는 **수백 대에 동시에 잘못된 설정을 배포**할 수 있습니다
- 대응: 단계적 배포(Canary Deployment), 변경 전 --check 실행

**2. 테스트 부족**
- 충분한 테스트 없이 운영 환경에 바로 적용하면 예상치 못한 문제가 발생할 수 있습니다
- 대응: 테스트 환경 구축, CI/CD 파이프라인에 자동 테스트 포함

**3. 보안 위험**
- 자동화 스크립트에 인증 정보가 평문으로 저장되면 보안 사고로 이어질 수 있습니다
- 대응: Ansible Vault, HashiCorp Vault 등으로 인증 정보 암호화

**4. 지식 편중**
- 자동화를 구축한 사람만 시스템을 이해하고, 다른 팀원은 모르는 상황
- 대응: 문서화, 코드 리뷰 문화, 팀 교육

**5. 과도한 자동화**
- 자동화할 필요 없는 일회성 작업까지 자동화하려다 오히려 시간을 낭비하는 경우
- 대응: "이 작업을 앞으로 몇 번 더 할 것인가?"를 기준으로 판단

#### 핵심 개념: 안전한 자동화를 위한 원칙

| 원칙 | 실천 방법 |
|------|----------|
| 점진적 배포 | 1대 → 10대 → 전체 순서로 배포 |
| 롤백 계획 | 변경 전 반드시 백업, 복원 절차 수립 |
| 변경 창 준수 | 업무 시간 외 유지보수 시간에 배포 |
| 코드 리뷰 | 동료가 Playbook을 검토한 후 실행 |
| 모니터링 | 배포 후 네트워크 상태 자동 확인 |

자동화는 "도구"일 뿐이며, 결국 **네트워크에 대한 깊은 이해가 바탕**이 되어야 합니다. 자동화 도구를 잘 다루는 것도 중요하지만, 그 전에 네트워크 기본기를 탄탄히 다지는 것이 우선입니다.
