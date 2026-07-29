# Phase 2 — Virtuelle Infrastruktur

🇬🇧 [English version](README.md)

---

## 1. Zweck

Phase 1 hat das Netzwerk auf Papier und in Cisco Packet Tracer entworfen. Phase 2 setzt diesen Entwurf in eine laufende Virtualisierungsplattform um: vier virtuelle Maschinen auf einem physischen Host, verbunden mit einem isolierten Labornetz, mit dokumentierter Hardware-Grundlage und einer reproduzierbaren Snapshot-Strategie.

Ziele dieser Phase:

- Host- und Hypervisor-Umgebung vollständig und nachprüfbar dokumentieren
- Alle vier virtuellen Maschinen in einen bekannten, saubere Ausgangszustand bringen
- Jeden Konfigurationskonflikt vor der Installation der Verzeichnisdienste beseitigen
- Eine Sicherungspunkt-Strategie etablieren, die alle späteren Phasen wiederholbar macht

Diese Phase enthält bewusst **keine** Dienstinstallation. Ihr einziges Ergebnis ist eine Plattform, auf die man sich verlassen kann.

---

## 2. Host-Umgebung

| Komponente | Wert |
| --- | --- |
| Host-Betriebssystem | Ubuntu 24.04.4 LTS |
| CPU | Intel Core i7-10750H @ 2,60 GHz, 6 Kerne / 12 Threads |
| Hardware-Virtualisierung | Intel VT-x (aktiviert) |
| Arbeitsspeicher | 16 GB gesamt (15 GiB nutzbar) |
| Speicher (Labor) | 1-TB-HDD, eingebunden unter `/media/HDD` — 916 GB gesamt, 513 GB frei |
| libvirt | 10.0.0 |
| Hypervisor | QEMU/KVM 8.2.2 |
| Grafikkarte | NVIDIA GeForce GTX 1660 Ti (kein Passthrough) |

Der Host ist ein Notebook — das ist die zentrale Einschränkung dieses Labors. Jede Entwurfsentscheidung in diesem Projekt ist von 16 GB Arbeitsspeicher und einer einzelnen mechanischen Festplatte für die Abbilder geprägt.

### Speicherbudget

Die vier virtuellen Maschinen fordern zusammen 13 GB Arbeitsspeicher, der Host selbst benötigt etwa 4 GB. Alle vier gleichzeitig zu betreiben ist deshalb nicht möglich.

**Betriebsregel: maximal drei virtuelle Maschinen laufen gleichzeitig.** Das ist kein Mangel, sondern eine dokumentierte Kapazitätsentscheidung. In der Phase des Domänenbeitritts werden `LAB-DC01` und `LAB-CL01` benötigt; `LAB-SEC01` und `LAB-WEB01` bleiben ausgeschaltet.

---

## 3. Spezifikation der virtuellen Maschinen

| Name | Rolle | Betriebssystem | IP-Adresse | RAM | vCPU | NIC-Modell | Datenträgerbus |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `LAB-DC01` | Domänencontroller, DNS, DHCP | Windows Server 2019 (180-Tage-Evaluierung) | 192.168.100.10 | 4 GB | 4 | e1000e | SATA |
| `LAB-WEB01` | Web- und SSH-Server | CentOS Stream 9 | 192.168.100.20 | 2 GB | 2 | virtio | VirtIO |
| `LAB-CL01` | Domänenclient | Windows 10 22H2 | DHCP (192.168.100.100–150) | 4 GB | 2 | e1000e | SATA |
| `LAB-SEC01` | Sicherheit und Analyse | Parrot Security 7.2 | 192.168.100.50 | 3 GB | 4 | virtio | VirtIO |

Alle vier Maschinen sind mit dem isolierten libvirt-Netzwerk `homelab` verbunden (Bridge `virbr1`, 192.168.100.0/24, NAT-Weiterleitung).

### Maschinennamen

Die Maschinen waren ursprünglich nach ihren Betriebssystemen benannt (zum Beispiel `win2019`, `centos9`). Sie wurden mit `virsh domrename` auf rollenbasierte Namen umbenannt.

Die Begründung ist in ADR-004 festgehalten: ein Betriebssystem kann ersetzt werden, eine Rolle nicht. `LAB-DC01` bleibt der Domänencontroller, auch wenn sich die Windows-Version ändert, und der Name ist für jeden verständlich, der ein Diagramm oder eine Protokolldatei liest.

> `virsh domrename` schlägt fehl, wenn für die Maschine bereits Snapshots existieren. Das Umbenennen musste daher **vor** den ersten Snapshots erfolgen. Diese Reihenfolge wird leicht übersehen und ist teuer zu korrigieren.

---

## 4. Speicheraufteilung und Thin Provisioning

| Maschine | Abbilddatei | Virtuelle Größe | Tatsächliche Größe |
| --- | --- | --- | --- |
| `LAB-DC01` | `/media/HDD/Virtual_OS/Windows_Server_2019/winserver.qcow2` | 40 GiB | 9,53 GiB |
| `LAB-WEB01` | `/media/HDD/Virtual_OS/CentOS/centos-stream9.qcow2` | 35 GiB | 5,95 GiB |
| `LAB-CL01` | `/media/HDD/Virtual_OS/Windows_10/win10.qcow2` | 40 GiB | 14,2 GiB |
| `LAB-SEC01` | `/media/HDD/Virtual_OS/Parrot/Parrot-security-7.2_amd64.qcow2` | 64 GiB | 12,5 GiB |
| **Gesamt** | | **179 GiB** | **42,2 GiB** |

Die virtuellen Maschinen gehen davon aus, 179 GiB Speicher zu besitzen, tatsächlich belegt sind 42,2 GiB. Das ist die Wirkung des `qcow2`-Formats, das Blöcke erst beim ersten Schreibzugriff belegt.

**Die betriebliche Folge muss klar sein:** die Summe aller virtuellen Datenträger darf die physische Kapazität des Hosts übersteigen — eine bewusste Überbuchung des Speicherplatzes. Solange die Gastsysteme ihre Datenträger nicht füllen, passiert nichts. Wenn sie es tun, geht dem Host der Platz aus und **alle** betroffenen virtuellen Maschinen können gleichzeitig ausfallen, nicht nur die verursachende. In Produktionsumgebungen wird der freie Speicherplatz deshalb unabhängig von den Gastsystemen überwacht.

Alle Abbilder liegen auf der 1-TB-HDD und nicht auf der 512-GB-SSD (ADR-007). Die SSD wird mit einer Windows-Dual-Boot-Installation geteilt und hat nicht die Kapazität. Der Preis ist die Datenträgerleistung, die bei der Active-Directory-Installation und bei Snapshot-Vorgängen spürbar ist.

---

## 5. Konfigurationsänderungen

### 5.1 Entfernen des eingebauten DHCP-Servers

Das libvirt-Netzwerk `homelab` enthielt ursprünglich einen `<dhcp>`-Block, wodurch libvirt `dnsmasq` als DHCP-Server für diese Bridge betreibt. In Phase 3 wird der Domänencontroller zum DHCP-Server für dasselbe Subnetz.

Zwei DHCP-Server in einem Netzsegment erzeugen ein Wettrennen: der Client nimmt das Angebot an, das zuerst eintrifft. Antwortet der falsche Server, erhält der Client eine Adresse ohne den richtigen DNS-Server, und der Domänenbeitritt scheitert mit einer Fehlermeldung, die DHCP nicht einmal erwähnt.

Der `<dhcp>`-Block wurde daher entfernt. Beide Zustände sind als Nachweis erhalten:

- `configs/network-homelab-before.xml`
- `configs/network-homelab-after.xml`

Prüfung: `virsh net-dumpxml homelab | grep -c dhcp` gibt `0` zurück.

> Dies ist zugleich ein praktisches Beispiel für einen nicht autorisierten DHCP-Server. Der libvirt-`dnsmasq`-Prozess läuft unter Linux und fragt Active Directory nicht um Autorisierung — der Schutzmechanismus, auf den sich Windows-DHCP-Server verlassen, hätte ihn also nicht aufgehalten. Siehe ADR-002.

### 5.2 Auswerfen der Installationsmedien

Bei drei Maschinen waren noch die Installations-ISO-Abbilder eingebunden, und bei `LAB-DC01` war das **falsche** Abbild eingebunden — ein Windows-10-ISO auf dem künftigen Domänencontroller.

```bash
virsh change-media LAB-DC01  sdb --eject --config
virsh change-media LAB-WEB01 sda --eject --config
virsh change-media LAB-CL01  sdb --eject --config
```

Ein eingebundenes Installationsmedium ist nicht harmlos. Ändert sich die Startreihenfolge oder wird der Datenträger nicht mehr startfähig, startet die Maschine unbemerkt erneut das Installationsprogramm — im schlimmsten Fall überschreibt eine unbeaufsichtigte Installation das System.

### 5.3 Datenträgerbus: SATA gegenüber VirtIO

Die beiden Windows-Maschinen nutzen den emulierten SATA-Controller, die beiden Linux-Maschinen VirtIO. VirtIO ist die paravirtualisierte Schnittstelle und messbar schneller, weil keine echte Hardware emuliert werden muss.

**Die Windows-Maschinen wurden bewusst nicht umgestellt.** Windows lädt nur den Speichertreiber, mit dem es installiert wurde. Ein Controller-Wechsel an einer bestehenden Installation führt unmittelbar zum Stopp-Fehler `INACCESSIBLE_BOOT_DEVICE`, weil das Betriebssystem seinen eigenen Systemdatenträger nicht mehr sieht. Eine korrekte Migration erfordert, den VirtIO-Treiber zuerst in das laufende System einzubringen und danach den Bus zu wechseln.

Der Leistungsverlust wurde in Kauf genommen, weil ein defekter Domänencontroller mehr kostet als die gewonnene Leistung. Das korrekte Verfahren ist dokumentiert, damit die Entscheidung als Entscheidung erkennbar bleibt und nicht als Versäumnis.

### 5.4 libvirt: System gegenüber Session

Beim Auslesen der Datenträger-Metadaten meldeten drei von vier Abbildern für den normalen Benutzer `Permission denied`, eines nicht:

```
qemu-img: Could not open '/media/HDD/Virtual_OS/Windows_Server_2019/winserver.qcow2': Permission denied
```

Die Ursache ist, dass libvirt in zwei unabhängigen Instanzen existiert:

| Verbindung | Läuft als | Eigentümer der Abbilder | Verhalten |
| --- | --- | --- | --- |
| `qemu:///system` | root / Systemdienst | `libvirt-qemu:kvm`, Modus `600` | Maschinen starten mit dem Host, unabhängig von einer Anmeldesitzung |
| `qemu:///session` | angemeldeter Benutzer | benutzereigen, Modus `644` | Maschinen gehören nur einem Benutzer |

Drei Maschinen gehören zur System-Instanz, `LAB-SEC01` zur Benutzersitzung. Der Berechtigungsfehler ist damit **korrektes Verhalten** und kein Defekt. Datenträgerinformationen werden mit `sudo` gelesen.

Die Dateieigentümer wurden bewusst **nicht** geändert. Eigentümer oder Modus eines von libvirt verwalteten Abbilds zu ändern, bricht das Sicherheitsmodell und kann den Start der Maschine verhindern. Produktionsumgebungen verwenden ausschließlich die System-Instanz, weil ein Dienst nicht davon abhängen darf, dass ein Benutzer angemeldet ist.

---

## 6. Überprüfung

| Test | Befehl | Erwartetes Ergebnis | Status |
| --- | --- | --- | --- |
| Hypervisor verfügbar | `virsh version` | libvirt 10.0.0, QEMU 8.2.2 | ✅ |
| Hardware-Virtualisierung | `lscpu \| grep Virtualization` | `VT-x` | ✅ |
| Beide Netzwerke aktiv | `virsh net-list --all` | `homelab` und `default` aktiv, Autostart | ✅ |
| Kein DHCP im Labornetz | `virsh net-dumpxml homelab \| grep -c dhcp` | `0` | ✅ |
| Gateway-Adresse vorhanden | `ip -4 addr show virbr1` | `192.168.100.1/24` | ✅ |
| Alle Maschinen definiert | `virsh list --all` | vier Maschinen, persistent | ✅ |
| RAM und vCPU wie geplant | `virsh dominfo <vm>` | entspricht der Tabelle in Abschnitt 3 | ✅ |
| Netzwerkschnittstellen verbunden | `virsh domiflist <vm>` | Netzwerk `homelab`, MAC-Adresse stabil | ✅ |
| Keine Installationsmedien eingebunden | `virsh domblklist <vm>` | optische Laufwerke zeigen `-` | ✅ |
| Datenträgergrößen plausibel | `sudo qemu-img info <disk>` | virtuelle und tatsächliche Größe wie in Abschnitt 4 | ✅ |

> `ip -4 addr show virbr1` meldet `NO-CARRIER` und `state DOWN`, solange keine virtuelle Maschine mit der Bridge verbunden ist. Das ist erwartet: eine Bridge ohne aktiven Port hat kein Trägersignal. Die Adresse ist konfiguriert, und die Bridge wird betriebsbereit, sobald die erste Maschine startet.

---

## 7. Snapshot-Strategie

Für alle vier Maschinen wurde im ausgeschalteten Zustand ein Basis-Snapshot erstellt:

```bash
virsh snapshot-create-as --domain <vm> \
  --name "phase02-baseline-clean" \
  --description "Clean baseline after renaming, media ejection and DHCP removal" \
  --atomic
```

Das Namensschema ist `phase<NN>-<Zustand>`, zum Beispiel `phase03-pre-adds` und `phase03-post-adds-working`. Ein Snapshot-Name muss zwei Fragen ohne weitere Erklärung beantworten: **wann** im Projekt er entstand und **welchen Zustand** er sichert.

Aus der Praxis dieses Projekts folgen zwei Regeln:

1. Vor jedem schwer umkehrbaren Vorgang einen Snapshot erstellen.
2. Nach jeder großen Änderung, die erfolgreich war und geprüft wurde, ebenfalls einen Snapshot erstellen. Eine Rückkehr in den Zustand vor einer zwanzigminütigen Installation kostet diese Installation erneut.

Snapshots werden im Zustand `shutoff` erstellt. Ein Offline-Snapshot enthält nur den Datenträgerzustand und ist damit konsistent; ein Snapshot einer laufenden Maschine muss auch den Arbeitsspeicher erfassen, was Platz und Zeit kostet.

> Ein Snapshot ist **keine** Datensicherung. Er liegt in derselben `qcow2`-Datei auf derselben physischen Festplatte. Fällt diese Festplatte aus, sind die Snapshots mit ihr verloren. Die Datensicherung wird in Phase 8 gesondert behandelt.

---

## 8. Ergebnis

Die Virtualisierungsplattform ist dokumentiert, geprüft und reproduzierbar:

- Host- und Hypervisor-Versionen erfasst, Hardware-Virtualisierung bestätigt
- Vier Maschinen mit rollenbasierten Namen, korrekten Ressourcen und geprüfter Netzanbindung
- Alle Installationsmedien ausgeworfen, kein konkurrierender DHCP-Server
- Speicheraufteilung dokumentiert, einschließlich Thin Provisioning und dessen Risiko
- Sicherungspunkt für jede Maschine vorhanden

Die Plattform ist bereit für Active Directory in Phase 3.

---

## 9. Gelernte Lektionen

**Die Ausgabe lesen, nicht nur den Befehl ausführen.** Das falsche ISO-Abbild auf dem künftigen Domänencontroller, die unterschiedlichen Datenträgerbusse und die zwei libvirt-Instanzen wurden durch sorgfältiges Lesen der Inventarausgabe gefunden, nicht durch einen auftretenden Fehler. Fehler, die vor dem Schaden gefunden werden, kosten nichts.

**Ein Berechtigungsfehler ist nicht automatisch ein Problem.** `Permission denied` bei drei von vier Abbildern sah nach einem Defekt aus und war tatsächlich das korrekte Verhalten der System-Instanz von libvirt. Die falsche Reaktion — `chown` auf die Abbilddateien — hätte aus einem eingebildeten Problem ein echtes gemacht.

**Manche Änderungen erfordern eine bestimmte Reihenfolge.** `virsh domrename` funktioniert nicht mehr, sobald Snapshots existieren. Wer zuerst Snapshots erstellt, muss sie löschen, um eine Maschine umzubenennen. Reihenfolge ist Teil eines Plans, nicht ein Detail.

**Thin Provisioning verlagert das Risiko, es beseitigt es nicht.** 179 GiB zugesagt gegen 42,2 GiB belegt ist effizient — bis es das nicht mehr ist. Der Ausfall verläuft nicht allmählich, sondern trifft alle Maschinen auf demselben Speicher gleichzeitig.

**Nicht jede Verbesserung sollte umgesetzt werden.** VirtIO ist schneller als SATA, und die Umstellung eines installierten Windows-Systems zerstört den Startvorgang. Die bessere Option zu kennen und die sicherere zu wählen, ist eine Entscheidung, die aufgeschrieben gehört — sonst sieht sie wie Unwissen aus.

---

## 10. Nächste Phase

**Phase 3 — Active Directory Domain Services:** Installation und Hochstufung von `LAB-DC01` zum Domänencontroller für die neue Struktur `corp.homelab.internal`, DNS mit Forward- und Reverse-Zone, autorisierter DHCP-Bereich, Zeitsynchronisation sowie die Struktur der Organisationseinheiten und Gruppen.

[⬅ Phase 1 — Netzwerkplanung](../01-network-design/README.de.md) · [Zurück zur Projektübersicht](../../README.de.md)
