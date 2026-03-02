# WLC Configuration — Catalyst 9800 Series

## Overview

The Cisco Catalyst 9800 Wireless LAN Controller is the control plane for all APs in the stadium deployment. Deployed as an **on-site HA (High Availability) cluster** in the venue data center, it manages 500–3,000+ APs via CAPWAP, centralizes policy enforcement, and provides a single management plane for the entire wireless network.

---

## High Availability (HA) Configuration

### HA Architecture
The stadium deployment requires **Stateful Switchover (SSO)** — an active/standby pair where the standby WLC maintains a synchronized copy of all AP and client state. Failover is seamless with no AP re-join required.

| Parameter | Value |
|-----------|-------|
| HA Mode | SSO (Stateful Switchover) |
| Primary WLC | WLC-STADIUM-01 |
| Standby WLC | WLC-STADIUM-02 |
| HA Interface | Dedicated L2 link between WLCs (RP port) |
| Keep-alive Timer | 1 second |
| Failover Priority | Primary: 1 (higher) / Standby: 2 |
| Platform | C9800-80 (supports 6,000 APs / 64,000 clients) |

```
! HA Configuration — Primary WLC
redundancy
 mode sso
!
chassis 1 priority 1
chassis 2 priority 2
!
interface GigabitEthernet0/0
 description HA-RP-Link
 ip address 192.168.255.1 255.255.255.252
 negotiation auto
!
wireless management interface Vlan10
```

### HA Considerations
- **Dedicated RP link**: Use a direct cable or short fiber between the two chassis — not through the production network
- **Bulk sync**: After initial pairing, the standby performs a bulk sync of all config, AP, and client state (~5–15 min for large deployments)
- **Software version**: Both WLCs must run the **identical** IOS-XE version
- **Licensing**: Smart License with DNA Advantage on both controllers

---

## WLAN Profiles

Each SSID is mapped to a WLAN profile → Policy profile → Policy tag. This three-tier model separates wireless settings, network policy, and site assignment.

### SSID Definitions

| SSID Name | WLAN Profile | Auth Type | VLAN | Encryption | Notes |
|-----------|-------------|-----------|------|------------|-------|
| Stadium-Guest | WP-GUEST | Captive Portal (CWA) | 30 | OWE / Open | Internet-only; rate-limited |
| Stadium-Staff | WP-STAFF | 802.1X (EAP-TLS) | 20 | WPA3-Enterprise | Dynamic VLAN via RADIUS |
| Stadium-Premium | WP-PREMIUM | PSK / Portal | 40 | WPA3-Personal | Suite/club access |
| Stadium-Media | WP-MEDIA | 802.1X | 50 | WPA3-Enterprise | Credential per event |
| Stadium-IoT | WP-IOT | MAB / EAP-TLS | 60 | WPA2-Enterprise | Device certificates |
| Stadium-POS | WP-POS | 802.1X / MAB | 70 | WPA2-Enterprise | PCI-DSS segment |

```
! WLAN Profile — Guest
wlan Stadium-Guest 1 Stadium-Guest
 security wpa akm owe
 security web-auth authentication-list GUEST-AUTH
 security web-auth parameter-map CAPTIVE-PORTAL-MAP
 no security wpa wpa2
 security wpa wpa3
 peer-blocking drop
 session-timeout 14400
 no shutdown
!
! Policy Profile — Guest
wireless profile policy PP-GUEST
 description Guest Internet-Only
 vlan VLAN0030
 aaa-override
 nac
 idle-timeout 1800
 session-timeout 14400
 qos-policy output GUEST-QOS
 no shutdown
```

### AAA / RADIUS Integration

```
! RADIUS Server Configuration
radius server ISE-PRIMARY
 address ipv4 10.10.10.60 auth-port 1812 acct-port 1813
 key <RADIUS_SECRET>
 automate-tester username probe-user probe-on
!
aaa group server radius ISE-GROUP
 server name ISE-PRIMARY
 server name ISE-SECONDARY
 deadtime 5
!
aaa authentication dot1x default group ISE-GROUP
aaa authorization network default group ISE-GROUP
aaa accounting dot1x default start-stop group ISE-GROUP
```

---

## RF Profiles

RF profiles control transmit power, channel assignment, and radio resource management (RRM) behavior. Stadium deployments require aggressive tuning for high-density.

### 5 GHz RF Profile (Primary)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Channel Width | 20 MHz | Maximizes channel reuse in high-density |
| DCA Channels | 36, 40, 44, 48, 52, 56, 60, 64, 100, 104, 108, 112, 116, 120, 124, 128, 132, 136, 140, 144, 149, 153, 157, 161, 165 | Use all available UNII channels |
| TPC Min Power | 1 dBm | Keep cells small |
| TPC Max Power | 11 dBm | Prevent oversized cells |
| TPC Threshold | -65 dBm | Aggressive reduction |
| RX-SOP Threshold | Medium (–76 dBm) | Ignore weak frames to reduce CCI |
| Data Rates | Disable 6, 9, 12 Mbps; mandatory 24 Mbps | Force clients to high rates |
| Band Select | Enabled, probe cycle count: 2 | Steer dual-band clients to 5 GHz |

```
! RF Profile — 5 GHz High Density
ap dot11 5ghz rf-profile HD-5GHZ-STADIUM
 channel-width 20
 tpc-min-power 1
 tpc-max-power 11
 tpc-threshold -65
 rx-sop threshold medium
 rate RATE_6M disable
 rate RATE_9M disable
 rate RATE_12M disable
 rate RATE_18M supported
 rate RATE_24M mandatory
 band-select probe-cycle 2
 band-select cycle-threshold 200
 no shutdown
```

### 2.4 GHz RF Profile

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Channel Width | 20 MHz | Only option for 2.4 GHz in high-density |
| DCA Channels | 1, 6, 11 | Non-overlapping channels only |
| TPC Min Power | 1 dBm | Minimize cell size |
| TPC Max Power | 5 dBm | Very small cells |
| Data Rates | Disable 1, 2, 5.5, 11 Mbps; mandatory 12 Mbps | Remove legacy rates |
| Status | **Disabled on most APs** | 2.4 GHz is liability in high-density — enable selectively |

> **⚠️ Best Practice**: In stadium deployments, disable 2.4 GHz on most APs. Only enable it on a sparse subset (~1 per sector) for legacy IoT devices that lack 5 GHz support.

### 6 GHz RF Profile (WiFi 6E)

| Parameter | Value |
|-----------|-------|
| Channel Width | 20 MHz (40 MHz in low-density zones) |
| Channels | All UNII-5 through UNII-8 (59 channels at 20 MHz) |
| Security | WPA3-SAE / OWE mandatory |
| Discovery | Reduced Neighbor Reports (RNR) via 5 GHz beacons |
| Client Steering | Prefer 6 GHz for 6E-capable clients |

---

## AP Join Process

### Discovery Methods (in order of preference)
1. **DHCP Option 43** — WLC IP embedded in DHCP offer on VLAN 10
2. **DNS** — `CISCO-CAPWAP-CONTROLLER.localdomain` resolves to WLC management IP
3. **Broadcast** — AP sends CAPWAP discovery on local subnet (only works if WLC is L2 adjacent)
4. **Priming** — Previously joined WLC IP stored in AP flash

### Join Sequence
1. AP boots → obtains IP via DHCP on VLAN 10
2. AP sends CAPWAP Discovery Request to WLC
3. WLC responds with Discovery Response
4. AP sends Join Request (with AP certificate)
5. WLC validates certificate → sends Join Response
6. DTLS tunnel established (UDP 5246/5247)
7. AP downloads config, firmware (if mismatched), and radio profiles
8. AP moves to **Run** state → begins serving clients

### AP Firmware Management
```
! Pre-download firmware to APs before maintenance window
ap image predownload
!
! Staggered upgrades — percentage-based rolling upgrade
ap upgrade staggered 25
```

---

## FlexConnect vs Local Mode

| Feature | Local Mode | FlexConnect |
|---------|-----------|-------------|
| Data Path | Centralized through WLC | Locally switched at AP |
| WAN Dependency | Full — AP stops serving if WLC unreachable | Partial — AP continues local switching |
| VLAN Trunking | Not required at AP | Required — AP switch port must trunk client VLANs |
| Scalability | WLC becomes data bottleneck | Scales better — WLC only handles control plane |
| Use Case | Small venues, centralized policy | Large venues, distributed switching |

### Stadium Recommendation: **Local Mode with Central Switching**

For stadium deployments with an **on-site WLC**, Local Mode is preferred:
- WLC is physically in the same building — latency is < 5 ms
- Centralized data plane simplifies VLAN management (no trunking needed at every access switch)
- All traffic policies enforced at a single point
- ARP suppression and client isolation handled by WLC

**Exception**: Use FlexConnect for **remote concourse areas or satellite buildings** (e.g., parking structures, external plazas) where a WAN link separates the AP from the data center.

---

## Policy Tags & Site Tags

The tag model maps APs to their WLAN + policy + RF configuration.

```
! Site Tag — Main Bowl
wireless tag site ST-MAIN-BOWL
 ap-profile AP-PROFILE-HD
 description "Main bowl seating area"
 local-site
!
! Policy Tag — Maps WLANs to Policy Profiles
wireless tag policy PT-STANDARD
 wlan WP-GUEST policy PP-GUEST
 wlan WP-STAFF policy PP-STAFF
 wlan WP-PREMIUM policy PP-PREMIUM
 wlan WP-MEDIA policy PP-MEDIA
 wlan WP-IOT policy PP-IOT
 wlan WP-POS policy PP-POS
!
! RF Tag
wireless tag rf RT-HIGH-DENSITY
 5ghz-rf-policy HD-5GHZ-STADIUM
 24ghz-rf-policy HD-24GHZ-STADIUM
!
! Assign tags to AP
ap <AP-MAC>
 site-tag ST-MAIN-BOWL
 policy-tag PT-STANDARD
 rf-tag RT-HIGH-DENSITY
```

---

## Monitoring & Troubleshooting

### Key Commands

```
show ap summary                        ! List all joined APs
show wireless client summary           ! Active client count
show wireless stats client detail      ! Client join/roam/failure stats
show ap auto-rf dot11 5ghz             ! Current channel + power assignments
show redundancy                        ! HA status
show wireless tag summary              ! Tag assignments
show wireless fabric summary           ! Fabric/SDA status (if applicable)
```

### Scale Targets

| Metric | C9800-80 | C9800-40 | C9800-L |
|--------|----------|----------|---------|
| Max APs | 6,000 | 2,000 | 250 |
| Max Clients | 64,000 | 32,000 | 5,000 |
| Max WLANs | 4,096 | 4,096 | 4,096 |
| Throughput | 80 Gbps | 40 Gbps | 5 Gbps |

For a 70,000-seat stadium, the **C9800-80** is the appropriate platform, supporting the full AP count and client capacity with headroom.
