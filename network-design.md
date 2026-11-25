# Secure Home Network Design Sheet
## Overall Best-Practice Baseline (Generic)

**Version:** 1.25  
**Date:** 2026-11-25  

**Purpose:** Generic, deployable foundation for any household. Hardware and software are clearly separated. Designed to be extended with specific requirements later.

**Primary project repository:** https://github.com/joselvelez/bastion  
All design documents, AI system code, scripts, prompts, and related work are maintained in this repo.

### High-Level Summary

This design provides a segmented, defense-in-depth home network with:

- **Core network:** VLAN-based segmentation behind a stateful firewall (default-deny).
- **Always-on Omarchy node:** Small dedicated Minisforum mini-PC running Omarchy (Arch-based) that hosts the Tailscale service (subnet router) and **Bastion** (the local AI system) for continuous network monitoring and protection.
- **Remote access:** Tailscale (WireGuard-based zero-trust mesh) as the primary method — no inbound port forwarding required. Runs on the Omarchy node.
- **Physical internet kill switch:** DIY ESP32 + micro servo button pusher that physically presses a modem power button or dedicated power switch. Controllable locally, via Tailscale, and automatically by **Bastion** when threat level is high.
- **AI communication requirement:** Every hardware and software component in the design must be able to communicate with **Bastion** (the AI system running on the Omarchy node). Components that cannot be observed or controlled by Bastion are not used.
- **Bastion (primary project repo):** All design, documentation, and implementation live in the public repository:  
  https://github.com/joselvelez/bastion (Apache 2.0).  
  This design document is maintained there as the single source of truth.

All major elements (segmentation, Omarchy always-on node, Tailscale, AI monitoring, and the physical kill switch) are first-class parts of the design.

## Table of Contents

- [1. Goals & Principles](#1-goals--principles)
- [2. High-Level Architecture](#2-high-level-architecture)
- [3. Recommended Network Zones](#3-recommended-network-zones)
- [4. Hardware Components](#4-hardware-components)
- [5. Software Components & Configuration](#5-software-components--configuration)
- [6. Tailscale Integration](#6-tailscale-integration)
- [7. Physical Internet Kill Switch — DIY ESP32 + Micro Servo Button Pusher](#7-physical-internet-kill-switch--diy-esp32--micro-servo-button-pusher)
- [8. Omarchy Always-On Node — Tailscale + Local AI Network Monitor](#8-omarchy-always-on-node--tailscale--local-ai-network-monitor)
- [9. Inter-Zone Traffic Policy (Baseline)](#9-inter-zone-traffic-policy-baseline)
- [10. Security Controls Checklist](#10-security-controls-checklist)
- [11. High-Level Implementation Order](#11-high-level-implementation-order)
- [12. Maintenance & Operations](#12-maintenance--operations)
- [13. Example Setup](#13-example-setup)

---

## 1. Goals & Principles

| Goal | Description |
|------|-------------|
| **Confidentiality** | Protect personal data and devices from external and internal threats |
| **Integrity** | Prevent unauthorized modification of devices, traffic, or configurations |
| **Availability** | Maintain reliable internet and local services |
| **Least Privilege** | Devices and users get only the access they need |
| **Defense in Depth** | Multiple layers (physical, network, host, application) |
| **Segmentation** | Isolate risk zones so a compromise in one area cannot freely reach others |
| **Observability** | Logging and monitoring so issues are detectable |
| **Controlled Connectivity** | Remote access via Tailscale; physical ability to disconnect internet on demand |
| **Automated Protection** | **Bastion** (local AI system on the always-on Omarchy node) monitors the network and can trigger the kill switch |

**Default stance:** Deny by default, allow explicitly.

---

## 2. High-Level Architecture

### 2.1 AI Communication Requirement

**Mandatory constraint**: Every hardware and software component used in this design must be able to communicate with **Bastion** (the AI system running on the Omarchy node).

- The AI must be able to observe the component (logs, status, telemetry, API, etc.).
- The AI must be able to act on or control the component where relevant (commands, configuration, triggers).
- Communication can occur over the local network, Tailscale, local APIs, syslog, or other controlled methods.
- Any component that cannot meet this requirement is excluded from the design.

This makes **Bastion** the central coordination point for the entire network.

## 2.3 ISP Connection Model

**Required:** Bridge / passthrough / IP passthrough.  
**Double NAT is not permitted**, except when using Starlink.

- The ISP device (modem/ONT or gateway) is used only as a Layer 2 handoff.
- One Ethernet cable connects the ISP device to the WAN port of the firewall.
- On the ISP device:
  - Wi-Fi is disabled
  - Routing, DHCP, and firewall features are disabled or bypassed
  - Bridge / passthrough / IP passthrough mode is enabled so the firewall receives a public IP
- The firewall is the only router, DHCP server, and firewall for the network.
- Physical kill switch controls power to the ISP device (or its uplink) when needed.

| Component | Observation | Control | Status | Required Integration |
|-----------|-------------|---------|--------|----------------------|
| Omarchy node | Yes | Yes | Pass | Primary AI host |
| Tailscale (on Omarchy) | Yes | Yes | Pass | Local API / logs |
| ESP32 kill switch | Yes | Yes | Pass | Status + trigger endpoints |
| Firewall (OPNsense or equivalent) | Partial | Limited | Needs work | Syslog to Omarchy + scriptable API or config interface |
| Managed switch | Partial | Limited | Needs work | SNMP (or API) reachable by Bastion |
| Access points | Partial | Limited | Needs work | SNMP / controller API or logs to Omarchy |
| End devices | Indirect (via firewall/telemetry) | Limited | Acceptable | Observation through network telemetry is sufficient |
| ISP modem | No | Via kill switch only | Pass | Power controlled by ESP32 |

**Specific switch models to consider** (must support reliable SNMP or equivalent for AI communication):

1. **MikroTik CRS310-8G+2S+IN** (Primary)  
   - 8× 2.5GbE + 2× 10G SFP+  
   - Full RouterOS with strong SNMP (v1/v2c/v3), API, and scripting. Excellent for AI observation and control. Best future-proof option.

2. **TP-Link TL-SG2008** (or equivalent JetStream L2+)  
   - 8× Gigabit  
   - Solid SNMP support, full 802.1Q VLANs, CLI + web. Reliable and easier to manage than MikroTik while still meeting the AI requirement.

3. **MikroTik CRS326-24G-2S+RM** (or similar CRS)  
   - 24× Gigabit + 2× SFP+  
   - Same RouterOS strengths (SNMP + API). Choose if higher port density is needed.

**Avoid** pure “Easy Smart” switches (e.g. basic NETGEAR GS308E) for this design — their SNMP support is limited or unreliable.

**Access points**: MikroTik **cAP ax** (ceiling/wall) and **wAP ax** (wall). Each SSID maps to a VLAN. Bastion is the management interface. Initial example setup uses one of each.

All core infrastructure (firewall, switch, APs) must be configured so the AI can observe them and act where relevant.

```
Internet
   │
   ▼
[ ISP Modem / ONT ]  ←── Physical Kill Switch (optional)
   │                     ESP32 + micro servo presses power button
   ▼                     or dedicated power switch in modem path
[ Firewall / Router ]  ← Core security device
   │                     (stateful firewall, inter-VLAN policy)
   ├── WAN interface     (no inbound ports required)
   │
   └── LAN / Trunk
         │
         ▼
   [ Managed Switch(es) ]
         │
         ├── VLAN 10  Trusted / Personal
         ├── VLAN 20  Servers / NAS / Home Lab
         │              └── [ Omarchy Always-On Node ]
         │                    - Tailscale subnet router
         │                    - Bastion (local AI system)
         │                    - Can trigger kill switch
         ├── VLAN 30  IoT / Smart Home
         ├── VLAN 40  Guest
         ├── VLAN 50  Work / Sensitive (optional)
         └── VLAN 99  Management
               │
               ├── Wired clients
               └── Access Points (SSID → VLAN mapping)
```

**Key points:**
- Firewall is the single point of inter-zone routing and policy enforcement.
- All inter-VLAN traffic must pass through the firewall.
- Prefer 802.1Q VLAN tagging end-to-end.
- **Tailscale** runs primarily on the always-on **Omarchy node** (as subnet router). Provides remote access with no open inbound ports.
- **Omarchy always-on node** (Servers or Management VLAN) hosts Tailscale and **Bastion**, which continuously monitors the network and can trigger the kill switch.
- **Physical kill switch** sits at the modem power path. Controllable locally, via Tailscale, and automatically by **Bastion**.
- Hardware (Section 4) provides the physical and Layer 2/3 foundation; software (Section 5) implements policy, filtering, remote access, and the kill-switch logic.

---

## 3. Recommended Network Zones

| VLAN ID | Name              | Subnet Example     | Typical Devices                  | Default Policy |
|---------|-------------------|--------------------|----------------------------------|----------------|
| 10      | Trusted           | 192.168.10.0/24    | Laptops, phones, desktops        | Internet + controlled access to Servers/IoT |
| 20      | Servers           | 192.168.20.0/24    | NAS, home servers, media         | Internet (updates), limited inbound from Trusted |
| 30      | IoT               | 192.168.30.0/24    | Cameras, TVs, lights, sensors    | Internet only (or specific controller) |
| 40      | Guest             | 192.168.40.0/24    | Visitors                         | Internet only, client isolation |
| 50      | Work / Sensitive  | 192.168.50.0/24    | Work laptops, financial devices  | Internet + restricted local |
| 99      | Management        | 192.168.99.0/24    | Switches, APs, firewall mgmt, kill-switch ESP32 | Admin devices only |

**Notes:**
- Avoid using VLAN 1 for production traffic (common attack surface).
- Subnets can be adjusted (10.x.x.x or others) as long as they do not overlap.
- Add or remove zones based on household needs.
- Kill-switch ESP32 belongs on Management (or Servers) VLAN.

---

## 4. Hardware Components

Hardware provides the physical and Layer 2/3 foundation. Software (Section 5) runs on and configures these devices.

| Component          | Role                              | Key Requirements                          | Example Categories                  |
|--------------------|-----------------------------------|-------------------------------------------|-------------------------------------|
| **Firewall / Router** | Routing, stateful firewall, inter-VLAN policy | VLAN-capable, 2+ Intel NICs, 8 GB RAM | Protectli Vault (example: V1410), Beelink EQ14, or equivalent mini-PC running pfSense / OPNsense |
| **Managed Switch** | VLAN segmentation, PoE delivery   | 802.1Q support, managed, preferably PoE   | UniFi, TP-Link Omada, Netgear, Cisco SG series |
| **Access Points**  | Wireless coverage and SSID-to-VLAN mapping | Multi-SSID, VLAN tagging, WPA3 support    | MikroTik **cAP ax** (ceiling) and **wAP ax** (wall). Bastion is the management interface. |
| **Modem / ONT**    | ISP handoff                       | Bridge mode preferred                     | ISP-provided or third-party         |
| **Omarchy Always-On Node** | Tailscale subnet router + local AI monitor | Minisforum mini-PC (x86_64), 16–32 GB+ RAM preferred | Minisforum UM890 Pro (or MS series) running Omarchy |
| **Kill Switch Actuator** | Physically presses power button/switch | ESP32 + micro servo (SG90/MG90S)         | DIY (see Section 7)                 |
| **Cabling**        | Physical connectivity             | Cat5e minimum, Cat6 preferred             | —                                   |

**Hardware selection guidance:**
- Match firewall performance to your ISP speed (especially if enabling IDS/IPS).
- Prefer devices that receive regular security updates.
- Separate wireless (APs) from the firewall when possible for better coverage and flexibility.
- Ensure the firewall, switch, and APs all support 802.1Q VLANs.
- For the kill switch: ESP32 (S3 preferred) + metal-gear micro servo + simple mount.

---

## 5. Software Components & Configuration

Software runs on the hardware above and implements policy, filtering, remote access, monitoring, and kill-switch control.

### 5.1 Firewall Operating System / Firmware

- **Primary:** pfSense (preferred) or OPNsense
- Strong preference for Intel NICs (i226 preferred)
- Default-deny inter-VLAN policy
- Syslog forwarding to the Omarchy node (Bastion) is required
- Change all default credentials immediately.
- Disable remote management from the WAN.
- Disable UPnP and WPS.
- Enable a stateful firewall with a default-deny posture.
- Create explicit allow rules only for required traffic.
- Keep the OS/firmware updated.
- Install Tailscale (see Section 6).

### 5.2 Wireless Configuration (on Access Points)
- WPA3 (or WPA2-AES if WPA3 is unavailable).
- Strong unique passphrase (16+ characters).
- Separate SSIDs mapped to the appropriate VLANs (Trusted, IoT, Guest, etc.).
- Client isolation enabled on Guest and IoT SSIDs where supported.
- Disable legacy protocols (WEP, TKIP).

### 5.3 Network Services
- DHCP scopes defined per VLAN on pfSense (see Section 5.4).
- Static DHCP reservations for infrastructure devices and important servers.
- NTP for time synchronization across the network.
- Logging enabled and retained (local or remote syslog to Bastion).

### 5.4 DNS & DHCP

#### Design Decisions
| Component | Choice | Rationale |
|-----------|--------|-----------|
| **DHCP** | pfSense (one scope per VLAN) | Tightly coupled to interfaces and VLANs. Keeps addressing reliable even if Bastion is down. |
| **DNS filtering** | AdGuard Home on the Omarchy node (Bastion) | High-signal data stays where Bastion can observe and act. Clean API for future automation. |
| **Recursive resolution** | Unbound on pfSense (or local to Bastion) | AdGuard Home forwards non-blocked queries to Unbound. Avoids single upstream resolver seeing all queries. |
| **Client DNS** | Only the designated resolver is allowed | Firewall rules block outbound DNS (53/853) from clients except to AdGuard Home, preventing bypass. |

#### Why this split
- **Bastion is the management layer.** AdGuard Home’s query log and API give Bastion direct visibility without scraping syslog.
- **DHCP stays on pfSense** so new devices can still obtain addresses if Bastion is offline.
- **Filtering is a first-class signal** for threat detection. Placing it on Bastion makes the data native to the AI.
- **Separation of duties**: pfSense handles routing and addressing; Bastion handles observation and DNS policy.

#### DHCP Layout (same pattern on every VLAN)
| Range | Purpose | Example (Trusted /24) |
|-------|---------|------------------------|
| `.1` | Gateway (pfSense) | `192.168.10.1` |
| `.2`–`.49` | Reserved / static (infrastructure) | Bastion, switch, APs |
| `.50`–`.199` | DHCP pool | Normal clients |
| `.200`–`.249` | DHCP reservations | Devices that should keep a stable IP |
| `.250`–`.254` | Spare | Testing |

#### Per-VLAN DNS Behavior
| VLAN | DNS server handed out | Notes |
|------|-----------------------|-------|
| Trusted | AdGuard Home (Bastion IP) | Full filtering + local names |
| Servers | AdGuard Home | Same |
| IoT | AdGuard Home | Can use stricter lists |
| Guest | AdGuard Home | Guest-specific rules; short lease |
| Management | AdGuard Home | |

#### Key Settings
- **Lease time**: 8–24 hours for Trusted/Servers; 1–4 hours for Guest/IoT.
- **Register DHCP leases in DNS**: Enabled so local hostnames resolve.
- **Static mappings**: Bastion, switch, APs, firewall management.
- **Block outbound DNS** from clients except to the designated resolver.

#### Bastion Integration
- AdGuard Home query log + API → Bastion sees queried domains and can alert or adjust rules.
- pfSense DHCP leases → syslog or API so Bastion knows active devices.
- Optional later: Bastion can add/remove blocklist rules or whitelist domains via the AdGuard API.

### 5.5 Logging

#### Design Decisions
| Source | Method | Destination | Rationale |
|--------|--------|-------------|-----------|
| **pfSense** | Remote syslog (UDP 514) | Bastion | Firewall, DHCP, and system events. Native remote logging. |
| **MikroTik CRS310** | Remote syslog (UDP 514) | Bastion | Switch events, interface changes, auth. RouterOS remote logging. |
| **MikroTik APs** (cAP ax / wAP ax) | Remote syslog (UDP 514) | Bastion | Client associations, wireless events. Same as switch. |
| **AdGuard Home** | Local API + query log | Bastion (same host) | Query log is already on Bastion. API is the clean access path. |
| **Bastion itself** | Local journal / files | Local | Host and AI process logs stay on the node. |

#### Why centralized on Bastion
- Bastion is already the observation and management layer.
- One place for the AI to read firewall blocks, DHCP leases, switch events, AP client activity, and DNS queries.
- Avoids standing up a separate logging server.
- Fits the “single interface” principle.

#### What is sent
| Source | Minimum topics / content |
|--------|--------------------------|
| pfSense | System, firewall (blocks + key allows), DHCP |
| MikroTik switch | info, warning, error, critical + interface / auth topics |
| MikroTik APs | info, warning, error, critical + wireless topics |
| AdGuard Home | Query log via API (domains, clients, blocked vs allowed) |

#### Retention & access
- Bastion retains logs long enough for the AI to detect patterns (recommended: 7–30 days hot, longer if storage allows).
- Logs are readable by Bastion processes; no separate log UI is required for the user.
- Optional later: structured storage (e.g. simple database or log files) so Bastion can query efficiently.

#### Notes
- Use UDP 514 for simplicity on the local network. TCP or a higher port can be used later if reliability becomes an issue.
- NTP must be consistent across all devices so log timestamps can be correlated.
- Firewall rules must allow the infrastructure devices to reach Bastion on the syslog port.

### 5.6 Remote Access & Always-On Node
- **Tailscale** is the recommended primary method (detailed in Section 6). Runs on the Omarchy always-on node as the subnet router.
- No inbound port forwarding required.
- The Omarchy node also hosts **Bastion** for network monitoring (detailed in Section 8).

### 5.6 Kill Switch Control
- ESP32 firmware: ESP32-SwitchBot (includes local web UI + Tailscale) or ESPHome.
- Controllable locally, via Tailscale, and automatically by **Bastion**.
- See Section 7 for full details.

### 5.8 Optional Advanced Software Features
- IDS/IPS (Suricata or Snort).
- GeoIP blocking.
- Traffic shaping / QoS.
- Certificate-based authentication (802.1X) for higher-security environments.
- Endpoint protection on trusted devices.

---

## 6. Tailscale Integration

Tailscale is a zero-trust mesh VPN built on WireGuard. It is the recommended remote-access method in this design. It provides secure remote access without requiring inbound port forwarding on the home firewall and complements (rather than replaces) local VLAN segmentation and firewall policy.

All guidance below is drawn from current Tailscale documentation (last validated late 2025–2026).

### How Tailscale Impacts the Design

| Aspect | Impact | Recommendation |
|--------|--------|----------------|
| **Remote Access** | Primary method for reaching home resources from outside | Prefer Tailscale over a traditional self-hosted VPN server |
| **Inbound Ports** | None required in most cases | Keep the WAN interface fully closed to unsolicited inbound traffic |
| **Devices that cannot run Tailscale** | Reachable via a subnet router | Use a subnet router only for those devices; install the client directly whenever possible |
| **Access Control** | Identity-based + policy file (grants/ACLs) | Enforce least privilege in the Tailscale policy file *and* keep local firewall rules |
| **Local Segmentation** | Unchanged | VLANs and inter-zone firewall rules remain the primary defense against local compromise |
| **Exit Node** | Optional | Enable only when you specifically want remote traffic to exit via the home connection |
| **Kill Switch** | Remote triggering possible | ESP32 can run Tailscale or be reached via subnet router |

### Recommended Placement

**Preferred:** Install Tailscale on the always-on **Omarchy node** (Servers or Management VLAN). This node acts as the dedicated subnet router and also hosts the local AI monitoring model.

**Alternative:** Install Tailscale on the firewall itself (e.g., OPNsense `os-tailscale` plugin). The Omarchy node can still run the AI model and control the kill switch.

- On OPNsense (community-supported): Install the `os-tailscale` plugin via **System → Firmware → Plugins** (enable “Show community plugins”). Available since OPNsense v24.711.
- After authentication, the `tailscale0` interface appears. Assign it under **Interfaces → Assignments**, enable it, then add a Pass firewall rule on that interface (default policy otherwise blocks even traffic to the firewall host itself).
- Advertise only the specific subnets required.
- Optionally enable as an exit node if desired.

**Alternative:** Run a dedicated always-on subnet router (Linux device, NAS, or small VM) in a trusted or management zone. The local firewall continues to enforce zone policy.

### Subnet Router Guidance (per official docs)

- Subnet routers are available on all plans.
- Prefer installing the Tailscale client directly on devices whenever possible — this gives better security and performance (end-to-end encryption).
- A subnet router is appropriate for devices that cannot run the client (printers, some cameras, certain NAS units, etc.).
- Devices behind a subnet router do not count toward device limits.
- By default, subnet routers perform SNAT (traffic appears to come from the router). SNAT can be disabled if source IP preservation is required.
- Advertise routes with the appropriate platform method (e.g., `tailscale set --advertise-routes=...` on Linux, or the equivalent in the OPNsense plugin / admin console).
- Approve the advertised routes in the Tailscale admin console (Machines → Subnets) unless autoApprovers are configured in the policy file.
- Access rules in the Tailscale policy file still apply to traffic destined for the advertised subnets.

**Best practice:** Advertise only the subnets that remote users actually need. Do not advertise broad ranges such as the entire private address space or untrusted zones (IoT, Guest) by default.

### Firewall Considerations

- Most deployments require **no inbound ports**.
- Outbound traffic that helps connectivity (when both ends are behind difficult NATs):
  - TCP to port 443
  - UDP from the device’s WireGuard port (default 41641) to any destination
  - UDP to port 3478 (STUN)
- On OPNsense, after assigning `tailscale0`, create explicit Pass rules on that interface. Restrict sources (via aliases of specific Tailscale IPs or groups) rather than allowing the entire CGNAT range indiscriminately if tighter control is desired.
- Local inter-VLAN firewall rules remain in force and continue to protect the network from compromised devices on the LAN.
- For better direct (non-relay) connections from LAN clients that themselves run Tailscale, OPNsense’s default NAT behavior can be adjusted with static port mapping or by enabling NAT-PMP (see Tailscale’s OPNsense documentation for current steps).

### Benefits in This Design

- No open inbound ports on the WAN.
- Identity-based access that works across CGNAT and restrictive networks.
- Complements rather than replaces VLAN segmentation and local firewall policy.
- Secure remote administration of infrastructure without exposing management interfaces to the internet.
- Enables remote triggering of the physical kill switch without additional exposure.

---

## 7. Physical Internet Kill Switch — DIY ESP32 + Micro Servo Button Pusher

Optional but first-class feature. A low-trust, fully local physical kill switch for internet access. An ESP32 microcontroller drives a micro servo that physically presses a button (modem power button or a dedicated power switch in the modem’s power path). Avoids cloud-dependent smart plugs; control stays under the owner’s firmware and network policy.

### Design Goals

- Physical actuation (presses a real button).
- Fully local control possible (no mandatory vendor cloud).
- Scriptable / automatable.
- Compatible with Tailscale for remote triggering without opening inbound ports.
- Placeable on a restricted VLAN.

### Recommended Implementation

**Primary reference project:** [psychoStark/ESP32-SwitchBot](https://github.com/psychoStark/ESP32-SwitchBot)

Features relevant to this design:
- Micro servo (SG90 / MG90S) presses a physical button.
- Local web UI (no third-party app required).
- Embedded Tailscale support for remote control over the existing Tailscale network.
- Interactive servo calibration.
- Standalone operation (Home Assistant optional).

**Alternative (lighter):** ESP32 + ESPHome with a template button that moves the servo to a press angle, holds briefly, then returns to the rest position. Reachable locally or via Tailscale.

### Hardware Details

| Component              | Recommendation                          | Notes |
|------------------------|-----------------------------------------|-------|
| Microcontroller        | ESP32 or ESP32-S3 (4 MB+ flash)        | S3 preferred for the reference project |
| Actuator               | TowerPro SG90 or MG90S micro servo     | Metal-gear (MG90S) preferred for durability |
| Power                  | 5 V USB supply (or better regulated 5 V) | Adequate for occasional button presses |
| Mounting               | 3D-printed or simple bracket           | Search “SG90 button pusher”, “DIY fingerbot”, or “servo switchbot mount” |
| Optional enclosure     | Small project box or 3D-printed case   | Protects electronics |

**Basic wiring:**
- Servo signal → free GPIO (example: GPIO 1 or GPIO 13)
- Servo VCC → 5 V
- Servo GND → GND (common ground with ESP32)

### Placement Options

1. Directly adjacent to the modem’s physical power button (if it fully powers the device off).
2. **Preferred:** On a dedicated physical power switch or large button inserted in the modem’s power path. The button can still be pressed by hand; the servo automates it.
3. On the rocker of a power strip that only powers the modem.

### Control Methods

| Method                   | Local? | Remote?       | Notes |
|--------------------------|--------|---------------|-------|
| Local web UI             | Yes    | Via Tailscale | Built into ESP32-SwitchBot |
| Home Assistant / ESPHome | Yes    | Via Tailscale | Template button entity |
| Script / API / curl      | Yes    | Via Tailscale | Simple HTTP endpoint |
| Physical press           | Yes    | —             | Always available if using a dedicated switch |

### Security Placement Recommendations

- Put the ESP32 on the **Management** or **Servers** VLAN (or a dedicated restricted VLAN).
- Deny internet access from that VLAN for the device itself (it only needs local reachability or Tailscale).
- If using Tailscale on the ESP32, apply least-privilege grants in the Tailscale policy file.
- Keep the local network firewall rules in place so the device cannot reach untrusted zones.

---

## 8. Omarchy Always-On Node — Dedicated Design

A small, always-on machine running **Omarchy** (Arch-based Linux distro by DHH). It is the primary host for Tailscale and **Bastion**. This section defines its role, placement, capabilities, and operational requirements in more detail.

### 8.1 Purpose & Roles

| Role | Description |
|------|-------------|
| **Tailscale Subnet Router** | Always-on Tailscale node that advertises only the required home subnets. Primary remote-access path. |
| **Bastion** | Local AI system that continuously evaluates network telemetry and threat level. |
| **Kill Switch Controller** | Can autonomously trigger the ESP32 physical kill switch when the AI determines the threat level is high enough. |
| **Telemetry Collector** | Receives and stores relevant logs/telemetry for the AI model to analyze. |

### 8.2 Placement & Network Requirements

- **VLAN**: Servers (VLAN 20) preferred, or Management (VLAN 99).
- **Addressing**: Static IP or DHCP reservation.
- **Access**:
  - Reachable from Trusted VLAN for administration (least privilege).
  - Can reach the ESP32 kill switch (local network or via Tailscale).
  - Can receive logs from the firewall (and optionally other sources).
  - No unnecessary internet access beyond Tailscale coordination and controlled model/package updates.
- **Physical**: Prefer a small form-factor machine that can run 24/7 with low power.

### 8.3 AI Monitoring Design

**Bastion** is the core protection mechanism on this node.

**Telemetry sources (baseline):**
- Firewall syslog (allowed/denied connections, interface events)
- DNS query logs (from local resolver)
- Optional: NetFlow / traffic summaries, Tailscale connection events

**Threat evaluation (conceptual):**
- The model scores anomalies, suspicious patterns, and known-bad indicators.
- A configurable threshold determines when the threat level is “high enough” to act.
- All decisions and actions are logged.

### 8.3.1 Recommended AI Models

**Primary recommendations** (both open-weight, Apache 2.0, actively maintained, practical on a 32 GB Minisforum):

1. **Qwen3.8 27B** (or latest Qwen3.5 27B-class) via Ollama  
   - Strong community adoption and proven tool-use support.  
   - Excellent general reasoning for log analysis and anomaly detection.

2. **Thinking Machines Inkling 2 Mini** via Ollama (or equivalent)  
   - 26.4B total / low active parameters (looped transformer).  
   - Explicitly designed for local deployment on 32 GB systems.  
   - Strong agentic / tool-use performance.  
   - Newer but high-profile lab (Mira Murati / Thinking Machines).

**Other solid alternatives**:
- Gemma 4 26B / 31B (Apache 2.0)
- Smaller Qwen3.5 models (9B / 4B) for lower resource use or lower latency

All inference remains fully local. The model receives structured telemetry summaries and produces structured decisions (alerts and optional kill-switch triggers).

**Best practices**:
- Prefer models with reliable structured output.
- Start with human-in-the-loop or rate limits on automatic kill-switch actions.
- Keep the model updated from official / community sources.
- Either primary model is acceptable; choose based on real-world testing on your hardware.

**Safeguards:**
- Physical button always remains usable by a human.
- All AI-initiated kill-switch events are logged with reasoning summary.
- Rate limiting or confirmation window possible (to be defined).
- Human override must always take precedence.

### 8.4 Software Stack

| Component | Recommendation | Notes |
|-----------|----------------|-------|
| **Base OS** | Omarchy (Arch + Hyprland) | Can be run with minimal desktop or effectively headless for always-on use. Harden accordingly. |
| **Tailscale** | Official client | Configure as subnet router; advertise only needed routes. |
| **Local LLM** | Ollama running Qwen3.8 27B (or latest Qwen3.5 27B-class) | Primary recommendation. Apache 2.0, actively maintained, strong tool use. Fits on 32 GB systems. Alternatives: Gemma 4 series or smaller Qwen models. |
| **Telemetry** | syslog-ng / rsyslog + simple scripts or lightweight collector | Feed structured events to the AI. |
| **Kill-switch control** | Local script or service that calls the ESP32 endpoint | Invoked by the AI decision logic. |
| **Hardening** | Minimal services, strong auth, regular updates, firewall on host | Treat as a high-value system. |

### 8.5 Hardware Guidance — Minisforum Platform

The design is built around a **Minisforum mini-PC** as the Omarchy always-on node.

**Recommended models:**
- **Primary recommendation**: Minisforum **UM890 Pro** (Ryzen 9 8945HS) — excellent Linux support, dual 2.5GbE, up to 96 GB RAM, quiet, efficient.
- **Alternatives**: MS-A2 or MS-01 series if you want stronger networking (10G) or more expansion.

**Minimum / preferred configuration:**
- **CPU**: Modern AMD Ryzen (HS or HX class) or recent Intel
- **RAM**: 16 GB minimum, **32 GB preferred** for comfortable local AI
- **Storage**: 512 GB+ NVMe SSD (1 TB preferred)
- **Networking**: Gigabit or better Ethernet (wired)
- **Power**: Low idle consumption preferred for 24/7 operation

**Why Minisforum:**
- Mature x86_64 platform with strong Linux driver support
- Good RAM expandability
- Compact, quiet form factor suitable for always-on placement
- Avoids the current maturity issues of Apple Silicon for a critical node

Place the Minisforum node on the Servers or Management VLAN with a static IP or DHCP reservation.

### 8.6 Operational Notes

- The node must be highly available — it is the preferred Tailscale entry point and the AI monitor.
- Document a fallback path if the Omarchy node is unavailable (e.g., Tailscale on the firewall as secondary).
- Regularly test the AI → kill-switch trigger path.
- Keep the local model and decision logic under local control (no mandatory cloud dependency for core function).

### 8.8 Single Interface Principle & Dashboard

**Core requirement**: The Omarchy node is the primary interface for the entire network.  

Normal day-to-day management and observation does **not** require:
- Tailscale admin console
- Firewall web UI
- Individual device interfaces

All routine visibility and control is handled through the Omarchy server.

#### Interaction methods
1. **Direct chat with Bastion** — natural language queries and commands.
2. **AI-managed dashboard** — visual overview maintained by the AI.

#### Dashboard contents (baseline)
The AI is responsible for collecting data and keeping the dashboard current. Minimum views:

- Network health overview
- Current activity / traffic summary
- Threat level (scoring and thresholds to be refined later)
- Connected devices
- Recent suspicious activity

#### Design notes
- The AI model drives both the chat interface and the dashboard.
- Data sources remain local (firewall logs, Tailscale events, network telemetry, etc.).
- Dashboard can be a simple local web UI, terminal UI, or other presentation layer that the AI updates.
- Human override and physical kill switch remain available outside this interface when needed.

---

## 9. Inter-Zone Traffic Policy (Baseline)

| From → To          | Trusted | Servers | IoT     | Guest   | Management |
|--------------------|---------|---------|---------|---------|------------|
| **Trusted**        | —       | Allow*  | Allow*  | Deny    | Allow (admin) |
| **Servers**        | Deny    | —       | Deny    | Deny    | Deny       |
| **IoT**            | Deny    | Deny    | —       | Deny    | Deny       |
| **Guest**          | Deny    | Deny    | Deny    | —       | Deny       |
| **Management**     | Deny    | Deny    | Deny    | Deny    | —          |
| **Any → Internet** | Allow   | Allow   | Allow   | Allow   | Allow (limited) |

\* Allow only specific ports/services as needed (e.g., Trusted → Servers for SMB/NFS/media, Trusted → IoT for control apps).

**Additional rules:**
- Block RFC1918 ranges from IoT and Guest except where explicitly required.
- No management interfaces reachable from untrusted zones.
- Logging of denied traffic for review.
- Kill-switch ESP32 should have no unnecessary outbound internet access.

---

## 10. Security Controls Checklist

- [ ] Firewall in place with default-deny policy
- [ ] VLAN segmentation implemented
- [ ] Unique strong credentials on all network devices
- [ ] Remote management disabled from internet
- [ ] UPnP and WPS disabled
- [ ] WPA3 (or WPA2-AES) on all wireless
- [ ] Guest network isolated
- [ ] IoT devices isolated
- [ ] DNS filtering active
- [ ] Tailscale installed on Omarchy node (or firewall) as subnet router
- [ ] Only required subnets advertised and approved in the Tailscale admin console
- [ ] Tailscale policy file (grants/ACLs) configured for least privilege
- [ ] Omarchy always-on node deployed and hardened
- [ ] Bastion running on Omarchy node and receiving network telemetry
- [ ] AI-to-kill-switch trigger path tested (with human override confirmed)
- [ ] Firewall rules on the Tailscale interface reviewed and restricted as appropriate
- [ ] Logging enabled
- [ ] Automatic updates or regular manual update process
- [ ] Inventory of connected devices maintained
- [ ] Backup of firewall/switch configurations
- [ ] (Optional) Physical internet kill switch — ESP32 + micro servo assembled, mounted, calibrated, and tested
- [ ] Kill-switch device placed on restricted VLAN with no unnecessary internet access

---

## 11. High-Level Implementation Order

1. Document current devices and requirements.
2. Select and acquire hardware (firewall, switch, APs; optional kill-switch parts).
3. Put ISP modem into bridge mode.
4. Install and harden firewall.
5. Deploy and harden the Omarchy always-on node.
6. Install and configure Tailscale on the Omarchy node (subnet router).
7. Deploy Bastion on Omarchy node and connect it to network logs.
8. Create VLANs and subnets.
9. Configure managed switch and APs (trunk ports, SSID-to-VLAN).
10. Write firewall rules (start restrictive, open only what is needed).
11. Deploy DNS filtering.
12. Assemble, mount, and configure the ESP32 kill switch; test AI trigger path.
13. Migrate devices zone by zone, testing connectivity.
14. Enable monitoring/logging and document the final design.

---

## 13. Example Setup

This section captures the specific components chosen for this design. It serves as a concrete example of how the generic baseline can be implemented.

### Selected Components

| Role | Model / Choice | Notes |
|------|----------------|-------|
| **Always-on Omarchy node** | Minisforum mini-PC (e.g. UM890 Pro or equivalent) | Runs Omarchy, Tailscale subnet router, and Bastion |
| **Core switch** | **MikroTik CRS310-8G+2S+IN** | 8× 2.5GbE + 2× 10G SFP+. Full RouterOS with strong SNMP + API for AI communication |
| **Firewall / Router** | Protectli Vault FW4C running pfSense | Stateful firewall, VLAN routing, default-deny. 4× Intel 2.5GbE ports, fanless. Beelink EQ14 is an acceptable lower-cost alternative. |
| **Physical kill switch** | DIY ESP32 + micro servo (SG90/MG90S) | Physically presses modem power button; controllable by AI |
| **Access Points** | 1× MikroTik **cAP ax** + 1× MikroTik **wAP ax** | Ceiling and wall mount options. SSID-to-VLAN mapping. Bastion is the management interface. |

### Notes
- Additional components (access points, exact firewall hardware, etc.) will be added here as they are selected.
- All components must satisfy the AI communication requirement.

- Review firewall logs weekly (or set alerts).
- Update firmware/OS monthly or when security patches are released.
- Audit connected devices periodically.
- Test backups and Tailscale access quarterly.
- Test the physical kill switch periodically.
- Review and refine firewall rules as needs change.
- Keep a simple network diagram and IP plan up to date.

---

*This design treats segmentation, Tailscale remote access, and the optional physical kill switch as core, visible elements rather than afterthoughts. Local network segmentation and firewall policy remain the primary defenses against threats originating inside the home. Always refer to current Tailscale documentation for platform-specific configuration steps.*
