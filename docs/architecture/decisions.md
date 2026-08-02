# Architecture Decision Log

> Part of **Phase 0 — Foundation**. Every significant technical decision in this lab is
> recorded here with its context, the chosen option, the alternatives that were rejected
> and the resulting consequences. The purpose is to show *why* the environment looks the
> way it does, not only *what* was installed.

| ID | Decision | Date | Status |
| --- | --- | --- | --- |
| [ADR-001](#adr-001) | Reuse the existing `192.168.100.0/24` subnet | 2026-07-27 | Accepted |
| [ADR-002](#adr-002) | Disable the libvirt DHCP server in favour of Windows DHCP | 2026-07-27 | Accepted |
| [ADR-003](#adr-003) | Use `corp.homelab.internal` as the AD domain name | 2026-07-27 | Accepted |
| [ADR-004](#adr-004) | Rename all virtual machines to a role-based scheme | 2026-07-27 | Accepted |
| [ADR-005](#adr-005) | Use CentOS Stream 9 as the Linux server platform | 2026-07-27 | Accepted |
| [ADR-006](#adr-006) | Implement one flat subnet, document segmentation as target state | 2026-07-27 | Accepted |
| [ADR-007](#adr-007) | Store all virtual disks on the HDD, not the SSD | 2026-07-27 | Accepted |
| [ADR-008](#adr-008) | Bilingual documentation (English primary, German for overviews) | 2026-07-27 | Accepted |
| [ADR-009](#adr-009) | Publish demo credentials openly | 2026-07-27 | Accepted |
| [ADR-010](#adr-010) | Use libvirt snapshots as rollback points per phase | 2026-07-27 | Accepted |

---

## ADR-001

### Reuse the existing `192.168.100.0/24` subnet

**Context** — A libvirt network named `homelab` already existed with the subnet
`192.168.100.0/24` and the gateway `192.168.100.1`.

**Decision** — Keep this subnet and build the addressing concept around it.

**Alternatives considered** — Renumbering to `10.10.10.0/24` for a more "enterprise"
look.

**Reasoning** — Renumbering would require reconfiguring the bridge and every guest
without adding any technical value. `192.168.100.0/24` is a valid RFC 1918 private range.
Effort is better invested in the structure *within* the subnet.

**Consequences** — The documented range allocation (see `docs/ip-plan/`) provides the
structure that a raw subnet choice would not.

---

## ADR-002

### Disable the libvirt DHCP server in favour of Windows DHCP

**Context** — The `homelab` network was configured with the libvirt built-in dnsmasq DHCP
server handing out `192.168.100.128 - .254`. Phase 3 introduces a Windows DHCP server on
`LAB-DC01`.

**Decision** — Remove the `<dhcp>` block from the libvirt network definition and let
`LAB-DC01` be the only DHCP server. NAT forwarding is kept so guests retain internet
access for updates.

**Alternatives considered**

1. Keep both — rejected: two DHCP servers in one broadcast domain answer client requests
   in a race, so clients receive non-deterministic configuration. Typical symptom is an
   intermittently failing domain join with misleading error messages.
2. Switch the network to isolated mode — rejected: guests would lose internet access,
   which is required for Windows updates and package installation.

**Consequences** — Before the change the original definition is exported to
`configs/network-homelab-before.xml` so the modification is traceable. Address assignment
is centralised on the domain controller, which matches real company practice.

---

## ADR-003

### Use `corp.homelab.internal` as the AD domain name

**Context** — The Active Directory DNS domain name must be chosen before promotion and
cannot be changed easily afterwards.

**Decision** — `corp.homelab.internal` with the NetBIOS name `HOMELAB`.

**Alternatives considered**

1. `homelab.local` — rejected: `.local` is reserved for multicast DNS (RFC 6762) and
   Microsoft explicitly advises against it.
2. A single-label name such as `HOMELAB` — rejected: single-label AD domains require
   additional client configuration and are unsupported for new deployments.
3. A subdomain of a publicly owned domain — rejected: no domain is owned for this lab.

**Reasoning** — `.internal` is reserved by ICANN for private internal use, so it can
never collide with a public name. The dedicated `corp.` label follows the Microsoft
recommendation of placing the directory in its own subdomain rather than at the apex.

**Consequences** — Logon names use the NetBIOS form `HOMELAB\username`.

---

## ADR-004

### Rename all virtual machines to a role-based scheme

**Context** — The existing libvirt domains were named `winserver`, `centos-stream9`,
`win10` and `Parrot_Sec`, describing the operating system rather than the function.

**Decision** — Rename to `LAB-DC01`, `LAB-WEB01`, `LAB-CL01` and `LAB-SEC01`, matching
the guest hostname exactly. The full scheme is documented in
`docs/architecture/naming-convention.md`.

**Reasoning** — Operating systems get replaced, roles remain. Identical names in
`virsh list` and inside the guest remove a whole category of confusion during
troubleshooting.

**Consequences** — Renaming a libvirt domain requires the machine to be shut down
(`virsh domrename`). Performed once, at the start, before any dependency exists.

---

## ADR-005

### Use CentOS Stream 9 as the Linux server platform

**Context** — A CentOS Stream 9 virtual machine was already installed.

**Decision** — Keep CentOS Stream 9 for the Linux server role.

**Alternatives considered** — Rocky Linux 9 or AlmaLinux 9 (downstream RHEL rebuilds),
Debian 12.

**Reasoning** — Stream 9 shares the RHEL 9 toolchain (`dnf`, `firewalld`, SELinux,
`systemd`), which is the ecosystem most relevant to enterprise environments in Germany.
It is important to state the trade-off honestly: CentOS Stream is the upstream development
branch of RHEL, not a stability-focused downstream rebuild. For a production deployment,
RHEL or Rocky Linux would be the correct choice; for a learning environment the
administrative experience is equivalent.

**Consequences** — All Linux procedures in this lab transfer directly to RHEL-family
systems.

---

## ADR-006

### Implement one flat subnet, document segmentation as target state

**Context** — Network segmentation with routing between a server and a client network
would demonstrate more advanced networking knowledge, but requires an additional router
instance and considerable configuration effort.

**Decision** — Implement a single flat subnet `192.168.100.0/24`. Design the segmented
target architecture in Cisco Packet Tracer in Phase 1 and document it as a planned
expansion.

**Reasoning** — Distinguishing between the current state and a planned target state is
itself part of infrastructure planning. A fully working flat network with complete
documentation is worth more than a half-finished segmented one.

**Consequences** — Phase 1 delivers two diagrams: the implemented architecture and the
target architecture. The addressing plan already reserves ranges for the second segment.

---

## ADR-007

### Store all virtual disks on the HDD, not the SSD

**Context** — The host has a 512 GB SSD (shared with a Windows dual boot) and a 1 TB HDD.
All `qcow2` images currently reside on the HDD.

**Decision** — Keep all virtual disks on the HDD.

**Reasoning** — Preserving SSD endurance and free space was prioritised over guest
performance. The lab is not performance-critical; only one-off operations such as the
Active Directory installation take noticeably longer.

**Consequences** — Longer installation and boot times are accepted. To limit total I/O
load, no more than three virtual machines run at the same time.

---

## ADR-008

### Bilingual documentation, English primary and German for overviews

**Context** — The project targets German employers, while the platform audience on GitHub
is international.

**Decision** — The main README and every phase README exist in English (`README.md`) and
German (`README.de.md`). Deep technical detail such as command listings, scan output and
troubleshooting notes is maintained in English only.

**Reasoning** — A German reader finds every conceptual document in German, while the
technical layer stays in the language technical documentation is normally written in.
Translating every command listing would double the effort without adding information.

**Consequences** — Both language versions cross-link at the top of the document.

---

## ADR-009

### Publish demo credentials openly

**Context** — The documentation shows administrative logons and password dialogs.

**Decision** — Use clearly marked demonstration credentials and publish them, together
with an explicit notice in the README.

**Reasoning** — Reproducibility is a documentation goal. The environment is a
non-routable virtual network, and none of these credentials is reused anywhere else.
Redacting them would reduce the value of the documentation without adding real security.

**Consequences** — A hard rule applies: no credential used in this repository is ever
reused in a real environment. Screenshots are reviewed before every commit to ensure no
unintended data is visible.

---

## ADR-010

### Use libvirt snapshots as rollback points per phase

**Context** — Configuration errors in Active Directory, DNS or the firewall can leave a
guest in a state that is faster to rebuild than to repair.

**Decision** — Create a named libvirt snapshot before each critical phase, with the
machine shut down. Naming follows `docs/architecture/naming-convention.md`.

**Alternatives considered** — Full clones (rejected: disk space) and reinstalling on
failure (rejected: time).

**Consequences** — Snapshots are taken offline to avoid large memory-state files on the
HDD, and are deleted once a phase is verified as working, to reclaim space.

---

## ADR-011

### Forward external DNS queries to public resolvers instead of the NAT gateway

**Context** — ADR-003 configured the domain controller to forward all non-domain queries to
`192.168.100.1`, the libvirt NAT gateway, which passes them to the host resolver. During
Phase 5 the CentOS server could not install packages: the mirror names returned a CNAME
chain without a final A record, so the query succeeded technically but delivered no usable
address. Queries sent directly to public resolvers returned complete answers for the same
names.

**Decision** — Configure `8.8.8.8` and `1.1.1.1` as the forwarders of the domain
controller. The NAT gateway remains the default gateway for all guests; only name
resolution bypasses the upstream resolver.

**Alternatives considered** — Entering the public resolvers on each guest (rejected: the
domain controller must stay the only DNS server for the clients, otherwise domain lookups
fail) and adding static host entries for the mirrors (rejected: not maintainable, and it
hides the underlying problem).

**Consequences** — Name resolution no longer depends on the behaviour of the upstream
resolver, and the failure is fixed in exactly one place for all guests. In return the lab
now requires the two public resolvers to be reachable, and external DNS traffic leaves the
host unencrypted. In a production environment this decision would be made differently,
because an internal resolver is normally mandatory for policy and logging reasons.
