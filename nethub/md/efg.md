# EFG 장비 참조 — GUI 고정 기준 정보

EFG (Edge Gateway Firewall) = UniFi Dream Machine Enterprise (UDMENT)  
실제 운영 설정이 바뀌면 UniFi Controller 확인 후 이 문서와 `source/config/ap_inventory.json`을 함께 갱신한다.

---

## System Summary

| 항목 | 값 |
|------|-----|
| Gateway IP | `192.168.11.1` |
| UniFi Network | `10.2.105` (UniFi OS: `5.0.16`) |
| Gateway 수 | 1 |
| Switch 수 | 33 |
| AP 수 | 29 |
| 클라이언트 | 약 173대 |
| WAN | Comcast Business 500Mbps (Static IPv4, Failover Only) |
| WAN DNS | Primary `1.1.1.1`, Secondary `75.75.75.75` |
| Syslog 대상 | `192.168.11.7:51415` UDP (Logstash) |
| EFG Port 1 | WAN1 → Comcast |
| EFG Port 2 | `M2F_ITRM_S24` |
| EFG Port 3 | `M2F_ITRM_S48` |
| EFG Port 4 | `M2F_ITRM_CCTV_SW16` |
| EFG Port 6 | `E2F_ITRM_S24B` |

---

## EFG의 RCA 역할

| 기능 | RCA 영향 |
|------|---------|
| DHCP Server | DHCP 실패 시 WiFi 정상이어도 인터넷 불가 |
| DNS | DNS 실패 시 연결은 되어도 서비스 불가 |
| Firewall / Policy Engine | 잔존 정책 시 특정 트래픽 차단 가능 |
| QoS / Traffic Management | QoS 오류 시 AP No-Service 발생 |
| IDS/IPS (CyberSecure) | 오탐 시 정상 트래픽 차단 가능 |
| Logging / Disk / Kernel | **디스크 Full → system_issue → AP Stuck** (확인된 RCA) |

확인된 RCA (V25.2): `EFG /boot/firmware 디스크 Full → system_issue → qos_error → AP Stuck/No-Service`

---

## VLAN / 네트워크 구성

| Network | VLAN | Subnet |
|---------|-----:|--------|
| Core Network | 1 | `192.168.0.0/18` (DHCP: `192.168.12.36`~`192.168.63.254`) |
| NVCampus | 70 | `192.168.64.0/19` |
| ChapelNoahsArk | 110 | `192.168.110.0/24` |
| ChapelNewSong | 120 | `192.168.120.0/24` |
| ChapelWisdom | 130 | `192.168.130.0/24` |
| ChapelVision | 140 | `192.168.140.0/24` |
| ChapelLightHouse | 150 | `192.168.150.0/24` |
| ChapelBridge | 160 | `192.168.160.0/24` |
| NVGuestVLAN | 168 | `192.168.168.0/22` |
| NVFinance | 100 | `192.168.100.0/24` |
| KidsRegistration | 200 | `192.168.200.0/24` |
| WireGuard VPN | — | `192.168.96.0/24` |

Core Network: Gateway `192.168.11.1`, DNS 포함 `192.168.11.7`, DHCP option 43: `192.168.11.8`, Lease 86400s

---

## WiFi SSID 구성

| SSID | Network (VLAN) | AP 수 | 대역 | 보안 |
|------|---------|------:|------|------|
| `NVIntraNet` | Native | 24 | 2.4/5/6GHz | WPA2/WPA3 Enterprise |
| `NVCampus` | NVCampus (70) | 22 | 5GHz | WPA2 |
| `NVLegacy` | Native | 17 | 2.4/5GHz | WPA2 |
| `NVPastor` | Native | 10 | 5GHz | WPA2 |
| `KidsRegistration` | KidsReg (200) | 3 | 2.4/5GHz | WPA2 |
| `ChapelNewSong` | ChapelNewSong (120) | 1 | 2.4/5GHz | WPA2 |
| `ChapelWisdom` | ChapelWisdom (130) | 1 | 2.4/5GHz | WPA2 |
| `ChapelNoahsArk` | ChapelNoahsArk (110) | 1 | 2.4/5GHz | WPA2 |
| `NVGuest` | NVGuestVLAN (168) | 1 | 5/6GHz | Open |
| `NVIdentity` | Native | 2 | 5/6GHz | WPA2/WPA3 Enterprise |

RF 기본값: 2.4GHz 20MHz / 5GHz 80MHz / 6GHz 320MHz / DFS 활성 / Roaming Assistant -80dBm

---

## AP 인벤토리 (29대)

| AP | Name | Model | IP | Site | Floor | Location | Parent Device | Parent Port |
|---|---|---|---|---|---|---|---|---|
| AP01 | `M1F_AP1_OFFICE` | U6 LR | `192.168.11.49` | Main | 1F | Office | `M1F_OFFICE_S8` | Port 3 |
| AP02 | `M1F_AP2_PATHWAY` | U7 Pro Wall | `192.168.11.146` | Main | 1F | Pathway | `M2F_ITRM_S24` | Port 3 |
| AP03 | `M1F_AP3_CHOIR_R103` | AC Pro | `192.168.11.128` | Main | 1F | Choir R103 | `M2F_ITRM_S24` | Port 10 |
| AP04 | `M1F_AP4_VIP_ROOM` | U6 LR | `192.168.11.139` | Main | 1F | VIP Room | `M1F_OFFICE_S8` | Port 2 |
| AP05 | `M1F_AP5_BONDANG` | U6 LR | `192.168.15.244` | Main | 1F | Bondang | `M1F_BONDANG_S8` | Port 2 |
| AP06 | `M1F_AP6_LITTLE_LAMB_R104` | AC Pro | `192.168.45.70` | Main | 1F | Little Lamb R104 | `M1F_104_SW5` | Port 3 |
| AP07 | `M2F_AP1_IT_ROOM` | AC Pro | `192.168.11.132` | Main | 2F | IT Room | `M2F_ITRM_S24` | Port 23 |
| AP08 | `M2F_AP2_PASTOR_A1` | U6 LR | `192.168.53.80` | Main | 2F | Pastor A1 | `M2F_PRM_A_S16` | Port 7 |
| AP09 | `M2F_AP3_PASTOR_A2` | AC Pro | `192.168.31.54` | Main | 2F | Pastor A2 | `M2F_PRM_A_S16` | Port 4 |
| AP10 | `M2F_AP4_MEDIA` | AC Pro | `192.168.11.136` | Main | 2F | Media | `M2F_MEDIA_L_S8` | Port 3 |
| AP11 | `M2F_AP5_PASTOR_B` | U7 Pro | `192.168.18.103` | Main | 2F | Pastor B | `M2F_PRM_B_S16` | Port 2 |
| AP12 | `M2F_AP6_R200_CR` | U7 Pro Wall | `192.168.100.107` | Main | 2F | R200 CR | `USW-Lite-8-PoE` | Port 3 |
| AP13 | `E1F_AP1_KITCHEN` | AC Pro | `192.168.17.40` | Education | 1F | Kitchen | `E1F_KITCHEN_S8` | Port 2 |
| AP14 | `E1F_AP2_WISDOM` | AC HD | `192.168.45.47` | Education | 1F | Wisdom | `E1F_WISDOM_SW8` | Port 2 |
| AP15 | `E1F_AP3_R302` | AC Pro | `192.168.27.218` | Education | 1F | R302 | `E2F_ITRM_S24A` | Port 5 |
| AP16 | `E1F_AP4_R301` | AC HD | `192.168.15.212` | Education | 1F | R301 | `E1F_NEWSONG_S8` | Port 3 |
| AP17 | `E1F_AP5_R321` | AC Pro | `192.168.48.244` | Education | 1F | R321 | `E1F_R321_SW5` | Port 2 |
| AP18 | `E1F_AP6_NEWSONG` | U7 Pro | `192.168.63.226` | Education | 1F | NewSong | `E1F_NEWSONG_S8` | Port 2 |
| AP19 | `E1F_AP6_RODEM` | U6 LR | `192.168.12.168` | Education | 1F | Rodem | `E2F_ITRM_S24A` | Port 16 |
| AP20 | `E1F_AP7_NOAHS_ARK` | U6 LR | `192.168.46.60` | Education | 1F | Noahs Ark | `E1F_NOAHS_ARK_S5` | Port 5 |
| AP21 | `E1F_AP8_WISDOM2` | AC Pro | `192.168.27.211` | Education | 1F | Wisdom2 | `E1F_WISDOM_SW8` | Port 3 |
| AP22 | `E1F_AP9_R310` | U6 LR | `192.168.31.155` | Education | 1F | R310 | `E2F_ITRM_S24B` | Port 3 |
| AP23 | `E2F_AP1_R412` | AC Pro | `192.168.11.123` | Education | 2F | R412 | `E2F_ITRM_S24A` | Port 11 |
| AP24 | `E2F_AP2_R407` | AC Pro | `192.168.11.124` | Education | 2F | R407 | `E2F_R407_S8` | Port 2 |
| AP25 | `E2F_AP3_VISION` | AC Pro | `192.168.11.125` | Education | 2F | Vision | `E2F_VISION_S5` | Port 1 |
| AP26 | `E2F_AP4_LIGHTHOUSE` | AC Pro | `192.168.14.151` | Education | 2F | Lighthouse | `E2F_LIGHTHOUSE_SW5` | Port 1 |
| AP27 | `E2F_AP5_R430` | U6 LR | `192.168.13.166` | Education | 2F | R430 | `E2F_ITRM_S24A` | Port 7 |
| AP28 | `E2F_AP6_PC_ROOM` | AC Pro | `192.168.11.137` | Education | 2F | PC Room | `E2F_PC_ROOM_S16` | Port 1 |
| AP29 | `E2F_AP7_R420` | U6 LR | `192.168.63.40` | Education | 2F | R420 | `E2F_R420_S8` | Port 4 |

AP 모델 수: AC Pro 14대 / U6 LR 9대 / AC HD 2대 / U7 Pro 2대 / U7 Pro Wall 2대

---

## 주요 스위치 목록 (AP Parent Device 해석)

| Name | Model | IP | Parent |
|---|---|---|---|
| `EFG` | UDMENT | `192.168.11.1` | Comcast |
| `M2F_ITRM_S48` | US 48 | `192.168.11.2` | EFG Port 3 |
| `M2F_ITRM_S24` | US 24 PoE 250W | `192.168.24.24` | EFG Port 2 |
| `E2F_ITRM_S24B` | USW Enterprise 24 | `192.168.57.29` | EFG Port 6 |
| `E2F_ITRM_S24A` | US 24 PoE 250W | `192.168.11.3` | E2F_ITRM_S24B Port 23 |
| `M2F_PRM_A_S16` | USW Lite 16 PoE | `192.168.38.176` | M2F_ITRM_S24 Port 5 |
| `M2F_PRM_B_S16` | USW Lite 16 PoE | `192.168.29.118` | M2F_ITRM_S48 Port 42 |
| `M1F_OFFICE_S8` | USW Lite 8 PoE | `192.168.20.6` | M2F_ITRM_S48 Port 35 |
| `M1F_BONDANG_S8` | USW Lite 8 PoE | `192.168.36.176` | M2F_ITRM_S24 Port 12 |
| `E1F_KITCHEN_S8` | USW Lite 8 PoE | `192.168.54.2` | E2F_ITRM_S24A Port 2 |
| `E1F_WISDOM_SW8` | USW Lite 8 PoE | `192.168.36.94` | E2F_ITRM_S24A Port 3 |
| `E1F_NEWSONG_S8` | USW Lite 8 PoE | `192.168.62.232` | E2F_ITRM_S24B Port 11 |
| `E2F_PC_ROOM_S16` | USW Lite 16 PoE | `192.168.35.1` | E2F_AP6_PC_ROOM Port 1 |

---

## 보안 / 방화벽 설정

| 항목 | 값 |
|------|-----|
| IPS | On (Notify and Block) |
| 탐지 그룹 | Botnets, Malware, Hacking, Exploits, Peer-to-Peer 등 |
| Region Blocking | Off |
| Encrypted DNS | Off |

RCA 참고: IDS/IPS 오탐 시 정상 트래픽 차단 가능 — AP No-Service처럼 보일 수 있음.

---

## VPN / 라우팅

| 항목 | 값 |
|------|-----|
| VPN 이름 | `NVC_VPN` (WireGuard) |
| WireGuard 포트 포워드 | TCP/UDP → `192.168.59.140:51820` |
| Static route | CCTV `10.0.1.0/24` |

---

## 장애 유형별 분석 포인트

### A형 — AP Stuck (AP 먹통)

| 증상 | 대표 로그 | EFG 연관 확인 |
|------|-----------|--------------|
| AP online이지만 client 0명 | `vap_timeout`, `channel_invalid` (ch=0) | 같은 시간대 system_issue 여부 |
| 5GHz VAP freeze | `radio_reset`, `WAL_DBGID_DEV_RESET` | EFG disk/I/O 오류 선행 여부 |
| TX OVERFLOW 반복 | `TX OVERFLOW wifi1ap3` | RRM scan trigger 직전 여부 |

### B형 — AP No-Service / 성능 붕괴

| 증상 | 대표 로그 | EFG 연관 확인 |
|------|-----------|--------------|
| 연결은 되나 인터넷 불가 | `dhcp_fail`, `dns_fail` | EFG DHCP/DNS 부하 |
| 속도 극저하 | `qos_error`, `policy_drop` | QoS/Policy 설정 |
| 특정 VLAN만 문제 | `network.vlan` | VLAN DHCP 범위 고갈 |

---

## GUI 사용 원칙

- 실시간 상태는 로그와 SSH/API 결과를 우선한다
- AP 이름/IP/모델/위치 기준: `source/config/ap_inventory.json`
- WAN 공인 IP, 비밀번호, 토큰, private key는 이 문서에 기록하지 않는다
- Syslog 대상과 Logstash UDP 포트는 반드시 `192.168.11.7:51415`로 일치시킨다
