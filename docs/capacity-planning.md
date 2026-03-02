# Capacity Planning

## Overview

Capacity planning for stadium WiFi is a math problem first and an RF problem second. The goal is to ensure that every connected device gets usable throughput during peak demand — typically halftime at a sold-out event when 60,000+ people simultaneously check scores, post photos, and stream replays.

---

## Client Density Math

### Assumptions

| Parameter | Value | Notes |
|-----------|-------|-------|
| Venue Capacity | 70,000 seats | Sold-out scenario |
| Device Multiplier | 1.2× | Some attendees carry 2 devices (phone + tablet/watch) |
| Estimated Devices | 84,000 | 70,000 × 1.2 |
| WiFi Adoption Rate | 70–80% | Not everyone connects |
| **Concurrent WiFi Clients** | **~60,000** | Peak connected |
| Target Clients per AP (5 GHz) | 30–50 | Performance degrades beyond 50 |

### AP Count Calculation

```
Total APs needed = Concurrent Clients ÷ Clients per AP

Conservative (30/AP):  60,000 ÷ 30 = 2,000 APs
Moderate (40/AP):      60,000 ÷ 40 = 1,500 APs
Aggressive (50/AP):    60,000 ÷ 50 = 1,200 APs
```

**Recommended: 1,500–2,000 APs** for a 70,000-seat venue on 5 GHz. Add headroom for concourses, suites, and back-of-house areas.

### Per-Area Breakdown

| Area | Est. Clients | APs | Clients/AP | Notes |
|------|-------------|-----|------------|-------|
| Bowl Seating (Lower) | 20,000 | 500 | 40 | Under-seat mounting |
| Bowl Seating (Upper) | 20,000 | 500 | 40 | Under-seat or railing |
| Concourses (Main) | 10,000 | 150 | 67 | Higher density OK — transient traffic |
| Suites & Club Level | 5,000 | 125 | 40 | Higher throughput expectations |
| Press Box & Media | 500 | 25 | 20 | Low density, high throughput |
| Back-of-House | 2,000 | 50 | 40 | Staff, operations |
| Exterior / Plazas | 2,500 | 50 | 50 | Outdoor APs |
| **Total** | **~60,000** | **~1,400** | — | Add 10–15% spare = **~1,600 APs** |

---

## Throughput Estimates

### Per-Client Throughput Targets

| User Class | Target Down | Target Up | Rationale |
|------------|-----------|----------|-----------|
| Guest (Fan) | 5–10 Mbps | 2–5 Mbps | Social media, scores, basic video |
| Premium | 15–25 Mbps | 5–10 Mbps | Higher quality streaming |
| Media | 25–50 Mbps | 25–50 Mbps | File uploads, live feeds |
| Staff | 5–10 Mbps | 2–5 Mbps | Ticketing, internal apps |
| IoT | 0.5–2 Mbps | 0.5–1 Mbps | Telemetry, POS transactions |

### Aggregate Throughput Math

**Peak demand scenario** (halftime, 60,000 clients active):

```
Guest clients:    50,000 × 5 Mbps  = 250 Gbps aggregate demand
Premium clients:   5,000 × 15 Mbps =  75 Gbps
Media clients:       500 × 30 Mbps =  15 Gbps
Staff clients:     2,000 × 5 Mbps  =  10 Gbps
IoT devices:       2,500 × 1 Mbps  =   2.5 Gbps
─────────────────────────────────────────────
Total:                               ~352 Gbps theoretical demand
```

**Reality check**: Not all clients are active simultaneously. Apply a **concurrency factor of 10–20%** (percentage of clients actively transferring data at any given instant):

```
Realistic peak throughput = 352 Gbps × 15% = ~53 Gbps
```

This is the backhaul and internet bandwidth target.

---

## Airtime Fairness

### The Problem
WiFi is a shared medium. A single client transmitting at 6 Mbps (legacy rate) consumes the same airtime as a client at 300 Mbps sending 50× more data. One slow client degrades performance for everyone on that AP.

### Mitigations

| Technique | Implementation | Effect |
|-----------|---------------|--------|
| Disable legacy rates | Minimum mandatory rate: 24 Mbps (5 GHz) | Prevents slow clients from hogging airtime |
| Band steering | WLC steers dual-band clients to 5/6 GHz | Reduces 2.4 GHz congestion |
| Airtime fairness (ATF) | WLC policy per SSID or client group | Allocates equal airtime regardless of data rate |
| Client limits per AP | Max 50 clients per radio | Hard cap prevents overloading |
| OFDMA (WiFi 6/6E) | Multi-user uplink/downlink scheduling | Serves multiple clients per transmission opportunity |

### ATF Configuration

```
! Enable Airtime Fairness on WLC
wireless airtime-fairness mode enforce

! Per-SSID ATF Policy
wireless airtime-fairness policy GUEST-ATF
 weight 50
!
wireless airtime-fairness policy MEDIA-ATF
 weight 80
!
wireless airtime-fairness policy IOT-ATF
 weight 20
```

---

## Band Steering

Band steering encourages dual-band clients to connect on 5 GHz instead of 2.4 GHz. The WLC suppresses probe responses on 2.4 GHz for clients that are also heard on 5 GHz.

### Configuration

```
! RF Profile — Band Select
ap dot11 5ghz rf-profile HD-5GHZ-STADIUM
 band-select probe-cycle 2
 band-select cycle-threshold 200
 band-select expire-suppression 60
 band-select client-rssi -75
```

| Parameter | Value | Meaning |
|-----------|-------|---------|
| Probe Cycle | 2 | Suppress 2.4 GHz probes for 2 cycles |
| Cycle Threshold | 200 ms | Time between probe cycles |
| Client RSSI | -75 dBm | Only steer clients with good 5 GHz signal |
| Expire Suppression | 60 sec | Re-evaluate after 60 seconds |

### 802.11k/v/r (Fast Roaming)

Essential in a stadium where fans move between sections:

- **802.11k**: Neighbor reports — client learns adjacent AP channels without scanning
- **802.11v**: BSS Transition Management — WLC suggests better AP to client
- **802.11r**: Fast BSS Transition (FT) — pre-authenticated roaming, < 50 ms handoff

```
! Enable on WLAN Profile
wlan Stadium-Guest 1 Stadium-Guest
 dot11r ft
 dot11r ft-over-ds
 dot11k neighbor-list
 dot11v bss-transition
```

---

## QoS & WMM

### WiFi Multimedia (WMM) Access Categories

| AC | Priority | Traffic Type | Stadium Use |
|----|----------|-------------|-------------|
| AC_VO (Voice) | Highest | VoIP, real-time comms | Staff PTT, media voice |
| AC_VI (Video) | High | Video streaming, live feeds | Media uploads, premium streaming |
| AC_BE (Best Effort) | Normal | Web, social media, email | Guest traffic (default) |
| AC_BK (Background) | Low | Bulk transfers, updates | App updates, backups |

### QoS Policy

```
! QoS Policy — Guest (rate-limited)
policy-map GUEST-QOS
 class class-default
  police rate 10000000 burst 1250000
   conform-action transmit
   exceed-action drop
  set dscp default
!
! QoS Policy — Media (prioritized)
policy-map MEDIA-QOS
 class VOICE
  set dscp ef
  priority level 1
 class VIDEO
  set dscp af41
  bandwidth remaining percent 40
 class class-default
  set dscp default
  bandwidth remaining percent 50
```

### Per-Client Rate Limiting

| VLAN/SSID | Downstream Limit | Upstream Limit |
|-----------|-----------------|----------------|
| Guest | 10 Mbps | 5 Mbps |
| Premium | 25 Mbps | 10 Mbps |
| Media | No limit (or 100 Mbps) | No limit (or 100 Mbps) |
| Staff | 15 Mbps | 5 Mbps |
| IoT | 2 Mbps | 1 Mbps |
| POS | 5 Mbps | 2 Mbps |

---

## Backhaul Bandwidth Requirements

### AP to Access Switch (Copper)

Each AP can generate up to **1 Gbps** on its wired uplink (Cat6a, 1GbE). With WiFi 6/6E APs, **mGig (2.5/5 Gbps)** uplinks are recommended.

| AP Radio Capability | Theoretical Max | Realistic Throughput | Recommended Uplink |
|--------------------|-----------------|--------------------|-------------------|
| WiFi 5 (802.11ac Wave 2) | 1.7 Gbps | 400–600 Mbps | 1 GbE |
| WiFi 6 (802.11ax) | 4.8 Gbps | 600–1,200 Mbps | 2.5 GbE (mGig) |
| WiFi 6E (802.11ax, tri-band) | 7.8 Gbps | 1,000–2,000 Mbps | 5 GbE (mGig) |

### Access Switch to Core (Fiber)

| Segment | Bandwidth | Connection |
|---------|-----------|------------|
| Access Switch Uplink (per closet) | 2× 10 GbE or 1× 25 GbE | Fiber (SM or MM, < 300 m) |
| Core Switch Aggregate | 100 GbE (multiple links, LAG) | Fiber (SM) |
| Core to WLC | 2× 40 GbE or 100 GbE | Fiber (short run in DC) |
| Core to Firewall | 40–100 GbE | Fiber |
| Firewall to Internet | 10–40 Gbps | Carrier circuits (diverse paths) |

### Internet Circuit Sizing

Based on the realistic peak throughput estimate of ~53 Gbps:

```
Internet bandwidth = Peak throughput × oversubscription factor

Conservative: 53 Gbps ÷ 2 = ~25 Gbps  (2:1 oversubscription)
Moderate:     53 Gbps ÷ 3 = ~18 Gbps  (3:1 oversubscription)
Aggressive:   53 Gbps ÷ 4 = ~13 Gbps  (4:1 oversubscription)
```

**Recommendation: 20–40 Gbps of diverse internet circuits** (multiple ISPs, multiple physical paths). Consider:
- 2× 10 Gbps dedicated internet access (DIA) from different providers
- 1× 10 Gbps burstable/CDN peering
- Content caching appliance on-site (Google GGC, Netflix OCA, Akamai) to reduce upstream demand by 30–50%

### Power Budget (PoE)

| AP Model | PoE Draw | PoE Standard |
|----------|----------|-------------|
| C9130AXI | 25.5 W | PoE+ (802.3at) |
| C9136I | 30 W | PoE+ (802.3at) / UPoE |
| C9166I | 30–51 W | UPoE (802.3bt) |

For 1,600 APs at 30 W each: **48 kW** total PoE budget. Plan access switch selection accordingly:
- Cisco C9300-48UXM: 48 ports mGig, 1,440 W PoE budget → supports ~48 APs
- Need approximately **34 access switches** minimum for APs alone

---

## Summary — Design Targets

| Metric | Target |
|--------|--------|
| Venue Capacity | 70,000 seats |
| Concurrent WiFi Clients | ~60,000 |
| Total APs | 1,500–2,000 |
| Clients per AP (5 GHz) | 30–50 |
| Per-Client Throughput (Guest) | 5–10 Mbps |
| Internet Bandwidth | 20–40 Gbps |
| Backhaul per Closet | 10–25 GbE fiber |
| Core Aggregate | 100+ GbE |
| AP-to-WLC Latency | < 20 ms RTT |
| Roaming Handoff | < 50 ms (802.11r) |
| WLC Platform | C9800-80 (HA SSO pair) |
| PoE Budget | ~48 kW |
