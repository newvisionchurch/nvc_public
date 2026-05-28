# EFG Settings Summary

This document summarizes the image-based reference:

- `docs/ORG/EFG  기본 설정 이미지.pdf`

Use this as a compact operational reference. When a precise setting matters,
open the PDF and verify the live UniFi controller because some captured Traffic
Logging / NetFlow states differ between pages.

## System Snapshot

- UniFi site/device name: `EFG`.
- Gateway IP: `192.168.11.1`.
- UniFi Network version shown in the PDF: `10.2.105`.
- UniFi OS version shown in the PDF: `5.0.16`.
- Inventory snapshot: 1 gateway, 33 switches, 29 APs, about 173 clients.
- WAN provider: Comcast Business.
- WAN is configured as a static IPv4 connection. The public WAN address is kept
  in the PDF/reference and is not repeated here.
- Expected ISP speed in the WAN settings: 500 Mbps down / 500 Mbps up.

## Device Inventory

Model counts shown in the PDF include:

- AC Pro: 14
- U6 LR: 9
- U7 Pro: 2
- U7 Pro Wall: 2
- AC HD: 2
- USW Lite 8 PoE: 10
- USW Flex Mini: 10
- USW Lite 16 PoE: 4
- US 8 60W: 3
- US 24 PoE 250W: 2
- USW Pro Max 16 PoE: 2
- US 48: 1
- USW Enterprise 24 PoE: 1
- EFG: 1

Switch and AP names follow the Main/Education building naming pattern:

- `M1F_*`, `M2F_*`
- `E1F_*`, `E2F_*`

This naming convention is important because the Logstash pipeline derives
`site`, `floor`, `location`, and `ap.priority` from AP names.

## AP Inventory and Floor Context

Main building AP examples:

- `M1F_AP1_OFFICE`
- `M1F_AP2_PATHWAY`
- `M1F_AP3_CHOIR_R103`
- `M1F_AP4_VIP_ROOM`
- `M1F_AP5_BONDANG`
- `M1F_AP6_LITTLE_LAMB_R104`
- `M2F_AP1_IT_ROOM`
- `M2F_AP2_PASTOR_A1`
- `M2F_AP3_PASTOR_A2`
- `M2F_AP4_MEDIA`
- `M2F_AP5_PASTOR_B`
- `M2F_AP6_R200_CR`

Education building AP examples:

- `E1F_AP1_KITCHEN`
- `E1F_AP2_WISDOM`
- `E1F_AP3_R302`
- `E1F_AP4_R301`
- `E1F_AP5_R321`
- `E1F_AP6_NEWSONG`
- `E1F_AP6_RODEM`
- `E1F_AP7_NOAHS_ARK`
- `E1F_AP8_WISDOM2`
- `E1F_AP9_R310`
- `E2F_AP1_R412`
- `E2F_AP2_R407`
- `E2F_AP3_VISION`
- `E2F_AP4_LIGHTHOUSE`
- `E2F_AP5_R430`
- `E2F_AP6_PC_ROOM`
- `E2F_AP7_R420`

The floor-map pages show 5 GHz primary coverage. Green/yellow/red heatmap areas
should be used as supporting context for low RSSI, retry, roaming, or AP-stuck
investigations.

## WiFi Profiles

SSID/network relationships shown in the PDF include:

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

Default WiFi speed/channel settings shown:

- 2.4 GHz channel width: 20 MHz
- 5 GHz channel width: 80 MHz default, with per-AP examples using 40 MHz
- 6 GHz channel width: 320 MHz
- Extended 5 GHz DFS is enabled in the default WiFi speed screen.
- 5 GHz Roaming Assistant is enabled with a default threshold around `-80 dBm`.
- Wireless meshing appears disabled.
- UniFi Auto-Link and WiFiman support appear enabled.

## Networks and VLANs

Networks shown in the PDF:

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

Core Network details shown:

- Gateway: `192.168.11.1`
- DHCP range: `192.168.12.36` to `192.168.63.254`
- DNS servers include public resolvers plus `192.168.11.7`
- DHCP option 43 is enabled with `192.168.11.8`
- Lease time: 86400 seconds
- mDNS enabled
- DHCP guarding disabled

NVCampus details shown:

- Gateway: `192.168.70.1`
- DHCP range: `192.168.64.31` to `192.168.95.254`
- VLAN ID: 70
- Isolate network enabled
- Internet access enabled
- mDNS enabled
- DHCP server enabled

## Network Global Settings

Settings shown in the Networks screens:

- Default security posture: Allow All
- Gateway mDNS proxy: Auto
- IGMP snooping: enabled
- Querier selection: Advanced
- Querier switches: All
- Auto unknown traffic handling: enabled
- Fast leave: disabled
- L3 network isolation: disabled globally
- Device isolation: disabled globally
- RADIUS server: Default, authentication on EFG
- Spanning tree protocol: RSTP
- Rogue DHCP server detection: enabled
- Jumbo frames: disabled
- 802.1X control: disabled
- Ethernet profile example: `VLAN100_untagged`, native VLAN `NVFinance`, tagged VLANs Allow All

Network lists shown:

- `WireGuard VPN`: `192.168.96.0/24`
- `CCTV subnet`: `10.0.1.0/24`

## Internet and Gateway Ports

WAN and port details shown:

- Internet 1: WAN1, Comcast Business, static IPv4.
- WAN mode: Failover Only.
- Port 1: WAN1 to Comcast Business.
- Port 2: connected to `M2F_ITRM_S24`.
- SFP+ 1: connected to `M2F_ITRM_S48`.
- SFP+ 2: connected to `M2F_ITRM_CCTV_SW16`.
- SFP28 4: connected to `E2F_ITRM_S24B`.
- Flow Control appears disabled.
- Automatic Speed Test is enabled daily at 09:00.
- Smart Queues appear disabled.
- IGMP Proxy is enabled for Core Network.
- UPnP appears disabled.
- IPv6 appears disabled on the WAN settings.

WAN DNS shown:

- Primary: `1.1.1.1`
- Secondary: `75.75.75.75`

## VPN and Port Forwarding

VPN settings shown:

- Teleport appears enabled.
- VPN server: `NVC_VPN`
- VPN type: WireGuard
- Local IP: All
- Interface: All

Port forwarding shown:

- Name: Wireguard
- Type: Port Forwarding
- Protocol: TCP/UDP
- Source: Any
- Forward target: `192.168.59.140:51820`
- WAN interface: Internet 1

## CyberSecure and Security

Protection settings shown:

- Region Blocking: off
- Encrypted DNS: off
- Identification: Device and Traffic
- Block Page: enabled with UniFi SSL Certificate
- Intrusion Prevention: on
- Detection Mode: Notify and Block
- Selected networks include Core, NVFinance, KidsRegistration, NVCampus, and
  multiple Chapel networks.

Active detection groups shown include:

- Botnets and Threat Intelligence
- Viruses, Malware and Spyware
- Hacking and Exploits
- Peer to Peer and Dark Web
- Attacks and Reconnaissance
- Protocol Vulnerabilities

This matters for RCA because IDS/IPS and security detections can look like
network-service failures from the client perspective.

## Traffic Logging, Syslog, and SIEM

Consistent settings shown:

- Activity Logging / Syslog is configured to use a SIEM Server.
- Server address: `192.168.11.7`
- Port: `51415`
- Debug Logs: enabled
- Data retention: Auto
- SNMP monitoring appears disabled
- Logging levels: Auto

Flow logging settings shown:

- Flow Logging: All Traffic
- Additional flows include UniFi Services.
- Some captures show Gateway DNS enabled, while other captures show it disabled.
- Some captures show NetFlow/IPFIX enabled with port `2055`, version 10, hash
  sampling, and sampling rate 512; other captures show NetFlow/IPFIX disabled.

Before changing Logstash assumptions around NetFlow or DNS-flow visibility,
verify the live UniFi controller.

## Policy, Firewall, NAT, and Routing

Policy/routing items shown:

- Static route for CCTV destination `10.0.1.0/24`.
- WireGuard port forward to the internal WireGuard target.
- Source NAT and masquerade NAT rules for Core, NVFinance, KidsRegistration,
  ChapelVision, and grouped networks.
- Dynamic routing is not configured.

Simple firewall/policy items shown:

- Protect core network from chapel networks: Block, Local.
- Block NewSong and Wisdom: Block, Local.
- NVCampus Sunday Mass Limit: Allow, Internet, custom schedule.

Advanced firewall defaults shown include:

- Accept ICMP, IGMP, multicast, and established/related traffic.
- Drop invalid traffic.
- Allow WireGuard server/port-forward traffic.
- Guest rules allow DNS, DHCP, RADIUS, and hotspot portal flows.
- Guest and virtual-network traffic has default restricted/blocked paths.
- IPv6 default rules are present, but IPv6 is not the main active WAN mode.

## Event Log Categories

The UniFi events screen shows a high volume of client lifecycle events.
Important event filters visible in the PDF include:

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

For Logstash and Kibana, do not treat high-volume WiFi lifecycle events as root
cause by themselves. Use them as supporting context around higher-severity
events.

## RCA Implications

- EFG is the core gateway and must be analyzed together with AP logs.
- VLAN and network names matter for scoping incidents.
- The syslog target and Logstash UDP port must stay aligned.
- Security, firewall, QoS, DHCP, DNS, and system-resource events can all present
  to users as WiFi/AP trouble.
- Floor maps and AP names provide location context for RF and AP-stuck analysis.
- If the live controller differs from this PDF, update this summary and the
  relevant operational docs.
