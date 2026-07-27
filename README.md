# Enterprise Infrastructure HomeLab

**🇬🇧 English** | [🇩🇪 Deutsch](README.de.md)

![Status](https://img.shields.io/badge/status-in%20progress-yellow)
![Platform](https://img.shields.io/badge/platform-KVM%20%2F%20QEMU-blue)
![OS](https://img.shields.io/badge/OS-Windows%20Server%202019%20%7C%20CentOS%20Stream%209-lightgrey)
![Docs](https://img.shields.io/badge/docs-EN%20%7C%20DE-green)

A virtualized lab environment that simulates the core IT infrastructure of a small company.
Built and documented as a hands-on portfolio project for an **Ausbildung as Fachinformatiker
für Systemintegration** in Germany.

---

## 1. Project Goal

This lab is intentionally **not** an attempt to build a large enterprise datacenter.
The goal is a realistic, fully documented environment that demonstrates practical skills in:

- **Virtualization** — KVM/QEMU with libvirt
- **Networking** — IP planning, DHCP, DNS, routing concepts
- **Windows Administration** — Active Directory, domain services, user management
- **Linux Administration** — services, firewall, permissions, SSH
- **Security Basics** — network scanning and service verification
- **Backup & Recovery** — backup concept including a tested restore
- **Technical Documentation** — reproducible, bilingual, version-controlled

Every component in this lab exists for a reason. Nothing is installed just to
increase the service count.

---

## 2. Architecture

Current implementation — a single flat, NAT-connected virtual network:

```
                        Internet
                            |
                    [ Ubuntu 24.04 Host ]
                     KVM / QEMU / libvirt
                     virbr1 - NAT Gateway
                       192.168.100.1
                            |
                 Virtual Network "homelab"
                    192.168.100.0/24
                            |
     +--------------+--------------+--------------+
     |              |              |              |
  LAB-DC01      LAB-CL01       LAB-WEB01      LAB-SEC01
 WinSrv 2019    Windows 10    CentOS Str. 9     Parrot
 AD / DNS /      Domain        Web / SSH       Security
    DHCP         Client         Server        Workstation
 .100.10         DHCP          .100.20         .100.50
```

> A diagram of the planned expansion (segmented client network with routing)
> is documented in [Phase 1](phases/01-network-design/).

---

## 3. Host Environment

| Component | Specification |
| --- | --- |
| Device | Lenovo Legion 5 |
| CPU | Intel Core i7-10750H (6C / 12T) |
| RAM | 16 GB |
| Host OS | Ubuntu 24.04.4 LTS |
| Hypervisor | KVM / QEMU via Virtual Machine Manager (libvirt) |
| VM Storage | 1 TB HDD |

---

## 4. Virtual Machines

| Hostname | Role | Operating System | IP Address | RAM | vCPU |
| --- | --- | --- | --- | --- | --- |
| `LAB-DC01` | Domain Controller, DNS, DHCP | Windows Server 2019 | 192.168.100.10 | 4 GB | 4 |
| `LAB-WEB01` | Web & SSH Server | CentOS Stream 9 | 192.168.100.20 | 2 GB | 2 |
| `LAB-CL01` | Domain Client | Windows 10 | DHCP | 4 GB | 2 |
| `LAB-SEC01` | Security Workstation | Parrot Security 7.2 | 192.168.100.50 | 3 GB | 4 |

---

## 5. Network & IP Concept

| Range | Purpose |
| --- | --- |
| `192.168.100.1 - .9` | Network infrastructure (gateway) |
| `192.168.100.10 - .49` | Servers (static) |
| `192.168.100.50 - .99` | Management & security systems (static) |
| `192.168.100.100 - .150` | DHCP scope served by `LAB-DC01` |
| `192.168.100.151 - .254` | Reserved for future expansion |

- **AD Domain:** `corp.homelab.internal`
- **NetBIOS Name:** `HOMELAB`
- **Internal DNS:** `LAB-DC01` (192.168.100.10), forwarding to 192.168.100.1

Full details and design decisions: [`docs/ip-plan/`](docs/ip-plan/) and
[`docs/architecture/decisions.md`](docs/architecture/decisions.md)

---

## 6. Project Phases

| # | Phase | Focus | Status |
| --- | --- | --- | --- |
| 0 | [Foundation](docs/architecture/) | Repository, IP plan, naming, decisions | 🟨 In progress |
| 1 | [Network Design](phases/01-network-design/) | Topology & addressing in Cisco Packet Tracer | ⬜ Planned |
| 2 | [Virtual Infrastructure](phases/02-virtual-infrastructure/) | libvirt networking, VM provisioning | ⬜ Planned |
| 3 | [Windows Server](phases/03-windows-server/) | Active Directory, DNS, DHCP, OU structure | ⬜ Planned |
| 4 | [Windows Client](phases/04-windows-client/) | Domain join, authentication testing | ⬜ Planned |
| 5 | [Linux Server](phases/05-linux-server/) | SSH, nginx, firewalld, permissions | ⬜ Planned |
| 6 | [Security Testing](phases/06-security-testing/) | Network scanning, service discovery | ⬜ Planned |
| 7 | [Documentation](phases/07-documentation/) | Troubleshooting, lessons learned | ⬜ Planned |
| 8 | [Backup & Recovery](phases/08-backup-recovery/) | Backup concept and verified restore | ⬜ Planned |

---

## 7. Repository Structure

```
homelab/
├── docs/
│   ├── architecture/       Naming conventions, decision log
│   ├── ip-plan/            Addressing concept
│   ├── lessons-learned/    Insights per phase
│   └── troubleshooting/    Problems and solutions
├── phases/                 One documented folder per phase
├── configs/                Exported configuration files
├── diagrams/               Network and architecture diagrams
├── packet-tracer/          Cisco Packet Tracer source files
├── screenshots/            Verification screenshots per phase
└── assets/                 Additional images
```

---

## 8. Documentation Standard

Every configuration step in this repository answers five questions:

1. **Purpose** — What was installed and why?
2. **Configuration** — How was it configured?
3. **Verification** — How was the result tested?
4. **Result** — What was achieved?
5. **Lessons Learned** — What went wrong and how was it solved?

---

## 9. Credentials Notice

> ⚠️ **Isolated lab environment.**
> All credentials in this repository are intentionally published demo values, used
> exclusively inside a non-routable virtual network. They are never reused in any
> real or production environment.
> Windows Server 2019 is used under its 180-day evaluation license.

---

## 10. Out of Scope (for now)

Deliberately postponed to keep the project focused and maintainable:
Kubernetes, Docker Swarm, Terraform, Ansible, Grafana, Prometheus, ELK Stack,
Jenkins, GitLab, SIEM, cloud infrastructure, high availability.

These are documented as potential expansion phases.

---

## Author

**AmirMohammad Eftekhari**
Aspiring Fachinformatiker für Systemintegration

- GitHub: [@eftekhari-amirmohammad](https://github.com/eftekhari-amirmohammad)
- Portfolio: [eftekhari-amirmohammad.github.io](https://eftekhari-amirmohammad.github.io)

## License

Released under the [MIT License](LICENSE).
