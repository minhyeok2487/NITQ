# Table of contents

* [Welcome](README.md)

## 01 Routing

* [01-01. OSPF Multi-Area 설계](01-routing/01-01-ospf-multi-area.md)
* [01-02. OSPF 인증과 Stub Area](01-routing/01-02-ospf-stub-authentication.md)
* [01-03. EIGRP 고급 설정과 마이그레이션](01-routing/01-03-eigrp-advanced.md)
* [01-04. BGP 기본 구성과 Peering](01-routing/01-04-bgp-peering.md)
* [01-05. BGP Path Attribute와 경로 제어](01-routing/01-05-bgp-path-attribute.md)
* [01-06. Route Redistribution](01-routing/01-06-route-redistribution.md)
* [01-07. VRF (Virtual Routing and Forwarding)](01-routing/01-07-vrf.md)
* [01-08. PBR (Policy-Based Routing)](01-routing/01-08-pbr.md)

## 02 Switching

* [02-01. VLAN 설계와 Inter-VLAN Routing](02-switching/02-01-vlan-intervlan.md)
* [02-02. STP/RSTP/MST](02-switching/02-02-stp-rstp-mst.md)
* [02-03. EtherChannel (LACP/PAgP)](02-switching/02-03-etherchannel.md)
* [02-04. Private VLAN](02-switching/02-04-private-vlan.md)
* [02-05. DHCP Snooping과 DAI](02-switching/02-05-dhcp-snooping-dai.md)

## 03 Firewall - Cisco

* [03-01. ASA 기본 구성과 Security Level](03-firewall-cisco/03-01-asa-security-level.md)
* [03-02. ASA NAT (Auto NAT / Manual NAT)](03-firewall-cisco/03-02-asa-nat.md)
* [03-03. ASA ACL과 MPF (Modular Policy Framework)](03-firewall-cisco/03-03-asa-acl-mpf.md)
* [03-04. FTD와 FMC 기본 구성](03-firewall-cisco/03-04-ftd-fmc.md)
* [03-05. Cisco Zone-Based Firewall](03-firewall-cisco/03-05-zone-based-firewall.md)

## 04 Firewall - Checkpoint

* [04-01. GAiA OS와 SmartConsole 초기 설정](04-firewall-checkpoint/04-01-gaia-smartconsole.md)
* [04-02. Security Policy와 Rule Base](04-firewall-checkpoint/04-02-security-policy-rule-base.md)
* [04-03. Checkpoint NAT (Hide NAT / Static NAT)](04-firewall-checkpoint/04-03-checkpoint-nat.md)
* [04-04. SIC (Secure Internal Communication)](04-firewall-checkpoint/04-04-sic.md)
* [04-05. Checkpoint Logging과 SmartView](04-firewall-checkpoint/04-05-logging-smartview.md)
* [04-06. Application Control과 URL Filtering](04-firewall-checkpoint/04-06-application-control-url-filtering.md)

## 05 VPN

* [05-01. IPSec Site-to-Site VPN (Cisco-Cisco)](05-vpn/05-01-ipsec-site-to-site-cisco.md)
* [05-02. IPSec Site-to-Site VPN (Cisco-Checkpoint)](05-vpn/05-02-ipsec-site-to-site-cisco-checkpoint.md)
* [05-03. GRE over IPSec](05-vpn/05-03-gre-over-ipsec.md)
* [05-04. DMVPN (Dynamic Multipoint VPN)](05-vpn/05-04-dmvpn.md)
* [05-05. SSL VPN / Remote Access VPN](05-vpn/05-05-ssl-vpn-remote-access.md)
* [05-06. Checkpoint Remote Access VPN](05-vpn/05-06-checkpoint-remote-access-vpn.md)

## 06 HA & Redundancy

* [06-01. HSRP / VRRP / GLBP](06-ha-redundancy/06-01-hsrp-vrrp-glbp.md)
* [06-02. Cisco ASA Failover (Active/Standby)](06-ha-redundancy/06-02-asa-failover.md)
* [06-03. Cisco ASA Active/Active Failover](06-ha-redundancy/06-03-asa-active-active.md)
* [06-04. Checkpoint ClusterXL (HA / Load Sharing)](06-ha-redundancy/06-04-checkpoint-clusterxl.md)
* [06-05. 이중화 설계 종합 시나리오](06-ha-redundancy/06-05-redundancy-design.md)

## 07 Security

* [07-01. AAA (RADIUS / TACACS+)](07-security/07-01-aaa-radius-tacacs.md)
* [07-02. 802.1X 포트 인증](07-security/07-02-802-1x.md)
* [07-03. IPS / IDS 구성과 탐지](07-security/07-03-ips-ids.md)
* [07-04. uRPF (Unicast Reverse Path Forwarding)](07-security/07-04-urpf.md)
* [07-05. CoPP (Control Plane Policing)](07-security/07-05-copp.md)

## 08 Troubleshooting

* [08-01. OSPF Neighbor 장애](08-troubleshooting/08-01-ospf-neighbor-troubleshoot.md)
* [08-02. BGP Peering 장애](08-troubleshooting/08-02-bgp-peering-troubleshoot.md)
* [08-03. IPSec VPN 터널 장애](08-troubleshooting/08-03-ipsec-vpn-troubleshoot.md)
* [08-04. NAT 통신 장애](08-troubleshooting/08-04-nat-troubleshoot.md)
* [08-05. Checkpoint 정책 설치 실패](08-troubleshooting/08-05-checkpoint-policy-install-fail.md)
* [08-06. 패킷 캡처와 분석 (tcpdump, fw monitor)](08-troubleshooting/08-06-packet-capture-analysis.md)

## 09 Design

* [09-01. Enterprise 3-Tier 아키텍처](09-design/09-01-enterprise-3-tier.md)
* [09-02. DMZ 설계](09-design/09-02-dmz-design.md)
* [09-03. 망분리 설계](09-design/09-03-network-separation.md)
* [09-04. DC 네트워크 설계 (Spine-Leaf)](09-design/09-04-dc-spine-leaf.md)
* [09-05. 재해복구(DR) 네트워크 설계](09-design/09-05-dr-network-design.md)
