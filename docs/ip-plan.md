# IP and VLAN Plan
IP and VLAN Plan

All addresses in this repository are private lab addresses (RFC 1918 for IPv4, RFC 4193 ULA for IPv6). No production, household, or public network details are published.

Status: Phase 0 design. Updated as each phase is built.

Design principles:
One summarizable supernet for the whole lab, so it can be advertised or filtered as a single route.
One /16 per site, so each site summarizes cleanly in later routing labs.
VLAN ID matches the third octet (VLAN 20 = 10.20.20.0/24), so any address shows its VLAN.
/24 per VLAN: readability over conservation, standard for access networks.
Segmentation by trust level, with default-deny between zones.
Dual-stack ready: every VLAN has a reserved IPv6 /64.
Address allocation

Lab supernet: 10.20.0.0/14 (10.20.0.0 – 10.23.255.255)

Block |	Purpose	| Deployment Phase
10.20.0.0/16	Site 20: physical lab (FW6A, switch, Proxmox)	0
10.21.0.0/16	Site 21: simulated branch	5
10.22.0.0/16	Site 22: simulated branch	5
10.23.0.0/16	Site 23: simulated branch	5

Site 20 detail
Range	VLAN	Name	Purpose	Phase
10.20.0.0/26	none	LOOPBACKS	/32 router IDs and loopbacks	1
10.20.0.64/26	none	TRANSIT	/31 point-to-point links (RFC 3021)	1
10.20.10.0/24	10	MGMT	Management: FW6A, switch, Proxmox	0
10.20.11.0/24	none	ADMIN	Break-glass admin port (FW6A physical port)	0
10.20.20.0/24	20	LAB	Lab VMs and test hosts	0
10.20.30.0/24	30	SERVICES	DNS, monitoring, NetBox	0
10.20.40.0/24	40	DMZ	Services reachable from outside the lab	2
10.20.50.0/24	50	GUEST	Untrusted devices, internet only	2
10.20.100.0/24	none	VPN	Remote-access VPN clients	2
10.20.128.0/17	none	VIRTUAL	CML and Containerlab topologies	1
none	999	BLACKHOLE	Native VLAN on trunks; unused ports. No gateway, no DHCP	0


Host addressing convention (every /24)
Range |	Use
.1	Default gateway (FW6A)
.2 – .9	Network devices
.10 – .99	Static servers and DHCP reservations
.100 – .199	DHCP pool
.200 – .254	Reserved

Static assignments
Hostname |	Address	VLAN |	Role
fw01	10.20.10.1	10	Firewall (gateway on each VLAN at .1)
fw01	10.20.11.1	none	Break-glass admin interface
sw01	10.20.10.2	10	TL-SG108E management
pve01	10.20.10.5	10	Proxmox VE host
dns01	10.20.30.53	30	Pi-hole + Unbound

DNS domain: lab.home.arpa (reserved for home networks by RFC 8375).

DHCP scopes
VLAN |	Scope |	Gateway	DNS server
10 MGMT	10.20.10.100 – .199	10.20.10.1	10.20.30.53
ADMIN	10.20.11.100 – .110	10.20.11.1	10.20.30.53
20 LAB	10.20.20.100 – .199	10.20.20.1	10.20.30.53
30 SERVICES	10.20.30.100 – .199	10.20.30.1	10.20.30.53

Switch port map (sw01, TL-SG108E)
Port |	Connects to	Mode |	VLANs
1	fw01 trunk port	Trunk	Tagged 10, 20, 30
2	pve01	Trunk	Tagged 10, 20, 30
3	Admin laptop	Access	Untagged 10 (PVID 10)
4	Lab device	Access	Untagged 20 (PVID 20)
5 – 8	Unused	Access	Untagged 999 (PVID 999)

IPv6 (ULA, RFC 4193)
Prefix: fd12:8962:41f6::/48 
Each VLAN receives a /64, using the VLAN number as the subnet ID for readability:

VLAN |	IPv4 |	IPv6
10 MGMT	10.20.10.0/24	fd12:8962:41f6:10::/64
20 LAB	10.20.20.0/24	fd12:8962:41f6:20::/64
30 SERVICES	10.20.30.0/24	fd12:8962X:41f6:30::/64

Every VLAN is /64, as required for SLAAC.

Change log
Date	Change
2026-10-12	Initial Phase 0 addressing plan