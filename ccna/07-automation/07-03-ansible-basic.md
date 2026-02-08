# 07-03. Ansible 기본

> 네트워크 엔지니어 면접 시나리오

> CCNA 200-301 수준의 Ansible 기본 개념 관련 면접 질문과 답변

---

### 면접관: "Ansible이 무엇이고, 네트워크 자동화에서 왜 많이 사용되는지 설명해 주세요."

Ansible은 **Red Hat에서 개발한 오픈소스 IT 자동화 도구**입니다. 네트워크 자동화 분야에서 특히 인기가 높은 이유는 다음과 같습니다.

**Ansible의 핵심 특징:**

**1. Agentless (에이전트 불필요)**
- 관리 대상 장비에 별도의 소프트웨어(에이전트)를 설치할 필요가 없습니다
- 서버 관리 시에는 **SSH**, 네트워크 장비 관리 시에도 **SSH**를 사용합니다
- Chef나 Puppet과 달리 에이전트 설치/관리 부담이 없습니다

**2. YAML 기반 (쉬운 문법)**
- Playbook을 **YAML**로 작성하므로 프로그래밍 경험이 없어도 이해하기 쉽습니다
- 네트워크 엔지니어가 Python을 몰라도 시작할 수 있습니다

**3. 멱등성 (Idempotency)**
- 같은 Playbook을 여러 번 실행해도 **결과가 동일**합니다
- 이미 적용된 설정은 다시 적용하지 않아 안전합니다

**4. 풍부한 네트워크 모듈**
- Cisco IOS, NX-OS, Juniper, Arista 등 **다양한 벤더의 모듈**을 기본 제공합니다

#### 핵심 비교: Ansible vs 수동 CLI 작업

| 구분 | 수동 CLI | Ansible |
|------|---------|---------|
| 30대 스위치 VLAN 추가 | 30번 SSH 접속, 30번 명령어 입력 | 1번 Playbook 실행 |
| 소요 시간 | 수 시간 | 수 분 |
| 실수 가능성 | 높음 (타이핑 실수) | 낮음 (코드로 검증) |
| 기록 관리 | 수동 문서화 | Playbook 자체가 문서 |
| 재현성 | 보장 불가 | 동일 결과 보장 |

---

### 면접관: "Ansible의 Inventory 파일이 무엇인지, 그리고 어떻게 작성하는지 보여주세요."

Inventory 파일은 **Ansible이 관리할 대상 장비들의 목록**을 정의하는 파일입니다. 어떤 장비에 어떻게 접속할지에 대한 정보를 담고 있습니다.

#### 핵심 설정: INI 형식 Inventory

```ini
# inventory.ini - 기본 INI 형식

[core_switches]
core-sw-01 ansible_host=192.168.1.1
core-sw-02 ansible_host=192.168.1.2

[access_switches]
acc-sw-01 ansible_host=192.168.2.1
acc-sw-02 ansible_host=192.168.2.2
acc-sw-03 ansible_host=192.168.2.3

[routers]
rt-01 ansible_host=10.0.0.1
rt-02 ansible_host=10.0.0.2

# 그룹의 그룹 정의
[all_switches:children]
core_switches
access_switches

# 그룹별 공통 변수
[all_switches:vars]
ansible_network_os=ios
ansible_connection=network_cli
ansible_user=admin
ansible_password=cisco123
```

#### 핵심 설정: YAML 형식 Inventory

```yaml
# inventory.yml - YAML 형식
all:
  children:
    core_switches:
      hosts:
        core-sw-01:
          ansible_host: 192.168.1.1
        core-sw-02:
          ansible_host: 192.168.1.2
      vars:
        ansible_network_os: ios
        ansible_connection: network_cli
    routers:
      hosts:
        rt-01:
          ansible_host: 10.0.0.1
```

Inventory에서 그룹을 잘 나누면, 특정 그룹에만 선택적으로 작업을 실행할 수 있어 매우 유용합니다. 예를 들어 코어 스위치에만 특정 설정을 적용하고 싶다면 `core_switches` 그룹을 대상으로 지정하면 됩니다.

---

### 면접관: "Ansible Playbook의 구조를 설명하고, 간단한 예시를 보여주세요."

Playbook은 **Ansible에서 자동화 작업을 정의하는 YAML 파일**입니다. "어떤 장비에, 어떤 작업을, 어떤 순서로 실행할지"를 정의합니다.

#### 핵심 개념: Playbook 구성 요소

| 구성 요소 | 설명 |
|----------|------|
| `name` | Play 또는 Task의 이름 (설명) |
| `hosts` | 작업 대상 호스트/그룹 |
| `gather_facts` | 장비 정보 자동 수집 여부 |
| `tasks` | 실행할 작업 목록 |
| `vars` | 변수 정의 |
| `handlers` | 특정 조건에서만 실행되는 작업 |

```yaml
# vlan_config.yml - VLAN 설정 Playbook
---
- name: 액세스 스위치에 VLAN 설정 적용
  hosts: access_switches
  gather_facts: no

  vars:
    vlans:
      - id: 100
        name: SALES
      - id: 200
        name: ENGINEERING
      - id: 300
        name: MANAGEMENT

  tasks:
    - name: VLAN 생성
      cisco.ios.ios_vlans:
        config:
          - vlan_id: "{{ item.id }}"
            name: "{{ item.name }}"
            state: active
        state: merged
      loop: "{{ vlans }}"

    - name: 설정 저장
      cisco.ios.ios_command:
        commands:
          - write memory

    - name: VLAN 설정 확인
      cisco.ios.ios_command:
        commands:
          - show vlan brief
      register: vlan_output

    - name: VLAN 결과 출력
      debug:
        var: vlan_output.stdout_lines
```

이 Playbook을 실행하면 `access_switches` 그룹의 모든 스위치에 VLAN 100, 200, 300이 자동으로 생성됩니다. 스위치가 3대든 30대든 **한 번의 실행으로 모두 적용**됩니다.

---

### 면접관: "ios_command와 ios_config 모듈의 차이점을 설명해 주세요."

이 두 모듈은 Cisco IOS 장비를 관리할 때 가장 많이 사용하는 Ansible 모듈입니다.

#### 핵심 비교

| 구분 | ios_command | ios_config |
|------|-----------|------------|
| 용도 | **show 명령어** 실행 (조회) | **설정 변경** (configuration mode) |
| 모드 | Privileged EXEC 모드 | Global Configuration 모드 |
| 변경 여부 | 읽기 전용 (변경하지 않음) | 설정을 변경함 |
| 멱등성 | 해당 없음 (항상 실행) | 지원 (이미 있는 설정은 건너뜀) |

```yaml
# ios_command 사용 예시 - 장비 정보 조회
- name: 인터페이스 상태 확인
  cisco.ios.ios_command:
    commands:
      - show ip interface brief
      - show running-config | section interface
  register: output

# ios_config 사용 예시 - 설정 변경
- name: 인터페이스 설정
  cisco.ios.ios_config:
    lines:
      - description Connected to Server-01
      - switchport mode access
      - switchport access vlan 100
      - no shutdown
    parents: interface GigabitEthernet1/0/1

# ios_config에서 설정 저장
- name: 설정 저장
  cisco.ios.ios_config:
    save_when: modified
```

`ios_config`의 가장 큰 장점은 **멱등성**입니다. 이미 `switchport access vlan 100`이 설정되어 있다면 다시 적용하지 않습니다. 이는 불필요한 변경을 방지하여 안전한 자동화를 가능하게 합니다.

---

### 면접관: "멱등성(Idempotency)이 네트워크 자동화에서 왜 중요한가요?"

멱등성은 **같은 작업을 여러 번 실행해도 결과가 동일한 성질**을 말합니다.

네트워크 자동화에서 이것이 중요한 이유를 예를 들어 설명하겠습니다.

**멱등성이 없는 경우 (위험):**
```
# 단순 스크립트 - 실행할 때마다 추가됨
interface GigabitEthernet1/0/1
 ip address 10.0.1.1 255.255.255.0   ← 이미 있어도 또 실행
 ip address 10.0.1.1 255.255.255.0   ← 중복 실행
```

**멱등성이 있는 경우 (안전):**
```
# Ansible - 이미 설정되어 있으면 건너뜀
TASK [인터페이스 설정] *****
ok: [sw-01]    ← changed가 아닌 ok (변경 없음)
```

실무적으로 중요한 이유는 다음과 같습니다.

- **장애 복구**: Playbook 실행 중 일부 장비에서 실패해도, 다시 실행하면 성공한 장비는 건너뛰고 실패한 장비만 재시도됩니다
- **정기 실행**: 설정이 변경되지 않은 장비는 그대로 유지되므로, 주기적으로 Playbook을 실행하여 **설정 일관성을 유지**할 수 있습니다
- **안전성**: 실수로 두 번 실행해도 문제가 발생하지 않습니다

Ansible에서 멱등성이 보장되는 모듈은 실행 결과가 `changed`(변경됨) 또는 `ok`(이미 적용됨)로 표시되어, 실제로 무엇이 변경되었는지 명확하게 파악할 수 있습니다.

---

### 면접관: "Ad-hoc 명령어란 무엇이고, 언제 사용하나요?"

Ad-hoc 명령어는 **Playbook을 작성하지 않고 CLI에서 직접 Ansible 명령어를 실행**하는 방식입니다. 일회성 작업이나 빠른 확인에 유용합니다.

```bash
# 모든 스위치에 ping 테스트
ansible all_switches -m ping -i inventory.ini

# 특정 그룹의 장비에 show 명령어 실행
ansible core_switches -m ios_command \
  -a "commands='show ip interface brief'" \
  -i inventory.ini

# 모든 라우터의 버전 확인
ansible routers -m ios_command \
  -a "commands='show version'" \
  -i inventory.ini

# 특정 장비 하나에만 명령어 실행
ansible core-sw-01 -m ios_command \
  -a "commands='show running-config'" \
  -i inventory.ini
```

#### 핵심 비교: Ad-hoc vs Playbook

| 구분 | Ad-hoc | Playbook |
|------|--------|----------|
| 사용 시기 | 빠른 확인, 일회성 작업 | 반복 작업, 복잡한 워크플로 |
| 작성 방식 | CLI 한 줄 | YAML 파일 |
| 재사용성 | 낮음 | 높음 (파일로 저장) |
| 복잡도 | 단순한 작업만 가능 | 조건, 반복, 핸들러 등 복잡한 로직 가능 |
| 버전 관리 | 어려움 | Git으로 관리 가능 |

실무에서는 "지금 당장 모든 스위치의 상태를 확인해야 할 때"처럼 급한 조회 작업에 Ad-hoc을 사용하고, 반복적으로 수행할 설정 작업은 Playbook으로 작성합니다.

---

### 면접관: "Ansible Playbook을 처음 작성할 때 가장 중요한 주의사항은 무엇인가요?"

Ansible Playbook을 작성할 때 특히 주의해야 할 사항들입니다.

**1. YAML 문법 주의**
```yaml
# 올바른 예시 (공백 2칸 들여쓰기)
- name: VLAN 설정
  hosts: switches
  tasks:
    - name: VLAN 생성
      cisco.ios.ios_vlans:
        config:
          - vlan_id: 100

# 잘못된 예시 (탭 사용 - 오류 발생)
- name: VLAN 설정
	hosts: switches    # 탭은 YAML에서 허용되지 않음!
```

**2. 테스트 환경에서 먼저 실행**
- `--check` 모드(Dry Run)를 사용하여 실제 변경 없이 미리 확인합니다
- `--limit` 옵션으로 특정 장비 1대에만 먼저 테스트합니다

```bash
# Dry Run (실제 변경 없이 확인)
ansible-playbook vlan_config.yml --check

# 특정 장비 1대에만 테스트
ansible-playbook vlan_config.yml --limit core-sw-01

# 단계별 확인하며 실행
ansible-playbook vlan_config.yml --step
```

**3. 변수와 비밀번호 관리**
- 비밀번호는 Playbook에 직접 쓰지 않고, **Ansible Vault**를 사용하여 암호화합니다
- 환경별로 다른 변수는 `group_vars/`, `host_vars/` 디렉토리에 분리합니다

**4. 설정 백업 먼저**
- 설정을 변경하기 전에 반드시 **현재 설정을 백업**하는 Task를 포함합니다

```yaml
- name: 설정 변경 전 백업
  cisco.ios.ios_config:
    backup: yes
    backup_options:
      dir_path: ./backups
      filename: "{{ inventory_hostname }}_backup.cfg"
```

이러한 주의사항을 지키면 안전하고 체계적인 네트워크 자동화를 시작할 수 있습니다.
