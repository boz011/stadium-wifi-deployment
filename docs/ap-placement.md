# AP Placement & RF Design

## Overview

AP placement in a stadium is fundamentally different from enterprise office deployments. The challenge is extreme client density (500–1,000+ devices per AP coverage area), hostile RF environments (concrete, steel, human body absorption), and the need for highly controlled, small cell sizes. Every AP placement decision directly impacts capacity, interference, and user experience.

---

## High-Density Design Principles

### 1. Small Cells, Many APs
- Target **25–50 clients per radio** for acceptable performance
- At 60,000 concurrent devices on 5 GHz: **1,200–2,400 APs** minimum
- Smaller cells = more spatial reuse = more aggregate capacity

### 2. Minimize Co-Channel Interference (CCI)
- Use **20 MHz channels only** — wider channels reduce the number of usable non-overlapping channels
- On 5 GHz with DFS: up to **25 non-overlapping 20 MHz channels**
- On 6 GHz (WiFi 6E): up to **59 non-overlapping 20 MHz channels**
- Design cell boundaries so APs on the same channel are physically separated

### 3. Control Cell Size
- **TPC (Transmit Power Control)**: Set aggressively low (1–11 dBm on 5 GHz)
- **RX-SOP**: Raise the receive sensitivity threshold to ignore distant frames
- **Directional antennas**: Point RF energy exactly where clients sit — not into the sky or opposite stands

### 4. Separate Bands by Role
| Band | Strategy | Use Case |
|------|----------|----------|
| 5 GHz | **Primary** — all capable clients here | Fans, staff, media, premium |
| 2.4 GHz | **Minimal** — disable on most APs | Legacy IoT only (sensors, some POS) |
| 6 GHz | **Growth** — WiFi 6E clients only | Future-proofing; media/premium first |

---

## Mounting Strategies

### Under-Seat Mounting (Preferred for Bowl Seating)

APs are mounted **beneath seats**, pointing upward into the seating rows above. This is the gold standard for stadium bowl coverage.

**Advantages:**
- Antenna is extremely close to clients (< 3 meters)
- Human body absorption works in your favor — bodies in adjacent rows attenuate the signal, naturally creating small cells
- Minimal line-of-sight to distant APs = reduced CCI
- Invisible to spectators

**Placement Pattern:**
- One AP every **8–12 seats** along a row
- Stagger APs between alternating rows to create a honeycomb pattern
- Skip rows to control vertical cell overlap

```
Row 10:  [--AP--]  [--------]  [--AP--]  [--------]  [--AP--]
Row 11:  [--------]  [--------]  [--------]  [--------]  [--------]
Row 12:  [--------]  [--AP--]  [--------]  [--AP--]  [--------]
Row 13:  [--------]  [--------]  [--------]  [--------]  [--------]
Row 14:  [--AP--]  [--------]  [--AP--]  [--------]  [--AP--]
```

**AP Model**: Cisco Catalyst 9136I or 9166 (internal omnidirectional antennas suitable for under-seat)

### AP-on-a-Stick / Railing Mount

APs mounted on handrails, structural beams, or dedicated poles in the seating bowl. Used where under-seat is impractical (e.g., bench seating, standing sections).

**Considerations:**
- Requires directional antennas aimed at specific seating sectors
- Greater distance to clients = larger cell sizes = fewer clients per AP
- More visible — coordinate with venue operations

### Ceiling/Overhead Mount (Concourses & Suites)

Standard enterprise-style ceiling mount for enclosed spaces.

| Area | AP Density | Model Recommendation |
|------|-----------|---------------------|
| Concourses | 1 AP per 1,500–2,500 sq ft | 9136I (tri-radio) |
| Suites / Club Level | 1 AP per suite or per 2 suites | 9130AXI |
| Press Box | 1 AP per 20–30 seats | 9136I |
| Locker Rooms | 1 AP per room | 9130AXI |
| Exterior Plazas | Outdoor AP per zone | 9163E (outdoor) |

---

## Channel Planning

### Dynamic Channel Assignment (DCA)

DCA is managed by the WLC's RRM (Radio Resource Management) engine. In high-density stadiums, customize DCA aggressively:

```
! Enable all UNII channels including DFS
ap dot11 5ghz channel add 52 56 60 64 100 104 108 112 116 120 124 128 132 136 140 144

! Set DCA sensitivity to HIGH
ap dot11 5ghz rrm channel-assignment mode auto
ap dot11 5ghz rrm channel-assignment sensitivity high

! Avoid channel changes during events
ap dot11 5ghz rrm channel-assignment freeze   ! Enable during game time
```

### Static Channel Plans (Alternative)

For venues with predictable, fixed AP locations, a **static channel plan** designed with a predictive site survey tool (Ekahau / iBwave) can outperform DCA:

- Assign channels manually using a channel reuse grid
- Validate with post-deployment survey
- Freeze DCA during events regardless

### Channel Plan Table (Example — One Section)

| AP Location | 5 GHz Channel | 5 GHz Power | 2.4 GHz |
|-------------|---------------|-------------|---------|
| Section 101, Row 5, Seat 10 | 36 | 3 dBm | Disabled |
| Section 101, Row 5, Seat 20 | 100 | 3 dBm | Disabled |
| Section 101, Row 8, Seat 15 | 149 | 3 dBm | Disabled |
| Section 101, Row 8, Seat 25 | 52 | 3 dBm | Disabled |
| Section 101, Row 11, Seat 10 | 44 | 3 dBm | Ch 1, 1 dBm |
| Section 101, Row 11, Seat 20 | 108 | 3 dBm | Disabled |

---

## Transmit Power Control (TPC)

### Goals
- Keep cells as small as possible
- Ensure sufficient signal at the client (-67 dBm or better for 5 GHz)
- Prevent signal bleed into adjacent sections

### TPC Settings

| Band | Min Power | Max Power | Target RSSI at Client |
|------|-----------|-----------|----------------------|
| 5 GHz | 1 dBm | 11 dBm | -60 to -67 dBm |
| 2.4 GHz | 1 dBm | 5 dBm | -65 to -70 dBm |
| 6 GHz | 1 dBm | 11 dBm | -60 to -67 dBm |

> **Note**: Client Tx power is uncontrollable. Some phones transmit at 15–18 dBm, creating asymmetric links. RX-SOP at the AP side helps mitigate this.

---

## Antenna Selection

| Scenario | Antenna Type | Pattern | Gain | Example |
|----------|-------------|---------|------|---------|
| Under-seat | Internal omni | Hemispherical (upward) | 4–6 dBi | Built-in on C9136I |
| Railing mount | External directional | 60°–90° sector | 6–8 dBi | Cisco ANT-45° patch |
| Concourse ceiling | Internal omni | Downward hemisphere | 4–5 dBi | Built-in on C9130AXI |
| Outdoor plaza | External omni | 360° horizontal | 5–8 dBi | C9163E built-in |
| Press box | Internal omni | Downward | 4 dBi | C9130AXI |

### Antenna Polarization
- Use **dual-polarized** antennas (horizontal + vertical) for spatial diversity
- In MIMO configurations, cross-polarized elements improve throughput in multipath-rich environments like stadiums

---

## Sector-Based Coverage Model

Divide the stadium into **sectors** for management and capacity planning:

```
┌─────────────────────────────────┐
│           NORTH STANDS          │
│  Sector N1  │  Sector N2  │ N3 │
├─────────────┼─────────────┼────┤
│    WEST     │   FIELD /   │ E  │
│   STANDS    │    PITCH    │ A  │
│  Sector W1  │             │ S  │
│  Sector W2  │             │ T  │
├─────────────┼─────────────┼────┤
│           SOUTH STANDS          │
│  Sector S1  │  Sector S2  │ S3 │
└─────────────────────────────────┘
```

Each sector gets:
- Its own **AP group** on the WLC
- Dedicated **access switch closet** (IDF)
- Independent **channel plan** designed to avoid inter-sector CCI
- Sector-specific **client density targets**

### Sector Capacity Example

| Sector | Seats | Est. Devices (1.2×) | APs (5 GHz) | Clients/AP |
|--------|-------|---------------------|-------------|------------|
| N1 | 8,000 | 9,600 | 240 | 40 |
| N2 | 8,000 | 9,600 | 240 | 40 |
| S1 | 8,000 | 9,600 | 240 | 40 |
| S2 | 8,000 | 9,600 | 240 | 40 |
| W1 | 5,000 | 6,000 | 150 | 40 |
| E1 | 5,000 | 6,000 | 150 | 40 |
| Suites | 3,000 | 3,600 | 90 | 40 |
| Concourses | — | 15,000 | 200 | 75 |
| **Total** | **~53,000** | **~66,000** | **~1,550** | — |

---

## Site Survey Process

### Predictive Survey (Pre-Deployment)
1. Obtain architectural drawings (CAD/BIM)
2. Import into Ekahau Pro or iBwave Design
3. Define wall/floor materials (concrete, steel, glass)
4. Place APs according to density targets
5. Simulate coverage, CCI, and capacity heatmaps
6. Iterate until targets are met

### Passive Survey (Post-Deployment)
1. Walk every section with survey tool (Ekahau Sidekick 2)
2. Capture RSSI, noise floor, channel utilization
3. Validate against predictive model
4. Adjust AP power, channels, and positions as needed

### Active Survey (Load Testing)
1. Simulate client load using traffic generators
2. Measure throughput, latency, roaming behavior
3. Test during a **full rehearsal or preseason event** with real crowd density
4. This is the only way to validate human body attenuation models

---

## 2.4 GHz vs 5 GHz vs 6 GHz Strategy

| Criterion | 2.4 GHz | 5 GHz | 6 GHz |
|-----------|---------|-------|-------|
| Channels (20 MHz) | 3 | 25 | 59 |
| Range | Long (problematic — CCI) | Medium (ideal) | Short (excellent for density) |
| Client Support | Universal | Universal (modern) | WiFi 6E/7 only (~15-25% of devices in 2025) |
| Stadium Role | **Disabled on most APs** | **Primary band** | **Future primary; early adoption** |
| IoT Use | Legacy sensors | Preferred | Not yet |

### Migration Path
1. **Today**: 5 GHz as primary, 2.4 GHz selectively for IoT
2. **2025–2027**: Begin enabling 6 GHz on tri-radio APs for 6E-capable devices
3. **2028+**: 6 GHz becomes primary as client adoption reaches 50%+; 5 GHz becomes secondary; 2.4 GHz fully deprecated
