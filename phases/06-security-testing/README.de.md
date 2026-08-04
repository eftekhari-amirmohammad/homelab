# Phase 6 — Sicherheitsüberprüfung

## 1. Zweck

Die vorherigen Phasen haben das Labor aufgebaut und gehärtet. Diese Phase stellt
eine andere Frage: **Wie sieht das Labor von außen tatsächlich aus?**

Härtung stellt her, was man beabsichtigt hat. Eine Überprüfung zeigt, was
tatsächlich vorhanden ist. Beides ist nicht dasselbe, und diese Phase existiert,
um den Unterschied zu messen. Ein eigenes Sicherheitssystem (`LAB-SEC01`, Parrot
Security 7.2) scannt die beiden Server des Labors, und jede Aussage aus den
Phasen 3 bis 5 wird gegen gemessene Nachweise geprüft, nicht gegen die
Erinnerung.

Die Phase wurde in zwei Durchgängen durchgeführt, weil der Host 16 GB RAM hat und
nie mehr als zwei virtuelle Maschinen gleichzeitig betreibt:

| Durchgang | Ziel | Rolle |
|---|---|---|
| 1 | `LAB-WEB01` (192.168.100.20) | CentOS Stream 9, nginx, SSH, firewalld |
| 2 | `LAB-DC01` (192.168.100.10) | Windows Server 2019, AD DS, DNS, DHCP |

## 2. Ausgangslage

- Phasen 0 bis 5 abgeschlossen: Netzplanung, Virtualisierung, Active Directory,
  domänenbeigetretener Client, gehärteter Linux-Server.
- `LAB-SEC01` existierte seit Phase 2 als virtuelle Maschine, war aber nie
  konfiguriert oder gestartet worden.
- Eine Sicherheitsüberprüfung hatte bis zu diesem Zeitpunkt nicht stattgefunden.

## 3. Umfang und Berechtigung

Ein Scan ist nur dann legitim, wenn sein Umfang vor Beginn schriftlich festgelegt
ist.

- **Zulässiges Zielnetz:** `192.168.100.0/24`, ein isoliertes NAT-Netz auf eigener
  Hardware ohne Route in ein fremdes Netz.
- **Zulässige Ziele:** `LAB-WEB01` und `LAB-DC01`, beide im Eigentum des Autors.
- **Quelladresse:** `192.168.100.50`, statisch vergeben, damit jeder Eintrag in
  einem Zielprotokoll eindeutig der Überprüfung zugeordnet werden kann.
- **Ausgeschlossen:** alles außerhalb des Labornetzes, einschließlich der
  Internetverbindung des Hostsystems.

Die feste Adresse des Scanners ist eine Absicht. Ein autorisierter Scan muss im
Protokoll eindeutig zuordenbar sein; sonst kann der Verteidiger eine Überprüfung
nicht von einem Angriff unterscheiden.

## 4. Werkzeuge

| Werkzeug | Version | Verwendung |
|---|---|---|
| `nmap` | 7.95 | Hostermittlung, Portscans, Versions- und Skriptscans |
| `curl` | 8.14.1 | Prüfung der HTTP-Antwortkopfzeilen |
| `dig` | bind 9 | Prüfung der Namensauflösung |
| `ss` | iproute2 | Auflisten lokaler lauschender Sockets |
| `sysctl` | procps | Auslesen der Kernel-Netzwerkparameter |

`LAB-SEC01` selbst wurde bewusst **nicht** gehärtet. Seine Rolle ist die eines
administrativen Werkzeugs in einem geschlossenen Labor, nicht die eines
exponierten Servers. Der Härtungsgrad richtet sich nach der Rolle des Systems —
nicht nach Gewohnheit.

## 5. Durchgang 1 — Linux-Webserver (LAB-WEB01)

### 5.1 Hostermittlung

```bash
sudo nmap -sn 192.168.100.0/24
```

Drei Hosts antworteten: das Gateway `.1`, der Webserver `.20` und der Scanner
selbst `.50`. Kein Name wurde aufgelöst, und der Scan dauerte 27,92 Sekunden —
ein auffälliger Wert für ein Netz mit drei aktiven Hosts. Die Ursache war nicht
das Netz: Der Domänencontroller, der in diesem Netz der DNS-Server ist, war
ausgeschaltet, sodass alle 253 Rückwärtsauflösungen in einen Zeitüberschritt
laufen mussten. **Die Laufzeit eines Werkzeugs kann selbst einen ausgefallenen
Dienst melden.**

Ein Scan beginnt immer mit der Frage, was überhaupt existiert, bevor gefragt wird,
was ein einzelner Host anbietet.

### 5.2 Portscan

```bash
sudo nmap -sS 192.168.100.20
```

```
Not shown: 988 filtered tcp ports (no-response), 10 filtered tcp ports (admin-prohibited)
22/tcp open  ssh
80/tcp open  http
```

Genau zwei offene Ports, passend zur firewalld-Dienstliste (`http ssh`) aus
Phase 5. Die Aussage aus Phase 5 hat der externen Messung also standgehalten.

Jeder offene Port muss begründet sein: `22` ist der einzige administrative Zugang
zur Maschine, `80` ist der Grund, aus dem die Maschine existiert.

### 5.3 Firewallverhalten und ICMP-Ratenbegrenzung

Die Erwartung vor dem Scan war, dass gesperrte Ports als `closed` erscheinen. Sie
erschienen als `filtered`, und zehn davon trugen die Begründung
`admin-prohibited`. Die Standardregel von firewalld ist `REJECT` mit einer
ICMP-Fehlermeldung und kein stilles Verwerfen; der Host teilt also aktiv mit,
dass eine Firewall vorhanden ist.

Das Verhältnis zehn zu 988 wurde nicht vermutet, sondern durch einen gezielten
Versuch erklärt:

```bash
sudo nmap -sS -p 8000-8004 --scan-delay 1s --reason 192.168.100.20
```

Mit einer Sekunde Abstand zwischen den Paketen antworteten **alle fünf** Ports mit
`admin-prohibited`. Der Linux-Kernel begrenzt ICMP-Fehlermeldungen
(`net.ipv4.icmp_ratelimit = 1000`, also eine Meldung pro Sekunde und Typ), sodass
ein schneller Scan den Antworten davonläuft. Das ist keine Schwäche der Firewall,
sondern ein Schutz davor, dass der Host als Verstärker gegen einen Dritten
missbraucht wird. **Der Kernel ist ebenfalls eine Sicherheitsschicht.**

Ein beobachtetes Ergebnis ist erst dann erklärt, wenn es absichtlich
wiederholbar ist.

### 5.4 Versionserkennung

```bash
sudo nmap -sV -p 22,80 192.168.100.20
```

```
22/tcp open  ssh     OpenSSH 9.9 (protocol 2.0)
80/tcp open  http    nginx 1.20.1
```

![nmap zeigt die genaue nginx-Version](../../screenshots/phase-06/06-01-nmap-version-disclosed-before.png)
*Abbildung 6.1 — Vor der Behebung: Der Webserver gibt seine genaue Versionsnummer
an einen nicht authentifizierten Client preis.*

Die genaue Versionsnummer ist ein Befund. Ein Angreifer muss Schwachstellen nicht
blind ausprobieren; die Nummer lässt sich in einer CVE-Datenbank nachschlagen, und
die Liste der passenden Angriffe liegt vor dem ersten Versuch vor.

### 5.5 Vollständiger Portscan

```bash
sudo nmap -sS -p- -T4 --max-retries 1 192.168.100.20
```

Alle 65535 Ports: nur `22` und `80` offen, 65405 ohne Antwort und 128
`admin-prohibited`, in 135,50 Sekunden. Die Anzahl der ICMP-Antworten entspricht
der in Abschnitt 5.3 gemessenen Ratenbegrenzung und bestätigt die Erklärung damit
aus einer zweiten Richtung.

Dieser Scan wurde durchgeführt, weil der Standardscan nur die 1000 häufigsten
Ports prüft — etwa 1,5 Prozent des Portbereichs. Die Aussage „es gibt nichts
weiter“ ist erst nach einem vollständigen Scan zulässig.

### 5.6 Spuren des Scanners im Webserverprotokoll

Jede Überprüfung muss von beiden Seiten betrachtet werden. Auf dem Ziel:

```bash
sudo grep "Nmap" /var/log/nginx/access.log | tail -3
```

```
"POST /sdk HTTP/1.1" 404 3971 "-" "Nmap Scripting Engine"
"GET /HNAP1 HTTP/1.1" 404 3971 "-" "Nmap Scripting Engine"
"GET /evox/about HTTP/1.1" 404 3971 "-" "Nmap Scripting Engine"
```

![nginx-Zugriffsprotokoll mit nmap-Anfragen](../../screenshots/phase-06/06-03-scanner-footprint-nginx-log.png)
*Abbildung 6.3 — Spuren des Scanners im nginx-Zugriffsprotokoll. Die Zeitstempel
mit `+0330` liegen vor der Zeitzonenkorrektur aus Abschnitt 7.2.*

Der Scan war für den Verteidiger vollständig sichtbar. Das Feld `User-Agent`
nannte das Werkzeug ehrlich, aber dieses Feld ist eine Selbstauskunft und keine
Identität: Ein Angreifer setzt es beliebig. Eine verlässliche Erkennung stützt
sich deshalb auf Verhaltensmuster — viele 404-Antworten auf nicht existierende
Pfade innerhalb von Sekunden — und nicht auf die Selbstauskunft des Clients.

Prävention verhindert manche Angriffe; die Protokollierung macht die übrigen
bemerkbar.

## 6. Durchgang 2 — Domänencontroller (LAB-DC01)

### 6.1 Preisgabe des Betriebssystems vor dem ersten Scan

Noch bevor der Scanner gestartet wurde, diente ein einfaches `ping` vom Host als
günstigste Erreichbarkeitsprüfung:

```
64 bytes from 192.168.100.10: icmp_seq=1 ttl=128 time=0.205 ms
```

`ttl=128` kennzeichnet ein Windows-System; Linux antwortet mit 64, wie
`LAB-WEB01` gezeigt hat. Die Betriebssystemfamilie ist damit preisgegeben, bevor
ein einziger Port gescannt wurde. Schon der TTL-Wert verrät das Betriebssystem —
Aufklärung beginnt vor dem ersten Scan.

Anders als `LAB-WEB01` antwortet der Domänencontroller auf ICMP-Echoanfragen. In
einer Domäne ist das eine bewusste betriebliche Entscheidung: Überwachung und
Diagnose hängen davon ab.

### 6.2 Hostermittlung und Rückwärtsauflösung

```
LAB-DC01.corp.homelab.internal (192.168.100.10)
3 hosts up, scanned in 1.98 seconds
```

![Hostermittlung löst den vollständigen Namen des Domänencontrollers auf](../../screenshots/phase-06/06-04-host-discovery-dc01-fqdn.png)
*Abbildung 6.4 — Bei laufendem DNS-Server löst die Rückwärtsauflösung den
vollständigen Namen des Domänencontrollers auf.*

Zwei Beobachtungen:

1. Derselbe Befehl brauchte in Durchgang 1 noch 27,92 Sekunden und jetzt 1,98
   Sekunden. Der Unterschied waren 253 DNS-Zeitüberschritte, nicht das Netz.
2. Die Rückwärtsauflösung liefert einem nicht authentifizierten Client den
   Hostnamen, das interne Namensschema `LAB-XX`, die Rolle der Maschine und den
   Domänennamen — kostenlos. Die in Phase 5 erstellten PTR-Einträge sind für
   Diagnose und Protokollauswertung betrieblich notwendig und verraten
   gleichzeitig das Namensschema. Der Zielkonflikt wird bewusst in Kauf genommen.

`192.168.100.20` fehlt in diesem Durchgang, weil `LAB-WEB01` aus Speichergründen
ausgeschaltet war. **Ein Scan liefert eine Momentaufnahme, kein Inventar.**

### 6.3 Portscan und Angriffsfläche

```bash
sudo nmap -sS 192.168.100.10
```

![zwölf offene Ports auf dem Domänencontroller](../../screenshots/phase-06/06-05-dc01-twelve-open-ports.png)
*Abbildung 6.5 — Zwölf offene Ports auf dem Domänencontroller, gegenüber zwei auf
dem Webserver.*

| Port | Dienst | Begründung |
|---|---|---|
| 53 | DNS | Namensauflösung für die gesamte Domäne |
| 88 | Kerberos | Authentifizierung |
| 135 | RPC-Endpunktzuordnung | Grundlage der Windows-RPC-Kommunikation |
| 139 | NetBIOS-Sitzung | älterer Namensdienst, mit der Rolle installiert |
| 389 | LDAP | Verzeichnisabfragen |
| 445 | SMB | Freigaben SYSVOL und NETLOGON, Verteilung der Gruppenrichtlinien |
| 464 | kpasswd | Kennwortänderungen |
| 593 | RPC über HTTP | zusammen mit der Domänenrolle installiert |
| 636 | LDAPS | LDAP über TLS |
| 3268 | Globaler Katalog | forestübergreifende Suche |
| 3269 | Globaler Katalog SSL | dasselbe über TLS |
| 5985 | WinRM | Fernverwaltung und Ausführung von Befehlen aus der Ferne |

**Zwei offene Ports gegen zwölf.** Der Unterschied ist kein Unterschied im
Aufwand, sondern in der Rolle. Ein Domänencontroller kann seine Portliste nicht
beliebig verkleinern, weil jeder dieser Ports ein Dienst ist, von dem die Domäne
abhängt. Er wird deshalb nicht durch geschlossene Ports geschützt, sondern durch
Netzsegmentierung (siehe ADR-006).

Positiver Befund: **Port 3389 (RDP) ist nicht offen.** Der Remotedesktop wurde nie
aktiviert, obwohl die Maschine über die grafische Konsole verwaltet wurde.

Das Firewallverhalten war erneut anders. Alle 988 gefilterten Ports antworteten
mit `no-response`, und **keine einzige** ICMP-Fehlermeldung wurde gesendet. Die
Windows-Firewall verwirft still, während firewalld mit einer Meldung ablehnt. In
einem Netz wurden damit drei unterschiedliche Verhaltensweisen beobachtet:

| Regel | nmap-Ergebnis | Geschwindigkeit | Preisgegebene Information |
|---|---|---|---|
| `REJECT` mit ICMP-Fehler | `filtered` | langsam | eine Firewall existiert |
| `REJECT` mit TCP-Reset | `closed` | schnell | eine Firewall existiert |
| `DROP` ohne jede Antwort | `filtered` | schnell | nichts |

Das Verhalten einer Firewall lässt sich nicht aus dem Betriebssystem ableiten —
nur messen.

### 6.4 Versionserkennung und protokollbedingte Preisgabe

```bash
sudo nmap -sV -p 53,88,135,389,445,3268,5985 192.168.100.10
```

![Preisgabe von Domäne und Serverzeit über LDAP und Kerberos](../../screenshots/phase-06/06-06-dc01-domain-info-disclosure.png)
*Abbildung 6.6 — Ohne jede Authentifizierung erfährt der Scanner den
Domänennamen, den AD-Standortnamen und die Serverzeit.*

```
88/tcp   Microsoft Windows Kerberos (server time: 2026-08-04 15:42:05Z)
389/tcp  Microsoft Windows Active Directory LDAP (Domain: corp.homelab.internal, Site: Default-First-Site-Name)
445/tcp  microsoft-ds?
Service Info: Host: LAB-DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Auf `LAB-WEB01` ließ sich die entsprechende Preisgabe mit einer einzigen
Anweisung schließen. Hier ist das nicht möglich:

- **LDAP muss die Domäne nennen, die es bedient,** sonst könnte kein Client eine
  Verbindung aufbauen.
- **Kerberos muss seine Uhrzeit veröffentlichen,** weil das Protokoll auf Zeit
  aufbaut: Tickets verfallen, und ein Zeitversatz von mehr als fünf Minuten macht
  die Authentifizierung unmöglich. Das abschließende `Z` belegt zudem, dass die
  Maschine in UTC läuft.

Was das Protokoll erfordert, lässt sich nicht abschalten — nur eingrenzen, und
zwar wiederum durch Segmentierung. Das ist das zweite unabhängige Argument für
ADR-006, und dieses beruht auf einer Messung und nicht auf Theorie.

Zwei Einschränkungen gehören in den Bericht:

- Port 53 wurde als `Simple DNS Plus` gemeldet. **Das ist falsch.** Der Dienst ist
  die in Phase 3 installierte Windows-DNS-Serverrolle. Die Versionserkennung
  vergleicht Antworten mit einer Signaturdatenbank und liefert die nächstliegende
  Übereinstimmung, die auch das falsche Produkt sein kann. Der Fehler wurde durch
  einen Abgleich mit der eigenen Dokumentation dieses Projekts entdeckt. Ein
  Werkzeug liefert eine Vermutung; die Bestätigung liefert der Prüfer.
- Port 445 antwortete mit `microsoft-ds?`. Das Fragezeichen bedeutet, dass der
  Dienst nicht bestätigt werden konnte. Modernes SMB gibt die Betriebssystemversion
  nicht von sich aus bekannt — ein positiver Befund und keine Lücke des Scans.

### 6.5 SMB-Sicherheitsprüfung

```bash
sudo nmap -p 445 --script smb2-security-mode,smb-os-discovery 192.168.100.10
```

![SMB-Signatur erforderlich und SMBv1 nicht verfügbar](../../screenshots/phase-06/06-07-smb-signing-required.png)
*Abbildung 6.7 — Die SMB-Signatur ist erforderlich, und das auf SMBv1 angewiesene
Skript liefert keine Ausgabe.*

```
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
```

- **Die Signatur ist aktiviert und erforderlich.** SMB-Relay-Angriffe, einer der
  häufigsten Wege zur Rechteausweitung in echten Netzen, funktionieren gegen
  diesen Host nicht.
- **Der Dialekt 3.1.1** ist die aktuelle SMB-Version, mit Integritätsprüfung vor
  der Authentifizierung und Verschlüsselungsunterstützung.
- **`smb-os-discovery` lieferte keine Ausgabe.** Dieses Skript ist auf SMBv1
  angewiesen; das Ausbleiben einer Antwort ist hier das eigentliche Ergebnis:
  SMBv1 — das Protokoll, über das sich WannaCry verbreitete — ist nicht
  verfügbar. Wenn der Nachweis eine Abwesenheit ist, wird die Kopfzeile des
  Bildschirmfotos Teil des Nachweises.

Eine ehrliche Anmerkung: Diese drei Eigenschaften sind **sichere Voreinstellungen
von Windows Server 2019** und keine Leistung dieses Projekts. Sichere
Voreinstellungen sind ein Verdienst des Herstellers — der Nachweis, dass sie
tatsächlich aktiv sind, ist der Beitrag der Prüfung. Ein Bericht, der nur
Schwächen auflistet, ist ein halber Bericht.

### 6.6 Vollständiger Portscan und dynamische RPC-Ports

```bash
sudo nmap -sS -p- -T4 --max-retries 1 192.168.100.10
```

![vollständiger Portscan mit neun zusätzlichen Ports](../../screenshots/phase-06/06-08-dc01-full-scan-nine-additional-ports.png)
*Abbildung 6.8 — Der vollständige Scan findet 21 offene Ports: neun mehr als der
Standardscan.*

**Insgesamt 21 offene Ports.** Der Standardscan der 1000 häufigsten Ports fand
zwölf; er hätte damit 43 Prozent der Angriffsfläche dieses Servers nicht gesehen.
Auf `LAB-WEB01` fand der vollständige Scan nichts Neues, hier neun zusätzliche
Ports. Der Nutzen eines vollständigen Scans lässt sich nicht an einem einzigen
Host beurteilen.

Die neun Ports sind `9389` (Active Directory-Webdienste) und acht dynamische
RPC-Endpunkte zwischen `49668` und `49696`.

**Diese acht Nummern sind nicht stabil.** Windows vergibt dynamische
RPC-Endpunkte beim Start aus dem Bereich `49152-65535`; sie ändern sich nach jedem
Neustart. Der korrekte Befund lautet deshalb „acht dynamische RPC-Ports im Bereich
49152-65535“ und nicht eine feste Liste. Ein Teil der Angriffsfläche hat keine
feste Nummer.

Das hat eine praktische Folge: Ein strenges Regelwerk mit einzelnen Ports ist für
einen Domänencontroller nicht möglich, solange der RPC-Bereich nicht in Windows
bewusst eingeschränkt wird. Die realistische Maßnahme ist die Segmentierung, nicht
die Portliste.

Dauer: 88,62 Sekunden gegenüber 135,50 Sekunden beim Linux-Host. Die Scanmethode
war in beiden Durchgängen identisch — nur deshalb sind die beiden Ergebnisse
überhaupt vergleichbar. Der Linux-Host war langsamer zu scannen, weil seine
ICMP-Ratenbegrenzung jede Ablehnung verzögerte. `DROP` macht einen Host stiller,
beschleunigt aber den Angreifer; `REJECT` gibt Informationen preis, kostet den
Angreifer aber Zeit.

## 7. Befunde und Behebung

### 7.1 Befund 1 — Preisgabe der nginx-Version (behoben)

**Befund:** Der Webserver gab `nginx 1.20.1` in seiner `Server`-Kopfzeile preis.

**Behebung** — eine Drop-in-Datei statt einer Änderung der Herstellerkonfiguration:

```bash
printf 'server_tokens off;\n' | sudo tee /etc/nginx/conf.d/security.conf
sudo nginx -t
sudo systemctl reload nginx
```

**Nachweis:**

```
80/tcp open  http    nginx
Server: nginx
```

![nmap zeigt keine Versionsnummer mehr](../../screenshots/phase-06/06-02-nmap-version-hidden-after.png)
*Abbildung 6.2 — Nach der Behebung: Der Dienst wird weiterhin erkannt, die
Versionsnummer fehlt.*

Zwei methodische Punkte sind hier wichtig:

- Der Nachweis des Zustands „vorher“ wurde **vor** der Änderung erfasst. Das
  Beweisfenster schließt sich in dem Moment, in dem die Änderung wirksam wird.
- `22/tcp OpenSSH 9.9` wurde vorher und nachher gemessen und hat sich **nicht**
  geändert. Das ist die Kontrollgruppe: Eine Änderung ist erst dann verstanden,
  wenn man auch weiß, was sie nicht verändert hat.

Dies ist kosmetische Härtung und keine Behebung einer Schwachstelle. Sie erhöht
den Aufwand für automatisierte Massenscans; unangreifbar wird der Server dadurch
nicht.

### 7.2 Befund 2 — abweichende Zeitzone (behoben)

**Befund:** Die Protokolleinträge des Scans trugen `+0330` (`Asia/Tehran`), während
der Domänencontroller und das gesamte Active Directory in UTC laufen. Eine
Korrelation von Ereignissen über mehrere Maschinen hätte in jeder einzelnen Zeile
eine Kopfrechnung erfordert, und eine Beweiskette, die Kopfrechnen erfordert, ist
eine schwache Beweiskette.

**Behebung:**

```bash
sudo timedatectl set-timezone UTC
```

**Nachweis** — und darauf kam es an: Nach der Änderung meldete `timedatectl` UTC,
während das nginx-Protokoll weiterhin `+0330` schrieb. Laufende Prozesse hatten
die alte Zeitzone geerbt. Erst nach `systemctl reload nginx`, wodurch neue
Arbeitsprozesse entstehen, zeigte das Protokoll `+0000`.

Eine Änderung auf Systemebene lässt laufende Prozesse zurück. Der Nachweis muss an
der Stelle erfolgen, um die es in der Aussage geht — hier in der Protokolldatei
und nicht in der Befehlsausgabe.

Zeitversatz und abweichende Zeitzone sind zwei verschiedene Probleme: Zeitversatz
zerstört Dienste; unterschiedliche Zeitzonen zerstören die Beweiskette. Diese
Entscheidung ist als ADR-012 festgehalten.

### 7.3 Vergleich der Angriffsflächen

| | `LAB-WEB01` | `LAB-DC01` |
|---|---|---|
| Rolle | Web- und SSH-Server | Domänencontroller, DNS, DHCP |
| Offene Ports (Top 1000) | 2 | 12 |
| Offene Ports (alle 65535) | 2 | 21 |
| Firewallregel | `REJECT` mit ICMP | stilles `DROP` |
| ICMP-Echo | gesperrt | absichtlich erlaubt |
| Versionspreisgabe | behoben | nicht behebbar, protokollbedingt |
| TTL | 64 (Linux) | 128 (Windows) |
| Dauer des vollständigen Scans | 135,50 s | 88,62 s |

Die zentrale Erkenntnis dieser Phase steht in den ersten beiden Zeilen: **Die
Angriffsfläche richtet sich nach der Rolle des Systems.** Der Webserver ließ sich
auf zwei Ports reduzieren, weil er zwei Dinge tut. Der Domänencontroller lässt
sich nicht reduzieren, weil jeder seiner 21 Ports ein Dienst ist, von dem die
Domäne abhängt.

## 8. Prüfmatrix

| # | Aussage | Methode | Ergebnis |
|---|---|---|---|
| 1 | Auf WEB01 sind nur `http` und `ssh` erreichbar | `nmap -sS` | ✅ 2 offene Ports |
| 2 | Auf WEB01 keine weiteren Ports außerhalb der Top 1000 | `nmap -sS -p-` | ✅ 2 von 65535 |
| 3 | firewalld lehnt mit ICMP-Fehler ab | `nmap --reason` | ✅ `admin-prohibited` |
| 4 | Das Verhältnis 988/10 entsteht durch Ratenbegrenzung | `--scan-delay 1s` | ✅ 5 von 5 |
| 5 | nginx gibt die Version nicht mehr preis | `nmap -sV`, `curl -sI` | ✅ `Server: nginx` |
| 6 | Die SSH-Kennung blieb unverändert (Kontrollgruppe) | `nmap -sV -p 22` | ✅ identisch |
| 7 | Der Scan ist im Webserverprotokoll sichtbar | `grep` in `access.log` | ✅ 404-Muster |
| 8 | WEB01 protokolliert in UTC | `access.log` nach Reload | ✅ `+0000` |
| 9 | DC01 ist ein Windows-System | `ping`, TTL | ✅ `ttl=128` |
| 10 | Die Rückwärtsauflösung liefert den DC-Namen | `nmap -sn` | ✅ FQDN aufgelöst |
| 11 | Alle AD-Dienste sind erreichbar | `nmap -sS` | ✅ 12 erwartete Ports |
| 12 | RDP ist nicht exponiert | `nmap -sS -p-` | ✅ 3389 nicht offen |
| 13 | Die Windows-Firewall verwirft still | `nmap -sS` | ✅ 0 ICMP-Fehler |
| 14 | Die SMB-Signatur ist erforderlich | `smb2-security-mode` | ✅ erforderlich |
| 15 | SMBv1 ist nicht verfügbar | `smb-os-discovery` | ✅ keine Ausgabe |
| 16 | Auf DC01 keine weiteren Ports außerhalb der Top 1000 | `nmap -sS -p-` | ❌ 9 weitere gefunden |
| 17 | Das DNS-Produkt wurde richtig erkannt | Abgleich mit Phase 3 | ❌ falsch erkannt |

Die Zeilen 16 und 17 bleiben bewusst in der Matrix. Eine Prüfmatrix, die nur
Haken enthält, wurde nicht als Prüfung verwendet, sondern als Dekoration.

## 9. Konfigurationsartefakte

| Datei | Inhalt |
|---|---|
| [`nmap-scan-results.txt`](../../configs/phase-06/nmap-scan-results.txt) | Rohausgabe Durchgang 1 (`LAB-WEB01`), mit Auswertung |
| [`nmap-scan-results-dc01.txt`](../../configs/phase-06/nmap-scan-results-dc01.txt) | Rohausgabe Durchgang 2 (`LAB-DC01`), mit Auswertung |

Beide Dateien wurden aus dem Terminal kopiert und nicht abgetippt. Ein
Konfigurationsartefakt, das abgetippt wurde, ist kein Nachweis mehr.

## 10. Nachweise

Alle acht Bildschirmfotos liegen in
[`screenshots/phase-06/`](../../screenshots/phase-06/). Die vorangestellte Nummer
trägt die Lesereihenfolge, und jeder Dateiname nennt die Aussage, die das Bild
belegt.

## 11. Ergebnis

Das Labor wurde erstmals von einem unabhängigen Standpunkt aus überprüft. Die
Firewallregeln, Dienstlisten und DNS-Einträge der Phasen 3 bis 5 wurden durch
externe Messung bestätigt und nicht angenommen. Zwei Schwächen wurden gefunden und
mit Vorher-Nachher-Nachweis behoben, vier positive Befunde wurden dokumentiert und
dort dem Hersteller zugeschrieben, wo es sich um Voreinstellungen handelt, und
zwei Vorhersagen des Autors wurden widerlegt und im Bericht belassen.

Das wichtigste Ergebnis ist keine behobene Schwäche. Es ist die Erkenntnis, dass
die beiden Server nicht am gleichen Maßstab gemessen werden können: Was auf dem
Webserver ein behebbarer Befund war, ist auf dem Domänencontroller eine
unvermeidbare Eigenschaft des Protokolls.

## 12. Gelernte Lektionen

1. **Härtung stellt her, was man beabsichtigt hat; eine Überprüfung zeigt, was
   tatsächlich vorhanden ist.** Beide Befunde dieser Phase hat erst die
   unabhängige Prüfung sichtbar gemacht, nicht die Härtung selbst.
2. **Das Verhalten einer Firewall lässt sich nicht aus dem Betriebssystem
   ableiten.** In einem Netz wurden drei Verhaltensweisen beobachtet, jede gemäß
   ihrer eigenen Standardregel.
3. **Der Standardscan erfasst 1,5 Prozent des Portbereichs.** Auf einem Host war
   das ausreichend, auf dem anderen verbarg er 43 Prozent der Angriffsfläche.
4. **Ein Scan liefert eine Momentaufnahme, kein Inventar.** `LAB-WEB01` fehlt in
   Durchgang 2, weil es ausgeschaltet war, nicht weil es nicht existiert.
5. **Auch die Laufzeit eines Werkzeugs ist ein Nachweis.** 27,92 Sekunden
   gegenüber 1,98 Sekunden meldeten einen Dienstausfall, bevor ein Port gescannt
   wurde.
6. **Manche Preisgabe ist das Protokoll selbst.** LDAP muss seine Domäne nennen,
   Kerberos seine Uhrzeit. Nicht abschaltbar, nur eingrenzbar.
7. **Ein Werkzeug liefert eine Vermutung, die Bestätigung liefert der Prüfer.**
   Das falsch erkannte DNS-Produkt wurde durch eine zweite Quelle entdeckt: die
   eigene Dokumentation dieses Projekts.
8. **Eine leere Ausgabe kann das Ergebnis sein.** Das stumme SMBv1-Skript ist der
   Nachweis, dass SMBv1 fehlt.
9. **Prüfe, was sich nicht geändert haben soll.** Die unveränderte SSH-Kennung
   macht das nginx-Ergebnis der nginx-Änderung zurechenbar.
10. **Prüfe dort, worum es in der Aussage geht.** Die Zeitzone war im System
    korrigiert und in der Protokolldatei weiterhin falsch, bis der Dienst neu
    geladen wurde.
11. **Sichere Voreinstellungen gehören dem Hersteller.** Der Prüfung gehört der
    Nachweis, dass sie aktiv sind.
12. **Ein Teil der Angriffsfläche hat keine feste Nummer.** Dynamische RPC-Ports
    ändern sich bei jedem Neustart; deshalb wird ein Domänencontroller durch
    Segmentierung geschützt und nicht durch eine Portliste.

## 13. Bekannte Einschränkungen

- **Der Nachweis auf der Zielseite von `LAB-DC01` wurde nicht erhoben.** Für
  `LAB-WEB01` belegte das nginx-Zugriffsprotokoll, dass der Scan für den
  Verteidiger sichtbar war; der entsprechende Nachweis für den Domänencontroller —
  Windows-Sicherheitsprotokoll und Firewallprotokollierung — wurde in diesem
  Durchgang nicht untersucht. Die Aussage wird deshalb nicht getroffen.
- **Beide Server wurden nie im selben Durchlauf gescannt.** Die 16 GB
  Hostspeicher erlauben zwei virtuelle Maschinen gleichzeitig, weshalb die beiden
  Durchgänge zeitlich getrennt sind. Die Vergleichstabelle in Abschnitt 7.3 fügt
  zwei Messungen zusammen, keine einzelne.
- **Keine authentifizierten Prüfungen, kein Schwachstellenscanner.** Die
  Überprüfung betrachtete die von außen sichtbare Angriffsfläche. Weder
  authentifizierte Konfigurationsprüfungen (etwa ein CIS-Benchmark) noch ein
  Schwachstellenscanner wie OpenVAS oder Nessus wurden eingesetzt. Aussagen zum
  Patchstand werden daher nicht getroffen.
- **UDP wurde nicht gescannt.** DNS, Kerberos und DHCP lauschen auch auf UDP;
  gemessen wurde ausschließlich TCP.
- **Auf `LAB-WEB01` bleibt `PasswordAuthentication yes` aktiv.** Eine
  schlüsselbasierte Anmeldung wäre die richtige Wahl; in einem Labor, das zu
  Demonstrationszwecken reproduzierbar sein muss, wurde die Kennwortanmeldung
  bewusst beibehalten.
- **Es ist kein IDS oder IPS vorhanden.** Der Scan wurde aus dem Protokoll des
  Webservers rekonstruiert. Eine Erkennung von Scans als solche ist nicht
  umgesetzt.
- **`LAB-SEC01` ist bewusst nicht gehärtet.** Es ist ein administratives Werkzeug
  in einem geschlossenen Labor und kein exponierter Server.
- **Die Zwischenablage von Gast zu Host funktioniert auf `LAB-SEC01` nicht**
  (`spice-vdagent`). Der Umweg über SSH war schneller als die Reparatur; der
  Defekt ist dokumentiert und nicht behoben.
- **Die acht dynamischen RPC-Portnummern im Artefakt gelten nur für die
  Startsitzung vom 04.08.2026.**

## 14. Nächste Phase

[Phase 7 — Dokumentation](../07-documentation)
