# 07-02. REST API와 JSON/YAML

> 네트워크 엔지니어 면접 시나리오

> CCNA 200-301 수준의 REST API, JSON, YAML 관련 면접 질문과 답변

---

### 면접관: "REST API가 무엇인지, 그리고 네트워크 관리에서 왜 중요한지 설명해 주세요."

REST(Representational State Transfer) API는 **HTTP 프로토콜을 기반으로 클라이언트와 서버 간에 데이터를 주고받는 인터페이스**입니다.

네트워크 관리에서 REST API가 중요한 이유는 다음과 같습니다.

**전통적 방식의 한계:**
- CLI로 장비에 접속하여 수동으로 `show` 명령어를 입력하고 텍스트 출력을 해석
- 장비가 많아지면 반복 작업이 비효율적
- 출력 형식이 일정하지 않아 자동화가 어려움

**REST API의 장점:**
- **구조화된 데이터**(JSON/XML)로 응답을 받아 프로그램에서 쉽게 처리 가능
- HTTP 메서드로 **표준화된 동작** 수행 가능
- 다양한 프로그래밍 언어에서 **쉽게 호출** 가능
- **자동화 스크립트**와 연동하여 대규모 네트워크 관리 가능

#### 핵심 개념: HTTP 메서드와 CRUD 매핑

| HTTP 메서드 | CRUD 동작 | 네트워크 관리 예시 |
|------------|----------|-----------------|
| GET | Read (조회) | 인터페이스 상태 조회, 라우팅 테이블 확인 |
| POST | Create (생성) | 새로운 VLAN 생성, ACL 추가 |
| PUT | Update (전체 수정) | 인터페이스 설정 전체 변경 |
| PATCH | Update (부분 수정) | 특정 설정값만 변경 |
| DELETE | Delete (삭제) | VLAN 삭제, ACL 항목 제거 |

---

### 면접관: "HTTP 상태 코드에 대해 설명해 주세요. API 호출 시 어떤 코드를 자주 보게 되나요?"

HTTP 상태 코드는 API 요청에 대한 **서버의 응답 결과**를 나타내는 3자리 숫자입니다.

#### 핵심 비교: 주요 상태 코드

| 상태 코드 | 의미 | 설명 |
|----------|------|------|
| **200** | OK | 요청 성공, 데이터 반환 |
| **201** | Created | 새로운 리소스 생성 성공 (POST 응답) |
| **204** | No Content | 요청 성공, 반환 데이터 없음 (DELETE 응답) |
| **400** | Bad Request | 잘못된 요청 (문법 오류, 잘못된 파라미터) |
| **401** | Unauthorized | 인증 실패 (토큰 없음 또는 만료) |
| **403** | Forbidden | 권한 없음 (인증은 됐지만 접근 불가) |
| **404** | Not Found | 요청한 리소스가 존재하지 않음 |
| **500** | Internal Server Error | 서버 내부 오류 |

실무에서 API를 다룰 때 가장 자주 만나는 것은 **200(성공)**, **401(인증 문제)**, **404(경로 오류)**입니다. 특히 401 에러는 API 토큰 만료 시 자주 발생하므로, 자동화 스크립트에서 토큰 갱신 로직을 포함하는 것이 좋습니다.

---

### 면접관: "JSON 형식에 대해 설명하고, 네트워크 장비 정보를 JSON으로 표현하는 예시를 보여주세요."

JSON(JavaScript Object Notation)은 **사람이 읽기 쉽고 기계가 파싱하기 쉬운 데이터 교환 형식**입니다. REST API에서 가장 많이 사용되는 데이터 형식이기도 합니다.

#### 핵심 개념: JSON 기본 규칙

- **중괄호 `{}`**: 객체(Object)를 나타냄
- **대괄호 `[]`**: 배열(Array)을 나타냄
- **키-값 쌍**: `"key": "value"` 형태
- 키는 반드시 **큰따옴표**로 감쌈
- 값의 타입: 문자열, 숫자, 불리언, 배열, 객체, null

네트워크 장비 정보를 JSON으로 표현하면 다음과 같습니다.

```json
{
  "device": {
    "hostname": "SW-CORE-01",
    "ip_address": "192.168.1.1",
    "device_type": "switch",
    "vendor": "Cisco",
    "model": "Catalyst 9300",
    "os_version": "IOS-XE 17.6",
    "interfaces": [
      {
        "name": "GigabitEthernet1/0/1",
        "status": "up",
        "speed": "1Gbps",
        "vlan": 10,
        "description": "Server-Web-01"
      },
      {
        "name": "GigabitEthernet1/0/2",
        "status": "down",
        "speed": "1Gbps",
        "vlan": 20,
        "description": "Server-DB-01"
      }
    ],
    "uptime_days": 45,
    "managed": true
  }
}
```

이 JSON을 프로그래밍에서 파싱하면 `device.interfaces[0].name`으로 "GigabitEthernet1/0/1"이라는 값을 바로 추출할 수 있습니다. CLI의 `show interfaces` 출력을 텍스트 파싱하는 것보다 훨씬 편리합니다.

---

### 면접관: "YAML 형식은 JSON과 어떻게 다른가요? 같은 정보를 YAML로 표현해 보세요."

YAML(YAML Ain't Markup Language)은 JSON과 마찬가지로 데이터를 구조화하는 형식이지만, **들여쓰기 기반**으로 더 읽기 쉬운 것이 특징입니다.

#### 핵심 비교: JSON vs YAML

| 구분 | JSON | YAML |
|------|------|------|
| 구분자 | 중괄호 `{}`, 대괄호 `[]` | 들여쓰기 (공백) |
| 가독성 | 보통 | 매우 좋음 |
| 주석 | 지원하지 않음 | `#`으로 주석 가능 |
| 사용처 | REST API 응답, 설정 파일 | Ansible Playbook, Docker Compose |
| 키 따옴표 | 필수 | 선택 (보통 생략) |

동일한 네트워크 장비 정보를 YAML로 표현하면 이렇습니다.

```yaml
# 네트워크 장비 정보
device:
  hostname: SW-CORE-01
  ip_address: 192.168.1.1
  device_type: switch
  vendor: Cisco
  model: Catalyst 9300
  os_version: "IOS-XE 17.6"
  interfaces:
    - name: GigabitEthernet1/0/1
      status: up
      speed: 1Gbps
      vlan: 10
      description: Server-Web-01
    - name: GigabitEthernet1/0/2
      status: down
      speed: 1Gbps
      vlan: 20
      description: Server-DB-01
  uptime_days: 45
  managed: true
```

YAML은 **Ansible Playbook 작성 시 주로 사용**되며, 네트워크 자동화 분야에서 매우 자주 접하게 됩니다. JSON은 API 통신에, YAML은 설정 파일에 주로 사용된다고 이해하시면 됩니다.

---

### 면접관: "API 인증 방식에 대해 설명해 주세요. 토큰 기반 인증은 어떻게 동작하나요?"

API 인증은 **허가된 사용자만 API에 접근할 수 있도록 하는 메커니즘**입니다. 주요 인증 방식은 다음과 같습니다.

**1. API Key 방식**
- 서버에서 발급한 고유 키를 HTTP 헤더에 포함하여 전송
- 간단하지만 키가 노출되면 보안 위험

**2. Basic Authentication**
- 사용자 이름과 비밀번호를 Base64로 인코딩하여 전송
- HTTPS와 함께 사용해야 안전

**3. Token 기반 인증 (가장 일반적)**
- 먼저 사용자 인증 후 **토큰을 발급**받음
- 이후 API 호출마다 토큰을 헤더에 포함
- 토큰에는 **만료 시간**이 있어 보안성이 높음

토큰 기반 인증 흐름 예시:

```
1단계: 토큰 발급 요청
POST /api/auth/token
Body: {"username": "admin", "password": "cisco123"}

응답:
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 3600
}

2단계: 발급받은 토큰으로 API 호출
GET /api/network/devices
Header: Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Cisco DNA Center 같은 장비에서도 이러한 토큰 기반 인증을 사용합니다.

---

### 면접관: "curl 명령어를 사용하여 네트워크 장비의 API를 호출하는 실제 예시를 보여줄 수 있나요?"

네, curl은 **CLI에서 HTTP 요청을 보낼 수 있는 도구**로 API 테스트에 자주 사용됩니다.

#### 핵심 설정: curl 기본 옵션

| 옵션 | 설명 |
|------|------|
| `-X` | HTTP 메서드 지정 (GET, POST 등) |
| `-H` | HTTP 헤더 추가 |
| `-d` | 요청 본문(Body) 데이터 전송 |
| `-k` | SSL 인증서 검증 무시 (테스트용) |
| `-u` | Basic Auth 인증 정보 |

```bash
# 1. 토큰 발급 요청 (Cisco DNA Center 예시)
curl -X POST "https://dnac-server/dna/system/api/v1/auth/token" \
  -u "admin:password" \
  -k

# 응답 예시: {"Token": "eyJ0eXAiOi..."}

# 2. 토큰을 사용하여 네트워크 장비 목록 조회
curl -X GET "https://dnac-server/dna/intent/api/v1/network-device" \
  -H "X-Auth-Token: eyJ0eXAiOi..." \
  -H "Content-Type: application/json" \
  -k

# 3. 새로운 설정 적용 (POST 예시)
curl -X POST "https://dnac-server/dna/intent/api/v1/template-programmer/template" \
  -H "X-Auth-Token: eyJ0eXAiOi..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "VLAN-Config",
    "templateContent": "vlan 100\n name SALES",
    "deviceTypes": [{"productFamily": "Switches"}]
  }' \
  -k
```

curl은 빠른 테스트에 유용하지만, 실제 자동화에서는 **Python의 requests 라이브러리**를 더 많이 사용합니다.

---

### 면접관: "Python으로 REST API를 호출하는 간단한 예시를 보여주시겠습니까?"

네, Python의 `requests` 라이브러리를 사용하면 매우 간결하게 API를 호출할 수 있습니다.

```python
import requests
import json

# SSL 경고 비활성화 (테스트 환경에서만 사용)
requests.packages.urllib3.disable_warnings()

# 변수 설정
BASE_URL = "https://192.168.1.100"
USERNAME = "admin"
PASSWORD = "cisco123"

# 1단계: 토큰 발급
auth_url = f"{BASE_URL}/dna/system/api/v1/auth/token"
response = requests.post(auth_url, auth=(USERNAME, PASSWORD), verify=False)
token = response.json()["Token"]

# 2단계: 장비 목록 조회
headers = {
    "X-Auth-Token": token,
    "Content-Type": "application/json"
}

devices_url = f"{BASE_URL}/dna/intent/api/v1/network-device"
response = requests.get(devices_url, headers=headers, verify=False)

# 3단계: 응답 데이터 처리
if response.status_code == 200:
    devices = response.json()["response"]
    for device in devices:
        print(f"호스트명: {device['hostname']}")
        print(f"IP 주소: {device['managementIpAddress']}")
        print(f"시리즈:  {device['series']}")
        print("---")
else:
    print(f"오류 발생: {response.status_code}")
```

이 스크립트는 Cisco DNA Center의 API를 호출하여 관리 중인 모든 네트워크 장비의 정보를 출력합니다. CLI에서 각 장비에 접속하여 `show version`을 실행하는 것보다 훨씬 효율적입니다.

---

### 면접관: "API를 활용한 네트워크 관리에서 주의해야 할 점은 무엇인가요?"

API를 활용할 때 주의해야 할 사항은 여러 가지가 있습니다.

**1. 보안**
- API 키나 토큰을 코드에 하드코딩하지 않아야 합니다
- 환경 변수나 별도의 보안 저장소(Vault)에 인증 정보를 보관합니다
- 반드시 HTTPS를 사용하여 통신을 암호화합니다

**2. Rate Limiting (호출 제한)**
- 대부분의 API에는 시간당 호출 횟수 제한이 있습니다
- 제한을 초과하면 **429 Too Many Requests** 오류가 발생합니다
- 적절한 대기 시간(sleep)을 두고 호출해야 합니다

**3. 에러 처리**
- 네트워크 장애, 타임아웃 등 다양한 오류 상황에 대한 처리가 필요합니다
- try-except 구문으로 예외를 처리하고, 재시도 로직을 구현합니다

**4. 버전 관리**
- API 버전이 변경되면 기존 스크립트가 동작하지 않을 수 있습니다
- API 호출 URL에 버전 정보(v1, v2)를 명시하는 것이 좋습니다

**5. 변경 전 검증**
- POST, PUT, DELETE 같은 변경 작업은 반드시 테스트 환경에서 먼저 검증합니다
- 운영 환경 적용 전 dry-run 기능이 있다면 반드시 활용합니다
