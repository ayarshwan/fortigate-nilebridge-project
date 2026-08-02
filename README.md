# NileBridge Services — FortiGate HQ & Branch Security Project

FortiOS graduation project implementing a small-company, two-site network secured with FortiGate: routed internet access with NAT, a route-based site-to-site IPsec VPN, local captive-portal authentication, static web filtering, and application control — built and validated in EVE-NG.

> **Status:** Core (mandatory) requirements implemented and validated. HQ SD-WAN (dual-WAN Internet failover) is implemented and validated as a bonus. The Branch remains on its original WAN1 design for a simpler, resource-efficient scope.

## Repository Structure

```
fortigate-graduation-project/
├── README.md
├── docs/
│   └── NileBridge_FortiGate_Final_Report.pdf
├── topology/
│   ├── Topology.png
│   └── addressing-table.md
├── configs/
│   ├── HQ-NGFW-1-sanitized.conf
│   ├── BR1-FGT-sanitized.conf
│   └── linux-isp-persistent.sh
└── evidence/
    ├── 01-interfaces.png
    ├── 02-firewall-policies.png
    ├── 03-ipsec-up.png
    ├── 04-authentication.png
    ├── 05-web-filter.png
    ├── 06-application-control.png
    ├── 07-connectivity-tests.png
    ├── 08-sdwan-members-and-sla.png
    ├── 09-sdwan-wan1-primary.png
    └── 10-sdwan-failover.png
```

## Table of Contents

1. [Project Scenario](#project-scenario)
2. [Requirements Mapping](#requirements-mapping)
3. [Topology](#topology)
4. [Addressing Plan](#addressing-plan)
5. [Objects & Profiles](#objects--profiles)
6. [Device Configuration](#device-configuration)
   - [Linux-ISP](#linux-isp)
   - [HQ-NGFW-1](#hq-ngfw-1)
   - [BR1-FGT](#br1-fgt)
7. [Firewall Policies](#firewall-policies)
8. [HQ SD-WAN (Bonus)](#hq-sd-wan-bonus)
9. [Site-to-Site IPsec VPN](#site-to-site-ipsec-vpn)
10. [Authentication & Security Profiles](#authentication--security-profiles)
11. [Validation Results](#validation-results)
12. [Deviations, Limitations & Troubleshooting](#deviations-limitations--troubleshooting)
13. [Repository Structure](#repository-structure)
14. [Future Work](#future-work)
15. [Publication & Safety Notes](#publication--safety-notes)

---

## Project Scenario

**NileBridge Services** is modeled as a small business with one headquarters office (~20 users) and one branch office (~8 users). The design goals were:

- Secure, logged internet access for both HQ and Branch users.
- A private, encrypted site-to-site connection between both offices.
- Local user authentication before HQ employees get browser-based internet access.
- Visibility and control over web traffic and applications (block a demo hostname, block common P2P apps).
- Automatic HQ Internet failover from WAN1 to WAN2 via SD-WAN, while the Branch stays on its original WAN1 design to keep the scope simple.
- A topology compact enough to run on a 16 GB laptop.

The lab uses one Linux node as a simulated ISP router connecting two FortiGates (HQ and Branch), each with its own LAN and two WAN uplinks.

---

## Requirements Mapping

| Requirement | Implementation | Status |
|---|---|---|
| Two FortiGate devices | `HQ-NGFW-1`, `BR1-FGT` | ✅ PASS |
| WAN connection | `Linux-ISP` — 4 routed WAN segments + internet NAT | ✅ PASS |
| Two end devices | `HQ-PC-1`, `HQ-Web-Client`, `BR1-PC-1` | ✅ PASS |
| Account configuration | Named local admin `projectadmin` on both FortiGates | ✅ PASS |
| Interface configuration | Static LAN / WAN1 / WAN2 addressing on both FortiGates | ✅ PASS |
| Firewall policy | NAT internet policies + bidirectional no-NAT VPN policies | ✅ PASS |
| Routing | Default routes + remote-LAN routes via IPsec interface | ✅ PASS |
| Authentication | Local user `employee1` in captive-portal group `HQ_Internet_Users` | ✅ PASS |
| Security profile #1 | `HQ-Web-Filter` — static URL filtering | ✅ PASS |
| Security profile #2 | `SMALLCO-APP` — application logging + P2P block | ✅ PASS |
| VPN | Route-based IKEv2 site-to-site IPsec tunnel | ✅ PASS |
| SD-WAN (bonus) | HQ dual-WAN SD-WAN with WAN1 priority, WAN2 backup, SLA monitoring, and tested failover | ✅ PASS |

---

## Topology

```
                          ┌────────────────────┐
                          │   Net / Internet    │
                          │  (ens3 mgmt/uplink)  │
                          └──────────┬──────────┘
                                     │
                     ┌───────────────┴────────────────┐
                     │            Linux-ISP             │
                     │   Routing, NAT, test web server   │
                     │        198.51.100.10/32           │
                     └───┬───────────────────────────┬──┘
        WAN1 100.65.0.0/24│                           │100.65.1.0/24 WAN1
        WAN2 100.66.0.0/24│                           │100.66.1.0/24 WAN2
             ┌────────────┴──────┐          ┌─────────┴───────────┐
             │     HQ-NGFW-1      │  IKEv2   │       BR1-FGT        │
             │  FortiOS 7.2.1     │◄────────►│    FortiOS 7.2.1      │
             │ port4 LAN|2/3 WAN  │  IPsec   │  port4 LAN | 2/3 WAN  │
             │ SD-WAN: port2      │  (bound  │  port2 = active WAN1  │
             │ primary/port3      │  to WAN1)│  port3 = available,   │
             │ backup (virtual-   │          │  not added to SD-WAN  │
             │ wan-link zone)     │          │                       │
             └─────────┬──────────┘          └───────────┬──────────┘
                        │ 10.0.11.0/24                    │172.20.1.0/24
                  ┌─────┴──────┐                          │
                  │   HQ-SW1    │                    ┌─────┴─────┐
                  │ (L2 switch) │                    │  BR1-PC-1  │
                  └──┬───────┬──┘                    │172.20.1.60 │
                     │       │                        └───────────┘
              ┌──────┴─┐ ┌───┴────────┐
              │HQ-PC-1 │ │HQ-Web-Client│
              │.50     │ │.70          │
              └────────┘ └─────────────┘
```

![Actual EVE-NG Topology](topology/Topology.png)
*Figure 1: Actual implemented topology showing HQ, Branch, and Linux-ISP.*

**Notes on the actual build:**
- The interface mapping follows the reference project guide: **`port4` = LAN, `port2` = WAN1, `port3` = WAN2** on both FortiGates.
- At **Headquarters**, `port2` and `port3` are members of the `virtual-wan-link` SD-WAN zone, with `port2` (WAN1) as the priority member and `port3` (WAN2) as the backup for automatic Internet failover.
- At the **Branch**, `port2` remains the sole active WAN interface; `port3` is addressed but intentionally **not** added to SD-WAN, keeping the Branch configuration simple.
- An access switch (`HQ-SW1`) was added at HQ so both the ping-test client (`HQ-PC-1`) and the browser-capable client (`HQ-Web-Client`, used for captive-portal and web-filter evidence) can share the HQ LAN.
- `Linux-ISP` provides four routed WAN gateway subnets, source-NATs toward the real uplink (`ens3`), and hosts a local test web server for deterministic connectivity/web-filter testing.

---

## Addressing Plan

| Device | Interface | Purpose | IPv4 Address | Gateway / Note |
|---|---|---|---|---|
| HQ-NGFW-1 | port4 | HQ LAN / captive portal | 10.0.11.254/24 | – |
| HQ-NGFW-1 | port2 | HQ WAN1 — SD-WAN primary member | 100.65.0.101/24 | 100.65.0.254 |
| HQ-NGFW-1 | port3 | HQ WAN2 — SD-WAN backup member | 100.66.0.101/24 | 100.66.0.254 |
| BR1-FGT | port4 | Branch LAN | 172.20.1.254/24 | – |
| BR1-FGT | port2 | WAN1 to Linux-ISP (active) | 100.65.1.111/24 | 100.65.1.254 |
| BR1-FGT | port3 | Branch WAN2 — available, not added to SD-WAN in final scope | 100.66.1.111/24 | Not added to SD-WAN |
| Linux-ISP | ens3 | Management / real internet uplink | 192.168.16.134/24 | 192.168.16.254 |
| Linux-ISP | ens5 | HQ WAN1 gateway | 100.65.0.254/24 | – |
| Linux-ISP | ens6 | HQ WAN2 gateway | 100.66.0.254/24 | – |
| Linux-ISP | ens7 | Branch WAN1 gateway | 100.65.1.254/24 | – |
| Linux-ISP | ens8 | Branch WAN2 gateway | 100.66.1.254/24 | – |
| Linux-ISP | loopback | Local test web server | 198.51.100.10/32 | – |
| HQ-PC-1 | eth0 | HQ VPCS ping-test client | 10.0.11.50/24 | 10.0.11.254 |
| HQ-Web-Client | ens3 | HQ Debian browser client | 10.0.11.70/24 | 10.0.11.254 |
| BR1-PC-1 | eth0 | Branch VPCS client | 172.20.1.60/24 | 172.20.1.254 |

---

## Objects & Profiles

| Type | Name | Purpose / Value |
|---|---|---|
| Address object | `HQ_LAN` | 10.0.11.0/24 |
| Address object | `BRANCH_LAN` | 172.20.1.0/24 |
| VPN tunnel | `HQ-to-BR1` / `BR1-to-HQ` | Route-based IKEv2 tunnel |
| Local user | `employee1` | HQ demonstration employee |
| User group | `HQ_Internet_Users` | Allowed captive-portal users |
| Web Filter | `HQ-Web-Filter` | Static hostname blocking |
| Application Control | `SMALLCO-APP` | App logging + P2P blocking |
| Named administrator | `projectadmin` | Local admin account on both FortiGates |
| SD-WAN zone | `virtual-wan-link` | HQ Internet egress zone containing `port2` + `port3` |
| Performance SLA | `ISPSLA` | ICMP target `198.51.100.10`; monitors both HQ WAN members |
| SD-WAN rule | `WAN1-Primary` | Priority order: member 1 (`port2`), then member 2 (`port3`) |

---

## Device Configuration

> ⚠️ All example secrets below are placeholders. See [Publication & Safety Notes](#publication--safety-notes).

### Linux-ISP

Acts as the simulated ISP: routes between the four WAN subnets, source-NATs toward the real uplink, and hosts a deterministic local test page.

```bash
# Interface addressing (adjust interface names to your image, e.g. ens5/ens6/ens7/ens8)
sudo ip addr add 100.65.0.254/24 dev ens5   # HQ WAN1 gateway
sudo ip addr add 100.66.0.254/24 dev ens6   # HQ WAN2 gateway
sudo ip addr add 100.65.1.254/24 dev ens7   # Branch WAN1 gateway
sudo ip addr add 100.66.1.254/24 dev ens8   # Branch WAN2 gateway
sudo ip link set ens5 up
sudo ip link set ens6 up
sudo ip link set ens7 up
sudo ip link set ens8 up

# Routing + NAT toward the real uplink (ens3)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -P FORWARD ACCEPT
sudo iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE

# Local deterministic test server
sudo ip addr add 198.51.100.10/32 dev lo
sudo mkdir -p /opt/testweb
echo "<h1>NileBridge ISP Test Page</h1>" | sudo tee /opt/testweb/index.html
sudo nohup python3 -m http.server 80 --bind 198.51.100.10 --directory /opt/testweb &
```

> **Lab-only note:** `FORWARD ACCEPT` is intentionally permissive for an isolated EVE-NG lab and must not be copied into a production Linux router without a real firewall policy. Runtime `ip`/`sysctl`/`iptables`/server commands do not persist across reboot — re-apply or add a systemd/NetworkManager persistence mechanism before a final demo.

### HQ-NGFW-1

```
config system global
    set hostname "HQ-NGFW-1"
end

config system admin
    edit "projectadmin"
        set accprofile "super_admin"
        set vdom "root"
        set password "<REDACTED>"
    next
end

config system interface
    edit "port4"
        set alias "HQ-LAN"
        set mode static
        set ip 10.0.11.254 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
    edit "port2"
        set alias "WAN1-to-Linux-ISP"
        set mode static
        set ip 100.65.0.101 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port3"
        set alias "WAN2-SDWAN-Backup"
        set mode static
        set ip 100.66.0.101 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
end

config firewall address
    edit "HQ_LAN"
        set subnet 10.0.11.0 255.255.255.0
    next
    edit "BRANCH_LAN"
        set subnet 172.20.1.0 255.255.255.0
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.65.0.254
        set device "port2"
        set comment "Initial default route through Linux-ISP (later migrated to the SD-WAN zone below)"
    next
end
```

### BR1-FGT

```
config system global
    set hostname "BR1-FGT"
end

config system admin
    edit "projectadmin"
        set accprofile "super_admin"
        set vdom "root"
        set password "<REDACTED>"
    next
end

config system interface
    edit "port4"
        set alias "Branch-LAN"
        set mode static
        set ip 172.20.1.254 255.255.255.0
        set allowaccess ping https ssh
        set role lan
    next
    edit "port2"
        set alias "WAN1-to-Linux-ISP"
        set mode static
        set ip 100.65.1.111 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port3"
        set alias "WAN2-reserved"
        set mode static
        set ip 100.66.1.111 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
end

config firewall address
    edit "HQ_LAN"
        set subnet 10.0.11.0 255.255.255.0
    next
    edit "BRANCH_LAN"
        set subnet 172.20.1.0 255.255.255.0
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 100.65.1.254
        set device "port2"
        set comment "Primary default route through Linux-ISP"
    next
end
```

---

## Firewall Policies

| Device | ID | Name | Source Intf | Dest Intf | Source | Destination | NAT | UTM Profiles | Log |
|---|---|---|---|---|---|---|---|---|---|
| HQ-NGFW-1 | 1 | HQ-LAN-to-Internet | port4 | virtual-wan-link | HQ_LAN | all | Enabled | `HQ-Web-Filter`, `SMALLCO-APP` | All |
| HQ-NGFW-1 | 10 | HQ-LAN-to-BR1-VPN | port4 | HQ-to-BR1 | HQ_LAN | BRANCH_LAN | Disabled | – | All |
| HQ-NGFW-1 | 11 | BR1-VPN-to-HQ-LAN | HQ-to-BR1 | port4 | BRANCH_LAN | HQ_LAN | Disabled | – | All |
| BR1-FGT | 1 | BR-LAN-to-Internet | port4 | port2 | BRANCH_LAN | all | Enabled | `HQ-Web-Filter`, `SMALLCO-APP` | All |
| BR1-FGT | 10 | BR-LAN-to-HQ-VPN | port4 | BR1-to-HQ | BRANCH_LAN | HQ_LAN | Disabled | – | All |
| BR1-FGT | 11 | HQ-VPN-to-BR-LAN | BR1-to-HQ | port4 | HQ_LAN | BRANCH_LAN | Disabled | – | All |

Policies are evaluated top-to-bottom; the LAN-to-internet policy must stay above any later deny rule, and the VPN policies use dedicated tunnel interfaces so they never conflict with the internet policy.

> ℹ️ HQ policy 1 (`HQ-LAN-to-Internet`) was migrated from physical `port2` to the `virtual-wan-link` SD-WAN zone once HQ SD-WAN was enabled — see [HQ SD-WAN (Bonus)](#hq-sd-wan-bonus). NAT and both security profiles were retained unchanged. The Branch policy 1 still points directly at `port2`, since the Branch was not migrated to SD-WAN.

```
config firewall policy
    edit 1
        set name "HQ-LAN-to-Internet"
        set srcintf "port4"
        set dstintf "virtual-wan-link"
        set action accept
        set srcaddr "HQ_LAN"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set nat enable
        set utm-status enable
        set inspection-mode flow
        set webfilter-profile "HQ-Web-Filter"
        set application-list "SMALLCO-APP"
        set logtraffic all
    next
    edit 10
        set name "HQ-LAN-to-BR1-VPN"
        set srcintf "port4"
        set dstintf "HQ-to-BR1"
        set action accept
        set srcaddr "HQ_LAN"
        set dstaddr "BRANCH_LAN"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 11
        set name "BR1-VPN-to-HQ-LAN"
        set srcintf "HQ-to-BR1"
        set dstintf "port4"
        set action accept
        set srcaddr "BRANCH_LAN"
        set dstaddr "HQ_LAN"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end
```

*(Branch policy set mirrors the above with source/destination interfaces and addresses swapped — see [Appendix configs](#repository-structure).)*

---

## HQ SD-WAN (Bonus)

SD-WAN was implemented **only at Headquarters** to demonstrate automatic Internet failover without adding complexity to the Branch. The HQ Internet policy and default route were migrated from physical `port2` to the `virtual-wan-link` SD-WAN zone. The Branch was intentionally left unchanged on its original WAN1 path.

| Parameter | Implemented Value |
|---|---|
| Scope | Headquarters only; Branch unchanged for simplification |
| SD-WAN zone | `virtual-wan-link` |
| Member 1 / Primary | `port2` — 100.65.0.101/24, gateway 100.65.0.254 |
| Member 2 / Backup | `port3` — 100.66.0.101/24, gateway 100.66.0.254 |
| Performance SLA | `ISPSLA`; ICMP target `198.51.100.10` |
| SLA thresholds | Latency 200 ms, jitter 50 ms, packet loss 10% |
| SD-WAN rule | `WAN1-Primary`; mode priority; members 1 then 2 |
| HQ Internet policy | Outgoing interface changed to `virtual-wan-link`; NAT and security profiles retained |
| HQ default route | SD-WAN zone `virtual-wan-link` |
| Failover trigger | Linux-ISP `ens5` disabled to simulate a WAN1 outage |
| Validated result | `port2` dead, `port3` alive, Internet continued through `port3` |

```
config system sdwan
    set status enable
    config members
        edit 1
            set interface "port2"
            set gateway 100.65.0.254
        next
        edit 2
            set interface "port3"
            set gateway 100.66.0.254
        next
    end
    config health-check
        edit "ISPSLA"
            set server "198.51.100.10"
            set protocol ping
            set members 1 2
            config sla
                edit 1
                    set latency-threshold 200
                    set jitter-threshold 50
                    set packetloss-threshold 10
                next
            end
        next
    end
    config service
        edit 1
            set name "WAN1-Primary"
            set mode priority
            set dst "all"
            set src "all"
            set health-check "ISPSLA"
            set priority-members 1 2
        next
    end
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set distance 1
        set sdwan-zone "virtual-wan-link"
    next
end

config firewall policy
    edit 1
        set dstintf "virtual-wan-link"
    next
end
```

### Failover Validation

During normal operation, the SD-WAN priority rule selected `port2`. When Linux-ISP interface `ens5` was disabled to simulate a WAN1 outage, `ISPSLA` marked `port2` dead while `port3` remained alive, and Internet traffic continued through WAN2. After `ens5` was restored, `port2` again became the preferred path.

```
HQ-NGFW-1 # diagnose sys sdwan health-check

Normal state:
  Seq(1 port2): state(alive), packet-loss(0.000%), sla_map=0x1
  Seq(2 port3): state(alive), packet-loss(0.000%), sla_map=0x1

WAN1 failure test (Linux-ISP ens5 down):
  Seq(1 port2): state(dead), packet-loss(32.000%), sla_map=0x0
  Seq(2 port3): state(alive), packet-loss(0.000%), sla_map=0x1

Result: Internet traffic continued through port3 (WAN2 backup).
```

> ⚠️ **Scope note:** The site-to-site IPsec tunnel remains bound to HQ `port2` (WAN1) and is **outside** the SD-WAN zone. This SD-WAN test validates **Internet failover only** — the VPN itself may temporarily drop during a WAN1 outage and returns once WAN1 is restored. Extending VPN redundancy across both WAN members would require a separate design (e.g. a second tunnel on `port3` with SD-WAN-based routing).

---

## Site-to-Site IPsec VPN

A route-based IKEv2 tunnel links `10.0.11.0/24` (HQ) and `172.20.1.0/24` (Branch). Static routes send each remote LAN toward the local tunnel interface, and dedicated no-NAT policies permit bidirectional traffic.

| Parameter | Value |
|---|---|
| HQ tunnel name | `HQ-to-BR1` |
| Branch tunnel name | `BR1-to-HQ` |
| IKE version | IKEv2 |
| HQ peer | 100.65.1.111 |
| Branch peer | 100.65.0.101 |
| Phase 1 proposal | DES-SHA256, DH group 14 *(lab limitation — see warning below)* |
| Phase 2 selectors | 10.0.11.0/24 ↔ 172.20.1.0/24 |
| Authentication | Pre-shared key — **redacted** |
| NAT across VPN | Disabled |
| Observed state | Established / Up |

> ⚠️ **Security warning:** DES is obsolete and must never be used in production. The low-encryption FortiGate image used in this lab did not expose AES proposals, so DES-SHA256 was used only to complete this isolated educational lab. A production deployment should use **AES-256-GCM** or another modern, currently-approved proposal.

```
config vpn ipsec phase1-interface
    edit "HQ-to-BR1"
        set interface "port2"
        set ike-version 2
        set peertype any
        set net-device disable
        set proposal des-sha256          # LAB ONLY — use aes256-sha256 in production
        set dhgrp 14
        set remote-gw 100.65.1.111
        set psksecret "<REDACTED>"
    next
end

config vpn ipsec phase2-interface
    edit "HQ-to-BR1-P2"
        set phase1name "HQ-to-BR1"
        set proposal des-sha256
        set dhgrp 14
        set src-subnet 10.0.11.0 255.255.255.0
        set dst-subnet 172.20.1.0 255.255.255.0
        set auto-negotiate enable
    next
end

config router static
    edit 10
        set dst 172.20.1.0 255.255.255.0
        set device "HQ-to-BR1"
        set comment "Branch LAN through IPsec"
    next
end
```

---

## Authentication & Security Profiles

### Captive Portal
The HQ LAN interface (`port4`) uses `security-mode captive-portal`. Local user `employee1` belongs to group `HQ_Internet_Users`. The browser-capable `HQ-Web-Client` authenticates successfully, and subsequent traffic is logged as an identified user.

```
config user local
    edit "employee1"
        set type password
        set passwd "<REDACTED>"
    next
end

config user group
    edit "HQ_Internet_Users"
        set member "employee1"
    next
end

config system interface
    edit "port4"
        set security-mode captive-portal
        set security-groups "HQ_Internet_Users"
    next
end
```

### Static Web Filter — `HQ-Web-Filter`
Attached in flow-based inspection mode with certificate inspection. A hosts-file entry mapped `allowed.demo.local` / `blocked.demo.local` to the Linux test server (HTTP, to avoid unnecessary TLS certificate prompts). `allowed.demo.local` opened successfully; `blocked.demo.local` was denied by the static URL filter.

### Application Control — `SMALLCO-APP`
Permits and logs normal/unknown applications while blocking selected P2P signatures (BitTorrent, BitTorrent.Sync, BitTorrent.Bleep, eDonkey).

| Log field | Observed value |
|---|---|
| Source | 10.0.11.70 |
| Authenticated user | employee1 |
| Source interface | HQ-LAN (port4) |
| Policy | HQ-LAN-to-Internet |
| Application sensor | SMALLCO-APP |
| Detected application | HTTP.BROWSER_Firefox |
| Category / Risk | Web.Client / Low |
| Action | Pass |

---

## Validation Results

| Test | Action | Result |
|---|---|---|
| HQ local gateway | HQ client → 10.0.11.254 | ✅ PASS |
| HQ local test server | HQ-Web-Client → 198.51.100.10 | ✅ PASS |
| HQ internet | HQ-Web-Client → 8.8.8.8 | ✅ PASS |
| HQ to Branch VPN | HQ-Web-Client → 172.20.1.60 | ✅ PASS |
| Branch local gateway | BR1-PC-1 → 172.20.1.254 | ✅ PASS |
| Branch local test server | BR1-PC-1 → 198.51.100.10 | ✅ PASS |
| Branch internet | BR1-PC-1 → 8.8.8.8 | ✅ PASS |
| Branch to HQ VPN | BR1-PC-1 → 10.0.11.70 | ✅ PASS |
| IPsec state | VPN > IPsec Tunnels | ✅ PASS (Up) |
| Captive portal | employee1 browser login | ✅ PASS |
| Web Filter | allowed / blocked hostname test | ✅ PASS |
| Application Control | Firefox HTTP traffic logged | ✅ PASS |
| Configuration backup | Encrypted backups, both sites | ✅ PASS |
| HQ SD-WAN normal path | HQ-Web-Client → 8.8.8.8 with both links alive | ✅ PASS — packet sniffer showed traffic leaving via `port2` |
| HQ SD-WAN health check | `ISPSLA` monitors `port2` and `port3` | ✅ PASS — both members alive, 0% packet loss before failure |
| HQ SD-WAN failover | Disable Linux-ISP `ens5` (WAN1) | ✅ PASS — `port2` dead, `port3` alive, Internet continued via WAN2 |
| HQ SD-WAN recovery | Re-enable Linux-ISP `ens5` (WAN1) | ✅ PASS — WAN1 returned and remained the preferred member |

---

## Deviations, Limitations & Troubleshooting

| Issue | Observed situation | Resolution / Impact |
|---|---|---|
| FortiOS version | Guide targeted 7.6.x; lab image was 7.2.1 build 1254 | Documented version; expect minor GUI label differences |
| VPN encryption | AES proposals unavailable in the low-encryption image | Used DES-SHA256 for the isolated lab only; replace with AES-256-GCM in production |
| Captive portal + VPCS | VPCS cannot open a browser to authenticate | Used `HQ-Web-Client` for authenticated HQ validation |
| Static hostname test | `blocked.demo.local` initially failed to resolve | Corrected `/etc/hosts`, verified with `getent` |
| Linux persistence | Runtime `ip`/`sysctl`/`iptables`/server commands don't survive reboot | Persist via systemd/NetworkManager before final demo |
| Trial license | VM showed trial/free-license constraints | Kept design compact; avoided unnecessary policies/features |
| SD-WAN static route syntax | The trial image rejected `virtual-wan-link` as a `device` in the default route | Used `set sdwan-zone "virtual-wan-link"` for the HQ default route instead |
| Static-route entry limit | The trial image reached its maximum number of static-route entries | Avoided an extra SLA host route; validated reachability by sourcing the test ping from 100.66.0.101 |
| HQ-only SD-WAN scope | Configuring SD-WAN at both sites would add time/complexity without changing the learning objective | Implemented and tested SD-WAN at HQ only; Branch remains on WAN1 |
| Management session during WAN1 failure | The management path (HTTPS/8443) used WAN1, so the session dropped when `ens5` was disabled | Added a temporary management path through HQ WAN2 for failover testing |
| VPN during HQ WAN1 outage | The IPsec tunnel is bound to HQ `port2` | Internet fails over to WAN2, but the VPN may drop until WAN1 is restored |

---

## Future Work

Work should resume only from the saved configuration checkpoints (`HQ-NGFW-1-core-working.conf`, `BR1-FGT-core-working.conf`, `HQ-NGFW-1-SDWAN-Final.conf`), kept **outside** the public repository.

1. **Extend SD-WAN to the Branch** — the Branch's `port3` is already cabled and addressed; adding it to a `virtual-wan-link` zone with its own SLA health-check would bring the same WAN1↔WAN2 failover to the Branch site.
2. **VPN redundancy across WAN members** — the site-to-site IPsec tunnel is currently bound only to HQ `port2` (WAN1), so it can drop during a WAN1 outage even though Internet access fails over correctly. A second tunnel on `port3`, or SD-WAN-aware VPN routing, would let the VPN itself survive a WAN1 failure.
3. **Persistent management path** — the temporary WAN2 management route used during failover testing should be made permanent (or replaced with an out-of-band management interface) so admin access survives a WAN1 outage.

Before any real-world deployment: replace DES with modern cryptography (AES-256-GCM), persist the Linux router configuration, apply least-privilege administrator profiles, and enable MFA.

---

## Publication & Safety Notes

- Passwords, pre-shared keys, serial numbers, tokens, and private configuration backups are **excluded or redacted** from this repository.
- Public configuration copies replace administrator passwords, user passwords, VPN pre-shared keys, serial numbers, tokens, and personal names with placeholders (`<REDACTED>`).
- Original encrypted backups must stay in a private folder and must **not** be uploaded to GitHub.
- Screenshots are cropped to remove browser tabs, personal accounts, remote-access identifiers, and unrelated desktop information.
- Linux-ISP persistence scripts are reviewed to ensure they contain no private credentials before publication.
