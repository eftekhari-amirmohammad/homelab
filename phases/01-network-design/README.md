# Phase 1 — Network Design

**🇬🇧 English** | [🇩🇪 Deutsch](README.de.md)

[← Back to project overview](../../README.md)

![Phase](https://img.shields.io/badge/phase-01-blue)
![Status](https://img.shields.io/badge/status-completed-brightgreen)
![Tool](https://img.shields.io/badge/tool-Cisco%20Packet%20Tracer-informational)

---

## 1. Purpose

Before installing a single service, the network had to be planned and validated.
The goal of this phase was to answer three questions **on paper and in a simulator**,
not by trial and error on live systems:

1. Does the addressing concept from [`docs/ip-plan/`](../../docs/ip-plan/) actually work?
2. Will a client receive a correct lease from a central DHCP server, including gateway and DNS?
3. What would this network look like in a production environment, and how does that differ
   from what the available hardware allows?

Two topologies were therefore built in Cisco Packet Tracer:

| File | Purpose |
| --- | --- |
| [`homelab-current-state.pkt`](../../packet-tracer/homelab-current-state.pkt) | The network as actually implemented on the KVM host |
| [`homelab-target-architecture.pkt`](../../packet-tracer/homelab-target-architecture.pkt) | The segmented design that would be used in production |

---

## 2. Current State — As Implemented

![Packet Tracer topology of the implemented network](../../screenshots/phase-01/01-01-packet-tracer-topology-current-state.png)
*Figure 1.1 — Implemented topology: a single flat subnet `192.168.100.0/24`.*

### 2.1 Modelling note

On the real host, the Linux bridge `virbr1` performs **two roles at once**: it switches
traffic at Layer 2 between the virtual machines, and it holds the IP address
`192.168.100.1` acting as the Layer 3 NAT gateway to the internet.

Packet Tracer has no object that behaves this way, so the bridge is modelled as two
separate devices — a switch (`LAB-SW01`) and a router (`HOST-GW`). This is a deliberate
simplification of the simulation, not a difference in the real configuration.

### 2.2 Port map

| Device | Local interface | Connected to | Remote interface |
| --- | --- | --- | --- |
| `HOST-GW` | `GigabitEthernet0/0` | `LAB-SW01` | `GigabitEthernet0/1` |
| `LAB-DC01` | `FastEthernet0` | `LAB-SW01` | `FastEthernet0/1` |
| `LAB-WEB01` | `FastEthernet0` | `LAB-SW01` | `FastEthernet0/2` |
| `LAB-CL01` | `FastEthernet0` | `LAB-SW01` | `FastEthernet0/3` |
| `LAB-SEC01` | `FastEthernet0` | `LAB-SW01` | `FastEthernet0/4` |

### 2.3 Addressing

| Device | IP address | Subnet mask | Gateway | DNS | Assignment |
| --- | --- | --- | --- | --- | --- |
| `HOST-GW` | 192.168.100.1 | 255.255.255.0 | — | — | static |
| `LAB-DC01` | 192.168.100.10 | 255.255.255.0 | 192.168.100.1 | 192.168.100.10 | static |
| `LAB-WEB01` | 192.168.100.20 | 255.255.255.0 | 192.168.100.1 | 192.168.100.10 | static |
| `LAB-SEC01` | 192.168.100.50 | 255.255.255.0 | 192.168.100.1 | 192.168.100.10 | static |
| `LAB-CL01` | 192.168.100.101 | 255.255.255.0 | 192.168.100.1 | 192.168.100.10 | DHCP |

All clients point to `192.168.100.10` as their **only** DNS server. No public resolver is
configured, because a domain member that resolves names through an external DNS server
cannot find the `_ldap._tcp` service records of the domain — which would break the domain
join in Phase 4.

---

## 3. Configuration

### 3.1 Gateway router

```
hostname HOST-GW
no ip domain-lookup
!
interface GigabitEthernet0/0
 description LAN - homelab virtual network (192.168.100.0/24)
 ip address 192.168.100.1 255.255.255.0
 no shutdown
```

The router interface came up only after `no shutdown`. Cisco router interfaces are
`administratively down` by default, unlike switch ports, which are enabled out of the box.

### 3.2 DHCP service on `LAB-DC01`

![DHCP pool configuration on LAB-DC01](../../screenshots/phase-01/01-02-dhcp-pool-configuration.png)
*Figure 1.2 — DHCP pool `HOMELAB-Clients` configured on the simulated domain controller.*

| Setting | Value |
| --- | --- |
| Pool name | `HOMELAB-Clients` |
| Start IP address | 192.168.100.100 |
| Subnet mask | 255.255.255.0 |
| Maximum number of users | 51 (range `.100` – `.150`) |
| Default gateway (option 003) | 192.168.100.1 |
| DNS server (option 006) | 192.168.100.10 |

The range deliberately matches the DHCP scope defined in
[`docs/ip-plan/`](../../docs/ip-plan/). Two DHCP parameters of the real design cannot be
represented in Packet Tracer — the **lease duration** (8 days) and **option 015 Domain
Name** (`corp.homelab.internal`). Both are configured on the real Windows Server in Phase 3.

### 3.3 Removing the DHCP conflict on the host

The libvirt network `homelab` originally ran its own dnsmasq DHCP server on the same
subnet. Two DHCP servers in one broadcast domain answer client requests in a race,
which produces unpredictable addresses and DNS settings — and a failed domain join.

The built-in DHCP was therefore removed while keeping NAT and the gateway address.
The before/after configurations are documented in
[`configs/`](../../configs/) and the decision in
[ADR-002](../../docs/architecture/decisions.md).

---

## 4. Verification

![DHCP lease and connectivity test on LAB-CL01](../../screenshots/phase-01/01-03-client-dhcp-lease-and-connectivity-test.png)
*Figure 1.3 — `LAB-CL01` receives a valid lease and reaches gateway and servers.*

| # | Test | Command | Expected result | Result |
| --- | --- | --- | --- | --- |
| 1 | Client receives lease | `ipconfig /all` on `LAB-CL01` | Address from `.100`–`.150`, GW `.1`, DNS `.10` | ✅ 192.168.100.101 |
| 2 | Client → gateway | `ping 192.168.100.1` | 4/4 replies | ✅ 4/4 |
| 3 | Client → domain controller | `ping 192.168.100.10` | 4/4 replies | ✅ 4/4 |
| 4 | Client → web server | `ping 192.168.100.20` | 4/4 replies | ✅ 4/4 |
| 5 | Web server → gateway | `ping 192.168.100.1` | 4/4 replies | ✅ 4/4 |
| 6 | Web server → domain controller | `ping 192.168.100.10` | 4/4 replies | ✅ 4/4 |
| 7 | Web server → security workstation | `ping 192.168.100.50` | 4/4 replies | ✅ 4/4 |

All seven tests passed without packet loss.

---

## 5. Target Architecture — Design Only

![Packet Tracer topology of the target architecture](../../screenshots/phase-01/01-04-packet-tracer-topology-target-architecture.png)
*Figure 1.4 — Target architecture: servers and clients in separate, routed segments.*

The implemented lab uses one flat subnet, because 16 GB of host memory and a one-week
time budget do not justify additional routing infrastructure. In a production environment
this would not be acceptable: every client would sit in the same broadcast domain as every
server, with no point at which traffic could be filtered.

The target design therefore separates the two:

| Segment | VLAN | Subnet | Gateway | Members |
| --- | --- | --- | --- | --- |
| Server segment | 10 | 192.168.100.0/24 | 192.168.100.1 | `LAB-DC01`, `LAB-WEB01`, `LAB-SEC01` |
| Client segment | 20 | 192.168.110.0/24 | 192.168.110.1 | `LAB-CL01`, `LAB-CL02` |

```
interface GigabitEthernet0/0
 description Server segment - VLAN 10
 ip address 192.168.100.1 255.255.255.0
!
interface GigabitEthernet0/1
 description Client segment - VLAN 20
 ip address 192.168.110.1 255.255.255.0
```

### Consequences of segmentation

1. **Inter-VLAN routing is required.** Clients reach the servers only through
   `CORE-RT01`, which is exactly the point of the design — it creates a single place where
   access control lists can restrict which clients may reach which services.
2. **DHCP needs a relay.** A DHCP discover message is a broadcast, and a router does not
   forward broadcasts. Without help, clients in VLAN 20 would never reach the DHCP server
   in VLAN 10. The solution is `ip helper-address 192.168.100.10` on the client-facing
   interface, which forwards the request as a unicast to the domain controller.
3. **The addressing plan already accounts for it.** `192.168.110.0/24` is not invented for
   this diagram; it is the client subnet reserved in `docs/ip-plan/` from the start.

Recorded as [ADR-006](../../docs/architecture/decisions.md).

---

## 6. Result

- The addressing concept was validated in simulation before touching any virtual machine.
- Central DHCP with correct gateway and DNS options was proven to work end to end.
- Full Layer 2 and Layer 3 reachability was confirmed by seven documented tests.
- Both the implemented network and a production-grade target design exist as
  reproducible, version-controlled Packet Tracer files.

---

## 7. Lessons Learned

**A router port is not a switch port.** Both links to the router stayed down until
`no shutdown` was issued. Switch ports are enabled by default; router interfaces are not.
A down link in a diagram is not proof of a wiring error — it is usually an unconfigured
interface.

**The first client lease was `.101`, not `.100`.** Toggling the client between static and
DHCP sent a second request while the first address was still bound, so the server handed
out the next free address. The documentation records the address that was actually
assigned rather than the one that was expected — a document that contradicts reality is
worse than no document.

**The default Packet Tracer pool `serverPool` cannot be deleted.** It remains in the
configuration but is harmless: it is bound to a different subnet, and a DHCP server only
answers from a pool matching the subnet the request arrived on. The same scope-selection
logic applies to a real Windows DHCP server.

**A simulator has limits, and naming them is part of the work.** Lease duration and DHCP
option 015 could not be modelled, and the Linux bridge had to be split into two devices.
Documenting those gaps is more valuable than presenting a diagram that silently pretends
to be complete.

---

## 8. Next Phase

[Phase 2 — Virtual Infrastructure](../02-virtual-infrastructure/): transferring this
validated design onto the KVM host, verifying the virtual machines and their network
attachment.
