# Naming Conventions

> Part of **Phase 0 — Foundation**. Consistent naming is defined once, before the first
> service is installed, and applies to every object created in this lab.

---

## 1. Hostnames

### Pattern

```
LAB-<ROLE><NN>
```

| Element | Meaning | Rules |
| --- | --- | --- |
| `LAB` | Site / environment prefix | fixed |
| `<ROLE>` | Functional role code | 2 - 4 uppercase letters |
| `<NN>` | Sequence number | two digits, starting at `01` |

### Role codes

| Code | Role |
| --- | --- |
| `DC` | Domain Controller |
| `WEB` | Web server |
| `CL` | Client workstation |
| `SEC` | Security workstation |
| `FS` | File server (future) |
| `BKP` | Backup server (future) |

### Applied names

| Hostname | Length | Role |
| --- | --- | --- |
| `LAB-DC01` | 8 | Domain Controller, DNS, DHCP |
| `LAB-WEB01` | 9 | Web and SSH server |
| `LAB-CL01` | 8 | Domain client |
| `LAB-SEC01` | 9 | Security workstation |

**Constraint:** all names stay within the 15-character NetBIOS limit and contain only
letters, digits and hyphens, so they are valid in DNS as well.

---

## 2. libvirt Domain Names

The libvirt domain name matches the guest hostname exactly, so that `virsh list` and
the running operating system report the same identity.

| Original name | New name |
| --- | --- |
| `winserver` | `LAB-DC01` |
| `centos-stream9` | `LAB-WEB01` |
| `win10` | `LAB-CL01` |
| `Parrot_Sec` | `LAB-SEC01` |

---

## 3. Active Directory Objects

### Organizational Unit structure

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

Separating users, groups and computers into dedicated OUs allows Group Policy to be
linked precisely instead of applying it to the whole domain.

### User accounts

| Pattern | Example | Notes |
| --- | --- | --- |
| `<first initial>.<lastname>` | `m.mueller` | lowercase, no umlauts (`ue` instead of `ü`) |

| Display name | Logon name | Department |
| --- | --- | --- |
| Max Mueller | `m.mueller` | Sales |
| Sabine Schmidt | `s.schmidt` | IT |

### Security groups

```
GRP_<Scope>_<Resource>_<Permission>
```

| Example | Meaning |
| --- | --- |
| `GRP_HOMELAB_IT_Admins` | IT administrators |
| `GRP_HOMELAB_Share_Sales_RW` | Read/write on the Sales share |
| `GRP_HOMELAB_Share_Sales_RO` | Read-only on the Sales share |

---

## 4. Repository Files and Folders

| Object | Convention | Example |
| --- | --- | --- |
| Folders | lowercase, `kebab-case` | `phases/03-windows-server/` |
| Documents | lowercase, `kebab-case` | `naming-convention.md` |
| Conventional files | uppercase as required | `README.md`, `LICENSE` |
| German translations | `<name>.de.md` | `README.de.md` |

No spaces and no uppercase in paths. This avoids issues between Linux, Windows and Git.

---

## 5. Screenshots

### Pattern

```
screenshots/phase-<NN>/<NN>-<step>-<slug>.png
```

| Example |
| --- |
| `screenshots/phase-03/03-01-server-manager-add-roles.png` |
| `screenshots/phase-03/03-02-adds-installation-complete.png` |
| `screenshots/phase-04/04-01-domain-join-dialog.png` |

### Rules

1. PNG format only.
2. Crop to the relevant window; no host desktop, taskbar or browser tabs.
3. Verify before committing that no password entry, license key, personal e-mail
   address or public IP address is visible.
4. 4 to 8 screenshots per phase. More reduces readability.
5. Always reference with a caption:

   `![ADDS role installation](../../screenshots/phase-03/03-01-server-manager-add-roles.png)`
   `*Figure 3.1 — Adding the Active Directory Domain Services role.*`

---

## 6. Snapshots

### Pattern

```
phase<NN>-<state>-<subject>
```

| Snapshot name | Machine | Taken |
| --- | --- | --- |
| `phase02-baseline-clean` | all | before any configuration |
| `phase03-pre-adds` | `LAB-DC01` | before installing Active Directory |
| `phase03-post-adds-working` | `LAB-DC01` | after a verified working domain |
| `phase04-pre-domain-join` | `LAB-CL01` | before joining the domain |
| `phase05-pre-hardening` | `LAB-WEB01` | before firewall and SSH changes |

---

## 7. Git Commits

[Conventional Commits](https://www.conventionalcommits.org/) is used.

```
<type>(<scope>): <imperative summary>
```

| Type | Use for |
| --- | --- |
| `feat` | new artifact (topology file, script, configuration) |
| `docs` | documentation |
| `config` | exported configuration files |
| `fix` | corrections |
| `chore` | repository maintenance |

| Scope | Use for |
| --- | --- |
| `phase-01` ... `phase-08` | work belonging to a specific phase |
| *(omitted)* | repository-wide changes |

**Rules:** English, imperative mood (`add`, not `added`), lowercase summary, no trailing
period, at least two commits per phase so that progress is visible in the history.

### Examples

```
docs(phase-03): document Active Directory Domain Services installation
config(phase-05): add nginx and firewalld configuration files
chore: initialize repository structure and gitignore
```
