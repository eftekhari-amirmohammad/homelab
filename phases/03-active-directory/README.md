# Phase 3 — Active Directory Domain Services

🇩🇪 [Deutsche Version](README.de.md)

---

## 1. Purpose

Phase 3 turns `LAB-DC01` from a standalone Windows server into the central directory service of the laboratory. From this point on, the domain owns identity, name resolution and address assignment.

Deliverables of this phase:

- A new Active Directory forest `corp.homelab.internal` with the domain controller `LAB-DC01`
- DNS with a forward zone, a reverse zone and a forwarder to the internet
- A DHCP server authorised in Active Directory with an active scope for clients
- Reliable time synchronisation, which Kerberos depends on
- An organisational unit structure and a group concept that follow Microsoft's AGDLP model
- Two test users with complete attributes

---

## 2. Design Decisions

| Decision | Value | Reason |
| --- | --- | --- |
| Domain name | `corp.homelab.internal` | `.local` is reserved for mDNS by RFC 6762 and conflicts with AD name resolution. Single-label names such as `HOMELAB` cause DNS routing problems. A subdomain of a name that cannot collide with the public internet is the safe choice (ADR-003) |
| NetBIOS name | `HOMELAB` | Short, readable in logon dialogs as `HOMELAB\username`. The wizard proposed `CORP`, which carries no information |
| Forest and domain functional level | Windows Server 2016 | The highest level Windows Server 2019 supports. The functional level defines the minimum operating system version a domain controller may run |
| Server roles | DNS server, Global Catalog | Required for a first domain controller in a new forest |
| RODC | disabled | A read-only domain controller is intended for insecure branch locations, not for a single-controller environment |
| Static IP address | 192.168.100.10 | A domain controller must never receive its address by DHCP, because it serves DHCP itself |
| Own DNS server | 192.168.100.10 | The server points to itself, because it becomes the authoritative DNS server for the domain |
| Time zone | UTC | See section 6 |

---

## 3. Preparing the Server

Before the role installation, the host name and the network configuration were set with PowerShell:

```powershell
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Disabled
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.100.10 -PrefixLength 24 -DefaultGateway 192.168.100.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.100.10
Rename-Computer -NewName "LAB-DC01" -Force
Restart-Computer -Force
```

DHCP is disabled on the interface **before** the static address is assigned. Assigning a static address while DHCP is still active can leave the interface in an inconsistent state.

> Before this change, the interface held the address `169.254.237.170`. That is an APIPA address, which Windows assigns to itself when no DHCP server answers — the expected result after the libvirt DHCP server was removed in Phase 2. **An address in the 169.254.0.0/16 range always means one thing: the client did not reach a DHCP server.**

Both the computer name and the network configuration must be final before promotion. Renaming a domain controller afterwards is possible but affects DNS records, Kerberos service principal names and replication, and is therefore avoided.

---

## 4. Installing and Promoting the Domain Controller

The role was installed through Server Manager, deliberately using the graphical interface: the wizard pages document the configuration decisions better than a single PowerShell command can.

![Selecting the Active Directory Domain Services role in Server Manager](../../screenshots/phase-03/03-01-server-manager-add-adds-role.png)
*Figure 3.1 — Adding the Active Directory Domain Services role, including the management tools.*

![Role installation completed successfully](../../screenshots/phase-03/03-02-adds-role-installation-succeeded.png)
*Figure 3.2 — `Configuration required. Installation succeeded on LAB-DC01`. The role is installed, the server is not yet a domain controller.*

**Installing the role and promoting the server are two separate operations:**

| Step | What happens |
| --- | --- |
| Role installation | Files, services and management tools are copied to disk. The server remains a standalone server |
| Promotion | The forest and domain are created, the NTDS database is built, the server becomes a domain controller |

![Deployment configuration with a new forest](../../screenshots/phase-03/03-03-deployment-configuration-new-forest.png)
*Figure 3.3 — `Add a new forest` with the root domain name `corp.homelab.internal`.*

![Domain controller options](../../screenshots/phase-03/03-04-domain-controller-options.png)
*Figure 3.4 — Functional level Windows Server 2016, DNS server and Global Catalog enabled, RODC disabled, DSRM password set.*

The **Directory Services Restore Mode** password is not the administrator password. It grants access to a special boot mode in which the AD database can be repaired offline, while the directory service itself is not running — which is exactly the situation in which the normal domain logon is unavailable. A domain controller whose DSRM password is unknown cannot be repaired.

![Prerequisites check passed](../../screenshots/phase-03/03-05-prerequisites-check-passed.png)
*Figure 3.5 — `All prerequisite checks passed successfully`, with two expected warnings.*

### The DNS delegation warning

The wizard reports:

> A delegation for this DNS server cannot be created because the authoritative parent zone cannot be found.

The parent zone of `corp.homelab.internal` would be `homelab.internal`, which does not exist and is not supposed to exist. **The warning is correct and was ignored deliberately.** In a domain that is connected to public DNS, the same warning would be a real problem, because external name resolution into the domain would fail.

---

## 5. DNS Configuration

The promotion creates the forward zone `corp.homelab.internal` and the special zone `_msdcs.corp.homelab.internal` automatically. Two things were added:

```powershell
Add-DnsServerPrimaryZone -NetworkID "192.168.100.0/24" -ReplicationScope "Domain"
Add-DnsServerForwarder -IPAddress 192.168.100.1
ipconfig /registerdns
```

![DNS zones and forwarder](../../screenshots/phase-03/03-07-dns-zones-and-forwarder.png)
*Figure 3.6 — Forward zone, reverse zone `100.168.192.in-addr.arpa` and the forwarder to the gateway.*

**Reverse lookup zone** resolves in the opposite direction, from address to name. The zone name reverses the octets because DNS builds its hierarchy from right to left. Without it, management tools and log files show raw IP addresses instead of server names, which makes troubleshooting harder.

**Forwarder** answers every query that does not belong to the domain. The DNS server only knows `corp.homelab.internal`; everything else is passed to `192.168.100.1`, the libvirt NAT gateway. This is why clients need exactly one DNS server — the domain controller — and still resolve internet names.

**`-ReplicationScope Domain`** stores the zone inside the AD database instead of a text file, so a future second domain controller receives it through replication.

---

## 6. DHCP Configuration

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
Add-DhcpServerInDC -DnsName LAB-DC01.corp.homelab.internal -IPAddress 192.168.100.10

Add-DhcpServerv4Scope -Name "HOMELAB-Clients" -Description "Client range for lab workstations" `
  -StartRange 192.168.100.100 -EndRange 192.168.100.150 -SubnetMask 255.255.255.0 `
  -LeaseDuration 8.00:00:00 -State Active

Set-DhcpServerv4OptionValue -ScopeId 192.168.100.0 -Router 192.168.100.1 `
  -DnsServer 192.168.100.10 -DnsDomain "corp.homelab.internal"
```

![DHCP scope and options](../../screenshots/phase-03/03-08-dhcp-scope-and-options.png)
*Figure 3.7 — Active scope `HOMELAB-Clients` with the configured options.*

| Option | Name | Value | Purpose |
| --- | --- | --- | --- |
| 003 | Router | 192.168.100.1 | Default gateway |
| 006 | DNS Servers | 192.168.100.10 | The domain controller, and only the domain controller |
| 015 | DNS Domain Name | corp.homelab.internal | DNS suffix, so short names resolve |
| 051 | Lease | 691200 seconds | 8 days, added automatically from `-LeaseDuration` |

Option numbers are defined in RFC 2132 and are identical on every DHCP server implementation, whether Windows, Cisco or `isc-dhcp` on Linux.

**Option 006 is critical.** A second DNS server such as `8.8.8.8` in this list would cause clients to send domain SRV queries to a server that cannot answer them, and domain join fails with an error message that does not mention DNS.

### Authorisation in Active Directory

`Add-DhcpServerInDC` authorises the server. Windows DHCP servers query Active Directory before answering any client, and an unauthorised server starts its service but hands out no addresses.

This protects against a **rogue DHCP server** — a home router or laptop plugged into a company network that starts handing out wrong addresses and gateways. **The limit of this protection must be understood:** only domain-joined Windows servers ask the question. A router or a Linux host does not, and hands out addresses regardless. The mechanism protects the network from administrator mistakes, not from an attacker.

> This is exactly why the libvirt `dnsmasq` DHCP server had to be removed manually in Phase 2 (ADR-002). It runs on Linux and would never have asked for authorisation.

---

## 7. Time Synchronisation

```powershell
w32tm /config /manualpeerlist:"pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
```

![Time synchronisation and UTC time zone](../../screenshots/phase-03/03-09-time-synchronization-utc.png)
*Figure 3.8 — Time zone set to UTC after synchronisation.*

Kerberos writes a timestamp into every ticket. If the clocks of client and domain controller differ by more than **five minutes**, the ticket is rejected and authentication fails — with an error message that says nothing about time. The tolerance exists to prevent replay attacks.

### Time hierarchy in a domain

```
External NTP source
    └── Domain controller holding the PDC Emulator role   ← LAB-DC01
            └── Other domain controllers
                    └── Member servers and clients
```

Clients never take their time from the internet; they take it from the domain controller. Only the machine holding the PDC Emulator role synchronises externally, which is what `/reliable:yes` declares. The `nltest` output confirms the flags `PDC`, `TIMESERV` and `GTIMESERV`.

### Time zone: an observed anomaly

After a successful synchronisation, the server clock was one hour ahead of the host. The cause was not NTP: **Iran abolished daylight saving time in 2022, and this unpatched Windows Server 2019 still contained the obsolete rule.** The internal UTC clock was correct; only the conversion to local time was wrong.

Disabling dynamic daylight saving time through the registry did not help. Instead of patching the rule further, the server was set to **UTC**, which has no daylight saving time and therefore no rule that can expire.

This matches common practice in production environments: **servers run in UTC, workstations in local time.** Kerberos compares timestamps in UTC, so a client on a different display time is not affected. The correct long-term fix is to install Windows updates, which deliver current time zone data — the equivalent of the `tzdata` package on Linux.

---

## 8. Organisational Unit Structure

```
corp.homelab.internal
└── OU=HOMELAB
    ├── OU=Users
    │   ├── OU=IT
    │   ├── OU=Sales
    │   └── OU=Management
    ├── OU=Groups
    ├── OU=Computers
    │   ├── OU=Workstations
    │   └── OU=Servers
    └── OU=ServiceAccounts
```

All organisational units were created with `-ProtectedFromAccidentalDeletion $true`.

**Why a parent organisational unit instead of using the domain root?** Group Policy applied at the domain root also applies to the domain controllers themselves. A parent unit provides a clean application point.

**Why separate users and computers?** Group Policy has two independent halves, `Computer Configuration` and `User Configuration`. Separating the objects allows password policy to target computers and desktop policy to target users without interference.

**Why a separate unit for service accounts?** Service accounts need different rules: passwords that do not expire, and no interactive logon. Separating them from day one is a security decision, not cosmetics.

**Why deletion protection?** Deleting an organisational unit deletes everything inside it. In a company with several hundred users, one wrong click destroys a working day.

---

## 9. Group Concept: AGDLP

![Groups and nesting](../../screenshots/phase-03/03-10-ad-groups-agdlp-structure.png)
*Figure 3.9 — Global role groups, domain local resource groups, and the nesting between them.*

| Group | Scope | Meaning |
| --- | --- | --- |
| `GRP_GG_IT_Users` | Global | Role: IT department staff |
| `GRP_GG_Sales_Users` | Global | Role: Sales department staff |
| `GRP_GG_Management_Users` | Global | Role: Management staff |
| `GRP_DL_FileShare-Sales_Read` | Domain Local | Access: read on the Sales file share |
| `GRP_DL_FileShare-Sales_Modify` | Domain Local | Access: modify on the Sales file share |
| `GRP_DL_Servers_Admin` | Domain Local | Access: local administrator on member servers |

Microsoft's model is **A**ccount → **G**lobal group → **D**omain **L**ocal group → **P**ermission:

1. The user account goes into a **global group** representing an organisational role
2. The global group becomes a member of a **domain local group** representing one specific access right
3. The **permission** is granted only to the domain local group — never to a user, never to a global group

**Why this indirection pays off:** when an employee moves from Sales to Marketing, exactly one change is needed — the group membership. All previous access ends and all new access begins automatically. Without the model, every share and every resource has to be checked by hand, which in practice does not happen. The result is **permission creep**: users retain access they should no longer have, one of the most common negative findings in security audits.

| Type | Possible members | Where permissions can be granted |
| --- | --- | --- |
| Global | Accounts and global groups from the **same domain** | In **any domain** of the forest |
| Domain Local | Accounts and groups from **any domain** | Only in the **same domain** |

Global groups answer "who are you", domain local groups answer "what may you access". In a single-domain forest such as this laboratory the difference is not visible in practice — the model is nevertheless implemented correctly from the start, because in a multi-domain environment it is essential.

Naming convention: `GRP_<Scope>_<Resource>_<Permission>`.

---

## 10. User Accounts

![Users and group membership](../../screenshots/phase-03/03-11-ad-users-and-group-membership.png)
*Figure 3.10 — Two users with department and title, and the group membership of `m.mueller`.*

| Display name | Logon name | UPN | Organisational unit | Department | Group |
| --- | --- | --- | --- | --- | --- |
| Markus Mueller | `m.mueller` | `m.mueller@corp.homelab.internal` | `OU=IT,OU=Users,OU=HOMELAB` | IT | `GRP_GG_IT_Users` |
| Sabine Schmidt | `s.schmidt` | `s.schmidt@corp.homelab.internal` | `OU=Sales,OU=Users,OU=HOMELAB` | Sales | `GRP_GG_Sales_Users` |

**`SamAccountName` and `UserPrincipalName` are both required.** The first is the legacy logon name (`HOMELAB\m.mueller`, maximum 20 characters), the second the modern mail-style form. Some applications accept only one of the two, so an account without a UPN cannot sign in to certain services. The pattern `firstinitial.lastname` is the most common convention in German companies.

**`Department` and `Title` were filled in deliberately.** These attributes are not decoration: Group Policy filtering, dynamic group membership and directory synchronisation tools rely on them. An account with empty attributes is an incomplete account.

**Documented simplification:** the accounts were created with `-ChangePasswordAtLogon $false`. In a production environment this must be `$true`, because an administrator should not know a user's password. The laboratory value was chosen so that repeated domain join tests do not require a password change each time.

---

## 11. Verification

![Domain verification in PowerShell](../../screenshots/phase-03/03-06-domain-verification-powershell.png)
*Figure 3.11 — Domain, forest, services, domain controller locator and SRV record.*

| Test | Command | Result | Status |
| --- | --- | --- | --- |
| Domain created | `Get-ADDomain` | `corp.homelab.internal`, NetBIOS `HOMELAB`, `Windows2016Domain` | ✅ |
| Forest created | `Get-ADForest` | `Windows2016Forest`, Global Catalog `LAB-DC01` | ✅ |
| Critical services running | `Get-Service NTDS, DNS, Netlogon, kdc` | all four `Running` | ✅ |
| Domain controller locatable | `nltest /dsgetdc:corp.homelab.internal` | `LAB-DC01`, flags `PDC GC KDC TIMESERV WRITABLE` | ✅ |
| SRV records present | `nslookup -type=SRV _ldap._tcp.corp.homelab.internal` | port 389, `lab-dc01.corp.homelab.internal` | ✅ |
| Reverse resolution | `nslookup 192.168.100.10` | `LAB-DC01.corp.homelab.internal` | ✅ |
| Internet resolution through forwarder | `Resolve-DnsName www.google.com` | A and AAAA records returned | ✅ |
| DHCP authorised | `Get-DhcpServerInDC` | `192.168.100.10`, `lab-dc01.corp.homelab.internal` | ✅ |
| DHCP scope active | `Get-DhcpServerv4Scope` | `HOMELAB-Clients`, `State: Active` | ✅ |
| Time source external | `w32tm /query /status` | `Stratum: 3`, source `pool.ntp.org` | ✅ |
| Organisational units created | `Get-ADOrganizationalUnit -Filter *` | ten units in the designed hierarchy | ✅ |
| Group nesting correct | `Get-ADGroupMember GRP_DL_Servers_Admin` | `GRP_GG_IT_Users`, `objectClass: group` | ✅ |

The most important of these tests is the SRV record lookup. A Windows client never knows the address of a domain controller in advance; it asks DNS which host offers the LDAP service for the domain. **If those records are missing, the domain does not exist as far as clients are concerned, no matter how healthy the server is.**

---

## 12. Snapshots

| Name | State |
| --- | --- |
| `phase02-baseline-clean` | Clean baseline from Phase 2 |
| `phase03-pre-adds` | Static IP address and host name set, before the ADDS installation |
| `phase03-post-adds-working` | Verified working domain controller with DNS, DHCP, time synchronisation, organisational units, groups and users |

The second restore point exists for a specific reason. Reverting to `phase03-pre-adds` after a failed domain join would mean rebuilding the entire domain — twenty minutes of installation plus every organisational unit, group and user. Reverting to `phase03-post-adds-working` takes seconds.

---

## 13. Result

`corp.homelab.internal` is operational and verified. The domain controller provides directory service, name resolution, address assignment and time as a reference. The organisational unit and group structure follows a documented model rather than growing by accident, and the laboratory is ready for the domain join in Phase 4.

---

## 14. Lessons Learned

**Distinguish an error from a side effect.** `nslookup -type=SRV` returned the SRV record correctly but printed `Server: UnKnown` and `DNS request timed out` beforehand. The failure belonged to the reverse lookup of the DNS server's own name, not to the query being made. Reading which part of an output failed prevents hours of searching for a problem that does not exist.

**A warning that is understood may be ignored; a warning that is not understood may not.** The DNS delegation warning during promotion is correct for an internal domain and would be a serious problem in an internet-connected one. The difference between "I did not see the warning" and "I understood the warning" is the whole distance between the two.

**NTP fixes the clock, not the time zone.** Windows keeps time internally in UTC; the time zone is a display layer. A correct UTC clock with an obsolete daylight saving rule still shows the wrong time. Time zone rules are data, not program logic, and a server that is never updated eventually shows the wrong time even with perfect synchronisation.

**The first solution is not always the right one.** Disabling dynamic daylight saving time through the registry did not remove the obsolete rule. Switching to UTC solved the problem and matched industry practice at the same time. Recognising that a workaround is not working is more valuable than defending it.

**A simulator is not a server.** In Cisco Packet Tracer, the DHCP lease duration cannot be configured and option 015 does not exist. Comparing the model from Phase 1 with the real implementation shows exactly where the limits of the tool are — knowing those limits is part of using it competently.

**Structure is the part that is actually difficult.** Installing Active Directory is a wizard. Deciding why the organisational units are arranged this way, why the groups follow AGDLP, and why the domain name is not `.local` is the work that a wizard cannot do.

---

## 15. Next Phase

**Phase 4 — Domain Join and Client Management:** snapshot `phase04-pre-domain-join`, joining `LAB-CL01` to `corp.homelab.internal`, verification of the DHCP lease and the DNS suffix, and authentication tests with `m.mueller` including `nltest /dsgetdc:`.

[⬅ Phase 2 — Virtual Infrastructure](../02-virtual-infrastructure/README.md) · [Back to project overview](../../README.md)
