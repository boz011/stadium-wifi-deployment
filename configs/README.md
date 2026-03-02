# Configs

This directory is reserved for **sanitized** configuration snippets.

> ⚠️ **Never commit production credentials, IP addresses, or SNMP community strings.**

Place sanitized examples here such as:

- `wlc-base.cfg` — Catalyst 9800 base configuration
- `switch-access.cfg` — PoE access switch port profiles
- `switch-core.cfg` — Core/distribution switch VLAN and routing config
- `firewall-acls.cfg` — Captive portal and rate-limiting rules
- `dhcp-scopes.cfg` — DHCP pool definitions per VLAN

Replace sensitive values with placeholders like `<WLC_MGMT_IP>`, `<RADIUS_SECRET>`, etc.
