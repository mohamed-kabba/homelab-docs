# Home Lab

![VMware ESXi](https://img.shields.io/badge/VMware_ESXi_8.0.3-3B9EFF?style=flat-square&logo=vmware&logoColor=white)
![Proxmox VE](https://img.shields.io/badge/Proxmox_VE_9.2-3B9EFF?style=flat-square&logo=proxmox&logoColor=white)
![FortiGate](https://img.shields.io/badge/FortiGate-3B9EFF?style=flat-square&logo=fortinet&logoColor=white)
![MikroTik](https://img.shields.io/badge/MikroTik-3B9EFF?style=flat-square&logo=mikrotik&logoColor=white)
![Cisco](https://img.shields.io/badge/Cisco-3B9EFF?style=flat-square&logo=cisco&logoColor=white)
![Zabbix](https://img.shields.io/badge/Zabbix_7.4-3B9EFF?style=flat-square&logo=zabbix&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-3B9EFF?style=flat-square)
![NetBox](https://img.shields.io/badge/NetBox-3B9EFF?style=flat-square)

A production-patterned virtualisation and networking environment running on six second-hand
machines in Rostock. Two hypervisor platforms, sixteen virtual machines, three routed network
segments behind a hardware firewall, and a self-hosted monitoring and SIEM stack built for
roughly **€500**, entirely from used hardware.

It exists so that I can practise the work I want to be paid for: designing, deploying,
breaking and repairing infrastructure that has to keep running.

---

## At a glance

| | |
|---|---|
| **Physical machines** | Five hypervisor hosts, one bare-metal Windows Server |
| **Hypervisors** | VMware ESXi 8.0.3 (×3) · Proxmox VE 9.2.3 (×2) |
| **Management** | vCenter Server · Proxmox Datacenter Manager |
| **Virtual machines** | 16 running, all treated as production |
| **Network segments** | 3 routed subnets |
| **Edge firewall** | FortiGate 60D |
| **Remote access** | Tailscale only — nothing exposed to the public internet |
| **Monitoring** | Zabbix 7.4 across 25 hosts · Grafana · Wazuh SIEM |
| **Total hardware spend** | ~€500, all bought used |

---

## Architecture

The estate sits behind a residential internet connection I do not administer, which shaped
most of the design. Three segments, described from the outside in:

**House network.** The building's own Wi-Fi. I have no administrative access to this router,
so it functions as an uncontrolled upstream rather than as part of the lab.

**MikroTik network.** A MikroTik RB951Ui-2HnD takes a DHCP address from the house access
point over a **wireless point-to-point link**. Housing rules forbid running cable to the
access point, so the uplink is wireless by necessity — an approach carried over from four
years running a wireless ISP. The MikroTik then serves two of its own networks: a wireless
LAN for devices that can only reach the lab over Wi-Fi, and a wired LAN for the management
PC and other cabled clients. It routes between them and provides DHCP.

**Lab network.** Created by the FortiGate 60D, which sits behind the MikroTik and forms the
perimeter for everything that matters. A single port feeds the Cisco 2960C, and every
hypervisor host hangs off that switch. The NAS is dual-homed, with one interface on the
MikroTik network and one on the lab network.

DNS and Active Directory for the estate are served by a bare-metal Windows Server.

### A note on segmentation

This is **routed segmentation, not VLAN segmentation**. The estate is small and single-user,
so the switch runs a single VLAN and separation is achieved at the routing boundaries between
the MikroTik and the FortiGate. Introducing VLANs and trunking here would add configuration
without adding isolation the routed design does not already provide. It is on the roadmap as
a deliberate exercise once the CCNA work makes it useful.

---

## Physical hosts

| Host | Hardware | CPU | RAM | Storage | Role |
|---|---|---|---|---|---|
| esxi1 | Lenovo ThinkCentre M700 | i5-6500T @ 2.50 GHz | 24 GB | 256 GB NVMe + 350 GB HDD | Management and monitoring |
| esxi2 | Lenovo ThinkCentre M910t | i5-8500 @ 3.00 GHz | 32 GB | 256 GB NVMe + 932 GB HDD | Services and lab tooling |
| esxi3 | Lenovo ThinkCentre M700 | i5-6400T @ 2.20 GHz | 16 GB | 256 GB SSD + 1 TB HDD | Applications and dashboards |
| proxmox1 | HP ProDesk 400 G4 DM | i3-7100T @ 3.40 GHz | 16 GB | 256 GB NVMe + 1.14 TB HDD | Automation and remote access |
| proxmox2 | Apple Mac Mini (2011) | i5-2415M @ 2.30 GHz | 16 GB | 256 GB SSD + 256 GB HDD | Containerised workloads |
| dns | Fujitsu Esprimo G558 | i5-9500T @ 2.20 GHz | 4 GB | 256 GB SSD | Windows Server, bare metal |

All six were bought second-hand. The mix is deliberate rather than accidental — see
[Design decisions](#design-decisions). Measured draw sits around 81 Wh.

---

## Virtual machines

| VM | Host | Role |
|---|---|---|
| vCenter Server | esxi1 | Central management for the three ESXi hosts |
| Windows Server 2022 Core | esxi1 | Command-line Windows administration practice |
| Zabbix | esxi1 | Metrics collection and trigger-based monitoring |
| EVE-NG | esxi2 | Multi-vendor topology emulation for certification labs |
| Home Assistant | esxi2 | Smart-device control and power metering via a Shelly plug |
| Ollama | esxi2 | Local LLM — **shut down**, see [Incidents](#operational-incidents) |
| NetBox | esxi2 | IPAM and device inventory |
| Wazuh | esxi2 | SIEM and file integrity monitoring |
| Cisco Modelling Labs | esxi3 | Cisco-specific lab work alongside EVE-NG |
| Life Dashboard | esxi3 | Self-hosted Flask application (Nginx + Gunicorn) |
| Grafana | esxi3 | Visualisation dashboards |
| Tailscale | proxmox1 | Subnet router for remote access |
| n8n | proxmox1 | Workflow automation — **being rebuilt**, see [Incidents](#operational-incidents) |
| Datacenter Manager | proxmox1 | Central management for the Proxmox nodes |
| WordPress | proxmox2 | Containerised web development |
| Odoo | proxmox2 | ERP evaluation |

Containers: Portainer, Homarr, Homepage.

Every VM is treated as production and runs continuously. Machines are replaced only when a
second-hand purchase offers a meaningful specification increase.

**Why two emulation platforms.** EVE-NG carries multi-vendor topologies — the point is
combining Cisco, MikroTik and Fortinet images in one lab, which reflects how mixed estates
actually look. Cisco Modelling Labs runs alongside it for Cisco-specific work, partly to
evaluate it honestly against Packet Tracer rather than assuming the heavier tool is better.

---

## Network devices

| Device | Function | Configuration |
|---|---|---|
| MikroTik RB951Ui-2HnD | Lab entry point | Wireless point-to-point uplink, dual LAN, inter-network routing, DHCP |
| FortiGate 60D | Perimeter firewall | Two interfaces, minimal policy set |
| Cisco WS-C2960C-8TC-L | Access switch | SSH management, SNMP polled by Zabbix |
| QNAP TS-321P | Storage | RAID 1, dual-homed, ~38% utilised |

**On the FortiGate:** it runs a deliberately minimal policy set. The estate is single-user
and all remote access is handled at the Tailscale layer, so elaborate policy here would be
configuration for its own sake. The 60D is also past end-of-support and is run knowingly as
lab-only hardware; migrating to a virtual FortiGate instance is on the roadmap.

---

## Remote access

A dedicated Tailscale node acts as a subnet router, advertising the lab network to my
tailnet; several individual hosts also run their own clients.

**Nothing in the lab is reachable from the public internet.** There are no port forwards and
no inbound NAT. Every remote session — administration, monitoring, application access —
arrives over Tailscale. This removes the exposed VPN endpoint a firewall-terminated tunnel
would require, and works cleanly behind an upstream router I do not control, which a
traditional VPN would not.

Two iterations are queued: moving every host onto its own client rather than relying on the
single subnet router, and replacing the current default-allow tailnet policy with explicit
ACLs.

---

## Monitoring and security

**Zabbix 7.4** polls 25 hosts — hypervisors, virtual machines and network devices — using
vendor and community templates. Triggers cover CPU, memory, disk capacity, latency,
bandwidth and process state. The Cisco switch is monitored via SNMP; the ESXi estate through
the VMware templates.

**Grafana** provides dashboards over the collected data, organised by device class:
hypervisors, firewall, routers, switches, Linux servers, Windows servers, NAS and IoT.

**Wazuh** runs as the SIEM, handling security event collection and file integrity monitoring
across nine agents. It is currently on its default ruleset — interpreting the output and
writing custom rules is active learning rather than finished work, and I would rather say so
than imply a tuned deployment.

**NetBox** holds device and IPAM records, originally populated automatically by an n8n
workflow that scanned the network and updated inventory. That sync is currently paused
pending the n8n rebuild.

### Alerting: an honest status

Alerts were previously delivered to Telegram, routed through a local LLM that summarised each
event before sending. That pipeline has been **removed** — the LLM consumed more resources
than the feature justified. Zabbix continues to collect and trigger, but there is currently
no active delivery path. Restoring it, with a lighter model or none at all, is the top
roadmap item.

---

## Backup and recovery

| | |
|---|---|
| **Target** | QNAP TS-321P, RAID 1, ~38% utilised |
| **Contents** | Website backups, documentation, study material, ESXi configurations, personal media |
| **Schedule** | Nightly at 01:10 — external VPS to NAS |
| **Off-site** | None for lab data |
| **Restore tested** | No |

The only automated job pulls the externally hosted website back to the NAS each night. The
lab itself has no scheduled backup, no snapshot policy and no tested restore — an untested
backup is an assumption rather than a recovery plan, and the MikroTik failure described below
proved the point at my own expense. This is the next block of work after the current
documentation effort.

---

## Design decisions

**Why two hypervisors.** The plan was ESXi only. Then I bought a machine ESXi would not run
on, and rather than discard it I learned Proxmox. Running both has been more instructive than
standardising would have been: it forces solutions that are not the mainstream answer for
either platform, and it shows where each is genuinely stronger. Hardware compatibility on the
second-hand market was the real driver; licensing was secondary.

**Why FortiGate.** I hold Fortinet NSE 1–4 and have worked with FortiGate professionally.
Owning the physical device means configuration practice on real hardware rather than a
simulator, which matters for the certification path.

**Why Tailscale over the FortiGate's VPN.** Cost and flexibility. I have configured IPsec
tunnels in production, so this is not avoidance of the harder option — but licensing costs
are real on a €500 budget, and Tailscale traverses an upstream router I do not administer
without requiring anything of it. It also means no publicly reachable VPN endpoint at all.

**Why both Zabbix and Wazuh.** They answer different questions. Zabbix tells me whether
infrastructure is healthy; Wazuh tells me whether it has been tampered with. Running both
mirrors the separation between monitoring and security operations in a real environment.

**Why NetBox rather than a spreadsheet.** A spreadsheet would hold the data. It would not be
queryable by automation. NetBox exists here as the source of truth that scripts and workflows
read from, which is the pattern used at scale — practising it on sixteen VMs is the point.

**Why second-hand hardware.** €500 for six machines, sixteen VMs, vCenter, a SIEM and a
hardware firewall. The constraint is the lesson: small-form-factor business desktops deliver
a credible virtualisation estate at a fraction of the cost of used server hardware, with a
fraction of the noise and power draw — which matters when the lab lives in a flat.

---

## Operational incidents

**Local LLM starving the host.** Zabbix reported the Ollama VM at 100% CPU. The host is
CPU-only with no GPU, so inference was compute-bound and the VM was consuming capacity the
rest of the estate needed. The decision was to shut the service down rather than let a
non-essential feature degrade the platform — including the alert-summarising pipeline that
depended on it. Finding a model small enough to be worth the resources is ongoing.

**MikroTik hardware failure.** The router failed and had to be replaced. I had no
configuration backup, so the replacement was rebuilt from scratch. Entirely avoidable, and
the direct reason device configuration backups are now a priority rather than an intention.

**n8n major version upgrade.** Upgrading across several major versions changed the node set
and broke the existing workflows. Some have been rebuilt; the platform is currently dormant
while other work takes priority. The lesson was about upgrade paths on self-hosted platforms
with breaking changes — reading release notes is not optional, and neither is a rollback plan.

---

## Constraints and current work

Listed deliberately. An estate with no acknowledged weaknesses is an estate nobody has
assessed honestly. Each of these is a known gap with a plan attached rather than an oversight.

| Constraint | Status |
|---|---|
| No tested restore procedure | Next work block after documentation |
| No hypervisor snapshot policy | Same |
| No off-site copy of lab data | Planned |
| No UPS | Blocked on finding one that reports consumption accurately |
| FortiGate 60D past end-of-support | Migration to virtual FortiGate planned |
| No alert delivery while n8n is down | Top roadmap item |
| Wazuh on default rules | Active learning |
| Tailscale ACLs not configured | Planned hardening |
| NetBox sync paused | Returns with the n8n rebuild |

---

## Roadmap

- Restore alert delivery — lighter local model, or plain Zabbix to Telegram
- Rebuild n8n on the current version and restore NetBox synchronisation
- Establish hypervisor snapshot and restore-test procedures
- Migrate FortiGate from physical to virtual
- Add a Raspberry Pi for DNS and network-wide ad blocking
- Source a UPS that reports consumption accurately — the affordable options do not, which
  has kept this open longer than it should have
- Build out EVE-NG topologies for CCNA preparation
- Establish configuration backups for all network devices
- Introduce VLAN segmentation as a deliberate exercise

---

---

## Notes on this repository

Private address ranges are published as configured. They are RFC1918 addresses behind NAT and
carry no meaning outside this network. Hostnames are redacted, and no public addresses,
Tailscale addresses, credentials or configuration secrets appear anywhere in this repository.

---

**Mohamed Kabba** — IT Infrastructure Engineer, Rostock
[portfolio.kabba.tech](https://portfolio.kabba.tech) · mohamed@kabba.tech
