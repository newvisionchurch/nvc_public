# EFG GUI Reference

이 문서는 GUI에서 사용할 고정 EFG/UniFi 기준 정보를 정리한 자료이다.

원본 기준:

- `docs/ORG/EFG  기본 설정 이미지.pdf`
- `docs/efg-settings.md`

PDF는 이미지 기반이므로 OCR 자동 추출이 아니라 화면 확인 기반으로 정리했다.
실제 운영 설정이 바뀌면 UniFi Controller에서 다시 확인한 뒤 이 문서와
GUI 설정 파일을 함께 갱신한다.

## EFG의 RCA 역할

EFG는 단순한 gateway 장비가 아니다. 네트워크의 중심 제어 지점으로 다음 기능을 담당한다.

| 기능 | RCA 영향 |
|------|---------|
| DHCP Server | DHCP 실패 시 WiFi 정상이어도 인터넷 불가 |
| DNS | DNS 실패 시 연결은 되어도 서비스 불가 |
| Firewall / Policy Engine | Migration 정책 잔존 시 특정 트래픽 차단 가능 |
| QoS / Traffic Management | QoS 오류 시 AP No-Service 발생 |
| IDS/IPS (CyberSecure) | 오탐 시 정상 트래픽 차단 가능 |
| Logging / Disk / Kernel | **디스크 Full → system_issue → AP Stuck** (확인된 RCA) |

**확인된 주요 Root Cause (V25.2):**
```
EFG /boot/firmware 디스크 Full
→ system_issue (I/O error, syslog write 실패)
→ QoS 처리 실패 (qos_error)
→ AP Stuck / AP No-Service
```
→ AP 문제처럼 보이는 현상이 실제로는 EFG 시스템 자원 문제에서 시작된 연쇄 장애

## 장애 유형별 분석 포인트

### A형 — AP Stuck (AP 먹통)

| 증상 | 대표 로그 패턴 | EFG 연관 확인 포인트 |
|------|--------------|-------------------|
| AP online이지만 client 0명 | `vap_timeout`, `channel_invalid` (ch=0) | 같은 시간대 `system_issue` 여부 |
| 5 GHz VAP freeze | `radio_reset`, `WAL_DBGID_DEV_RESET` | EFG disk/I/O 오류 선행 여부 |
| TX OVERFLOW 반복 | `TX OVERFLOW wifi1ap3` | RRM scan trigger 직전 여부 |
| reboot 후 복구 | `osif_vap_init timeout` | 동일 모델/펌웨어 다중 AP 반복 여부 |

### B형 — AP No-Service / 성능 붕괴형

| 증상 | 대표 로그 패턴 | EFG 연관 확인 포인트 |
|------|--------------|-------------------|
| 연결은 되나 인터넷 불가 | `dhcp_fail`, `dns_fail` | EFG DHCP/DNS 부하 여부 |
| 속도 극저하 | `qos_error`, `policy_drop` | QoS/Policy 설정 확인 |
| 특정 VLAN만 문제 | `network.vlan`, `network.name` | VLAN 단위 DHCP 범위 고갈 여부 |
| 무선 품질 저하 | `low_rssi`, `retry_high`, `low_phy_rate` | RF 문제 vs 서비스 문제 분리 필요 |

### Dashboard 모니터링 지표 (Kibana)

| 지표 | 확인 방법 | 의미 |
|------|-----------|------|
| `system_issue` 급증 | `event.problem_class:system_issue AND observer.type:gateway` | EFG 자원 부족 최상위 경보 |
| AP fault time-series | `event.problem_class:ap_stuck` 1/5/10분 window | AP 장애 패턴 파악 |
| Top 문제 AP | `ap.name.keyword` GROUP BY | 반복 장애 AP 식별 |
| system → AP 상관 | system_issue spike 직후 AP fault 증가 여부 | 인과관계 확인 |
| QoS/DHCP 오류 | `event.problem_class:ap_no_service` | 서비스 계층 영향 |

## GUI 사용 원칙

- NAS/EFG/AP의 실시간 상태는 로그와 SSH/API 결과를 우선한다.
- 이 문서는 GUI의 고정 기준 정보로 사용한다.
- AP 이름, 모델, IP, 위치, 상위 스위치 정보는 `gui/config/ap_inventory.json`
  생성 또는 갱신 기준으로 사용한다.
- AP firmware는 PDF 장비 목록 표에서 확인되지 않아 `unknown`으로 둔다.
- WAN 공인 IP, 비밀번호, 토큰, private key는 이 문서에 기록하지 않는다.

## System Summary

| Field | Value |
| --- | --- |
| UniFi site/device name | `EFG` |
| Gateway IP | `192.168.11.1` |
| UniFi Network version shown | `10.2.105` |
| UniFi OS version shown | `5.0.16` |
| Gateway count | `1` |
| Switch count shown | `33` |
| AP count shown | `29` |
| Client count shown | about `173` |
| WAN provider | Comcast Business |
| WAN mode | Failover Only |
| WAN IPv4 mode | Static IPv4 |
| Expected WAN speed | 500 Mbps down / 500 Mbps up |
| WAN DNS primary | `1.1.1.1` |
| WAN DNS secondary | `75.75.75.75` |

## GUI Data Fields

권장 AP GUI 컬럼:

| GUI column | Source | Meaning |
| --- | --- | --- |
| `AP` | GUI inventory | AP display ID such as `AP01` |
| `Name` | UniFi device name | Canonical AP name used in logs |
| `Model` | UniFi device list | Hardware model |
| `IP` | UniFi device list | AP management IP |
| `Site` | Derived from name | `Main` or `Education` |
| `Floor` | Derived from name | `1F` or `2F` |
| `Location` | Derived from name | Human location label |
| `Parent Device` | UniFi device list | Upstream switch |
| `Parent Port` | UniFi device list | Upstream switch port |
| `Firmware` | Live UniFi/API later | Keep `unknown` until verified |
| `Stuck` | GUI log analysis | Runtime count |
| `Status` | GUI log analysis | Runtime health state |

## AP Inventory

이 표는 GUI AP ID를 현재 `gui/config/ap_inventory.json`의 순서와 맞춘 것이다.

| AP | Name | Model | IP | Site | Floor | Location | Parent Device | Parent Port | Firmware |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| AP01 | `M1F_AP1_OFFICE` | U6 LR | `192.168.50.49` | Main | 1F | Office | `M1F_OFFICE_S8` | Port 3 | unknown |
| AP02 | `M1F_AP2_PATHWAY` | U7 Pro Wall | `192.168.12.146` | Main | 1F | Pathway | `M2F_ITRM_S24` | Port 3 | unknown |
| AP03 | `M1F_AP3_CHOIR_R103` | AC Pro | `192.168.11.128` | Main | 1F | Choir R103 | `M2F_ITRM_S24` | Port 10 | unknown |
| AP04 | `M1F_AP4_VIP_ROOM` | U6 LR | `192.168.11.139` | Main | 1F | VIP Room | `M1F_OFFICE_S8` | Port 2 | unknown |
| AP05 | `M1F_AP5_BONDANG` | U6 LR | `192.168.15.244` | Main | 1F | Bondang | `M1F_BONDANG_S8` | Port 2 | unknown |
| AP06 | `M1F_AP6_LITTLE_LAMB_R104` | AC Pro | `192.168.45.70` | Main | 1F | Little Lamb R104 | `M1F_104_SW5` | Port 3 | unknown |
| AP07 | `M2F_AP1_IT_ROOM` | AC Pro | `192.168.11.132` | Main | 2F | IT Room | `M2F_ITRM_S24` | Port 23 | unknown |
| AP08 | `M2F_AP2_PASTOR_A1` | U6 LR | `192.168.53.80` | Main | 2F | Pastor A1 | `M2F_PRM_A_S16` | Port 7 | unknown |
| AP09 | `M2F_AP3_PASTOR_A2` | AC Pro | `192.168.31.54` | Main | 2F | Pastor A2 | `M2F_PRM_A_S16` | Port 4 | unknown |
| AP10 | `M2F_AP4_MEDIA` | AC Pro | `192.168.11.136` | Main | 2F | Media | `M2F_MEDIA_L_S8` | Port 3 | unknown |
| AP11 | `M2F_AP5_PASTOR_B` | U7 Pro | `192.168.18.103` | Main | 2F | Pastor B | `M2F_PRM_B_S16` | Port 2 | unknown |
| AP12 | `M2F_AP6_R200_CR` | U7 Pro Wall | `192.168.100.107` | Main | 2F | R200 CR | `USW-Lite-8-PoE` | Port 3 | unknown |
| AP13 | `E1F_AP1_KITCHEN` | AC Pro | `192.168.17.40` | Education | 1F | Kitchen | `E1F_KITCHEN_S8` | Port 2 | unknown |
| AP14 | `E1F_AP2_WISDOM` | AC HD | `192.168.45.47` | Education | 1F | Wisdom | `E1F_WISDOM_SW8` | Port 2 | unknown |
| AP15 | `E1F_AP3_R302` | AC Pro | `192.168.27.218` | Education | 1F | R302 | `E2F_ITRM_S24A` | Port 5 | unknown |
| AP16 | `E1F_AP4_R301` | AC HD | `192.168.15.212` | Education | 1F | R301 | `E1F_NEWSONG_S8` | Port 3 | unknown |
| AP17 | `E1F_AP5_R321` | AC Pro | `192.168.48.244` | Education | 1F | R321 | `E1F_R321_SW5` | Port 2 | unknown |
| AP18 | `E1F_AP6_NEWSONG` | U7 Pro | `192.168.63.226` | Education | 1F | NewSong | `E1F_NEWSONG_S8` | Port 2 | unknown |
| AP19 | `E1F_AP6_RODEM` | U6 LR | `192.168.12.168` | Education | 1F | Rodem | `E2F_ITRM_S24A` | Port 16 | unknown |
| AP20 | `E1F_AP7_NOAHS_ARK` | U6 LR | `192.168.46.60` | Education | 1F | Noahs Ark | `E1F_NOAHS_ARK_S5` | Port 5 | unknown |
| AP21 | `E1F_AP8_WISDOM2` | AC Pro | `192.168.27.211` | Education | 1F | Wisdom2 | `E1F_WISDOM_SW8` | Port 3 | unknown |
| AP22 | `E1F_AP9_R310` | U6 LR | `192.168.31.155` | Education | 1F | R310 | `E2F_ITRM_S24B` | Port 3 | unknown |
| AP23 | `E2F_AP1_R412` | AC Pro | `192.168.11.123` | Education | 2F | R412 | `E2F_ITRM_S24A` | Port 11 | unknown |
| AP24 | `E2F_AP2_R407` | AC Pro | `192.168.11.124` | Education | 2F | R407 | `E2F_R407_S8` | Port 2 | unknown |
| AP25 | `E2F_AP3_VISION` | AC Pro | `192.168.11.125` | Education | 2F | Vision | `E2F_VISION_S5` | Port 1 | unknown |
| AP26 | `E2F_AP4_LIGHTHOUSE` | AC Pro | `192.168.14.151` | Education | 2F | Lighthouse | `E2F_LIGHTHOUSE_SW5` | Port 1 | unknown |
| AP27 | `E2F_AP5_R430` | U6 LR | `192.168.13.166` | Education | 2F | R430 | `E2F_ITRM_S24A` | Port 7 | unknown |
| AP28 | `E2F_AP6_PC_ROOM` | AC Pro | `192.168.11.137` | Education | 2F | PC Room | `E2F_PC_ROOM_S16` | Port 1 | unknown |
| AP29 | `E2F_AP7_R420` | U6 LR | `192.168.63.40` | Education | 2F | R420 | `E2F_R420_S8` | Port 4 | unknown |

## AP Model Counts

| Model | Count |
| --- | ---: |
| AC Pro | 14 |
| U6 LR | 9 |
| U7 Pro | 2 |
| U7 Pro Wall | 2 |
| AC HD | 2 |

## Switch and Gateway Inventory

이 표는 AP의 `Parent Device` 해석에 필요한 주요 스위치/게이트웨이 목록이다.

| Name | Model | IP | Parent Device |
| --- | --- | --- | --- |
| `EFG` | EFG | `192.168.11.1` | Comcast Business |
| `M2F_ITRM_S48` | US 48 | `192.168.11.2` | EFG Port 3 |
| `M2F_ITRM_S24` | US 24 PoE 250W | `192.168.24.24` | EFG Port 2 |
| `M2F_ITRM_CCTV_SW16` | USW Pro Max 16 PoE | `192.168.56.163` | EFG Port 4 |
| `E2F_ITRM_S24B` | USW Enterprise 24 | `192.168.57.29` | EFG Port 6 |
| `E2F_ITRM_S24A` | US 24 PoE 250W | `192.168.11.3` | `E2F_ITRM_S24B` Port 23 |
| `E2F_ITRM_CCTV_SW16` | USW Pro Max 16 PoE | `192.168.53.127` | `M2F_ITRM_CCTV_SW16` Port 18 |
| `M2F_PRM_B_S16` | USW Lite 16 PoE | `192.168.29.118` | `M2F_ITRM_S48` Port 42 |
| `M2F_PRM_A_S16` | USW Lite 16 PoE | `192.168.38.176` | `M2F_ITRM_S24` Port 5 |
| `M2F_PRM_A_S8` | US 8 60W | `192.168.56.3` | `M2F_PRM_A_S16` Port 1 |
| `M2F_MEDIA_L_S8` | USW Lite 8 PoE | `192.168.20.21` | `M2F_ITRM_S48` Port 29 |
| `M2F_MEDIA_R_S5` | USW Flex Mini | `192.168.33.57` | `M2F_MEDIA_L_S8` Port 5 |
| `M2F_ITRM_S16` | USW Lite 16 PoE | `192.168.25.76` | `M2F_ITRM_S48` Port 39 |
| `M2F_FWALL_S8` | USW Lite 8 PoE | `192.168.59.138` | `M2F_FCCTV_S8` Port 4 |
| `M2F_FDOOR_S8` | US 8 60W | `192.168.19.202` | `M2F_FCCTV_S8` Port 5 |
| `M2F_FCCTV_S8` | US 8 60W | `192.168.30.42` | `M2F_ITRM_S24` Port 2 |
| `M1F_OFFICE_S8` | USW Lite 8 PoE | `192.168.20.6` | `M2F_ITRM_S48` Port 35 |
| `M1F_BRIDGE_S5` | USW Flex Mini | `192.168.55.108` | `M2F_ITRM_S48` Port 36 |
| `M1F_BONDANG_S8` | USW Lite 8 PoE | `192.168.36.176` | `M2F_ITRM_S24` Port 12 |
| `M1F_104_SW5` | USW Flex Mini | `192.168.33.213` | `M2F_ITRM_S48` Port 38 |
| `USW-Lite-8-PoE` | USW Lite 8 PoE | `192.168.100.141` | `M2F_FCCTV_S8` Port 8 |
| `E1F_KITCHEN_S8` | USW Lite 8 PoE | `192.168.54.2` | `E2F_ITRM_S24A` Port 2 |
| `E1F_WISDOM_SW8` | USW Lite 8 PoE | `192.168.36.94` | `E2F_ITRM_S24A` Port 3 |
| `E1F_NEWSONG_S8` | USW Lite 8 PoE | `192.168.62.232` | `E2F_ITRM_S24B` Port 11 |
| `E1F_R321_SW5` | USW Flex Mini | `192.168.33.254` | `E2F_ITRM_S24A` Port 17 |
| `E1F_NOAHS_ARK_S5` | USW Flex Mini | `192.168.30.138` | `E2F_ITRM_S24B` Port 10 |
| `E2F_PC_ROOM_S16` | USW Lite 16 PoE | `192.168.35.1` | `E2F_AP6_PC_ROOM` Port 1 |
| `E2F_R407_S8` | USW Lite 8 PoE | `192.168.43.163` | `E2F_ITRM_S24A` Port 8 |
| `E2F_R420_S8` | USW Lite 8 PoE | `192.168.52.67` | `E2F_ITRM_S24A` Port 4 |
| `E2F_VISION_S5` | USW Flex Mini | `192.168.33.85` | `E2F_AP3_VISION` Port 1 |
| `E2F_LIGHTHOUSE_SW5` | USW Flex Mini | `192.168.30.131` | `E2F_AP4_LIGHTHOUSE` Port 1 |

## Switch Model Counts

| Model | Count |
| --- | ---: |
| USW Lite 8 PoE | 10 |
| USW Flex Mini | 10 |
| USW Lite 16 PoE | 4 |
| US 8 60W | 3 |
| US 24 PoE 250W | 2 |
| USW Pro Max 16 PoE | 2 |
| US 48 | 1 |
| USW Enterprise 24 PoE | 1 |

## WiFi Profiles

| SSID | Network | AP Scope | Bands | Security |
| --- | --- | --- | --- | --- |
| `KidsRegistration` | `KidsRegistration (200)` | 3 APs | 2.4 GHz, 5 GHz | WPA2 |
| `NVCampus` | `NVCampus (70)` | 22 APs | 5 GHz | WPA2 |
| `NVIntraNet` | Native Network | 24 APs | 2.4 GHz, 5 GHz, 6 GHz | WPA2/WPA3 Enterprise |
| `NVLegacy` | Native Network | 17 APs | 2.4 GHz, 5 GHz | WPA2 |
| `ChapelNewSong` | `ChapelNewSong (120)` | `E1F_AP6_NEWSONG` | 2.4 GHz, 5 GHz | WPA2 |
| `ChapelWisdom` | `ChapelWisdom (130)` | `E1F_AP2_WISDOM` | 2.4 GHz, 5 GHz | WPA2 |
| `NVPastor` | Native Network | 10 APs | 5 GHz | WPA2 |
| `NVGuest` | `NVGuestVLAN (168)` | IT-only, 1 AP | 5 GHz, 6 GHz | Open |
| `ChapelNoahsArk` | `ChapelNoahsArk (110)` | `E1F_AP7_NOAHS_ARK` | 2.4 GHz, 5 GHz | WPA2 |
| `NVIdentity` | Native Network | 2 APs | 5 GHz, 6 GHz | WPA2/WPA3 Enterprise |

Default WiFi/RF settings shown:

- 2.4 GHz channel width: 20 MHz
- 5 GHz channel width: 80 MHz default, with per-AP examples using 40 MHz
- 6 GHz channel width: 320 MHz
- Extended 5 GHz DFS: enabled
- 5 GHz Roaming Assistant: enabled, threshold around `-80 dBm`
- Wireless meshing: appears disabled
- UniFi Auto-Link and WiFiman support: appear enabled

## Networks and VLANs

| Network | VLAN | Subnet | DHCP |
| --- | ---: | --- | --- |
| Core Network | 1 | `192.168.0.0/18` | Server |
| NVFinance | 100 | `192.168.100.0/24` | Server |
| KidsRegistration | 200 | `192.168.200.0/24` | Server |
| NVCampus | 70 | `192.168.64.0/19` | Server |
| ChapelWisdom | 130 | `192.168.130.0/24` | Server |
| ChapelNewSong | 120 | `192.168.120.0/24` | Server |
| ChapelNoahsArk | 110 | `192.168.110.0/24` | Server |
| ChapelVision | 140 | `192.168.140.0/24` | Server |
| ChapelLightHouse | 150 | `192.168.150.0/24` | Server |
| ChapelBridge | 160 | `192.168.160.0/24` | Server |
| NVGuestVLAN | 168 | `192.168.168.0/22` | Server |
| WireGuard VPN | none shown | `192.168.96.0/24` | VPN |
| CCTV subnet | none shown | `10.0.1.0/24` | Static route context |

Core Network details:

- Gateway: `192.168.11.1`
- DHCP range: `192.168.12.36` to `192.168.63.254`
- DNS includes public resolvers plus `192.168.11.7`
- DHCP option 43: enabled with `192.168.11.8`
- Lease time: 86400 seconds
- mDNS: enabled
- DHCP guarding: disabled

NVCampus details:

- Gateway: `192.168.70.1`
- DHCP range: `192.168.64.31` to `192.168.95.254`
- VLAN ID: 70
- Network isolation: enabled
- Internet access: enabled
- mDNS: enabled
- DHCP server: enabled

## Network Global Settings

| Setting | Value |
| --- | --- |
| Default security posture | Allow All |
| Gateway mDNS proxy | Auto |
| IGMP snooping | Enabled |
| Querier selection | Advanced |
| Querier switches | All |
| Auto unknown traffic handling | Enabled |
| Fast leave | Disabled |
| L3 network isolation | Disabled globally |
| Device isolation | Disabled globally |
| RADIUS server | Default, authentication on EFG |
| Spanning tree protocol | RSTP |
| Rogue DHCP server detection | Enabled |
| Jumbo frames | Disabled |
| 802.1X control | Disabled |

## Internet, VPN, and Routing

| Item | Value |
| --- | --- |
| Internet 1 | WAN1, Comcast Business, static IPv4 |
| WAN mode | Failover Only |
| EFG Port 1 | WAN1 to Comcast Business |
| EFG Port 2 | `M2F_ITRM_S24` |
| EFG Port 3 | `M2F_ITRM_S48` |
| EFG Port 4 | `M2F_ITRM_CCTV_SW16` |
| EFG Port 6 | `E2F_ITRM_S24B` |
| Flow Control | Appears disabled |
| Automatic Speed Test | Enabled daily at 09:00 |
| Smart Queues | Appears disabled |
| IGMP Proxy | Enabled for Core Network |
| UPnP | Appears disabled |
| IPv6 | Appears disabled on WAN |

VPN and port-forwarding:

- Teleport: appears enabled
- VPN server: `NVC_VPN`
- VPN type: WireGuard
- Local IP: All
- Interface: All
- Port forward name: `Wireguard`
- Port forward protocol: TCP/UDP
- Port forward target: `192.168.59.140:51820`
- WAN interface: Internet 1
- Static route: CCTV destination `10.0.1.0/24`

## CyberSecure and Security

| Setting | Value |
| --- | --- |
| Region Blocking | Off |
| Encrypted DNS | Off |
| Identification | Device and Traffic |
| Block Page | Enabled with UniFi SSL Certificate |
| Intrusion Prevention | On |
| Detection Mode | Notify and Block |

Active detection groups shown:

- Botnets and Threat Intelligence
- Viruses, Malware and Spyware
- Hacking and Exploits
- Peer to Peer and Dark Web
- Attacks and Reconnaissance
- Protocol Vulnerabilities

Selected networks include Core, NVFinance, KidsRegistration, NVCampus, and
multiple Chapel networks.

## Traffic Logging, Syslog, and SIEM

| Setting | Value |
| --- | --- |
| Activity Logging / Syslog | SIEM Server |
| SIEM server address | `192.168.11.7` |
| SIEM/syslog port | `51415` |
| Debug Logs | Enabled |
| Data retention | Auto |
| SNMP monitoring | Appears disabled |
| Logging levels | Auto |
| Flow Logging | All Traffic |
| Additional flows | UniFi Services |
| NetFlow/IPFIX | Conflicting captures: enabled in some, disabled in others |
| NetFlow/IPFIX port when enabled | `2055` |
| NetFlow/IPFIX version when enabled | Version 10 |
| NetFlow sampling when enabled | Hash sampling, sampling rate 512 |

Important GUI implication:

- GUI should use NAS Logstash JSONL output as the normal analysis source.
- Syslog target and Logstash UDP port must stay aligned at `192.168.11.7:51415`.
- NetFlow/DNS-flow visibility should not be assumed unless live settings are verified.

## Policy, Firewall, NAT, and Routing Notes

Policy/routing items shown:

- Static route for CCTV destination `10.0.1.0/24`
- WireGuard port forward to internal WireGuard target
- Source NAT and masquerade NAT rules for Core, NVFinance, KidsRegistration,
  ChapelVision, and grouped networks
- Dynamic routing is not configured

Simple firewall/policy items shown:

- Protect core network from chapel networks: Block, Local
- Block NewSong and Wisdom: Block, Local
- NVCampus Sunday Mass Limit: Allow, Internet, custom schedule

Advanced firewall defaults shown:

- Accept ICMP, IGMP, multicast, and established/related traffic
- Drop invalid traffic
- Allow WireGuard server/port-forward traffic
- Guest rules allow DNS, DHCP, RADIUS, and hotspot portal flows
- Guest and virtual-network traffic has default restricted/blocked paths
- IPv6 default rules exist, but IPv6 is not the main active WAN mode

## Event Log Categories

Important UniFi event categories visible in the PDF:

- AP Channel Change
- AP Dropped Traffic Detected
- Device Offline / Reconnected / Updated
- DFS Radar Detected
- Internet Down / Restored
- IP Address Conflict
- Multiple Devices Offline / Reconnected
- Packet Loss Detected
- Poor AP Link Speed
- Slow RADIUS Authentication
- STP Port Blocked
- Threat Detected
- VPN Client Connected / Disconnected
- WiFi Client Connected / Disconnected / Roamed
- WiFi Impersonation Detected
- Wired Client Connected / Disconnected

GUI and Logstash should treat high-volume WiFi lifecycle events as supporting
context, not direct root cause, unless they correlate with higher-severity
events such as `system_issue`, `rrm_scan_trigger`, `ap_stuck`, `qos_error`, or
`ap_no_service`.

## GUI Update Checklist

When this document changes, update these files as needed:

- `gui/config/ap_inventory.json`
- GUI AP table columns in `gui/main.py`
- Any future GUI reference data file derived from this document
- `docs/efg-settings.md` if the operational summary changes

Required AP inventory fields for GUI:

- `id`
- `name`
- `ip`
- `model`
- `firmware`
- `site`
- `floor`
- `location`
- `parent_device`
- `parent_port`
- `keywords`

