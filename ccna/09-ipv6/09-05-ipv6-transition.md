# 09-05. IPv4 → IPv6 전환 기술 (Dual-Stack/Tunneling/NAT64)

> 개요
IPv4에서 IPv6로의 전환을 위한 주요 기술인 Dual-Stack, 터널링(6to4, ISATAP, GRE), NAT64/DNS64의 개념과 적용 시나리오에 대한 면접 시나리오입니다.

---

### 면접관: "회사에서 IPv6 도입을 결정했습니다. 하지만 기존 IPv4 인프라를 즉시 폐기할 수는 없는 상황인데, IPv4에서 IPv6로 전환하는 기술에는 어떤 것들이 있나요?"

IPv4에서 IPv6로의 전환을 위한 세 가지 주요 기술이 있습니다.

| 전환 기술 | 개념 | 적합한 상황 |
|-----------|------|------------|
| Dual-Stack | 장비에서 IPv4와 IPv6를 동시 운영 | 신규 장비 도입 시, 점진적 전환 |
| 터널링 | IPv6 패킷을 IPv4 네트워크에 캡슐화하여 전송 | IPv4 전용 구간을 통과해야 할 때 |
| NAT64/DNS64 | IPv6 전용 호스트가 IPv4 서버와 통신 | IPv6 전용 네트워크에서 레거시 접근 |

이 세 가지 기술은 상호 배타적이지 않으며, 실제 전환 과정에서는 여러 기술을 조합하여 사용합니다.

전환의 일반적인 로드맵은 다음과 같습니다:

```
1단계: Dual-Stack 도입 (IPv4 + IPv6 동시 운영)
2단계: IPv6 우선 전환 (IPv6 선호, IPv4 보조)
3단계: IPv4 축소 (터널링/NAT64로 레거시 지원)
4단계: IPv6 전용 운영 (장기 목표)
```

---

### 면접관: "Dual-Stack에 대해 자세히 설명해 주세요. 실제 설정 방법도 보여주시겠습니까?"

Dual-Stack은 하나의 장비에서 IPv4와 IPv6 프로토콜 스택을 동시에 운영하는 방식입니다. 가장 직관적이고 권장되는 전환 방법입니다.

**Dual-Stack의 특징:**
- 모든 인터페이스에 IPv4 주소와 IPv6 주소를 모두 할당
- IPv4 트래픽은 IPv4로, IPv6 트래픽은 IPv6로 처리
- 애플리케이션이 목적지에 따라 적절한 프로토콜 선택
- DNS에 A 레코드(IPv4)와 AAAA 레코드(IPv6) 모두 등록

**라우터 Dual-Stack 설정:**

```
Router(config)# ip routing
Router(config)# ipv6 unicast-routing

Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip address 192.168.1.1 255.255.255.0
Router(config-if)# ipv6 address 2001:DB8:1::1/64
Router(config-if)# no shutdown

! IPv4 라우팅 프로토콜
Router(config)# router ospf 1
Router(config-router)# router-id 1.1.1.1
Router(config-router)# network 192.168.1.0 0.0.0.255 area 0

! IPv6 라우팅 프로토콜
Router(config)# ipv6 router ospf 1
Router(config-rtr)# router-id 1.1.1.1
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 ospf 1 area 0
```

**Dual-Stack의 장점:**
- 가장 안정적이고 호환성이 높은 방식
- 기존 IPv4 서비스에 영향 없이 IPv6를 추가
- 점진적 전환 가능

**Dual-Stack의 단점:**
- 두 가지 프로토콜 스택을 모두 관리해야 하므로 운영 복잡도 증가
- 장비의 메모리와 CPU 리소스가 더 필요
- 보안 정책도 IPv4와 IPv6 각각에 대해 설정해야 함
- IPv4 주소 부족 문제를 직접적으로 해결하지는 않음

---

### 면접관: "터널링 기술에 대해 설명해 주세요. 6to4 터널과 GRE 터널의 차이점은 무엇인가요?"

터널링은 IPv6 패킷을 IPv4 패킷 안에 캡슐화하여 IPv4 전용 네트워크를 통과시키는 기술입니다.

**주요 터널링 방식:**

| 터널 유형 | 설정 방식 | 특징 |
|-----------|----------|------|
| GRE 터널 | 수동 (point-to-point) | 가장 유연하고 널리 사용 |
| 6to4 | 자동 | IPv4 주소 기반으로 자동 터널 |
| ISATAP | 자동 | 인트라 사이트 자동 터널 |

**GRE 터널 설정 예시:**

두 개의 IPv6 네트워크가 IPv4 전용 백본을 통해 연결되는 시나리오입니다.

```
[IPv6 Site A] --- R1 ===== IPv4 백본 ===== R2 --- [IPv6 Site B]
                  1.1.1.1              2.2.2.2
```

**R1 설정:**

```
R1(config)# interface Tunnel0
R1(config-if)# ipv6 address 2001:DB8:FFFF::1/64
R1(config-if)# tunnel source 1.1.1.1
R1(config-if)# tunnel destination 2.2.2.2
R1(config-if)# tunnel mode gre ip

R1(config)# ipv6 route 2001:DB8:B::/48 Tunnel0
```

**R2 설정:**

```
R2(config)# interface Tunnel0
R2(config-if)# ipv6 address 2001:DB8:FFFF::2/64
R2(config-if)# tunnel source 2.2.2.2
R2(config-if)# tunnel destination 1.1.1.1
R2(config-if)# tunnel mode gre ip

R2(config)# ipv6 route 2001:DB8:A::/48 Tunnel0
```

**GRE vs 6to4 비교:**

| 항목 | GRE 터널 | 6to4 터널 |
|------|---------|-----------|
| 설정 | 수동 (터널 양끝 지정) | 자동 (IPv4 주소 기반) |
| 유연성 | 높음 | 제한적 |
| 멀티캐스트 | 지원 | 미지원 |
| 라우팅 프로토콜 | 터널 위에서 실행 가능 | 제한적 |
| 오버헤드 | GRE 헤더 추가 (24바이트) | IPv4 헤더만 추가 (20바이트) |
| 실무 활용도 | 높음 | 점점 감소 (보안 우려) |

---

### 면접관: "6to4 터널이 점점 사용되지 않는다고 하셨는데, 그 이유와 동작 원리를 설명해 주시겠습니까?"

6to4 터널의 동작 원리와 현재 상태를 설명드리겠습니다.

**6to4 동작 원리:**

6to4는 IPv4 공인 주소를 IPv6 프리픽스에 자동으로 매핑합니다.

```
6to4 프리픽스: 2002::/16
IPv4 주소를 16진수로 변환하여 포함

예: IPv4 주소 192.168.1.1
    192 = C0, 168 = A8, 1 = 01, 1 = 01
    6to4 주소: 2002:C0A8:0101::/48
```

**ISATAP 동작 원리:**

ISATAP(Intra-Site Automatic Tunnel Addressing Protocol)은 IPv4 인트라넷 내에서 IPv6 연결을 제공합니다.

```
ISATAP 인터페이스 ID: ::0000:5EFE:IPv4주소
예: IPv4 192.168.1.100
    인터페이스 ID: ::5EFE:C0A8:164
```

**6to4가 감소하는 이유:**

1. **릴레이 라우터 의존성**: 6to4 트래픽이 인터넷을 통과하려면 6to4 릴레이 라우터를 경유해야 하는데, 이 릴레이의 가용성과 성능이 보장되지 않습니다
2. **보안 문제**: 릴레이 라우터를 통한 중간자 공격 가능성
3. **비대칭 라우팅**: 패킷이 예상하지 못한 경로로 전달될 수 있음
4. **NAT 환경 호환 문제**: NAT 뒤에서는 6to4가 제대로 동작하지 않음
5. **Dual-Stack 확산**: 많은 네트워크가 이미 Dual-Stack을 지원하여 터널링 필요성이 줄어듦

RFC 7526에서는 6to4를 더 이상 사용하지 말 것을 권고하고 있습니다. CCNA 시험에서는 개념 이해 수준에서 출제될 수 있습니다.

---

### 면접관: "NAT64와 DNS64는 어떤 기술이고, 어떤 상황에서 사용하나요?"

NAT64와 DNS64는 IPv6 전용 호스트가 IPv4 전용 서버와 통신할 수 있게 해주는 기술입니다.

**NAT64 개념:**

```
[IPv6 전용 클라이언트] → [NAT64 장비] → [IPv4 전용 서버]
   2001:DB8::10           변환 수행         203.0.113.50
```

NAT64는 IPv6 패킷을 IPv4 패킷으로 변환(또는 그 반대)합니다. IPv6 주소와 IPv4 주소 사이의 변환 테이블을 유지합니다.

**DNS64 개념:**

DNS64는 NAT64와 함께 동작하며, IPv4 전용 서버의 A 레코드를 합성된 AAAA 레코드로 변환합니다.

```
동작 과정:
1. IPv6 클라이언트가 www.example.com의 AAAA 레코드 요청
2. DNS64 서버가 실제 DNS에 AAAA 레코드 조회 → 없음
3. DNS64 서버가 A 레코드 조회 → 203.0.113.50
4. DNS64가 합성 AAAA 레코드 생성:
   64:ff9b::203.0.113.50 = 64:ff9b::CB00:7132
5. 클라이언트에게 합성 AAAA 레코드 응답
6. 클라이언트가 64:ff9b::CB00:7132로 패킷 전송
7. NAT64 장비가 이를 IPv4 203.0.113.50으로 변환
```

**NAT64의 유형:**

| 유형 | 설명 | 사용 시나리오 |
|------|------|-------------|
| Stateless NAT64 | 1:1 매핑, 상태 테이블 없음 | 서버에 고정 매핑이 필요한 경우 |
| Stateful NAT64 | N:1 매핑, 상태 테이블 유지 | 다수 클라이언트가 IPv4 접근 필요 |

CCNA 수준에서는 NAT64와 DNS64의 개념과 왜 필요한지를 이해하는 것이 중요합니다.

---

### 면접관: "각 전환 기술을 어떤 시나리오에서 선택해야 하는지 정리해 주시겠습니까?"

네, 시나리오별 적합한 전환 기술을 정리하겠습니다.

**시나리오별 전환 기술 선택 가이드:**

| 시나리오 | 권장 기술 | 이유 |
|----------|----------|------|
| 신규 네트워크 구축 | Dual-Stack | 처음부터 두 프로토콜 지원 |
| IPv6 섬(island) 간 연결 | GRE 터널 | IPv4 백본을 통과하여 IPv6 사이트 연결 |
| IPv6 전용 모바일 네트워크 | NAT64/DNS64 | IPv4 인터넷 리소스 접근 필요 |
| 기업 내부 전환 초기 | Dual-Stack | 기존 서비스 영향 최소화 |
| 데이터센터 간 연결 | GRE 터널 또는 Dual-Stack | 안정성과 유연성 |
| ISP 환경 | Dual-Stack + NAT64 | 고객에게 IPv6 제공하면서 레거시 호환 |

**실무 전환 전략 수립 시 고려사항:**

1. **현재 인프라 분석**:
   - 기존 장비가 IPv6를 지원하는지 확인
   - 네트워크 장비의 소프트웨어 버전 확인
   - IPv6 지원을 위한 업그레이드 필요 여부 판단

2. **주소 계획**:
   - ISP로부터 IPv6 주소 블록 할당 받기
   - 내부 서브넷 체계 설계 (/48 → /64 계획)

3. **보안 정책 수립**:
   - IPv6 방화벽 규칙 설정
   - IPv6 ACL 구성
   - ICMPv6 허용 정책 (NDP에 필수)

4. **모니터링 체계**:
   - IPv6 트래픽 모니터링 도구 구축
   - 듀얼 스택 환경 통합 모니터링

---

### 면접관: "전환 기간 동안 보안 관점에서 특별히 주의해야 할 점은 무엇인가요?"

전환 기간의 보안 이슈는 매우 중요한 주제입니다.

**주요 보안 주의사항:**

1. **의도하지 않은 IPv6 터널**: Windows 등 운영체제에서 자동으로 6to4나 Teredo 터널이 활성화될 수 있습니다. 이러한 자동 터널은 보안 정책을 우회할 수 있으므로 비활성화해야 합니다.

2. **듀얼 스택 환경의 이중 보안**: IPv4와 IPv6 각각에 대해 보안 정책을 적용해야 합니다. IPv4에만 방화벽 규칙을 적용하고 IPv6를 무시하면 보안 구멍이 생깁니다.

```
! IPv4 ACL과 IPv6 ACL 모두 필요
R1(config)# ip access-list extended IPv4-FILTER
R1(config-ext-nacl)# deny ip any 10.0.0.0 0.255.255.255
R1(config-ext-nacl)# permit ip any any

R1(config)# ipv6 access-list IPv6-FILTER
R1(config-ipv6-acl)# deny ipv6 any 2001:DB8:BAD::/48
R1(config-ipv6-acl)# permit ipv6 any any
```

3. **Rogue RA(불법 라우터 광고) 방지**: 악의적인 호스트가 RA 메시지를 보내 기본 게이트웨이를 변조할 수 있습니다. RA Guard를 적용해야 합니다.

```
! 스위치에서 RA Guard 설정
Switch(config)# ipv6 nd raguard policy HOST-POLICY
Switch(config-nd-raguard)# device-role host
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# ipv6 nd raguard attach-policy HOST-POLICY
```

4. **터널 트래픽 검사**: GRE 등 터널을 사용하는 경우, 터널 내부의 패킷도 검사할 수 있는 보안 장비가 필요합니다.

5. **IPv6 스캐닝**: IPv6의 거대한 주소 공간은 전통적인 네트워크 스캐닝을 어렵게 하지만, EUI-64 패턴이나 알려진 주소 패턴(::1, ::2 등)을 기반으로 한 스캐닝은 가능합니다.

전환 기간은 보안 취약점이 가장 많이 발생하는 시기이므로, 단계별로 충분한 보안 검증을 수행하는 것이 중요합니다.

---

### 면접관: "마지막으로, 공존(coexistence) 기간을 효율적으로 관리하기 위한 모범 사례를 정리해 주세요."

듀얼 스택 공존 기간의 관리 모범 사례를 정리하겠습니다.

**운영 관리:**

1. **단계적 접근**: 한 번에 전체 네트워크를 전환하지 말고, 세그먼트별로 순차 전환
2. **파일럿 네트워크**: 소규모 테스트 네트워크에서 먼저 검증 후 확대
3. **롤백 계획**: 문제 발생 시 IPv4로 즉시 롤백할 수 있는 계획 수립
4. **문서화**: IPv6 주소 체계, 라우팅 정책, 보안 규칙 등을 철저히 문서화

**기술적 관리:**

```
! 정기적인 상태 확인 스크립트 항목
show ipv6 interface brief       ! IPv6 인터페이스 상태
show ipv6 route summary         ! IPv6 라우팅 테이블 요약
show ipv6 ospf neighbor         ! OSPFv3 네이버 상태
show ipv6 traffic               ! IPv6 트래픽 통계
show ipv6 neighbors             ! NDP 캐시 확인
```

**교육 및 인력:**
- 네트워크 엔지니어에 대한 IPv6 교육 필수
- IPv4와 IPv6를 모두 다룰 수 있는 운영 인력 확보
- 헬프데스크 직원에게도 기본적인 IPv6 트러블슈팅 교육

공존 기간은 수년이 걸릴 수 있으므로, 장기적인 관점에서 체계적으로 관리하는 것이 중요합니다.
