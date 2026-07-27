# IP Addressing Concept

> Part of **Phase 0 — Foundation**. This document defines the addressing scheme for the
> entire lab. It is written before any service is installed so that no address has to be
> changed later.

---

## 1. Subnet Overview

| Parameter | Value |
| --- | --- |
| Network address | `192.168.100.0/24` |
| Subnet mask | `255.255.255.0` |
| Usable host range | `192.168.100.1 - 192.168.100.254` |
| Default gateway | `192.168.100.1` (libvirt bridge `virbr1`, NAT) |
| Broadcast address | `192.168.100.255` |
| Total usable addresses | 254 |
| libvirt network name | `homelab` |
| Forward mode | NAT (outbound internet access for updates) |

---

## 2. Address Range Allocation

The subnet is divided into functional blocks. This makes the environment predictable
and mirrors how addressing is planned in real company networks.

| Range | Size | Purpose | Assignment |
| --- | --- | --- | --- |
| `.1 - .9` | 9 | Network infrastructure (gateway, future router) | static |
| `.10 - .49` | 40 | Servers | static |
| `.50 - .99` | 50 | Management and security systems | static |
| `.100 - .150` | 51 | DHCP scope, served by `LAB-DC01` | dynamic |
| `.151 - .254` | 104 | Reserved for future expansion | unused |

**Why reserve so much?** A production network is never planned at 100 percent capacity.
Leaving a documented reserve block is a deliberate design choice, not unused space.

---

## 3. Static Host Assignments

| IP Address | Hostname | Role | Operating System |
| --- | --- | --- | --- |
| `192.168.100.1` | *(host bridge)* | NAT gateway, upstream DNS forwarder | Ubuntu 24.04 / libvirt |
| `192.168.100.10` | `LAB-DC01` | Domain Controller, DNS, DHCP | Windows Server 2019 |
| `192.168.100.20` | `LAB-WEB01` | Web server (nginx), SSH | CentOS Stream 9 |
| `192.168.100.50` | `LAB-SEC01` | Security workstation | Parrot Security 7.2 |

### Dynamic Clients

| Hostname | Role | Address source |
| --- | --- | --- |
| `LAB-CL01` | Domain client | DHCP (`LAB-DC01`) |

---

## 4. DHCP Design

DHCP is provided by **`LAB-DC01`**, not by libvirt.

### Scope configuration

| Setting | Value |
| --- | --- |
| Scope name | `HOMELAB-Clients` |
| Start address | `192.168.100.100` |
| End address | `192.168.100.150` |
| Subnet mask | `255.255.255.0` |
| Lease duration | 8 days |

### Scope options

| Option | Name | Value |
| --- | --- | --- |
| 003 | Router | `192.168.100.1` |
| 006 | DNS Servers | `192.168.100.10` |
| 015 | DNS Domain Name | `corp.homelab.internal` |

> **Critical prerequisite:** the libvirt built-in DHCP server (dnsmasq) must be disabled
> on the `homelab` network before `LAB-DC01` starts serving DHCP. Two DHCP servers on the
> same broadcast domain cause non-deterministic client configuration and domain join
> failures. See `docs/architecture/decisions.md` (ADR-002).

---

## 5. DNS Design

| Layer | Server | Responsibility |
| --- | --- | --- |
| Internal | `LAB-DC01` (192.168.100.10) | Authoritative for `corp.homelab.internal`, AD service records |
| Forwarder | `192.168.100.1` | Resolves external names via the host |

**Rules applied in this lab:**

1. Every domain member uses **only** `192.168.100.10` as its DNS server.
2. No public DNS server (for example `8.8.8.8`) is configured on domain members.
   Windows may query a secondary DNS server that cannot resolve `_ldap._tcp` service
   records, which breaks authentication in a way that is hard to diagnose.
3. External resolution is handled by the forwarder chain, not by the clients.

### Zones

| Zone | Type | Notes |
| --- | --- | --- |
| `corp.homelab.internal` | Forward lookup, AD-integrated | Created automatically during promotion |
| `100.168.192.in-addr.arpa` | Reverse lookup | Created manually in Phase 3 |

---

## 6. Naming and Domain

| Parameter | Value |
| --- | --- |
| AD DNS domain | `corp.homelab.internal` |
| NetBIOS domain name | `HOMELAB` |
| Forest functional level | Windows Server 2016 |

Hostname rules are documented in `docs/architecture/naming-convention.md`.

---

## 7. Verification

The addressing concept is considered implemented when all of the following succeed.

### From the host

```bash
# libvirt network is active and no longer serves DHCP
virsh net-list --all
virsh net-dumpxml homelab | grep -c dhcp    # expected: 0

# Bridge holds the gateway address
ip -4 addr show virbr1
```

### From LAB-DC01 (PowerShell)

```powershell
ipconfig /all
Test-NetConnection 192.168.100.1 -InformationLevel Detailed
Resolve-DnsName corp.homelab.internal
nslookup -type=SRV _ldap._tcp.corp.homelab.internal 192.168.100.10
```

### From LAB-WEB01

```bash
ip -4 addr show
ip route
ping -c 3 192.168.100.10
nslookup lab-dc01.corp.homelab.internal 192.168.100.10
```

### From LAB-CL01

```powershell
ipconfig /all          # address must be inside 192.168.100.100 - .150
                       # DNS server must be 192.168.100.10 only
nltest /dsgetdc:corp.homelab.internal
```

---

## 8. Planned Expansion

The current implementation uses one flat subnet. The documented target architecture
introduces network segmentation, which is designed in Phase 1 and can be implemented later.

| Segment | Network | Purpose |
| --- | --- | --- |
| Server segment | `192.168.100.0/24` | Domain controller, application servers |
| Client segment | `192.168.110.0/24` | Workstations |
| Transfer / uplink | `192.168.99.0/30` | Router interconnect |

See `docs/architecture/decisions.md` (ADR-006) for the reasoning behind starting flat.
