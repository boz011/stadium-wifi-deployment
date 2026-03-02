# VLAN Design

## Overview

VLAN segmentation is the backbone of security and traffic management in a large-scale stadium WiFi deployment. Each user class, management function, and IoT system is isolated into its own broadcast domain with dedicated subnets, DHCP scopes, and access control policies. Inter-VLAN routing is performed at the distribution/core layer (L3 switching) — never at the access layer.

---

## VLAN Table

| VLAN ID | Name | Subnet | CIDR | Gateway | Purpose | ACL Notes |
|---------|------|--------|------|---------|---------|-----------|
| 10 | Management | 10.10.10.0 | /24 | 10.10.10.1 | AP management, WLC management, switch management | Restricted — no client access; SSH/HTTPS from NOC only |
| 20 | Staff / Corporate | 10.10.20.0 | /22 | 10.10.20.1 | Staff devices, ticketing systems, internal apps | Access to internal resources + internet; 802.1X required |
| 30 | Public / Guest WiFi | 10.10.32.0 | /17 | 10.10.32.1 | Fan/spectator internet-only access | Internet-only; no access to any RFC 1918 space; captive portal enforced |
| 40 | Premium / Suites | 10.10.64.0 | /20 | 10.10.64.1 | Premium seating, luxury suites, club level | Internet + limited internal services (streaming); higher bandwidth per client |
| 50 | Media / Press | 10.10.80.0 | /22 | 10.10.80.1 | Press box, broadcast crews, media workrooms | Internet + media upload endpoints; higher QoS priority |
| 60 | IoT & Sensors | 10.10.90.0 | /23 | 10.10.90.1 | IP cameras, environmental sensors, digital signage | No internet; internal only; device cert or MAB auth |
| 70 | Vendors / POS | 10.10.92.0 | /23 | 10.10.92.1 | Point-of-sale terminals, vendor equipment | PCI-DSS compliant segment; internet to payment gateway only |

### Subnet Sizing Rationale

- **VLAN 30 (Guest)** uses a /17 (32,766 hosts) to accommodate 30,000–80,000+ fan devices. In practice, use DHCP lease times of 2–4 hours and consider splitting into multiple /19 or /20 subnets mapped to venue sections to limit broadcast domain size.
- **VLAN 40 (Premium)** uses a /20 (4,094 hosts) — suites and club seats are a smaller population but expect higher throughput.
- **VLAN 10 (Management)** is intentionally small (/24) — only infrastructure devices reside here.

---

## Inter-VLAN Routing

All inter-VLAN routing is handled at the **distribution/core switch** using L3 SVIs (Switched Virtual Interfaces). The core switch acts as the default gateway for every VLAN.

```
! Core Switch — SVI Configuration Example
interface Vlan10
 description Management - APs + WLC
 ip address 10.10.10.1 255.255.255.0
 ip helper-address 10.10.10.50
 no ip proxy-arp
!
interface Vlan30
 description Public Guest WiFi
 ip address 10.10.32.1 255.255.128.0
 ip helper-address 10.10.10.50
 ip access-group GUEST-ACL in
 no ip proxy-arp
```

### Routing Policy

| Source VLAN | Destination | Action |
|-------------|-------------|--------|
| 10 (Mgmt) | Any | Permit (infrastructure) |
| 20 (Staff) | Internet + Internal | Permit |
| 30 (Guest) | Internet only | Permit; deny all RFC 1918 |
| 40 (Premium) | Internet + streaming | Permit; deny internal except media servers |
| 50 (Media) | Internet + upload endpoints | Permit |
| 60 (IoT) | Internal only | Deny internet; permit internal management |
| 70 (POS) | Payment gateways only | Deny all except whitelisted IPs |

---

## DHCP Scope Design

DHCP is centralized on a dedicated server (or WLC internal pools for smaller deployments). The core switch relays DHCP via `ip helper-address`.

| VLAN | DHCP Pool | Range | Lease Time | DNS | Notes |
|------|-----------|-------|------------|-----|-------|
| 10 | MGMT-POOL | 10.10.10.100–.250 | 8 hours | Internal DNS | Static assignments preferred for WLC/switches |
| 20 | STAFF-POOL | 10.10.20.100–10.10.23.250 | 8 hours | Internal DNS | 802.1X assigns VLAN dynamically |
| 30 | GUEST-POOL | 10.10.32.100–10.10.63.250 | 2 hours | Public DNS (8.8.8.8, 1.1.1.1) | Short lease to recycle IPs during events |
| 40 | PREMIUM-POOL | 10.10.64.100–10.10.79.250 | 4 hours | Public DNS | |
| 50 | MEDIA-POOL | 10.10.80.100–10.10.83.250 | 8 hours | Public DNS | |
| 60 | IOT-POOL | 10.10.90.100–10.10.91.250 | 24 hours | Internal DNS | Static preferred for cameras |
| 70 | POS-POOL | 10.10.92.100–10.10.93.250 | 12 hours | Internal DNS | |

### DHCP Options

- **Option 43** — WLC IP address for AP discovery (VLAN 10)
- **Option 60** — Vendor class identifier for Cisco APs
- **Option 150** — TFTP server for firmware delivery

---

## Security Policies Per VLAN

### VLAN 10 — Management
- **Access**: ACL restricts to NOC source IPs only
- **Protocols**: SSH, HTTPS, SNMP (v3 only), CAPWAP
- **No client devices** permitted on this VLAN
- **Port security**: Static MAC on switch management ports

### VLAN 20 — Staff / Corporate
- **Authentication**: 802.1X with RADIUS (ISE recommended)
- **Encryption**: WPA3-Enterprise (or WPA2-Enterprise minimum)
- **DNS**: Internal DNS with split-horizon
- **Posture**: NAC posture check recommended (antivirus, OS patch level)

### VLAN 30 — Public / Guest
- **Authentication**: Captive portal (email/social sign-in or sponsor-based)
- **Encryption**: WPA3-OWE (Opportunistic Wireless Encryption) or Open+Captive Portal
- **Rate limiting**: Per-client bandwidth cap (e.g., 10 Mbps down / 5 Mbps up)
- **ACL**: Deny all RFC 1918 destinations; permit DNS, HTTP/HTTPS only
- **Session timeout**: 4 hours; re-auth required
- **Client isolation**: Enable peer-to-peer blocking

### VLAN 40 — Premium / Suites
- **Authentication**: PSK or simplified captive portal (welcome page)
- **Rate limiting**: Higher cap (e.g., 25 Mbps down / 10 Mbps up)
- **QoS**: Higher WMM priority than VLAN 30

### VLAN 50 — Media / Press
- **Authentication**: 802.1X or credential-based portal
- **Rate limiting**: No per-client cap (or high cap: 50+ Mbps)
- **QoS**: Highest data priority; video/voice traffic marked appropriately
- **Dedicated uplink** bandwidth reservation recommended

### VLAN 60 — IoT & Sensors
- **Authentication**: MAB (MAC Authentication Bypass) or certificate-based 802.1X
- **No internet access** — internal communication only
- **Micro-segmentation**: SGT (Scalable Group Tags) via Cisco TrustSec recommended
- **Monitoring**: NetFlow/sFlow for anomaly detection

### VLAN 70 — Vendors / POS
- **Authentication**: 802.1X or MAB with profiling
- **PCI-DSS compliance**: Isolate from all other VLANs; encrypt POS traffic end-to-end
- **ACL**: Permit only payment processor IPs (whitelist)
- **Logging**: Full session logging for audit trail

---

## Design Considerations

### Broadcast Domain Size
Guest VLANs with 30,000+ clients create massive broadcast domains. Mitigations:
- **Split VLAN 30** into zone-based sub-VLANs (e.g., VLAN 30 = North stands, VLAN 31 = South stands)
- **Proxy ARP** on the gateway to suppress ARP broadcasts
- **ARP suppression** at the WLC level
- **mDNS gateway** to prevent Bonjour/mDNS storms

### Dynamic VLAN Assignment
Use RADIUS-returned VLAN attributes to place clients on the correct VLAN dynamically:
- Attribute 64: Tunnel-Type = VLAN
- Attribute 65: Tunnel-Medium-Type = 802
- Attribute 81: Tunnel-Private-Group-ID = VLAN-ID

This allows a single SSID to serve multiple user classes based on their authentication credentials.
