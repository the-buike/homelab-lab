### Home Lab Network Infrastructure
A production-style home lab built on MikroTik RouterOS, designed to simulate enterprise network segmentation, security policies, and infrastructure management.
Hardware

MikroTik hEX Router (firewall/gateway)
TP-Link TL-SG108E 8-port managed switch
HP EliteDesk Mini (Ubuntu Server 24.04.4 LTS)
2-bay NAS enclosure with 2× 2TB SATA HDDs
Internet via MoCA adapter over coax (Frontier)

Network Design
VLANNameSubnetPurposeVLAN 10Management10.0.10.0/24Admin access, Domain ControllerVLAN 20Services10.0.20.0/24Nextcloud, Jellyfin, Pi-holeVLAN 30DMZ10.0.30.0/24Public-facing servicesVLAN 40Surveillance10.0.40.0/24Cameras, Frigate NVR (planned)
What's Configured

VLAN segmentation on MikroTik hEX via RouterOS CLI
Inter-VLAN firewall rules — VLANs isolated from each other
DHCP servers per VLAN with dedicated address pools
Tailscale remote access mesh VPN
Config exported and version controlled

Planned

Windows Server 2022 VM — Active Directory, DNS, NPS/RADIUS
802.1X VLAN authentication backed by AD
Frigate NVR + camera VLAN (VLAN 40)
WiFi AP with VLAN tagging
