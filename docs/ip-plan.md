# IP and VLAN Plan

All addresses in this repository are private lab addresses (RFC 1918 for IPv4, RFC 4193 ULA for IPv6). No production, household, or public network details are published.

**Status:** Phase 0 design (v3). The machine-readable source of truth is [`data/ip-plan.yml`](../data/ip-plan.yml); this page mirrors it for humans. Change the YAML first, then this page.

## Design principles

1. **One summarizable supernet** for the whole lab: a single route to advertise, filter or block.
2. **One /16 per site** (IPv4) and **one /56 per site** (IPv6), so every site summarizes cleanly in later routing labs.
3. **VLAN ID matches the third octet** (VLAN 20 = 10.20.**20**.0/24), so any address shows its VLAN. Exceptions are listed explicitly below.
4. **/24 per VLAN**: readability over address conservation, standard for access networks.
5. **Segmentation by trust level**, with default-deny between zones. Addressing and policy are designed together.
6. **IPv4 and IPv6 at parity**: every IPv4 network has an IPv6 counterpart following the same structure.
7. **Automation-ready**: the plan exists as structured data that Ansible, Batfish and NetBox can consume.

## Address hierarchy

| Level | IPv4 | IPv6 | Notes |
|---|---|---|---|
| Lab supernet | 10.20.0.0/14 | `fd12:8962:41f6:2000::/54` | Sites 20–23 summarize to one route in each family |
| Site 20 (physical lab) | 10.20.0.0/16 | `fd12:8962:41f6:2000::/56` | FW6A, switch, Proxmox; Phase 0 |
| Site 21 (simulated branch) | 10.21.0.0/16 | `fd12:8962:41f6:2100::/56` | Phase 5 |
| Site 22 (simulated branch) | 10.22.0.0/16 | `fd12:8962:41f6:2200::/56` | Phase 5 |
| Site 23 (simulated branch) | 10.23.0.0/16 | `fd12:8962:41f6:2300::/56` | Phase 5 |

`<ULA>` is the lab's randomly generated 48-bit ULA prefix (`fdXX:XXXX:XXXX`), generated once per RFC 4193 and fixed permanently.

**IPv6 subnet ID layout (16 bits):** the first two hex digits are the site (`20` for Site 20), the last two identify the network within the site (`10` for VLAN 10). So VLAN 10 at Site 20 is `<ULA>:2010::/64`.

## Site 20 networks

| Network | IPv4 | IPv6 | VLAN | Usable IPv4 | Assigned | Phase |
|---|---|---|---|---|---|---|
| LOOPBACKS | 10.20.0.0/26 | `fd12:8962:41f6:2000::/64` | none | 64 × /32 | 0 | 1 |
| TRANSIT | 10.20.0.64/26 | `fd12:8962:41f6:2001::/64` | none | 32 × /31 links | 0 | 1 |
| MGMT | 10.20.10.0/24 | `fd12:8962:41f6:2010::/64` | 10 | 254 | 3 | 0 |
| ADMIN | 10.20.11.0/24 | `fd12:8962:41f6:2011::/64` | none | 254 | 1 | 0 |
| LAB | 10.20.20.0/24 | `fd12:8962:41f6:2020::/64` | 20 | 254 | 1 | 0 |
| SERVICES | 10.20.30.0/24 | `fd12:8962:41f6:2030::/64` | 30 | 254 | 2 | 0 |
| DMZ | 10.20.40.0/24 | `fd12:8962:41f6:2040::/64` | 40 | 254 | 0 | 2 |
| GUEST | 10.20.50.0/24 | `fd12:8962:41f6:2050::/64` | 50 | 254 | 0 | 2 |
| VPN | 10.20.100.0/24 | `fd12:8962:41f6:2070::/64` | none | 254 | 0 | 2 |
| VIRTUAL | 10.20.128.0/17 | `<fd12:8962:41f6:2080::/57` | none | 128 × /24 | 0 | 1 |
| BLACKHOLE | none | none | 999 | none | none | 0 |

- **LOOPBACKS:** one /32 (IPv4) and one /128 (IPv6) per router.
- **TRANSIT:** /31 point-to-point links per RFC 3021; /127 per RFC 6164 for IPv6.
- **VIRTUAL:** carved into /24s (IPv4) and /64s (IPv6) for CML and Containerlab topologies.
- **BLACKHOLE:** native VLAN on trunks and the access VLAN of unused switch ports. No gateway, no DHCP.

### Unallocated (reserved for growth)

| Range | Intended use |
|---|---|
| 10.20.0.128/25 | Additional loopbacks or transit |
| 10.20.1.0/24 – 10.20.9.0/24 | Infrastructure (future out-of-band or provisioning networks) |
| 10.20.12.0/24 – 10.20.99.0/24, except allocated VLANs | Future VLANs, following VLAN = third octet |
| 10.20.101.0/24 – 10.20.127.0/24 | Future remote-access or overlay networks |

### Documented exceptions

| Exception | Reason |
|---|---|
| **ADMIN (10.20.11.0/24) has no VLAN** | It is a dedicated physical firewall port (break-glass access), not a switched VLAN. Third octet 11 places it next to MGMT. |
| **VPN uses IPv6 subnet ID `70`**, not `100` | The network ID field is two hex digits (max `ff`), and IDs `80`–`ff` belong to VIRTUAL. |
| **IPv6 subnet IDs reuse decimal VLAN digits** | `2010` is written to read "site 20, VLAN 10". In hexadecimal, `10` is decimal 16. This is a readability convention; the values are still valid and unique. |

## Host addressing convention

| IPv4 host | IPv6 interface ID | Use |
|---|---|---|
| .1 | ::1 | Default gateway (fw01) |
| .2 – .9 | ::2 – ::9 | Network devices |
| .10 – .99 | ::10 – ::99 | Static servers and DHCP reservations |
| .100 – .199 | SLAAC / DHCPv6 | Dynamic clients |
| .200 – .254 | reserved | Reserved |

Static hosts reuse their IPv4 last octet as their IPv6 interface ID (pve01 at 10.20.10.5 is `<ULA>:2010::5`).

## Static assignments

| Hostname | IPv4 | IPv6 | Network | Role |
|---|---|---|---|---|
| fw01 | 10.20.10.1 | `fd12:8962:41f6:2010::1` | MGMT | Firewall; gateway (.1 / ::1) on every Site 20 network |
| fw01 | 10.20.11.1 | `fd12:8962:41f6:2011::1` | ADMIN | Break-glass admin interface |
| sw01 | 10.20.10.2 | `fd12:8962:41f6:2010::2` | MGMT | TL-SG108E management |
| pve01 | 10.20.10.5 | `fd12:8962:41f6:2010::5` | MGMT | Proxmox VE host |
| dns01 | 10.20.30.53 | `fd12:8962:41f6:2030::53` | SERVICES | Pi-hole + Unbound |

## DHCP scopes and DNS

| Network | DHCP scope | Gateway | DNS servers (primary, secondary) | Reverse DNS zone |
|---|---|---|---|---|
| MGMT | 10.20.10.100 – .199 | 10.20.10.1 | 10.20.30.53, 10.20.10.1 | `10.20.10.in-addr.arpa` |
| ADMIN | 10.20.11.100 – .110 | 10.20.11.1 | 10.20.30.53, 10.20.11.1 | `11.20.10.in-addr.arpa` |
| LAB | 10.20.20.100 – .199 | 10.20.20.1 | 10.20.30.53 | `20.20.10.in-addr.arpa` |
| SERVICES | 10.20.30.100 – .199 | 10.20.30.1 | 10.20.30.53 | `30.20.10.in-addr.arpa` |

**Secondary DNS (ADMIN and MGMT only):** fw01's built-in Unbound resolver. dns01 runs on pve01, so without a secondary, a pve01 outage would take name resolution down for the networks used to fix it.

- fw01's resolver listens on ADMIN and MGMT only. LAB and SERVICES stay on dns01, so lab DNS enforcement is unchanged.
- It carries host overrides for fw01, sw01, pve01 and dns01, so `lab.home.arpa` names still resolve during an outage.
- It runs a DNS blocklist. Clients don't treat the secondary purely as a fallback and may send some queries to it while dns01 is up, so filtering must hold on both. Those queries won't appear in Pi-hole's log, which is an accepted trade-off.

Site 20 as a whole: `20.10.in-addr.arpa`. IPv6 reverse zones are the nibble-reversed ULA /48 under `ip6.arpa`, defined once the prefix is set.

## Inter-zone policy (Phase 0)

Default deny. Rows are sources, columns are destinations. Rules are enforced on fw01.

| From \ To | MGMT | ADMIN | LAB | SERVICES | fw01 itself | Household network | Internet |
|---|---|---|---|---|---|---|---|
| **ADMIN** | Allow | n/a | Allow | Allow | Allow (GUI, SSH) | Deny | Allow |
| **MGMT** | n/a | Deny | Allow | Allow | Allow (GUI, SSH) | Deny | Allow |
| **LAB** | Deny | Deny | n/a | DNS to dns01 only | NTP only | Deny | Allow |
| **SERVICES** | Deny | Deny | Deny | n/a | NTP only | Deny | Allow |
| **Household network (WAN)** | Deny | Deny | Deny | Deny | Deny | n/a | n/a |

- **DNS enforcement:** LAB and SERVICES may not use TCP/UDP 53 or 853 to any server except dns01 (dns01 itself may query the internet).
- **DHCP** is permitted implicitly on interfaces where the DHCP service runs.
- **Management interfaces** (fw01 GUI and SSH) listen only on ADMIN and MGMT.
- **fw01 services for ADMIN and MGMT:** DNS (secondary resolver, UDP/TCP 53) and NUT (TCP 3493, MGMT only, pve01 as client).
- **Traffic within a VLAN** is switched and does not pass through fw01.

## Power and graceful shutdown

| Item | Value |
|---|---|
| UPS | ups01, CyberPower EC850LCD, NUT driver `usbhid-ups` |
| UPS data cable | USB to fw01 |
| NUT server | fw01, listening on 10.20.10.1 (TCP 3493) only |
| NUT clients | pve01 |
| Shutdown order | pve01 first, fw01 last; sw01 has no shutdown and loses power with the UPS |

fw01 shuts down last because clients must still reach the NUT server to receive the shutdown signal. The BIOS on both machines must be set to power on after AC power returns, or the lab stays off after an outage.


## Change log

| Date | Version | Change |
|---|---|---|
| 2026-10-12 | v1 | Initial Phase 0 addressing plan |
| 2026-10-12 | v2 | Added inter-zone policy, IPv6 parity (per-site /56, loopback and transit ranges), capacity, naming convention, reverse DNS zones, unallocated ranges, documented exceptions, and machine-readable `data/ip-plan.yml` |
| 2026-10-13 | v3 | Added fw01 as secondary DNS for ADMIN and MGMT, and UPS shutdown signaling (NUT) |
