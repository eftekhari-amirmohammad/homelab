# Phase 5 — Linux-Server: Dienste und Härtung

> Teil des [Homelab-Projekts](../../README.de.md) · Englische Fassung: [README.md](README.md)

## 1. Ziel

In dieser Phase wird die virtuelle Maschine mit CentOS Stream 9 zu einem produktiven Mitglied des Domänennetzes: ein Webserver mit statischer Adresse, korrekter Namensauflösung, gehärtetem SSH-Zugang und minimaler Angriffsfläche in der Firewall.

Das Ziel ist nicht „ein laufender Webserver“. Ein Webserver ist mit einem Befehl installiert. Das Ziel ist die Disziplin darum herum: ein dokumentierter Adressplan, eine geprüfte Kette der Namensauflösung, ein Berechtigungskonzept mit genau einem administrativen Konto und ein Nachweis für jede Behauptung.

## 2. Ausgangslage

| Punkt | Wert |
|---|---|
| VM-Name | `LAB-WEB01` |
| Betriebssystem | CentOS Stream 9 (Kernel `5.14.0-677.el9.x86_64`) |
| Installierte Pakete | 1180 |
| Hostname | nicht gesetzt (`localhost.localdomain`) |
| Netzwerk | `enp1s0`, getrennt, keine Adresse |
| Benutzer | `centos_user0` (ohne sudo-Rechte), `root` |
| Standardziel | `graphical.target` |
| Webserver | nicht installiert |

Die Maschine wurde in Phase 2 erstellt und in dieser Phase zum ersten Mal gestartet. Sie war nicht Mitglied der Domäne und hatte keine funktionierende Netzwerkverbindung.

## 3. Sicherungsstrategie

Vor der ersten Änderung wurde ein Sicherungspunkt erstellt:

```bash
virsh snapshot-create-as LAB-WEB01 phase05-pre-hardening \
  "Clean CentOS Stream 9 before network, service and hardening changes"
```

Härtung ist die eine Kategorie von Änderungen, die den Administrator selbst aussperren kann. Ein vorher erstellter Sicherungspunkt macht aus einem irreversiblen Fehler eine Rücksetzung von fünf Minuten.

## 4. Hostname und statische Adresse

Der Hostname wurde nach Linux-Konvention klein geschrieben gesetzt, während der libvirt-Name groß geschrieben bleibt:

```bash
hostnamectl set-hostname lab-web01
```

Für `enp1s0` war bereits ein NetworkManager-Profil vorhanden. Es wurde **geändert**, nicht ersetzt — ein zweites Profil für dieselbe Schnittstelle ist eine häufige Ursache für nicht vorhersagbares Verhalten nach einem Neustart.

```bash
nmcli connection modify enp1s0 \
  ipv4.method manual \
  ipv4.addresses 192.168.100.20/24 \
  ipv4.gateway 192.168.100.1 \
  ipv4.dns 192.168.100.10 \
  ipv4.dns-search corp.homelab.internal \
  ipv6.method disabled \
  connection.autoconnect yes
nmcli connection up enp1s0
```

IPv6 wurde bewusst deaktiviert: Das Labor ist eine reine IPv4-Umgebung, und ein halb konfigurierter zweiter Protokollstapel erzeugt Fehler, die schwer zu diagnostizieren sind.

Die Verbindung wurde anschließend von unten nach oben geprüft — Leitung, Adresse, Gateway, Nachbar, Route, Name — statt zuerst die Anwendung zu testen.

| Schicht | Test | Ergebnis |
|---|---|---|
| Adresse | `ip -4 addr show enp1s0` | `192.168.100.20/24` |
| Gateway | `ping 192.168.100.1` | 0 % Verlust, `ttl=64` |
| Nachbar | `ping 192.168.100.10` | 0 % Verlust, `ttl=128` |
| Internet | `ping 8.8.8.8` | 0 % Verlust, `ttl=111` |
| Interner Name | `getent hosts dc01.corp.homelab.internal` | aufgelöst |

Die TTL-Werte sind für sich genommen aussagekräftig: 64 kennzeichnet einen Linux-Host, 128 einen Windows-Host. Dieselbe Erkennung nutzt `nmap -O` in Phase 6.

## 5. Fehlerbehebung: Externe Namensauflösung

Die Paketinstallation schlug sofort fehl:

```
Errors during downloading metadata for repository 'baseos':
- Curl error (6): Couldn't resolve host name for https://mirrors.centos.org/...
```

Interne Namen wurden aufgelöst, die Internetverbindung über IP-Adressen funktionierte, und nur externe Namen scheiterten. Der Fehler war damit auf die Namensauflösung externer Zonen eingegrenzt.

```bash
dig A mirrors.centos.org @192.168.100.10 | grep -E "status:|ANSWER:"
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51839
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
```

`NOERROR` mit zwei Antworten sieht nach Erfolg aus. Die Prüfung der Eintragstypen zeigte, dass beide Antworten `CNAME`-Einträge waren und **kein A-Eintrag** enthalten war. Dieselbe Abfrage an öffentliche Resolver lieferte die vollständige CNAME-Kette **und neun IPv4-Adressen**.

Der Domänencontroller war in Phase 3 so konfiguriert worden, dass er an das libvirt-Gateway `192.168.100.1` weiterleitet, das wiederum den Resolver des Anbieters nutzt. Dieser entfernt für bestimmte Ziele die A-Einträge. Der Fehler lag damit weder in CentOS noch im Domänencontroller, sondern im Weiterleitungsziel.

Die Korrektur gehört in die Schicht, die für das Problem zuständig ist — die Namensauflösung für externe Adressen ist Aufgabe des Domänencontrollers, nicht des Clients:

```powershell
Set-DnsServerForwarder -IPAddress 8.8.8.8,1.1.1.1 -PassThru
Clear-DnsServerCache -Force
```

Nach der Änderung lieferte dieselbe Abfrage alle neun IPv4-Adressen, und die Paketinstallation war erfolgreich. Siehe [ADR-011](../../docs/architecture/decisions.md).

## 6. Webserver

```bash
dnf -y install nginx
systemctl enable --now nginx
```

Der GPG-Schlüssel `0x8483C65D` des Repositorys wurde bei der Installation importiert und sein Fingerabdruck bestätigt. Paketsignaturen sind die Lieferkettenkontrolle der Paketverwaltung und keine Formalität.

`enable --now` startet den Dienst sofort und registriert ihn für jeden weiteren Systemstart. Ein Dienst, der nur bis zum nächsten Neustart läuft, ist kein Dienst.

Eine deutschsprachige Startseite wurde nach `/usr/share/nginx/html/index.html` geschrieben, direkt im Wurzelverzeichnis erzeugt, damit sie die korrekte SELinux-Kennzeichnung erbt. SELinux blieb während der gesamten Phase aktiv.

Der Dienst wurde über drei unabhängige Sichten geprüft, denn „installiert“, „läuft“ und „erreichbar“ sind drei verschiedene Aussagen:

```bash
systemctl status nginx      # active (running), enabled
ss -tlnp | grep nginx       # 0.0.0.0:80 und [::]:80
curl -s -o /dev/null -w "%{http_code}\n" http://localhost   # 200
```

![nginx-Startseite im Browser](../../screenshots/phase-05/05-01-nginx-browser-access.png)
*Abbildung 5.1 — Der Webserver liefert seine Startseite an einen Browser auf dem Hostsystem.*

## 7. Firewall

Eine Anfrage vom Hostsystem wurde abgewiesen, während die lokale Anfrage erfolgreich war — das erwartete Verhalten eines korrekt arbeitenden Paketfilters.

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

firewalld führt eine Laufzeitkonfiguration und eine dauerhafte Konfiguration. `--permanent` allein ändert nur die gespeicherte Konfiguration, `--reload` aktiviert sie. Dieselbe Zweiteilung gibt es in libvirt mit `--live` und `--config`.

Die Wirkung wurde als Paar aus Vorher und Nachher dokumentiert. Der Zustand „vorher“ wurde authentisch wiederhergestellt, indem die Regel nur aus der Laufzeitkonfiguration entfernt wurde, sodass `--reload` sie anschließend aus der dauerhaften Konfiguration zurückholte:

```
BEFORE: 000    curl: (7) Failed to connect ... after 0 ms
AFTER:  200
```

Das sofortige Scheitern nach 0 ms kennzeichnet eine `REJECT`-Regel und kein stilles `DROP`. In Phase 6 erscheint derselbe Unterschied in der Ausgabe von `nmap` als `closed` gegenüber `filtered`.

![Firewall-Prüfung vorher und nachher](../../screenshots/phase-05/05-02-firewalld-http-verification.png)
*Abbildung 5.2 — Externer Zugriff vor und nach der Freigabe des HTTP-Dienstes in firewalld.*

Zwei standardmäßig geöffnete Dienste wurden entfernt:

| Dienst | Grund für die Entfernung |
|---|---|
| `cockpit` | Web-Verwaltungskonsole, in diesem Labor nie aktiviert |
| `dhcpv6-client` | IPv6 ist auf diesem Host deaktiviert |

Die resultierende Zone enthält genau zwei Dienste, und beide lassen sich in einem Satz begründen: `http` ist der Zweck der Maschine, `ssh` ihr einziger Weg zur Fernwartung.

## 8. Administrativer Benutzer

Das bei der Installation angelegte Konto hatte keine administrativen Rechte:

```
centos_user0 is not in the sudoers file. This incident will be reported.
```

Statt dem vorhandenen Konto Rechte zu geben, wurde ein eigenes administratives Konto angelegt:

```bash
useradd -m -G wheel -c "Lab Administrator" labadmin
passwd labadmin
```

Bei RHEL-Ableitungen entspricht die Gruppe `wheel` der Gruppe `sudo` auf Debian-Systemen. Die Rechteausweitung wurde mit dem billigsten harmlosen Befehl geprüft:

```bash
sudo whoami   # root
```

Der Fernzugriff als `labadmin` wurde **vor** jeder Härtungsmaßnahme geprüft. Einen Zugangsweg zu schließen, bevor ein anderer nachweislich funktioniert — und zwar über denselben Kanal, der später genutzt wird — ist die übliche Art, sich selbst auszusperren.

![Fernwartung über SSH](../../screenshots/phase-05/05-03-ssh-remote-administration.png)
*Abbildung 5.3 — SSH-Sitzung vom Ubuntu-Host zum Webserver als `labadmin`.*

## 9. Härtung des SSH-Dienstes

Die Härtung wurde als Drop-in-Datei umgesetzt und nicht durch Änderung der Herstellerkonfiguration. Eine Drop-in-Datei lässt sich mit einem Befehl entfernen, überlebt Paketaktualisierungen und ist ein sauberes Artefakt für das Repository.

| Einstellung | Vorher | Nachher | Begründung |
|---|---|---|---|
| `PermitRootLogin` | `without-password` | `no` | Keine direkte Root-Anmeldung; Handlungen bleiben zuordenbar |
| `MaxAuthTries` | `6` | `3` | Weniger Versuche je Verbindung |
| `LoginGraceTime` | `120` | `30` | Kürzeres Zeitfenster für halb offene Verbindungen |
| `X11Forwarding` | `yes` | `no` | Auf einem Server nicht erforderlich |
| `ClientAliveInterval` | `0` | `300` | Erkennt tote Sitzungen |
| `ClientAliveCountMax` | `3` | `2` | Schnelleres Aufräumen |
| `AllowUsers` | — | `labadmin` | Weiße Liste statt schwarzer Liste |

Die Konfiguration wurde vor der Aktivierung geprüft. `sshd_config` ist die eine Datei, deren Syntaxfehler ein entferntes System dauerhaft unerreichbar machen kann:

```bash
sshd -t && systemctl reload sshd
```

Es wurde `reload` statt `restart` verwendet, damit bestehende Sitzungen die Änderung überleben und nur neue Verbindungen nach den neuen Regeln bewertet werden.

![Wirksame SSH-Konfiguration](../../screenshots/phase-05/05-04-ssh-hardening-effective-config.png)
*Abbildung 5.4 — Reihenfolge der Drop-in-Dateien und die wirksame SSH-Konfiguration nach der Härtung.*

## 10. Fehlerbehebung: Reihenfolge der Drop-in-Dateien und Verbindungsstrafen

Während der Härtung traten zwei lehrreiche Probleme auf.

**Eine Einstellung wurde stillschweigend ignoriert.** Nach dem ersten Neuladen waren sechs von sieben Einstellungen aktiv, `X11Forwarding` stand jedoch weiterhin auf `yes`. Der Syntaxtest war erfolgreich, das Neuladen ebenfalls, und nirgends erschien eine Fehlermeldung.

```
/etc/ssh/sshd_config.d/50-redhat.conf:17   X11Forwarding yes
/etc/ssh/sshd_config.d/99-hardening.conf:5 X11Forwarding no
```

OpenSSH verwendet den **ersten** gefundenen Wert eines Schlüsselworts, nicht den letzten. Drop-in-Dateien werden in alphabetischer Reihenfolge gelesen, also gewann die Datei der Distribution. Der Inhalt der Datei war richtig, ihr Name war falsch. Die Umbenennung in `01-hardening.conf` löste das Problem.

**Der Server sperrte die eigene Adresse des Administrators aus.** Nach mehreren fehlgeschlagenen Anmeldeversuchen und abgelaufenen Anmeldefristen wurden alle weiteren Verbindungen von dieser Quelladresse sofort und ohne Passwortabfrage geschlossen:

```
drop connection #0 from [192.168.100.20]:60380 ... penalty: exceeded LoginGraceTime
```

OpenSSH 9.9 verhängt Strafen je Quelladresse (`crash:90 authfail:5 grace-exceeded:10 max:600`). Die verkürzte `LoginGraceTime` von 30 Sekunden ließ diese Schwelle deutlich leichter erreichen. Das ist kein Defekt einer der beiden Einstellungen, sondern ihr Zusammenspiel — und der Grund, warum jede Härtungsmaßnahme von außen und über einen zweiten, unabhängigen Zugangsweg geprüft werden muss. In diesem Labor war dieser zweite Weg die Konsole der virtuellen Maschine.

## 11. Prüfung der Zugriffssteuerung

Eine Sicherheitsregel ist erst dann nachgewiesen, wenn sie etwas abweist. Vom Hostsystem wurden drei Fälle geprüft:

| Konto | Erwartung | Ergebnis | Mechanismus |
|---|---|---|---|
| `labadmin` | erlaubt | `lab-web01` / `labadmin` | in `AllowUsers` geführt |
| `root` | abgewiesen | `Permission denied` | `PermitRootLogin no` |
| `centos_user0` | abgewiesen | `Permission denied` | nicht in `AllowUsers` |

Beide abgewiesenen Versuche wurden nach genau drei Passwortabfragen getrennt, was `MaxAuthTries 3` im Verhalten bestätigt und nicht nur in der Konfigurationsausgabe. Die beiden Abweisungen werden von zwei unabhängigen Mechanismen durchgesetzt — mehrschichtige Sicherheit.

![Prüfung der SSH-Zugriffssteuerung](../../screenshots/phase-05/05-05-ssh-access-control-test.png)
*Abbildung 5.5 — Eine erlaubte und zwei abgewiesene SSH-Anmeldungen, jeweils mit vorab genannter Erwartung.*

## 12. DNS-Einträge für den Webserver

Windows-Server tragen sich beim Domänenbeitritt selbst in das DNS ein. Ein Linux-Server, der kein Domänenmitglied ist, tut das nicht, daher wurden die Einträge auf dem Domänencontroller manuell angelegt:

```powershell
Add-DnsServerResourceRecordA -Name "lab-web01" `
  -ZoneName "corp.homelab.internal" `
  -IPv4Address "192.168.100.20" `
  -CreatePtr
```

Die Einträge sind statisch (`Timestamp: 0`) und werden deshalb nicht durch die DNS-Bereinigung entfernt. Dynamische Einträge sind für Clients angemessen, Servereinträge sollten statisch sein.

Der Reverse-Eintrag füllt die in Phase 3 angelegte Reverse-Lookup-Zone und erlaubt es Protokollen und Analysewerkzeugen, Namen statt Adressen anzuzeigen.

Die Prüfung erfolgte durch einen neutralen Zeugen und nicht durch den Webserver selbst:

![Namensauflösung vom Domänencontroller](../../screenshots/phase-05/05-06-dns-resolution-from-dc01.png)
*Abbildung 5.6 — Forward-Eintrag, Reverse-Eintrag und TCP-Verbindung auf Port 80, geprüft vom Domänencontroller.*

![Namensauflösung vom Webserver](../../screenshots/phase-05/05-07-dns-resolution-from-web01.png)
*Abbildung 5.7 — Forward, Reverse und Kurzname auf dem Webserver sowie eine HTTP-Anfrage über den Namen.*

## 13. Prüfmatrix

| # | Test | Befehl | Ergebnis |
|---|---|---|---|
| 1 | Statische Adresse aktiv | `ip -4 addr show enp1s0` | `192.168.100.20/24` |
| 2 | Gateway erreichbar | `ping 192.168.100.1` | 0 % Verlust |
| 3 | Domänencontroller erreichbar | `ping 192.168.100.10` | 0 % Verlust, `ttl=128` |
| 4 | Externe Namensauflösung | `dig +short A mirrors.centos.org` | 9 A-Einträge |
| 5 | Webserver läuft | `systemctl status nginx` | active, enabled |
| 6 | Lauschender Socket | `ss -tlnp` | `0.0.0.0:80` |
| 7 | Lokale HTTP-Anfrage | `curl http://localhost` | `200` |
| 8 | Externe HTTP-Anfrage | `curl http://192.168.100.20` | `200` |
| 9 | Firewall vor der Regel | Laufzeitentfernung + `curl` | `000` |
| 10 | Firewall nach der Regel | `--reload` + `curl` | `200` |
| 11 | Cockpit-Port geschlossen | `curl ...:9090` | `000` |
| 12 | Administrative Rechte | `sudo whoami` | `root` |
| 13 | SSH als `labadmin` | `ssh labadmin@...` | erfolgreich |
| 14 | SSH als `root` | `ssh root@...` | abgewiesen |
| 15 | SSH außerhalb der weißen Liste | `ssh centos_user0@...` | abgewiesen |
| 16 | Wirksame SSH-Konfiguration | `sshd -T` | 7 von 7 Einstellungen aktiv |
| 17 | Forward-Eintrag | `Resolve-DnsName ... -Type A` | `192.168.100.20` |
| 18 | Reverse-Eintrag | `Resolve-DnsName ... -Type PTR` | `lab-web01.corp...` |
| 19 | HTTP über den Namen | `curl http://lab-web01.corp...` | `200` |
| 20 | TCP-Test von DC01 | `Test-NetConnection ... -Port 80` | `True` |

## 14. Konfigurationsartefakte

Alle Dateien wurden aus dem laufenden System kopiert und nicht abgetippt:

| Datei | Inhalt |
|---|---|
| [`01-hardening.conf`](../../configs/phase-05/01-hardening.conf) | Drop-in-Datei zur SSH-Härtung |
| [`firewalld-public-zone.xml`](../../configs/phase-05/firewalld-public-zone.xml) | firewalld-Zone mit genau zwei Diensten |
| [`network-enp1s0.txt`](../../configs/phase-05/network-enp1s0.txt) | Wirksame IPv4-Konfiguration |
| [`nginx-index.html`](../../configs/phase-05/nginx-index.html) | Startseite des Webservers |

## 15. Sicherungspunkte

| Name | Erstellt | Zweck |
|---|---|---|
| `phase05-pre-hardening` | 2026-08-02 20:07 | Sauberer Zustand vor allen Änderungen der Phase 5 |

## 16. Ergebnis

`LAB-WEB01` ist ein statisch adressierter, über DNS auffindbarer Webserver im Domänennetz. Er ist über genau zwei Ports erreichbar, wird über ein einziges benanntes Konto verwaltet, und jede Konfigurationsentscheidung ist dokumentiert und aus den Artefakten dieses Repositorys nachvollziehbar.

## 17. Erkenntnisse

- Eine Änderung ohne Fehlermeldung ist nicht zwangsläufig wirksam geworden. Die wirksame Konfiguration muss zurückgelesen und darf nicht angenommen werden.
- OpenSSH verwendet den ersten gefundenen Wert eines Schlüsselworts. Muster aus anderen Konfigurationssystemen lassen sich nicht ungeprüft übertragen.
- Härtungsmaßnahmen wirken zusammen. Ein verkürztes Zeitlimit in Verbindung mit einem automatischen Strafsystem sperrte die eigene Adresse des Administrators aus.
- Jede Härtungsmaßnahme erfordert einen zweiten, unabhängigen Zugangsweg. Der Konsolenzugang ist bei solchen Arbeiten nicht optional.
- `NOERROR` in einer DNS-Antwort bedeutet nicht, dass die Antwort brauchbar ist. Eintragstyp und Anzahl müssen geprüft werden.
- Ein Fehler wird in der Schicht behoben, die dafür zuständig ist. Die externe Namensauflösung ist Aufgabe des Domänencontrollers, nicht des Clients.
- Diagnosewerkzeuge stellen unterschiedliche Fragen. `getent` zeigt, was eine Anwendung sieht, `dig` zeigt, was ein Namensserver antwortet.

## 18. Bekannte Einschränkungen

- **Die Passwortanmeldung über SSH ist weiterhin aktiv.** Eine schlüsselbasierte Anmeldung mit `PasswordAuthentication no` wäre die stärkere Konfiguration und wurde aus Zeitgründen zurückgestellt. Dies ist der wichtigste offene Punkt der Phase.
- **Die grafische Oberfläche ist weiterhin installiert und aktiv.** Ein Server würde üblicherweise `multi-user.target` verwenden. Die grafische Sitzung wurde beibehalten, weil die Maschine über eine Konsole bedient wird.
- **`centos_user0` existiert weiterhin**, ohne administrative Rechte und ohne SSH-Zugang. In einer Produktivumgebung würde das Konto entfernt oder gesperrt.
- **`getent hosts lab-web01`** liefert auf der Maschine selbst die Loopback-Adresse `::1`, weil das Namensdienstmodul `myhostname` den lokalen Hostnamen beantwortet, wenn die Adressfamilie nicht eingeschränkt ist. Für IPv4 und für alle anderen Hosts im Netz ist die Namensauflösung korrekt; betroffen ist nur die Sicht der Maschine auf sich selbst.
- **Kein TLS.** Der Webserver liefert unverschlüsseltes HTTP aus. Eine Zertifikatsinfrastruktur liegt außerhalb des Umfangs dieser Phase.
- **Keine Protokollüberwachung.** Fehlgeschlagene Anmeldungen werden in `/var/log/secure` festgehalten, aber nicht automatisch ausgewertet.

## 19. Nächste Phase

[Phase 6 — Sicherheitsüberprüfung](../06-security-testing/)
