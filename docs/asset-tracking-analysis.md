# Stadium Asset Tracking: 802.11ah vs BLE vs WiFi Location Services

## Short Answer

**Your 500-3,000 Cisco APs already have built-in BLE radios.** Enable BLE scanning, license Cisco Spaces, stick $10 BLE tags on your assets, and you're tracking in a week — fully integrated with your C9800 dashboard. HaLow adds a separate infrastructure for worse results.

---

## 1. What "Asset Tracking in a Stadium" Actually Means

| Category | Examples | Accuracy Needed | Update Frequency |
|----------|----------|----------------|-----------------|
| **High-value mobile equipment** | Portable radios, medical AEDs/kits, oxygen tanks | Room/zone (5-10m) | Every 5-15 min |
| **Accessibility equipment** | Wheelchairs, mobility scooters | Zone (10-15m) | Every 5-10 min |
| **F&B logistics** | Kegs, vendor carts, portable POS terminals | Section-level (15-25m) | Every 15-30 min |
| **Credentials/personnel** | VIP badges, staff credentials, security radios | Zone (5-10m) | Near real-time |
| **Maintenance equipment** | Pressure washers, floor scrubbers, tool carts | Building/zone | Every 30-60 min |
| **Game-day ops** | Barriers, stanchions, signage carts | Section-level | Periodic |

**Key insight:** Almost none of these require sub-meter precision. Zone-level (5-15m) is sufficient for 90%+ of stadium asset tracking.

---

## 2. 802.11ah for Asset Tracking — Honest Assessment

### Pros (on paper)
- **Range**: 100m–1km, easily covers entire venue from fewer APs
- **Battery life**: TWT enables 2-5 year coin-cell life for sensors
- **IP-native**: Full TCP/IP stack, no translation gateway
- **Capacity**: 8,191 stations per AP (designed for massive IoT)

### Cons (in your reality)

| Problem | Impact |
|---------|--------|
| **Zero Cisco WLC integration** | Catalyst 9800 has no HaLow radio, no CAPWAP support. Entirely separate infrastructure needed. |
| **No location engine** | HaLow APs don't ship with RSSI triangulation or FTM. You'd build your own. |
| **900 MHz ISM interference** | Shares band with LoRa, Z-Wave, industrial sensors, baby monitors. |
| **No enterprise vendor support** | Cisco, Aruba, Ruckus, Extreme — none ship HaLow APs. |
| **Separate NOC visibility** | Two wireless platforms to monitor on game day. |
| **Tag availability** | HaLow asset tags are niche ($20-40). BLE tags are commodity ($5-15). |
| **Accuracy** | ~15-25m (RSSI from sparse HaLow APs) — worse than BLE on existing APs. |

**Cost estimate for HaLow overlay**: 20-40 HaLow APs (~$300-500 each) + gateway/controller (~$2-5K) + custom integration + HaLow tags ($20-40 each) + ongoing management = **$50-100K+ with zero ecosystem leverage**.

---

## 3. WiFi-Based Location Services That Exist TODAY

### Cisco Spaces (formerly DNA Spaces / CMX)
- **How**: RSSI fingerprinting from existing APs — no additional hardware
- **Accuracy**: **5-10m** zone-level (RSSI), **1-3m** with Wi-Fi RTT (802.11mc FTM)
- **What it tracks**: Any WiFi client (associated or probing), BLE tags via AP BLE radio
- **Integration**: Native to Catalyst 9800 → Cisco Spaces cloud dashboard
- **Cost**: Cisco Spaces licensing (included in DNA Advantage/Premier or ~$100/AP/yr for Spaces ACT)

### Cisco Catalyst Center Location Analytics
- **How**: Leverages AP RSSI data + optional hyperlocation antenna modules
- **Accuracy**: 5-10m standard, **1-3m with Hyperlocation** (dedicated module on C9130/9136)

### Wi-Fi RTT / Fine Time Measurement (802.11mc)
- **How**: Time-of-flight ranging between FTM-capable AP and client/tag
- **Accuracy**: **1-2m** indoors
- **Requirement**: Both AP and client must support FTM. C9136AXI and C9166 support it.
- **Limitation**: Android 9+ supports it, iOS does not expose to third-party apps.

### Wi-Fi 6E/7 FTM Improvements
- **802.11az (Next Generation Positioning)**: Sub-meter accuracy target. Cisco C9166/9167 APs being updated.

---

## 4. BLE vs WiFi for Asset Tracking — BLE Is the Real Answer

| Factor | BLE (via AP radio) | WiFi RSSI | WiFi RTT (FTM) | 802.11ah |
|--------|-------------------|-----------|-----------------|----------|
| **Accuracy** | 3-8m (RSSI), 1-3m (AoA) | 5-15m | 1-2m | 15-25m |
| **Tag battery life** | 1-5 years (coin cell) | N/A | N/A | 2-5 years |
| **Tag cost** | **$5-15** | N/A | N/A | $20-40 |
| **Additional infrastructure** | **NONE** | NONE | NONE | Full overlay |
| **Management integration** | Cisco Spaces | Cisco Spaces | Cisco Spaces | Separate |
| **Ecosystem maturity** | Massive | Mature | Growing | Nascent |

### Cisco AP BLE Radios — What You Already Have
- **C9130AXI**: Built-in BLE 5.0 radio
- **C9136AXI**: Built-in BLE 5.0 + dedicated IoT radio
- **C9166**: Built-in BLE 5.2 radio
- All integrate with **Cisco Spaces** for BLE asset tracking out of the box

### How It Works
1. Attach a $5-15 BLE beacon tag (iBeacon/Eddystone) to each asset
2. Your 500-3000 APs act as BLE listeners
3. Cisco Spaces trilaterates position from multiple AP RSSI readings
4. Dashboard shows asset locations with geofence alerts

**With your AP density (1 per 100-150 seats), BLE accuracy will be at the high end (3-5m).**

### Compatible BLE Tag Vendors
- **AiRISTA Flow** (formerly Ekahau B4 tags)
- **Kontakt.io** — SmartBeacon
- **Minew** — low-cost tags, multiple form factors
- **Estimote** — industrial/ruggedized
- **HID Global** — badge-integrated BLE

---

## 5. UWB (802.15.4z) — The Precision Answer

| Factor | UWB | BLE |
|--------|-----|-----|
| **Accuracy** | **10-30cm** | 3-8m |
| **Infrastructure** | Dedicated UWB anchors ($200-500 each) | Existing APs |
| **Tag cost** | $30-100 | $5-15 |
| **Battery life** | 6-18 months | 1-5 years |
| **Use case** | Player/official tracking, precision logistics | General asset tracking |
| **Products** | Zebra MotionWorks, Qorvo DW3000, Ubisense | Dozens |

**Verdict**: Overkill for stadium ops. This is what the NFL uses for player tracking (Zebra RTLS), not for finding wheelchairs.

---

## 6. Recommended Phasing

### Phase 1 — Immediate (Zero Additional Infrastructure)
1. Enable BLE scanning on all APs (checkbox in WLC RF profile)
2. License Cisco Spaces (ACT tier for asset tracking)
3. Deploy BLE tags on priority assets: AEDs, wheelchairs, portable radios, vendor carts
   - 50-100 tags @ $10 each = **$500-1,000**
   - Total Phase 1 cost: **$5-15K** (mostly Spaces licensing)
4. Configure geofence alerts

### Phase 2 — Enhancement
5. Add BLE Angle-of-Arrival on C9136AXI APs (1-3m accuracy)
6. Wi-Fi RTT for tracked staff devices (Android) supporting FTM
7. Integrate with existing CMMS for automated asset audit

### Phase 3 — Only If Precision Required
8. UWB for specific high-value zones (medical staging, player tunnel)

### Do NOT Do
- ❌ Deploy 802.11ah infrastructure
- ❌ Deploy Ekahau RTLS (legacy approach)
- ❌ Build custom location engines

---

## 7. Does HaLow Add Anything the Existing Infrastructure Can't?

| HaLow Claimed Advantage | Existing Infrastructure |
|--------------------------|------------------------|
| Long range | AP density means BLE covers every square meter |
| Battery life (2-5yr) | BLE tags: 1-5 years on coin cell |
| IP-native | VLAN 60 already provides IP connectivity |
| 8,000+ devices/AP | 500+ APs × 20 tags = 10,000+ assets |
| Sub-1 GHz penetration | APs every 30-50 feet — penetration isn't the problem |

**The only scenario where HaLow wins**: Tracking assets across an **outdoor campus with no WiFi AP coverage** — parking lots, remote storage, practice fields 500m+ away. Even then, LoRaWAN with GPS tags is cheaper and more mature.
