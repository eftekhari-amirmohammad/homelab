# Phase 2 — Virtual Infrastructure

🇩🇪 [Deutsche Version](README.de.md)

---

## 1. Purpose

Phase 1 defined the network on paper and in Cisco Packet Tracer. Phase 2 turns that design into a running virtualisation platform: four virtual machines on a single physical host, connected to an isolated laboratory network, with a documented hardware baseline and a reproducible snapshot strategy.

The goals of this phase were:

- Document the host and hypervisor environment completely and verifiably
- Bring all four virtual machines to a known, clean baseline
- Remove every configuration conflict before installing domain services
- Establish a restore point strategy that makes every later phase repeatable

This phase deliberately contains **no** service installation. Its only product is a platform that can be trusted.

---

## 2. Host Environment

| Component | Value |
| --- | --- |
| Host OS | Ubuntu 24.04.4 LTS |
| CPU | Intel Core i7-10750H @ 2.60 GHz, 6 cores / 12 threads |
| Hardware virtualisation | Intel VT-x (enabled) |
| RAM | 16 GB total (15 GiB usable) |
| Storage (lab) | 1 TB HDD mounted at `/media/HDD` — 916 GB total, 513 GB free |
| libvirt | 10.0.0 |
| Hypervisor | QEMU/KVM 8.2.2 |
| GPU | NVIDIA GeForce GTX 1660 Ti (not passed through) |

The host is a laptop, which is the central constraint of this laboratory. Every design decision in this project is shaped by 16 GB of RAM and a single mechanical disk for virtual machine images.

### RAM budget

The four virtual machines request 13 GB of RAM in total, while the host itself needs roughly 4 GB. Running all four simultaneously is therefore not possible.

**Operating rule: a maximum of three virtual machines run at the same time.** This is not a defect but a documented capacity decision. In the domain-join phase, `LAB-DC01` and `LAB-CL01` are required; `LAB-SEC01` and `LAB-WEB01` remain switched off.

---

## 3. Virtual Machine Specifications

| Name | Role | Operating system | IP address | RAM | vCPU | NIC model | Disk bus |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `LAB-DC01` | Domain controller, DNS, DHCP | Windows Server 2019 (180-day evaluation) | 192.168.100.10 | 4 GB | 4 | e1000e | SATA |
| `LAB-WEB01` | Web and SSH server | CentOS Stream 9 | 192.168.100.20 | 2 GB | 2 | virtio | VirtIO |
| `LAB-CL01` | Domain client | Windows 10 22H2 | DHCP (192.168.100.100–150) | 4 GB | 2 | e1000e | SATA |
| `LAB-SEC01` | Security and analysis | Parrot Security 7.2 | 192.168.100.50 | 3 GB | 4 | virtio | VirtIO |

All four machines are attached to the isolated libvirt network `homelab` (bridge `virbr1`, 192.168.100.0/24, NAT forwarding).

### Machine names

The machines were originally named after their operating systems (for example `win2019`, `centos9`). They were renamed to role-based names using `virsh domrename`.

The reason is documented in ADR-004: an operating system can be replaced, a role cannot. `LAB-DC01` remains the domain controller even if the underlying Windows version changes, and the name is understandable to anybody reading a diagram or a log file.

> `virsh domrename` fails if snapshots already exist for the machine. Renaming therefore had to happen **before** the first snapshots were taken. This ordering constraint is easy to overlook and expensive to correct.

---

## 4. Storage Layout and Thin Provisioning

| Machine | Disk image | Virtual size | Actual size on disk |
| --- | --- | --- | --- |
| `LAB-DC01` | `/media/HDD/Virtual_OS/Windows_Server_2019/winserver.qcow2` | 40 GiB | 9.53 GiB |
| `LAB-WEB01` | `/media/HDD/Virtual_OS/CentOS/centos-stream9.qcow2` | 35 GiB | 5.95 GiB |
| `LAB-CL01` | `/media/HDD/Virtual_OS/Windows_10/win10.qcow2` | 40 GiB | 14.2 GiB |
| `LAB-SEC01` | `/media/HDD/Virtual_OS/Parrot/Parrot-security-7.2_amd64.qcow2` | 64 GiB | 12.5 GiB |
| **Total** | | **179 GiB** | **42.2 GiB** |

The virtual machines believe they own 179 GiB of storage, while only 42.2 GiB are actually occupied. This is the effect of the `qcow2` format, which allocates blocks only when they are first written.

**The operational consequence must be understood clearly:** the sum of all virtual disks may exceed the physical capacity of the host. As long as the guests do not fill their disks, nothing happens. When they do, the host runs out of space and **all** affected virtual machines can fail at once — not only the one that caused it. In production environments this is why free space on the storage layer is monitored independently of the guests.

All virtual machine images are stored on the 1 TB HDD rather than the 512 GB SSD (ADR-007). The SSD is shared with a Windows dual-boot installation and does not have the capacity. The price is disk performance, which is noticeable during the Active Directory installation and during snapshot operations.

---

## 5. Configuration Changes

### 5.1 Removing the built-in DHCP server

The libvirt network `homelab` originally contained a `<dhcp>` block, which means libvirt runs `dnsmasq` as a DHCP server for that bridge. In Phase 3 the domain controller becomes the DHCP server for the same subnet.

Two DHCP servers on one network segment produce a race condition: the client accepts whichever offer arrives first. When the wrong server answers, the client receives an address without the correct DNS server, and domain join fails with an error message that does not mention DHCP at all.

The `<dhcp>` block was therefore removed. Both states are preserved as evidence:

- `configs/network-homelab-before.xml`
- `configs/network-homelab-after.xml`

Verification: `virsh net-dumpxml homelab | grep -c dhcp` returns `0`.

> This is also a practical illustration of a rogue DHCP server. The libvirt `dnsmasq` process runs on Linux and does not ask Active Directory for authorisation, so the protection mechanism that Windows DHCP servers rely on would not have stopped it. See ADR-002.

### 5.2 Ejecting installation media

Three machines still had their installation ISO images attached, and `LAB-DC01` had the **wrong** image attached — a Windows 10 ISO on the future domain controller.

```bash
virsh change-media LAB-DC01  sdb --eject --config
virsh change-media LAB-WEB01 sda --eject --config
virsh change-media LAB-CL01  sdb --eject --config
```

An attached installation medium is not harmless. If the boot order changes or the disk becomes unbootable, the machine silently starts the installer again, and in the worst case an unattended installation overwrites the system.

### 5.3 Disk bus: SATA versus VirtIO

The two Windows machines use the emulated SATA controller, while the two Linux machines use VirtIO. VirtIO is the paravirtualised interface and is measurably faster because it avoids emulating real hardware.

**The Windows machines were deliberately not converted.** Windows loads only the storage driver it was installed with. Changing the controller of an existing installation produces an immediate `INACCESSIBLE_BOOT_DEVICE` stop error, because the operating system can no longer see its own system disk. A correct migration requires injecting the VirtIO driver into the running system first, then changing the bus.

The performance cost was accepted because a broken domain controller costs more than the performance gained. The correct procedure is documented so that the decision is visible as a decision and not as an oversight.

### 5.4 libvirt system versus session

While reading disk metadata, three of four images returned `Permission denied` for the normal user, and one did not:

```
qemu-img: Could not open '/media/HDD/Virtual_OS/Windows_Server_2019/winserver.qcow2': Permission denied
```

The cause is that libvirt has two independent instances:

| Connection | Runs as | Image ownership | Behaviour |
| --- | --- | --- | --- |
| `qemu:///system` | root / system service | `libvirt-qemu:kvm`, mode `600` | Machines start with the host, independent of a login session |
| `qemu:///session` | the logged-in user | user-owned, mode `644` | Machines belong to one user only |

Three machines belong to the system instance, `LAB-SEC01` belongs to the user session. The permission error is therefore **correct behaviour**, not a fault. Disk information is read with `sudo`.

The file ownership was deliberately **not** changed. Changing owner or mode of a libvirt-managed image breaks the security model and can prevent the machine from starting. Production environments use the system instance exclusively, because a service must not depend on a user being logged in.

---

## 6. Verification

| Test | Command | Expected result | Status |
| --- | --- | --- | --- |
| Hypervisor available | `virsh version` | libvirt 10.0.0, QEMU 8.2.2 | ✅ |
| Hardware virtualisation | `lscpu \| grep Virtualization` | `VT-x` | ✅ |
| Both networks active | `virsh net-list --all` | `homelab` and `default` active, autostart | ✅ |
| No DHCP in lab network | `virsh net-dumpxml homelab \| grep -c dhcp` | `0` | ✅ |
| Gateway address present | `ip -4 addr show virbr1` | `192.168.100.1/24` | ✅ |
| All machines defined | `virsh list --all` | four machines, persistent | ✅ |
| RAM and vCPU as planned | `virsh dominfo <vm>` | matches the table in section 3 | ✅ |
| Network interfaces attached | `virsh domiflist <vm>` | network `homelab`, MAC address stable | ✅ |
| No installation media attached | `virsh domblklist <vm>` | optical drives show `-` | ✅ |
| Disk sizes plausible | `sudo qemu-img info <disk>` | virtual and actual size as in section 4 | ✅ |

> `ip -4 addr show virbr1` reports `NO-CARRIER` and `state DOWN` while no virtual machine is attached to the bridge. This is expected: a bridge without an active port has no carrier. The address is configured and the bridge becomes operational as soon as the first machine starts.

---

## 7. Snapshot Strategy

A baseline snapshot was created for all four machines in the powered-off state:

```bash
virsh snapshot-create-as --domain <vm> \
  --name "phase02-baseline-clean" \
  --description "Clean baseline after renaming, media ejection and DHCP removal" \
  --atomic
```

The naming scheme is `phase<NN>-<state>`, for example `phase03-pre-adds` and `phase03-post-adds-working`. A snapshot name must answer two questions without further explanation: **when** in the project it was taken, and **what state** it preserves.

Two rules follow from this project's practice:

1. Take a snapshot **before** every operation that is difficult to undo.
2. Take a snapshot **after** every large change that succeeded and was verified. Reverting to a state before a twenty-minute installation costs that installation again.

Snapshots are taken in the `shutoff` state. An offline snapshot contains only the disk state and is therefore consistent; a snapshot of a running machine must also capture memory to be reliable, which costs space and time.

> A snapshot is **not** a backup. It lives inside the same `qcow2` file on the same physical disk. If that disk fails, the snapshots are lost with it. Backup is covered separately in Phase 8.

---

## 8. Result

The virtualisation platform is documented, verified and reproducible:

- Host and hypervisor versions recorded, hardware virtualisation confirmed
- Four machines with role-based names, correct resources and verified network attachment
- All installation media detached, no conflicting DHCP server
- Storage layout documented including thin provisioning and its risk
- Restore point present for every machine

The platform is ready for Active Directory in Phase 3.

---

## 9. Lessons Learned

**Read the output, do not merely run the command.** The wrong ISO image on the future domain controller, the mismatched disk buses and the two libvirt instances were all found by reading inventory output carefully, not by an error occurring. Errors found before they cause damage cost nothing.

**A permission error is not automatically a problem.** `Permission denied` on three of four disk images looked like a defect and was in fact the correct behaviour of the system libvirt instance. The wrong reaction — `chown` on the image files — would have created a real problem out of an imagined one.

**Some changes must happen in a specific order.** `virsh domrename` no longer works once snapshots exist. Whoever takes snapshots first has to delete them to rename a machine. Sequence is part of a plan, not a detail.

**Thin provisioning shifts risk instead of removing it.** 179 GiB promised against 42.2 GiB used is efficient until it is not. The failure mode is not gradual; it affects all machines on the same storage at once.

**Not every improvement should be applied.** VirtIO is faster than SATA, and switching an installed Windows system to it breaks the boot process. Knowing the better option and choosing the safer one is a decision that deserves to be written down, because otherwise it looks like ignorance.

---

## 10. Next Phase

**Phase 3 — Active Directory Domain Services:** installation and promotion of `LAB-DC01` to a domain controller for the new forest `corp.homelab.internal`, DNS with forward and reverse zones, an authorised DHCP scope, time synchronisation, and the organisational unit and group structure.

[⬅ Phase 1 — Network Design](../01-network-design/README.md) · [Back to project overview](../../README.md)
