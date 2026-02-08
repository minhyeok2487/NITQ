# NITQ (Network Interview Technical Questions)

### 네트워크 엔지니어 기술 면접 준비

네트워크 엔지니어 기술 면접 준비를 위한 시나리오 기반 학습 자료입니다.

단순한 개념 암기가 아닌, 실무 시나리오를 기반으로 토폴로지를 설계하고 이론을 설명하는 방식으로 구성되어 있습니다.

### 문서 구조

각 문서는 면접관-면접자 대화 흐름으로 구성됩니다.

1. **고객 요구/상황 제시** - 면접관이 실무 시나리오를 제시
2. **설계 답변** - 토폴로지 + 핵심 설정
3. **꼬리물기 이론 질문** - 설계에서 파생되는 심화 질문
4. **장애 발생** - 구성 후 발생할 수 있는 장애 상황
5. **트러블슈팅** - 확인 명령어와 원인 분석
6. **추가 요구사항** - 고객의 확장/변경 요청 대응

### 난이도

CCNP ENCOR/ENARSI + Checkpoint CCSA/CCSE 수준을 기준으로 합니다.

### 카테고리

| 카테고리 | 주제 | 문서 수 |
|---|---|---|
| [01 Routing](01-routing/) | OSPF, EIGRP, BGP, Redistribution, VRF, PBR | 8개 |
| [02 Switching](02-switching/) | VLAN, STP/RSTP/MST, EtherChannel, PVLAN, DHCP Snooping | 5개 |
| [03 Firewall - Cisco](03-firewall-cisco/) | ASA, NAT, ACL/MPF, FTD/FMC, ZBF | 5개 |
| [04 Firewall - Checkpoint](04-firewall-checkpoint/) | GAiA, Policy, NAT, SIC, Logging, App Control | 6개 |
| [05 VPN](05-vpn/) | IPSec, GRE, DMVPN, SSL VPN, Checkpoint RA VPN | 6개 |
| [06 HA & Redundancy](06-ha-redundancy/) | HSRP/VRRP/GLBP, ASA Failover, ClusterXL | 5개 |
| [07 Security](07-security/) | AAA, 802.1X, IPS/IDS, uRPF, CoPP | 5개 |
| [08 Troubleshooting](08-troubleshooting/) | OSPF, BGP, VPN, NAT, Checkpoint, 패킷 캡처 | 6개 |
| [09 Design](09-design/) | 3-Tier, DMZ, 망분리, Spine-Leaf, DR | 5개 |

### 참고 사항

- 토폴로지는 GNS3 / Packet Tracer / draw.io로 직접 구성합니다.
- CLI 설정은 반드시 랩 환경에서 검증 후 반영합니다.
- 내용 오류 발견 시 Issue 또는 PR로 알려주시면 감사하겠습니다.
