# Enterprise Infrastructure HomeLab

[🇬🇧 English](README.md) | **🇩🇪 Deutsch**

![Status](https://img.shields.io/badge/Status-in%20Arbeit-yellow)
![Plattform](https://img.shields.io/badge/Plattform-KVM%20%2F%20QEMU-blue)
![OS](https://img.shields.io/badge/OS-Windows%20Server%202019%20%7C%20CentOS%20Stream%209-lightgrey)
![Doku](https://img.shields.io/badge/Doku-EN%20%7C%20DE-green)

Eine virtualisierte Laborumgebung, die die IT-Infrastruktur eines kleinen Unternehmens
nachbildet. Aufgebaut und dokumentiert als praxisnahes Portfolio-Projekt für eine
**Ausbildung zum Fachinformatiker für Systemintegration**.

---

## 1. Projektziel

Dieses Labor ist bewusst **kein** Versuch, ein großes Rechenzentrum aufzubauen.
Das Ziel ist eine realistische, vollständig dokumentierte Umgebung, die praktische
Kenntnisse in folgenden Bereichen nachweist:

- **Virtualisierung** — KVM/QEMU mit libvirt
- **Netzwerktechnik** — IP-Konzept, DHCP, DNS, Routing-Grundlagen
- **Windows-Administration** — Active Directory, Verzeichnisdienste, Benutzerverwaltung
- **Linux-Administration** — Dienste, Firewall, Berechtigungen, SSH
- **IT-Sicherheit (Grundlagen)** — Netzwerk-Scans und Dienstüberprüfung
- **Datensicherung** — Backup-Konzept mit getesteter Wiederherstellung
- **Technische Dokumentation** — nachvollziehbar, zweisprachig, versioniert

Jede Komponente in diesem Labor hat einen konkreten Zweck. Es wird nichts installiert,
nur um die Anzahl der Dienste zu erhöhen.

---

## 2. Architektur

Aktuelle Umsetzung — ein flaches, per NAT angebundenes virtuelles Netzwerk:

```
                        Internet
                            |
                  [ Ubuntu 24.04 Host ]
                    KVM / QEMU / libvirt
                    virbr1 - NAT-Gateway
                      192.168.100.1
                            |
                Virtuelles Netz "homelab"
                    192.168.100.0/24
                            |
     +--------------+--------------+--------------+
     |              |              |              |
  LAB-DC01      LAB-CL01       LAB-WEB01      LAB-SEC01
 WinSrv 2019    Windows 10    CentOS Str. 9     Parrot
 AD / DNS /      Domänen-      Web / SSH       Sicherheits-
    DHCP          client         Server         arbeitsplatz
 .100.10         DHCP          .100.20         .100.50
```

> Ein Diagramm der geplanten Erweiterung (segmentiertes Clientnetz mit Routing)
> ist in [Phase 1](phases/01-network-design/) dokumentiert.

---

## 3. Host-Umgebung

| Komponente | Spezifikation |
| --- | --- |
| Gerät | Lenovo Legion 5 |
| CPU | Intel Core i7-10750H (6 Kerne / 12 Threads) |
| RAM | 16 GB |
| Host-Betriebssystem | Ubuntu 24.04.4 LTS |
| Hypervisor | KVM / QEMU über Virtual Machine Manager (libvirt) |
| VM-Speicher | 1 TB HDD |

---

## 4. Virtuelle Maschinen

| Hostname | Rolle | Betriebssystem | IP-Adresse | RAM | vCPU |
| --- | --- | --- | --- | --- | --- |
| `LAB-DC01` | Domänencontroller, DNS, DHCP | Windows Server 2019 | 192.168.100.10 | 4 GB | 4 |
| `LAB-WEB01` | Web- & SSH-Server | CentOS Stream 9 | 192.168.100.20 | 2 GB | 2 |
| `LAB-CL01` | Domänenclient | Windows 10 | DHCP | 4 GB | 2 |
| `LAB-SEC01` | Sicherheitsarbeitsplatz | Parrot Security 7.2 | 192.168.100.50 | 3 GB | 4 |

Um die 16 GB Arbeitsspeicher des Hosts nicht zu überschreiten, laufen nie mehr als drei
virtuelle Maschinen gleichzeitig.

---

## 5. Netzwerk- und IP-Konzept

| Bereich | Verwendung |
| --- | --- |
| `192.168.100.1 - .9` | Netzwerkinfrastruktur (Gateway) |
| `192.168.100.10 - .49` | Server (statisch) |
| `192.168.100.50 - .99` | Verwaltungs- und Sicherheitssysteme (statisch) |
| `192.168.100.100 - .150` | DHCP-Bereich, bereitgestellt von `LAB-DC01` |
| `192.168.100.151 - .254` | Reserviert für zukünftige Erweiterungen |

- **AD-Domäne:** `corp.homelab.internal`
- **NetBIOS-Name:** `HOMELAB`
- **Interner DNS:** `LAB-DC01` (192.168.100.10), Weiterleitung an 192.168.100.1

Ausführliche Details und Entwurfsentscheidungen: [`docs/ip-plan/`](docs/ip-plan/) und
[`docs/architecture/decisions.md`](docs/architecture/decisions.md)

---

## 6. Projektphasen

| # | Phase | Schwerpunkt | Status |
| --- | --- | --- | --- |
| 0 | [Grundlagen](docs/architecture/) | Repository, IP-Konzept, Namenskonvention | ✅ Abgeschlossen |
| 1 | [Netzwerkplanung](phases/01-network-design/) | Topologie & Adressierung in Cisco Packet Tracer | ✅ Abgeschlossen |
| 2 | [Virtuelle Infrastruktur](phases/02-virtual-infrastructure/) | libvirt-Netzwerk, Bereitstellung der VMs | ✅ Abgeschlossen |
| 3 | [Active Directory](phases/03-active-directory/) | Active Directory, DNS, DHCP, OU-Struktur | ✅ Abgeschlossen |
| 4 | [Windows-Client](phases/04-windows-client/) | Domänenbeitritt, Authentifizierungstests | ✅ Abgeschlossen |
| 5 | [Linux-Server](phases/05-linux-server/) | SSH, nginx, firewalld, Berechtigungen | ✅ Abgeschlossen |
| 6 | [Sicherheitstests](phases/06-security-testing/) | Netzwerk-Scans, Diensterkennung | ✅ Abgeschlossen |
| 7 | [Dokumentation](phases/07-documentation/) | Fehlerbehebung, Lessons Learned | ⬜ Geplant |
| 8 | [Datensicherung](phases/08-backup-recovery/) | Backup-Konzept und getestete Wiederherstellung | ⬜ Geplant |

---

## 7. Repository-Struktur

```
homelab/
├── docs/
│   ├── architecture/       Namenskonventionen, Entscheidungsprotokoll
│   ├── ip-plan/            Adressierungskonzept
│   ├── lessons-learned/    Erkenntnisse pro Phase
│   └── troubleshooting/    Probleme und Lösungen
├── phases/                 Ein dokumentierter Ordner pro Phase
├── configs/                Exportierte Konfigurationsdateien
├── diagrams/               Netzwerk- und Architekturdiagramme
├── packet-tracer/          Cisco Packet Tracer Quelldateien
├── screenshots/            Screenshots zur Überprüfung pro Phase
└── assets/                 Weitere Bilder
```

---

## 8. Dokumentationsstandard

Jeder Konfigurationsschritt in diesem Repository beantwortet fünf Fragen:

1. **Zweck** — Was wurde installiert und warum?
2. **Konfiguration** — Wie wurde es konfiguriert?
3. **Überprüfung** — Wie wurde das Ergebnis getestet?
4. **Ergebnis** — Was wurde erreicht?
5. **Lessons Learned** — Was ist schiefgelaufen und wie wurde es gelöst?

---

## 9. Hinweis zu Zugangsdaten

> ⚠️ **Isolierte Laborumgebung.**
> Alle in diesem Repository genannten Zugangsdaten sind bewusst veröffentlichte
> Demo-Werte und werden ausschließlich innerhalb eines nicht routbaren virtuellen
> Netzwerks verwendet. Sie werden in keiner realen oder produktiven Umgebung genutzt.
> Windows Server 2019 wird unter der 180-tägigen Evaluierungslizenz eingesetzt.

---

## 10. Nicht im Projektumfang (vorerst)

Bewusst zurückgestellt, um das Projekt fokussiert und wartbar zu halten:
Kubernetes, Docker Swarm, Terraform, Ansible, Grafana, Prometheus, ELK Stack,
Jenkins, GitLab, SIEM, Cloud-Infrastruktur, Hochverfügbarkeit.

Diese Themen sind als mögliche Erweiterungsphasen dokumentiert.

---

## 11. Arbeitsweise

Die gesamte Infrastruktur dieses Labors wurde von mir auf eigener Hardware geplant,
installiert, konfiguriert und überprüft. Jeder Screenshot dokumentiert ein System, das ich
selbst aufgebaut und getestet habe, und jedes unter *Lessons Learned* aufgeführte Problem
ist eines, auf das ich gestoßen bin und das ich selbst gelöst habe.

KI-Assistenten wurden bewusst als **Lern- und Dokumentationswerkzeug** eingesetzt — um meine
Architekturentscheidungen kritisch zu hinterfragen, diese Dokumentation strukturiert und
konsistent zu halten und die englische sowie deutsche Formulierung zu verbessern. Sie haben
die praktische Arbeit nicht ersetzt: Keine Konfiguration in diesem Repository wurde
übernommen, ohne sie selbst durchzuführen, zu testen und zu verstehen.

Den effektiven und transparenten Einsatz moderner Werkzeuge betrachte ich als Teil
professioneller IT-Arbeit — deshalb wird er hier ausdrücklich benannt.

---

## Autor

**AmirMohammad Eftekhari**
Angehender Fachinformatiker für Systemintegration

- GitHub: [@eftekhari-amirmohammad](https://github.com/eftekhari-amirmohammad)
- Portfolio: [eftekhari-amirmohammad.github.io](https://eftekhari-amirmohammad.github.io)

## Lizenz

Veröffentlicht unter der [MIT-Lizenz](LICENSE).
