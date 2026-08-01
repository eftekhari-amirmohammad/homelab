# Phase 4 — Windows Client: Domain Join and Authentication

🇩🇪 [Diese Seite auf Deutsch](README.de.md)

> Part of the [Homelab project](../../README.md) · Previous phase: [Phase 3 — Active Directory](../03-active-directory/README.md)

---

## 1. Purpose

Phase 3 built the domain. Phase 4 answers the only question that proves it works:

> **Can a client machine obtain its network configuration automatically, find the domain controller on its own, join the domain, and authenticate a regular user?**

A domain controller that no client ever joins is an untested assumption. This phase turns the assumption into evidence.

A second goal is deliberately built into this phase: it is the **final verification of Phase 2**. In Phase 2 the libvirt-internal DHCP server was removed so that the Windows DHCP server would be the only address provider on the segment. That change could not be proven at the time, because no DHCP client existed yet. It is proven here.

| Item | Value |
| --- | --- |
| Client VM | `LAB-CL01` |
| Operating system | Windows 10 Pro 22H2 |
| Address assignment | DHCP (`192.168.100.100–150`) |
| Domain | `corp.homelab.internal` |
| Test user | `HOMELAB\m.mueller` |
| Duration | ~60 minutes |

---

## 2. Starting Point

Before the first command was executed, the environment was in this state:

- `LAB-DC01` — shut down, holding snapshot `phase03-post-adds-working`
- `LAB-CL01` — shut down, Windows 10 installed, **workgroup** member, computer name `DESKTOP-8IHF3J4`
- libvirt network `homelab` — active, NAT, gateway `192.168.100.1`, **no DHCP block**
- No client had ever requested an address from the Windows DHCP scope

The boot order matters and was chosen deliberately: **domain controller first, client second**. A client that boots into a segment without a DHCP server falls back to an APIPA address (`169.254.x.x`) and would have to renew its lease manually afterwards, which destroys the evidence this phase is meant to collect.

---

## 3. Snapshot Strategy

Joining a domain is not a reversible setting. It creates a computer account in Active Directory, changes the machine SID relationship, and modifies local security policy. Leaving the domain afterwards does **not** restore the previous state.

For that reason a restore point was created **before** any change:

```bash
virsh snapshot-create-as --domain LAB-CL01 \
  --name phase04-pre-domain-join \
  --description "Windows 10 22H2, workgroup state, before joining corp.homelab.internal" \
  --atomic
```

The VM was powered off at that moment. A snapshot of a shut-down machine is the cleanest possible restore point, because no volatile memory state has to be captured or restored.

---

## 4. Verifying the DHCP Lease

### 4.1 Server-side state before the client booted

On `LAB-DC01`, the relevant services and the scope were verified first, and the lease list was confirmed to be **empty**:

```powershell
Get-Service NTDS, DNS, Netlogon, kdc, DHCPServer | Format-Table Name, Status -AutoSize
Get-DhcpServerv4Scope | Format-Table Name, ScopeId, StartRange, EndRange, State -AutoSize
Get-DhcpServerv4Lease -ScopeId 192.168.100.0
```

All five services reported `Running`, the scope `HOMELAB-Clients` reported `Active`, and no leases existed. The empty lease list is not a trivial detail: it means any address the client later receives can be traced unambiguously to this server.

### 4.2 First contact

Immediately after the first logon, Windows displayed its network location prompt. The prompt already carried the domain name:

![Windows network location prompt showing the DNS suffix corp.homelab.internal](../../screenshots/phase-04/04-01-network-prompt-domain-suffix.png)
*Figure 4.1 — Windows names the network after the DNS suffix received via DHCP option 015, before any manual configuration.*

This was the first evidence that the client had been served by the Windows DHCP server: no other device on the segment was capable of sending that suffix.

### 4.3 Full client configuration

```powershell
ipconfig /all | Select-String -Pattern "Description|Physical Address|DHCP Enabled|IPv4|Subnet|Default Gateway|DHCP Server|DNS Servers|Suffix|Lease"
```

![Filtered ipconfig output showing address, DHCP server and lease duration](../../screenshots/phase-04/04-02-client-dhcp-lease-verification.png)
*Figure 4.2 — The client received 192.168.100.100 from DHCP server 192.168.100.10, with an eight-day lease.*

| Field | Observed value | What it proves |
| --- | --- | --- |
| `DHCP Server` | `192.168.100.10` | The Windows DHCP server answered — not libvirt |
| `IPv4 Address` | `192.168.100.100` | First address of the configured range |
| `Lease Obtained` / `Expires` | 1 Aug → 9 Aug | Exactly 8 days = option 051 (`691200` s) |
| `Default Gateway` | `192.168.100.1` | Option 003 applied |
| `DNS Servers` | `192.168.100.10` only | Option 006 applied, no fallback resolver |
| `Connection-specific DNS Suffix` | `corp.homelab.internal` | Option 015 applied |
| `Physical Address` | `52-54-00-36-65-FA` | Matches the MAC documented in Phase 2 |

> **Phase 2 is hereby verified.** The removal of the libvirt DHCP block was correct and complete. Had it still been active, the client would have received an address from the `192.168.100.128–254` range with the gateway as its DNS server.

---

## 5. Joining the Domain

The join and the rename were attempted in a single operation, in order to save one reboot on the mechanical disk:

```powershell
Add-Computer -Domain corp.homelab.internal -NewName LAB-CL01 -Cred HOMELAB\Administrator -Force -Restart
```

This failed — partially.

![PowerShell error: the computer joined the domain but the rename failed](../../screenshots/phase-04/04-03-domain-join-rename-error.png)
*Figure 4.3 — The domain join succeeded; the simultaneous rename was rejected with "The directory service is busy".*

### 5.1 Analysis

The message is precise and worth reading carefully:

> `Computer 'DESKTOP-8IHF3J4' was successfully joined to the new domain 'corp.homelab.internal', but renaming it to 'LAB-CL01' failed ... The directory service is busy.`

Two separate operations were requested. The first succeeded, the second did not. `Add-Computer` performs them sequentially: it creates the computer account in Active Directory and then immediately requests a rename of that very object. The domain controller was still committing the freshly created account and rejected the second write.

### 5.2 Resolution

The rename was repeated as an independent operation and succeeded on the first attempt:

```powershell
Rename-Computer -NewName LAB-CL01 -DomainCred HOMELAB\Administrator -Force -Restart
```

The `-DomainCredential` parameter is required here and not before: once the machine is a domain member, its name is no longer a local setting but an object in Active Directory, and changing it requires write permission in the directory.

> **Conclusion drawn:** combining two dependent write operations to save one reboot was a poor trade. The attempted shortcut cost an additional reboot and an error, which is exactly what it was supposed to avoid. Dependent operations should be separated by default.

---

## 6. Domain Logon

After the reboot the logon screen offered `Other user` and reported the domain it would authenticate against:

![Windows logon screen showing that sign-in targets the HOMELAB domain](../../screenshots/phase-04/04-04-domain-logon-screen.png)
*Figure 4.4 — The client authenticates against HOMELAB, not against the local machine.*

Logon was performed with a **regular domain user**, not an administrator:

```
Username: HOMELAB\m.mueller
Password: Lab-User-2026!
```

The first logon took noticeably longer because Windows had to create a new user profile from `C:\Users\Default`. Subsequent logons of the same user load the existing profile and complete within seconds.

A successful logon is more than a convenience test. It is an **integration test of the entire Phase 3 work**: DNS resolution, Kerberos ticket issuance by the KDC, time synchronisation within the five-minute tolerance, and the user object itself all have to be correct simultaneously.

---

## 7. Authentication Verification

Three commands were executed as the logged-on standard user:

```powershell
hostname
whoami
nltest /dsgetdc:corp.homelab.internal
```

![Output of hostname, whoami and nltest confirming domain membership](../../screenshots/phase-04/04-05-domain-authentication-verification.png)
*Figure 4.5 — The machine identifies as LAB-CL01, the session identity comes from the domain, and the client located the domain controller by itself.*

| Command | Result | Interpretation |
| --- | --- | --- |
| `hostname` | `LAB-CL01` | The rename from section 5.2 took effect |
| `whoami` | `homelab\m.mueller` | Identity issued by the domain, not by the local SAM |
| `nltest` | `\\LAB-DC01.corp.homelab.internal` | Domain controller located |
| `nltest` | `Address: \\192.168.100.10` | Correct server, no other responder |
| `nltest` | `Dom Guid: 12fab4a3-…-3b6590a5ed0d` | Identical to the GUID recorded in Phase 3 |
| `nltest` | Flags `PDC GC KDC CLOSE_SITE FULL_SECRET` | Writable domain controller in the client's own site |

The prefix before the backslash in `whoami` is the shortest possible authentication test: it states where the identity came from. A local logon would have returned `lab-cl01\...`.

The `nltest` result deserves emphasis. The client was never told where the domain controller is. It discovered it through DNS SRV records — the mechanism configured in Phase 3.

> *Der Client findet den Domänencontroller über DNS-SRV-Einträge.*

All three commands ran **without administrative privileges**, which is itself part of the test: if domain discovery and authentication work for an ordinary user, the path used during every logon is confirmed to be open.

---

## 8. Least Privilege — An Observed Result

When an attempt was made to open PowerShell with `Run as administrator`, Windows demanded separate administrative credentials.

This was **not** a defect. `m.mueller` is a member of `GRP_GG_IT_Users` only, not of `Domain Admins` and not of the local `Administrators` group. The account therefore cannot elevate on its own workstation.

> *Ein normaler Domänenbenutzer kann auf dem Client keine administrativen Rechte erlangen — Nachweis des Prinzips der geringsten Rechte.*

This is the practical consequence of the group design from Phase 3. It was not configured separately in Phase 4; it follows from the fact that no unnecessary privileges were granted in the first place.

---

## 9. Placing the Computer Object

By default, a joined machine is placed in the container `CN=Computers`. This default was corrected:

```powershell
Get-ADComputer LAB-CL01 | Select-Object Name, DistinguishedName

Get-ADComputer LAB-CL01 | Move-ADObject -TargetPath "OU=Workstations,OU=Computers,OU=HOMELAB,DC=corp,DC=homelab,DC=internal"

Get-ADComputer LAB-CL01 | Select-Object Name, DistinguishedName
```

![Distinguished name before and after moving the computer object](../../screenshots/phase-04/04-06-computer-object-moved-to-ou.png)
*Figure 4.6 — The computer object was moved from the default container into the planned organisational unit.*

```
Before:  CN=LAB-CL01,CN=Computers,DC=corp,DC=homelab,DC=internal
After:   CN=LAB-CL01,OU=Workstations,OU=Computers,OU=HOMELAB,DC=corp,DC=homelab,DC=internal
```

### 9.1 Why this matters

`CN=Computers` is a **container**, not an organisational unit. Group Policy objects can only be linked to organisational units, sites, and domains — never to containers.

Leaving the machine in the default location would therefore make the entire OU structure created in Phase 3 useless for this client: no policy targeting `OU=Workstations` would ever apply to it.

> *Gruppenrichtlinien lassen sich nur mit Organisationseinheiten verknüpfen, nicht mit Containern.*

Moving a computer object does not affect domain membership. The machine's identity is bound to its SID and its machine account password, not to its position in the directory tree. The client does not notice the move.

The command was deliberately issued as a **before / action / after** sequence. `Move-ADObject` produces no output on success and no output on failure to match — the state has to be captured on both sides of the operation, or nothing has been proven.

---

## 10. Verification Matrix

| # | Test | Expected | Result |
| --- | --- | --- | --- |
| 1 | Client receives address automatically | from `.100–.150` | ✅ `192.168.100.100` |
| 2 | Address comes from the Windows server | `192.168.100.10` | ✅ |
| 3 | Lease duration | 8 days | ✅ 1 Aug → 9 Aug |
| 4 | Gateway via option 003 | `192.168.100.1` | ✅ |
| 5 | DNS via option 006, single entry | `192.168.100.10` | ✅ |
| 6 | DNS suffix via option 015 | `corp.homelab.internal` | ✅ |
| 7 | Domain join | successful | ✅ |
| 8 | Computer renamed | `LAB-CL01` | ✅ (second attempt) |
| 9 | Domain user logon | `HOMELAB\m.mueller` | ✅ |
| 10 | Session identity from domain | `homelab\m.mueller` | ✅ |
| 11 | Domain controller found via DNS | `LAB-DC01` | ✅ |
| 12 | Forest GUID matches Phase 3 | identical | ✅ |
| 13 | Standard user cannot elevate | denied | ✅ (intended) |
| 14 | Computer object in `OU=Workstations` | moved | ✅ |

---

## 11. Snapshots

| Name | Created | Purpose |
| --- | --- | --- |
| `phase02-baseline-clean` | 2026-07-27 | Untouched installation |
| `phase04-pre-domain-join` | 2026-08-01 00:03 | Workgroup state — for repeating the exercise |
| `phase04-post-domain-join` | 2026-08-01 01:05 | Known good state — starting point for later phases |

The client was shut down from within Windows rather than through `virsh`, so that the newly created user profile was written to disk completely. An interrupted profile write causes Windows to fall back to a temporary profile on the next logon.

Snapshot descriptions were written in full sentences on purpose. Six months later the name alone is not enough to reconstruct what state a snapshot represents.

---

## 12. Result

The client is a fully functional domain member:

- Automatic address configuration from the domain's own DHCP server
- Name resolution exclusively through the internal DNS server
- Domain controller discovery without any static configuration
- Kerberos authentication of a regular user account
- Computer object located in a policy-manageable organisational unit
- Standard user without local administrative rights

Together with Phase 3 this forms a small but complete and verified Windows domain.

---

## 13. Lessons Learned

1. **Do not combine dependent write operations to save time.** `Add-Computer` with `-NewName` failed on a busy directory service. Two separate commands would have been slower to type and faster to finish.

2. **An error message that contains the word "successfully" must be read to the end.** The join had worked; only the rename had not. Treating the whole operation as failed would have led to unnecessary and potentially harmful repetition.

3. **Defaults are decisions made by someone else.** `CN=Computers` is where Windows puts machines when nobody says otherwise. Accepting it would have silently disabled the OU structure for this client.

4. **Silent commands need external verification.** `Move-ADObject` reports nothing. State was therefore captured before and after the change.

5. **A refused privilege escalation can be a passing test.** The UAC prompt for `m.mueller` confirmed the group design rather than revealing a problem. Expected results should be defined before the test, not after it.

6. **One test can validate an earlier phase.** The DHCP lease was the first possible proof that the Phase 2 change was correct. Some decisions cannot be verified when they are made; they should be recorded and verified later rather than assumed.

---

## 14. Known Limitations

- Only one client was joined. Behaviour with concurrent logons or multiple machines was not tested.
- No Group Policy objects have been created yet; the OU structure is prepared but not yet used.
- The test user's password does not expire and does not have to be changed at first logon — a deliberate simplification for a lab environment, documented in Phase 3.
- Roaming profiles and folder redirection are out of scope; every profile is local to the machine.

---

## 15. Next Phase

[Phase 5 — Linux Server](../05-linux-server/): SSH hardening, nginx, firewalld, user and permission management on `LAB-WEB01`.
