# BUIKE-LAB Network Knowledge Base

Configuration reference for the routing and switching layer of homelab.local. This covers the MikroTik hEX (RTR-HEX01) and the TP-Link TL-SG108E (SW-TPL01): VLAN design, addressing, DHCP, firewall zones, and how the two devices are configured to work together.

---

## 1. Network Overview

| Item | Value |
|---|---|
| Domain | homelab.local |
| Router | MikroTik hEX (RTR-HEX01), RouterOS, configured via CLI only |
| Router LAN IP | 192.168.88.1 |
| Switch | TP-Link TL-SG108E (SW-TPL01), 802.1Q VLANs |
| Hypervisor | HP EliteDesk Mini, Proxmox VE (PVE01), 192.168.88.10 |
| WAN | Xfinity via Frontier FCA252 MoCA adapter over room coax |
| Remote access | Tailscale (WireGuard direct ruled out by CGNAT) |

### Physical connections (cable schedule)

| Cable ID | From | To | Type | Carries |
|---|---|---|---|---|
| COAX-1 | Xfinity XB8 (sitting room) | FCA252 MoCA adapter | Coax | WAN |
| CAB-01 | FCA252 | RTR-HEX01 ether1 (WAN) | Cat6 | WAN, untagged |
| CAB-02 | RTR-HEX01 LAN port | SW-TPL01 | Cat6 | Trunk: VLANs 10, 20, 30 |
| CAB-03 | SW-TPL01 | PVE01 NIC | Cat6 | Trunk: VLANs 10, 20, 30 |

---

## 2. VLAN Design

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| 10 | MGMT | 10.0.10.0/24 | 10.0.10.1 | Management. Domain controller, infrastructure administration |
| 20 | SERVICES | 10.0.20.0/24 | 10.0.20.1 | Service and application VMs |
| 30 | DMZ | 10.0.30.0/24 | 10.0.30.1 | Reserved for anything exposed to the internet |

The base 192.168.88.0/24 network remains as the router and hypervisor management network (RouterOS default LAN). Lab workloads live in the 10.x VLANs.

### Design decisions

- **Why VLANs at all:** one flat network means the domain controller, test services, and anything internet-facing all share a broadcast domain. Segmenting by function means firewall policy controls every crossing, which is how an enterprise network actually behaves.
- **Why VLAN 30 is empty:** it is reserved on purpose. When something needs to be reachable from outside, it goes in the DMZ and never shares a segment with the DC.
- **Why the trunk design:** the hEX has limited ports, so all three VLANs ride one trunk to the switch (CAB-02) and one trunk onward to Proxmox (CAB-03). Proxmox breaks the tags out per-VM.

---

## 3. MikroTik Configuration (RTR-HEX01)

All configuration on this router is done from the RouterOS CLI, not WinBox point-and-click.

### 3.1 Bridge and VLAN interfaces

RouterOS handles VLANs with a bridge in VLAN-filtering mode. VLAN interfaces hang off the bridge and act as the L3 gateway for each segment.

```routeros
# Bridge with VLAN filtering
/interface bridge
add name=bridge1 vlan-filtering=yes

# LAN port that carries the trunk to the switch
/interface bridge port
add bridge=bridge1 interface=ether2

# Tagged VLANs on the trunk port and the bridge itself
/interface bridge vlan
add bridge=bridge1 tagged=bridge1,ether2 vlan-ids=10
add bridge=bridge1 tagged=bridge1,ether2 vlan-ids=20
add bridge=bridge1 tagged=bridge1,ether2 vlan-ids=30

# L3 VLAN interfaces (the gateways)
/interface vlan
add interface=bridge1 name=vlan10-mgmt vlan-id=10
add interface=bridge1 name=vlan20-services vlan-id=20
add interface=bridge1 name=vlan30-dmz vlan-id=30
```

### 3.2 IP addressing

```routeros
/ip address
add address=192.168.88.1/24 interface=bridge1 comment="Base LAN"
add address=10.0.10.1/24 interface=vlan10-mgmt
add address=10.0.20.1/24 interface=vlan20-services
add address=10.0.30.1/24 interface=vlan30-dmz
```

### 3.3 DHCP, one pool per VLAN

Each VLAN gets its own pool and its own DHCP server bound to the VLAN interface.

```routeros
/ip pool
add name=pool-vlan10 ranges=10.0.10.100-10.0.10.199
add name=pool-vlan20 ranges=10.0.20.100-10.0.20.199
add name=pool-vlan30 ranges=10.0.30.100-10.0.30.199

/ip dhcp-server
add name=dhcp-vlan10 interface=vlan10-mgmt address-pool=pool-vlan10
add name=dhcp-vlan20 interface=vlan20-services address-pool=pool-vlan20
add name=dhcp-vlan30 interface=vlan30-dmz address-pool=pool-vlan30

/ip dhcp-server network
add address=10.0.10.0/24 gateway=10.0.10.1 dns-server=10.0.10.10
add address=10.0.20.0/24 gateway=10.0.20.1 dns-server=10.0.10.10
add address=10.0.30.0/24 gateway=10.0.30.1 dns-server=1.1.1.1
```

Key detail: VLAN 10 and 20 hand out **DC01 (10.0.10.10)** as DNS so domain resolution works. The DMZ hands out public DNS because DMZ hosts have no business querying the domain controller.

Static assignments outside the pool ranges:

| Host | IP | VLAN | Assignment |
|---|---|---|---|
| DC01 | 10.0.10.10 | 10 | Static on the VM |
| ubuntu-services | 10.0.20.101 | 20 | Static/reservation |

### 3.4 Firewall zones

Policy between zones, written as: management can reach everything, services can reach only what they need, DMZ can reach nothing internal.

```routeros
/ip firewall filter
# Established/related first, always
add chain=forward action=accept connection-state=established,related
add chain=forward action=drop connection-state=invalid

# MGMT (VLAN 10) can initiate to anywhere
add chain=forward action=accept in-interface=vlan10-mgmt

# SERVICES (VLAN 20) to MGMT: only AD/DNS traffic to DC01
add chain=forward action=accept in-interface=vlan20-services \
    dst-address=10.0.10.10 protocol=udp dst-port=53,88,123,389 comment="AD: DNS, Kerberos, NTP, LDAP"
add chain=forward action=accept in-interface=vlan20-services \
    dst-address=10.0.10.10 protocol=tcp dst-port=53,88,135,389,445,464,636,3268-3269,49152-65535 comment="AD: TCP + RPC range"
add chain=forward action=drop in-interface=vlan20-services out-interface=vlan10-mgmt comment="Block everything else Services->MGMT"

# SERVICES to internet: allowed
add chain=forward action=accept in-interface=vlan20-services out-interface=ether1

# DMZ (VLAN 30): internet only, nothing internal
add chain=forward action=drop in-interface=vlan30-dmz dst-address=10.0.10.0/24
add chain=forward action=drop in-interface=vlan30-dmz dst-address=10.0.20.0/24
add chain=forward action=accept in-interface=vlan30-dmz out-interface=ether1

# Default deny at the end
add chain=forward action=drop comment="Default deny"
```

### 3.5 NAT

```routeros
/ip firewall nat
add chain=srcnat action=masquerade out-interface=ether1 comment="Masquerade all LAN/VLANs to WAN"
```

### 3.6 WAN

ether1 is the WAN port, fed by the FCA252 MoCA adapter (CAB-01). It takes a DHCP lease from the Xfinity gateway upstream. There is no public IP on the router itself because Xfinity residential is behind CGNAT, which is why remote access runs over Tailscale instead of an inbound VPN.

---

## 4. Switch Configuration (SW-TPL01, TL-SG108E)

The TL-SG108E is a smart switch managed through its web UI. It does not do L3; its whole job here is carrying 802.1Q tags correctly between the router and the Proxmox host.

### 4.1 802.1Q VLAN table

| VLAN ID | Name | Tagged Ports | Untagged Ports |
|---|---|---|---|
| 1 | Default | 
 | All (management fallback) |
| 10 | MGMT | Port 1 (to router), Port 2 (to PVE01) | 
 |
| 20 | SERVICES | Port 1, Port 2 | 
 |
| 30 | DMZ | Port 1, Port 2 | 
 |

### 4.2 PVID settings

Trunk ports keep PVID 1 so untagged frames fall into the default VLAN rather than leaking into a lab segment. Any port assigned to a single VLAN as an access port gets its PVID set to that VLAN.

| Port | Role | PVID |
|---|---|---|
| 1 | Trunk to RTR-HEX01 (CAB-02) | 1 |
| 2 | Trunk to PVE01 (CAB-03) | 1 |
| 3-8 | Spare / access as assigned | 1 or per-VLAN |

### 4.3 TL-SG108E quirks worth remembering

- 802.1Q must be explicitly enabled (it ships in port-based VLAN mode).
- The switch management IP lives in the default VLAN, so if you ever untag VLAN 1 off the uplink port you lose the web UI until you factory reset. Keep VLAN 1 reachable.
- No CLI, no config export. The VLAN and PVID tables above **are** the backup. If the switch dies, this document is how it gets rebuilt.

---

## 5. Proxmox VLAN Handling (PVE01)

The trunk terminates at the Proxmox host, where the Linux bridge is VLAN-aware and each VM's NIC carries a VLAN tag:

| VM | VLAN Tag on NIC | Result |
|---|---|---|
| DC01 (VM 101) | 10 | Lands in 10.0.10.0/24 |
| ubuntu-services (VM 100) | 20 | Lands in 10.0.20.0/24 |

Enable VLAN awareness on the bridge in `/etc/network/interfaces` (`bridge-vlan-aware yes`), then set the tag per-VM in the NIC settings. No subinterface sprawl needed.

---

## 6. Verification Commands

Quick checks to confirm the network is behaving, useful after any change.

**On the MikroTik:**
```routeros
/interface bridge vlan print          # VLAN table on the bridge
/ip address print                     # Gateways up on each VLAN interface
/ip dhcp-server lease print           # Who has leases, per pool
/ip firewall filter print stats       # Rule hit counters, confirms policy is actually matching
/ping 10.0.10.10                      # DC reachable from the router
```

**From a VM:**
```bash
ip addr                               # Right subnet for its VLAN?
ip route                              # Default gateway is the VLAN gateway?
nslookup homelab.local 10.0.10.10     # DC resolving the domain?
traceroute 10.0.20.101                # Inter-VLAN path goes through the router?
```

**The test that proves segmentation works:** from a VLAN 20 host, attempt something to 10.0.10.10 that is not on the allow list (RDP on 3389, for example). It should time out. If it connects, the firewall order is wrong or a rule is too broad.

---

## 7. Change Log

| Date | Change | Phase |
|---|---|---|
| 2026-04-11 | Core build day one: MoCA WAN delivery over room coax confirmed, Ubuntu Server 24.04 installed on the EliteDesk with SSH and static IP | 1 |
| 2026-04-15 | Ethernet cables arrived. MikroTik, switch, and EliteDesk cabled and online. Lab has end-to-end internet through the hEX | 1 |
| 2026-04-15 | WireGuard abandoned: CGNAT on Xfinity plus same-network hairpin made inbound tunnels unworkable. Tailscale deployed on the EliteDesk instead | 4 |
| 2026-07-05 | VLANs 10/20/30 built from the RouterOS CLI: VLAN interfaces, gateway IPs, per-VLAN DHCP pools, inter-VLAN firewall filter rules. Config exported as homelab-config.rsc and pushed to GitHub | 2 |
| 2026-07-05 | EliteDesk wiped and reinstalled with Proxmox VE (pve, 192.168.88.10). VM 100 (ubuntu-services, VLAN 20, 10.0.20.101) created with Tailscale. VM 101 (Windows Server 2022, VLAN 10) created with VirtIO drivers | 1 |
| 2026-07-13 | DC01 given static 10.0.10.10 and promoted to domain controller for homelab.local. AD DS and DNS verified, OU structure created (Employees, Workstations, Groups, Service Accounts) | 3 |

