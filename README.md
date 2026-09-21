# Homelab: VLAN Segmentation + Monitoring

![Homelab VLAN Segmentation and Monitoring architecture diagram](./images/homelab-vlan-monitoring-banner(2).png)

I rebuilt my homelab's network segmentation from scratch, added a dedicated personal VLAN with one-way trust isolation, provisioned Ubuntu Server VMs, stood up a monitoring stack (Prometheus + SNMP Exporter + Grafana), extended remote access over Tailscale into the lab subnet, and then grew the single Proxmox host into a two-node cluster with both nodes running as VLAN-aware trunks.

## Stack

- **Router/Firewall:** MikroTik hEX S (RouterOS 7.20.7)
- **Switch:** TP-Link TL-SG108E (802.1Q managed)
- **Hypervisor:** Proxmox VE 9.2, two-node cluster named `homelab`
  - `pve`: HP EliteDesk 800 G1 (i5-4590T, 16GB RAM), management IP 10.0.10.4
  - `pve2`: HP EliteDesk 800 G2 Mini (quad-core i5, 16GB RAM, 240GB SSD), management IP 10.0.10.5
- **Monitoring:** Prometheus, SNMP Exporter, Grafana (all bare-metal systemd services, no containers)
- **Remote access:** Tailscale, with subnet routing into the lab VLANs
- **Public site:** Flask app on `ubuntu-services`, exposed through a Cloudflare Tunnel at labwithbuike.dev

## Physical setup

<details>
<summary>Rack (click to expand)</summary>

![Physical homelab rack](./images/rack-photo.jpg)

TP-Link TL-SG108E switch and patch panel at the top, both HP EliteDesk minis (`pve` and `pve2`) below them, and a 2-bay drive enclosure underneath.

</details>

## Starting point

Before this build, my MikroTik hEX S was already routing three VLANs on the Proxmox host:

| VLAN | Subnet | Purpose |
|------|--------|---------|
| 10 | 10.0.10.0/24 | Management |
| 20 | 10.0.20.0/24 | Lab servers |
| 30 | 10.0.30.0/24 | Lab clients |

My personal devices (laptop, phone) shared the untagged bridge network (192.168.88.0/24) with no isolation from the lab at all. That was the first thing I set out to fix.

## Phase 1: Adding a personal VLAN with one-way trust

The goal: my laptop should be able to reach into the lab to manage it, but nothing in the lab should be able to reach back out to my laptop.

### Renaming the existing VLANs

First I renamed the existing VLAN interfaces on the MikroTik to be more descriptive:

```
/interface vlan set [find name=vlan10-mgmt] name=vlan10-management
/interface vlan set [find name=vlan20-services] name=vlan20-lab-servers
/interface vlan set [find name=vlan30-dmz] name=vlan30-lab-clients
```

I kept the existing 10.0.x.0/24 addressing rather than renumbering to match a reference architecture doc I'd been using for planning, which used 10.10.x.0/24. Renumbering would have meant touching DHCP scopes and static IPs on my domain controller for no real benefit.

### Building VLAN 40

Created the new VLAN on the same trunk interface, with its own subnet, address, and DHCP pool:

```
/interface vlan add interface=ether2 name=vlan40-personal vlan-id=40
/ip address add address=10.0.40.1/24 interface=vlan40-personal network=10.0.40.0
/ip pool add name=pool-vlan40 ranges=10.0.40.100-10.0.40.200
/ip dhcp-server add address-pool=pool-vlan40 interface=vlan40-personal name=dhcp-vlan40
/ip dhcp-server network add address=10.0.40.0/24 dns-server=10.0.40.1 gateway=10.0.40.1
```

### Trusted device setup

I reserved a fixed IP for my laptop's MAC address and added it to a trusted address list, so the firewall rules could reference "my laptop specifically" rather than the whole VLAN 40 subnet:

```
/ip dhcp-server lease add address=10.0.40.10 mac-address=C8:A3:62:04:C8:2E server=dhcp-vlan40 comment="trusted-laptop"
/ip firewall address-list add address=10.0.40.10 list=trusted-lab-access comment="trusted laptop"
```

### Firewall isolation

Built an address list for the lab networks, then three rules: trusted device to lab is allowed but only on specific ports, anything else on VLAN 40 to the lab is blocked, and lab to VLAN 40 is blocked in every direction:

```
/ip firewall address-list add address=10.0.10.0/24 list=lab-networks comment="mgmt"
/ip firewall address-list add address=10.0.20.0/24 list=lab-networks comment="lab-servers"
/ip firewall address-list add address=10.0.30.0/24 list=lab-networks comment="lab-clients"

/ip firewall filter add action=accept chain=forward src-address-list=trusted-lab-access dst-address-list=lab-networks protocol=tcp dst-port=22,8291,3389,80,443,3000,9090 comment="trusted device to lab - allowed ports"
/ip firewall filter add action=drop chain=forward src-address=10.0.40.0/24 dst-address-list=lab-networks comment="vlan40 to lab - block untrusted"
/ip firewall filter add action=drop chain=forward src-address-list=lab-networks dst-address=10.0.40.0/24 comment="lab to vlan40 - block"
```

I went with allowing only specific management ports (SSH, Winbox, RDP, plus common web ports for anything I'd want to check in a browser) rather than the whole subnet, since a fully open personal-to-lab rule would mean a compromised laptop has unrestricted access to everything in the lab, not just the tools I actually use to manage it. Later I added port 8006 to the list for the Proxmox web UI.

### Physical switch config

This is where I found something I hadn't expected: 802.1Q VLAN filtering had never actually been enabled on my TP-Link TL-SG108E switch. The router-side VLANs had apparently been working through some other path, but the switch itself was passing everything through flat.

I enabled 802.1Q VLAN mode on the switch and built out all four VLANs properly:

| VLAN | Port 1 (MikroTik) | Port 2 (`pve`) | Port 3 (laptop) |
|------|-------------------|----------------|-----------------|
| 10 | Tagged | Tagged | Not member |
| 20 | Tagged | Tagged | Not member |
| 30 | Tagged | Tagged | Not member |
| 40 | Tagged | Not member | Untagged |

Then set Port 3's PVID to 40, so untagged frames arriving from my laptop get classified onto VLAN 40:

Switch web UI: 802.1Q PVID Setting → Port 3 → PVID 40 → Apply.

### The lockout

After making this change, I lost SSH and Winbox access to the router entirely, from any device on VLAN 40, including my laptop.

Root cause: none of the four VLAN interfaces were members of the router's `LAN` interface list, and the router's default input-chain firewall rule drops anything not arriving from an interface on that list. This wasn't unique to VLAN 40, it affected all four VLANs, I just hadn't noticed with 10/20/30 because I'd never tried to SSH into the router directly from any of them before.

Recovery: I moved my laptop's cable to an untouched switch port still on the old untagged network (192.168.88.0/24), which still had router access since the `bridge` interface was in the `LAN` list. From there I added all four VLAN interfaces to the list:

```
/interface list member add interface=vlan10-management list=LAN
/interface list member add interface=vlan20-lab-servers list=LAN
/interface list member add interface=vlan30-lab-clients list=LAN
/interface list member add interface=vlan40-personal list=LAN
```

Moved my laptop back to Port 3, and everything worked as intended: my laptop reachable into the lab on management ports, nothing in the lab able to reach back.

## Phase 1.5: Static IP reservations

Once everything was working, I locked in static DHCP reservations for the lab devices so their IPs would never drift on lease renewal, including for VMs provisioned later in this build:

```
/ip dhcp-server lease add address=10.0.20.108 mac-address=BC:24:11:C6:6C:CE server=dhcp-vlan20 comment="ubuntu-services"
/ip dhcp-server lease add address=10.0.20.109 mac-address=BC:24:11:6B:34:84 server=dhcp-vlan20 comment="monitoring01"
/ip dhcp-server lease add address=10.0.20.107 mac-address=BC:24:11:62:14:D3 server=dhcp-vlan20 comment="linux-practice01"
/ip dhcp-server lease add address=10.0.20.110 mac-address=0C:EF:15:05:DB:31 server=dhcp-vlan20 comment="TL-SG108E-switch"
```

## Phase 2: Provisioning the VMs

I cleared out old VMs and provisioned three fresh Ubuntu Server 26.04 VMs via a bash script on the Proxmox host, using `qm create`:

| VM | ID | Specs | Purpose |
|----|----|-------|---------|
| ubuntu-services | 102 | 2 vCPU / 2GB RAM / 60GB | General services, Flask site |
| monitoring01 | 103 | 2 vCPU / 2GB RAM / 30GB | Monitoring stack |
| linux-practice01 | 104 | 1 vCPU / 2GB RAM / 30GB | Disposable Linux practice sandbox |

The 4-core host CPU meant these three VMs alone (5 vCPU assigned) already oversubscribed the physical core count, which is fine since Proxmox timeslices and none of them need to be under heavy load simultaneously.

`linux-practice01`'s installer hung on "downloading and installing security updates" for over five minutes with 1GB RAM. I bumped it to 2GB and restarted the install, which resolved it. Likely the installer process plus package unpacking needed more headroom than 1GB comfortably provided.

All three came up correctly on VLAN 20 with the expected hostnames, confirmed with `hostnamectl status` on each.

<details>
<summary>Proof: all three VMs provisioned (click to expand)</summary>

![qm list showing all three VMs provisioned with correct specs](./images/qm-list-vms-provisioned.png)

</details>

## Phase 3: Monitoring stack

I chose Prometheus, SNMP Exporter, and Grafana over Zabbix. Zabbix is arguably more turnkey for pure SNMP polling, but this stack is closer to what I'll actually run into in cloud-focused work, which fits my current AWS-first cert path better.

### Prometheus

Installed via binary (not the snap package, for config flexibility) with a script that auto-detects the latest release version. Created a dedicated `prometheus` system user with no login shell, standard practice for service accounts.

First run failed. The script used `set -e` and died partway through when `mv consoles/` failed, since that Prometheus release no longer ships the legacy `consoles/` folder. That meant `prometheus.yml` never got moved into `/etc/prometheus/`, and systemd showed the service as briefly "active" right after each restart before it crashed on the missing config file. I only saw the actual error clearly with `journalctl -u prometheus`, since the live `systemctl status` output truncated the message. Fixed by manually moving the config file over.

```
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/

[Install]
WantedBy=multi-user.target
```

### SNMP Exporter

My install script hardcoded a version that was already out of date. I started checking the GitHub API for the actual latest release before downloading, going forward, and installed v0.30.1.

Enabled SNMP on the MikroTik:

```
/snmp set enabled=yes
/snmp community set [find default=yes] name=public
```

Tested the full chain manually before wiring it into Prometheus:

```bash
curl "http://localhost:9116/snmp?target=10.0.10.1&module=if_mib"
```

This returned real interface data for every VLAN and physical port, confirming SNMP Exporter could successfully query the router.

### Wiring Prometheus to SNMP Exporter

Added a scrape job to `prometheus.yml` that routes the actual SNMP query through the exporter via relabeling, which is the standard pattern for any Prometheus exporter that proxies to a third-party device:

```yaml
  - job_name: 'mikrotik'
    static_configs:
      - targets:
          - 10.0.10.1
    metrics_path: /snmp
    params:
      module: [if_mib]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9116
```

Confirmed via Prometheus's own Targets page that both jobs (self-monitoring and MikroTik) showed UP.

### Grafana

Install failed the first time because the GPG key download for the apt repo came back empty, so apt correctly refused to trust an unsigned source. Re-ran the key import using `gpg --dearmor` properly instead of assuming the raw curl output would work as a valid keyring:

```bash
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/grafana.key > /dev/null
```

After that, install and service start went cleanly. Connected Grafana to Prometheus as a data source (`http://localhost:9090`, confirmed with "Successfully queried the Prometheus API"), then built the **Homelab Network Overview** dashboard:

| Panel | Query | Notes |
|-------|-------|-------|
| VLAN Interface Traffic (Inbound) | `rate(ifHCInOctets{ifName=~"vlan.*"}[5m])` | Unit set to bytes/sec, legend `{{ifName}}` |
| Interface Status | `ifOperStatus{ifName=~"vlan.*"}` | Stat panel with value mappings (1 = UP in green, 2 = DOWN in red) instead of a flat line |
| Broadcast Traffic (Router Ports) | `rate(ifHCInBroadcastPkts{ifName=~"ether1\|ether2\|bridge"}[5m])` | Physical ports only, see note below |
| Total Bandwidth per VLAN (In + Out) | `sum by (ifName) (rate(ifHCInOctets{ifName=~"vlan.*"}[5m]) + rate(ifHCOutOctets{ifName=~"vlan.*"}[5m]))` | Inbound plus outbound combined |

**Broadcast panel: what went wrong first.** The panel came up empty. In Grafana's Explore view I found that the old 32-bit `ifInBroadcastPkts` counter returns 0 on every port, because RouterOS doesn't populate it. The 64-bit `ifHCInBroadcastPkts` does report real data, but only on the physical ports and the bridge, not on the VLAN sub-interfaces. Pointing the panel at the physical ports with `rate()` fixed it.

<details>
<summary>Proof: Grafana dashboard live (click to expand)</summary>

![Grafana dashboard showing VLAN interface traffic, status, broadcast traffic, and bandwidth](./images/grafana-dashboard.png)

</details>

### TP-Link switch SNMP

Checked whether the TL-SG108E supports SNMP so I could monitor it too. It's TP-Link's budget "Easy Smart" tier, one step below their SNMP-capable JetStream line, and there's no SNMP menu anywhere in its web UI. I'm treating this as a confirmed hardware limitation rather than something to work around.

## Phase 4: Remote access

I'd set up Tailscale on this host previously, but when I checked, it wasn't actually installed anymore, likely lost during an earlier Proxmox VE version migration. Reinstalled and reauthenticated:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

To reach the lab VMs over Tailscale without installing it on each one individually, I advertised the lab subnet from the host (later I also added `192.168.88.0/24` to the advertised routes):

```bash
tailscale up --advertise-routes=10.0.20.0/24
```

Approved the route in the Tailscale admin console. It didn't work at first, traffic just silently failed to route, because IP forwarding was disabled on the host by default. Tailscale warns about this during `up` but doesn't block the command on it. Fixed by explicitly enabling it:

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

in `/etc/sysctl.d/99-tailscale.conf`, applied with `sysctl -p`. After that, I could SSH into any of the lab VMs from my laptop over Tailscale regardless of which network I was physically on.

## Phase 5: Second Proxmox node and clustering

I added a second HP EliteDesk (`pve2`) to run more VMs and to practice clustering. Goal: two nodes joined into one cluster over the management VLAN, with both nodes set up the same way.

### pve2 couldn't reach its own gateway

I installed Proxmox on `pve2` with its management IP `10.0.10.5` set directly on `vmbr0`. It could not ping its gateway (`10.0.10.1`), and the error came from `10.0.10.5` itself ("Destination Host Unreachable"), which means the ARP request for the gateway never got an answer.

The NIC was up, and `tcpdump` on `vmbr0` showed only switch housekeeping frames (RSTP and loop-detection broadcasts) and no ARP or DHCP traffic from other hosts. That pointed at the switch port, not the host.

Root cause: I had added `pve2`'s switch port (port 4) as a **tagged** member of VLAN 10, but the host sends **untagged** frames, because the IP sits directly on the bridge. The switch decides which VLAN an untagged frame belongs to using the port's **PVID**, and port 4's PVID was still 1. So every frame landed in VLAN 1, which the gateway never sees.

The distinction that made it click:

- **VLAN membership (tagged/untagged)** controls how frames *leave* a port, with the tag kept or stripped.
- **PVID** controls which VLAN *untagged frames entering* a port are assigned to.

My first node worked without PVID changes because it tags VLAN 10 itself (`vmbr0.10`), so the switch reads the tag and ignores the PVID.

A quick fix would have been setting port 4 to untagged VLAN 10 with PVID 10, and that got `pve2` online. But it would have blocked any VM tagged for VLAN 20, so I converted both nodes to the same trunk design instead.

### Converting both nodes to VLAN-aware trunks

The clean design for a Proxmox node: the bridge is VLAN-aware, the host's management IP lives on a tagged VLAN interface, and each VM NIC gets its own VLAN tag. `pve2`:

```
auto vmbr0
iface vmbr0 inet manual
	bridge-ports nic0
	bridge-stp off
	bridge-fd 0
	bridge-vlan-aware yes
	bridge-vids 2-4094

auto vmbr0.10
iface vmbr0.10 inet static
	address 10.0.10.5/24
	gateway 10.0.10.1
```

Switch port 4 went back to PVID 1 with VLANs 10 and 20 tagged. `pve` got the same VLAN-aware bridge (it keeps its original untagged `192.168.88.10/24` on `vmbr0` for now), and I updated its `/etc/hosts` so the hostname resolves to `10.0.10.4`.

Current switch layout:

| VLAN | Port 1 (MikroTik) | Port 2 (`pve`) | Port 3 (laptop) | Port 4 (`pve2`) |
|------|-------------------|----------------|-----------------|-----------------|
| 10 | Tagged | Tagged | Not member | Tagged |
| 20 | Tagged | Tagged | Not member | Tagged |
| 30 | Tagged | Tagged | Not member | Not member |
| 40 | Tagged | Not member | Untagged (PVID 40) | Not member |

### Creating the cluster

Before changing anything on `pve`, I backed up its network config and `/etc/hosts`, then created the cluster over the management VLAN, and joined `pve2` from its side:

```bash
# on pve
pvecm create homelab --link0 10.0.10.4

# on pve2
pvecm add 10.0.10.4 --link0 10.0.10.5
```

The join failed the first time with `500 Can't connect to 10.0.10.4:8006 (hostname verification failed)`. `pve`'s web certificate had been generated when the node was `192.168.88.10`, so `10.0.10.4` wasn't in it. Regenerating the node certificate fixed it:

```bash
pvecm updatecerts --force
systemctl restart pveproxy
```

I confirmed the new certificate's Subject Alternative Names included `10.0.10.4` before retrying the join, which then succeeded. `pvecm status` now shows two nodes, quorate, both on the `10.0.10.x` link.

<details>
<summary>Proof: two-node cluster online (click to expand)</summary>

![Proxmox datacenter summary: cluster homelab, 2 nodes online, quorate](./images/proxmox-cluster.png)

</details>

### Repositories and updates

`pve2` failed its first `apt-get update` because a fresh Proxmox install ships with the enterprise repositories enabled, which return errors without a subscription. In the web UI I disabled the `pve-enterprise` and `ceph-enterprise` repos and enabled `pve-no-subscription`, then upgraded `pve2` first (it had no VMs), rebooted it into the new kernel, and confirmed it rejoined the cluster. `pve` is still on the older release until I have a maintenance window.

### Storage

I looked at turning a 2-bay drive enclosure into shared storage for the cluster, but it has no network port, so it would need a computer to serve it. Serving it from one of the Proxmox nodes would make that node a single point of failure for the storage, so I shelved the idea for now.

## Where things stand

| System | Access |
|--------|--------|
| `pve` | `ssh root@10.0.10.4` or `ssh root@192.168.88.10` (local), `ssh root@100.79.197.10` (Tailscale, anywhere) |
| `pve2` | `ssh root@10.0.10.5` |
| ubuntu-services | `ssh buike-lab@10.0.20.108` |
| monitoring01 | `ssh buike-lab@10.0.20.109` (Tailscale-reachable via subnet route) |
| linux-practice01 | `ssh buike-lab@10.0.20.107` (Tailscale-reachable via subnet route) |
| MikroTik | `ssh admin@10.0.10.1` |
| Grafana | `http://10.0.20.109:3000` |
| Proxmox web UI (either node) | `https://10.0.10.4:8006` |

## Design decisions

**Kept the existing 10.0.x.0/24 addressing** instead of matching a reference doc's 10.10.x.0/24 scheme. Renumbering working VLANs for no functional gain wasn't worth the disruption to DHCP scopes and static IPs already built on top of them.

**Folded the planned NAS/Samba VM into ubuntu-services** rather than running it as a separate VM. With only 16GB of host RAM and other VMs already running, this was a deliberate resource trade-off rather than following the original plan literally.

**Restricted personal-to-lab access to specific management ports** rather than allowing the whole VLAN 40 subnet through. A fully open rule would mean a compromised laptop has unrestricted access to everything in the lab.

**Built the monitoring stack bare-metal instead of in Docker.** This meant actually working through real systemd behavior, config file paths, and service dependencies by hand, rather than having Docker abstract that away. Worth revisiting as a containerized rebuild later, now with an actual understanding of what's happening underneath.

**Ran both Proxmox nodes as VLAN-aware trunks** with the management IP on a tagged interface (`vmbr0.10`), instead of leaving `pve2` on an untagged access port. The access-port shortcut worked, but it would have broken VLAN 20 for any VM placed on that node.

**Kept the cluster at two nodes for now.** With two votes, if one node is down the other can't start VMs or change configuration until it returns (running VMs keep running). A QDevice on a small always-on machine fixes that, and it's on the list.

## Things that broke

- Forgot the MikroTik admin password partway through. Recovered via Winbox's MAC-based connection instead of a factory reset, which would have wiped the whole existing VLAN/firewall config.
- A `!` in a new password broke RouterOS's terminal parser, since it treats `!` specially outside quotes.
- Adding VLAN 40 locked me out of the router entirely from any VLAN, not just the new one, because none of the VLAN interfaces were in the router's `LAN` interface list.
- Prometheus crash-looped on first start because the install script's `mv` command silently failed on a folder that no longer exists in newer releases, leaving the config file in the wrong place.
- SNMP Exporter install script had a stale hardcoded version.
- Grafana's apt repo key failed signature verification because the initial key download was empty.
- Tailscale had gone missing from the host at some point after being set up previously.
- Tailscale's subnet route was approved but non-functional until I manually enabled IP forwarding on the host.
- `linux-practice01`'s installer hung for 5+ minutes on 1GB RAM before I bumped it to 2GB.
- The Grafana broadcast panel was empty because RouterOS doesn't fill the 32-bit broadcast counter and the VLAN sub-interfaces don't report the 64-bit one.
- `pve2` couldn't reach its gateway because its switch port was tagged for VLAN 10 while the host sent untagged frames (PVID was still 1).
- The cluster join failed on a certificate hostname check because `pve`'s certificate predated the management IP change.
- `pve2`'s package update failed until I swapped the enterprise repositories for `pve-no-subscription`.

## What's still open

- Upgrade `pve` to match `pve2`, then reboot it to clear leftover VLAN 20 interfaces from the old bridge setup
- Retire the untagged `192.168.88.10` address on `pve` (update SSH shortcuts and the Tailscale subnet route first)
- Add a QDevice so the cluster keeps quorum when one node is down
- Shared or network-attached storage for VM disks and backups
- Containerize the Flask site with Docker, and automate backups (including encrypted backups to AWS S3)
- Site-to-site VPN
- Using `linux-practice01` for something
