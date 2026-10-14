# network-homelab

I'm building a home lab to get real hands-on experience with networking in various environments all from home. It's a small physical setup (a firewall, a managed switch, and a mini PC running Proxmox) that I'm using to practice the stuff I studied for my degree and my CCNA. I plan to break things on purpose in the lab and fix them without taking anything else down with it. This repo is where I keep the configs, diagrams, and notes as I go.

Status: Phase 0 (foundation) in progress.

## Hardware

| Device | Role |
| Protectli Vault FW6A (OPNsense) | Firewall/router: VLAN gateways, DHCP, firewall rules 
| TP-Link TL-SG108E | Managed switch: trunks and access ports |
| Lenovo ThinkCentre M920q, i7-9700T, 64 GB (Proxmox VE) | Hypervisor for VMs, services, and virtual topologies |

## Design

The whole lab sits behind its own firewall, so anything I break stays inside the lab.

See the [Phase 0 topology diagrams](diagrams/phase-0-topology.md) for the physical and logical layout.

- Separate VLANs for management, lab hosts, and services
- Default-deny between VLANs; only traffic I explicitly allow gets through
- Unused switch ports go to a dead-end VLAN with no gateway or DHCP
- A break-glass admin port on the firewall in case I lock myself out with a VLAN mistake
- VLAN-aware bridge on Proxmox, so one trunk carries every VLAN to the host
- Pi-hole + Unbound for DNS
- Nightly Proxmox backups, with a test restore before I start experimenting

### Addressing

The lab uses 10.20.0.0/14. The VLAN number matches the third octet, so any address tells you which VLAN it's in.

| Network | VLAN | Purpose |
|---|---|---|
| 10.20.10.0/24 | 10 MGMT | Firewall, switch, Proxmox management |
| 10.20.20.0/24 | 20 LAB | Lab VMs and test hosts |
| 10.20.30.0/24 | 30 SERVICES | DNS, later monitoring and NetBox |
| none | 999 BLACKHOLE | Unused switch ports |

Each /24 uses the same layout: .1 gateway, .2-.9 network devices, .10-.99 static servers, .100-.199 DHCP.

## Tools

Proxmox VE, OPNsense, Containerlab, Cisco Modeling Labs, netlab, Batfish, and Git/GitHub.

## Roadmap

### Phase 0: Foundation (in progress)

Done so far:
- Created this repo and set up Git
- Wrote the IP plan, then revised it to add an inter-zone policy matrix and IPv6 (ULA) addressing, plus a YAML version I checked with a script for overlapping subnets
- Downloaded Proxmox, OPNsense, and Memtest86+, verified every image's SHA-256 hash, and built a Ventoy USB with all three

Still to do:
- Install OPNsense on the FW6A and set up the VLANs and firewall rules
- Configure trunks and access ports on the switch
- Install Proxmox on the M920q once it arrives
- Set up Pi-hole + Unbound
- Set up backups and test a restore

### Later phases (planned)

- Phase 1: Campus switching and routing (STP, EtherChannel, OSPF) in CML and Containerlab
- Phase 2: Security edge (DMZ, NAT, WireGuard and IPsec VPNs)
- Phase 3: Monitoring (SNMPv3, syslog, NetFlow, Suricata)
- Phase 4: Automation (Python, Ansible, NetBox, Batfish, ZTP)
- Phase 5: WAN routing (BGP, DMVPN, IPv6)

Down the road I'd also like to get into EVPN-VXLAN, wireless with a Cisco AP and controller, and running some security tools locally.

## Lessons learned

Things that broke or tripped me up, and what I did about it.


- Windows' built-in extractor couldn't open the `.bz2` download. I had to use a different tool to unpack it.
- OPNsense ships with its LAN set to 192.168.1.1, which would have collided with the upstream router's network the moment I plugged in the WAN. I caught this while planning the build, so the first step after installing is changing the LAN address on the console, before the WAN is ever connected.
- My first IP plan was just a document without an inter-zone policy matrix and IPv6 (ULA) addressing. I made a YAML version I could check with a script. The script confirmed none of the subnets overlap, which is easy to get wrong by hand once a plan has a dozen networks in it.
- My hash-check script reported a match for an image that wasn't actually on the USB. It was checking a copy in a different folder. Now I confirm the script is pointed at the exact file before trusting its result.

## Structure

```
docs/       Notes for each phase
configs/    Sanitized configs (no passwords, keys, or identifying info)
diagrams/   Topology and addressing diagrams
```

Folders get added as I finish each phase.

## Contact

Mason Dettloff · [LinkedIn](https://linkedin.com/in/mason-richard-dettloff)
