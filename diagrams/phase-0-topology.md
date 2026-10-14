# Phase 0 topology

## Physical

How the hardware is cabled. Port numbers are switch ports on the TL-SG108E and physical ports on the FW6A.

```mermaid
flowchart TB
    NET(("Internet"))
    UP["Upstream router"]
    FW["fw01 · Protectli FW6A<br/>OPNsense"]
    SW["TL-SG108E switch<br/>10.20.10.2"]
    PVE["M920q · Proxmox VE<br/>10.20.10.5"]
    LAP["Admin laptop"]
    SPARE["Spare lab device"]
    BG["Break-glass laptop<br/>10.20.11.0/24"]
    DEAD["Ports 5-8<br/>VLAN 999, disabled"]

    NET --- UP
    UP ---|"FW port 1 · WAN (DHCP)"| FW
    FW ---|"FW port 2 → SW port 1<br/>trunk: 10, 20, 30"| SW
    FW -.-|"FW port 3 · break-glass"| BG
    SW ---|"SW port 2 · trunk: 10, 20, 30"| PVE
    SW ---|"SW port 3 · access VLAN 10"| LAP
    SW ---|"SW port 4 · access VLAN 20"| SPARE
    SW -.- DEAD
```

## Logical

The VLANs, their gateways on the firewall, and what lives in each one. Traffic between VLANs is denied unless a rule allows it.

```mermaid
flowchart TB
    FW["fw01 · OPNsense<br/>default-deny between VLANs"]

    subgraph MGMT["VLAN 10 · MGMT · 10.20.10.0/24"]
        M1["Gateway 10.20.10.1"]
        M2["Switch 10.20.10.2"]
        M3["Proxmox 10.20.10.5"]
        M4["Admin laptop · DHCP"]
    end

    subgraph LAB["VLAN 20 · LAB · 10.20.20.0/24"]
        L1["Gateway 10.20.20.1"]
        L2["Lab VMs · DHCP"]
    end

    subgraph SVC["VLAN 30 · SERVICES · 10.20.30.0/24"]
        S1["Gateway 10.20.30.1"]
        S2["dns01 · Pi-hole + Unbound<br/>10.20.30.53"]
    end

    subgraph ADMIN["Break-glass · 10.20.11.0/24 · no VLAN"]
        A1["Gateway 10.20.11.1"]
    end

    FW --- MGMT
    FW --- LAB
    FW --- SVC
    FW -.- ADMIN
```

Every /24 uses the same layout: .1 gateway, .2-.9 network devices, .10-.99 static servers, .100-.199 DHCP.
