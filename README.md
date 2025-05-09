# HackMyVM - Tryharder (Medium) - Penetration Test Report

![Tryharder VM Logo](Tryharder.png)

## VM Information & Report Details

| Kategorie        | Information                                                                 |
|------------------|-----------------------------------------------------------------------------|
| **VM Name**      | Tryharder                                                                   |
| **Platform**     | HackMyVM                                                                    |
| **Link zur VM**  | [Tryharder auf HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Tryharder) |
| **VM Autor**     | DarkSpirit                                                                  |
| **Schwierigkeit**| Medium                                                                      |
| **Berichtsdatum**| 29. April 2025                                                              |
| **Bericht von**  | Ben C.                                                                      |

## Einleitung

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "Tryharder" von HackMyVM mit dem Schwierigkeitsgrad "Medium". Ziel war es, systematisch Schwachstellen zu identifizieren und auszunutzen, um vollständigen Systemzugriff (Root-Rechte) zu erlangen. Alle Schritte, von der initialen Aufklärung über die Web-Enumeration, den initialen Zugriff bis hin zur Privilegienerweiterung auf mehreren Benutzerebenen, werden detailliert beschrieben und analysiert.

**Den vollständigen, interaktiven HTML-Bericht mit detaillierten Analysen, Befehlsausgaben und Empfehlungen finden Sie hier:**
[Live-Bericht: Tryharder - HackMyVM Medium](https://alientec1908.github.io/Tryharder_HackMyVM_Medium/)

## Phasen des Penetrationstests

Eine detaillierte Beschreibung der einzelnen Phasen und der angewendeten Techniken ist im [vollständigen HTML-Bericht](https://alientec1908.github.io/Tryharder_HackMyVM_Medium/) zu finden. Hier eine kurze Übersicht der Kernpunkte:

### 1. Reconnaissance
- Identifizierung der Ziel-IP (`192.168.2.186`) mittels `arp-scan`.
- Umfassender Portscan mit `nmap` offenbarte offene Ports:
  - **TCP/22:** OpenSSH 7.9p1 Debian 10+deb10u2
  - **TCP/80:** Apache httpd 2.4.59
- IPv6-Scan bestätigte ebenfalls die Erreichbarkeit von Port 80.

### 2. Web Enumeration
- Untersuchung der Webanwendung auf Port 80.
- `nikto` identifizierte fehlende HTTP Security Header.
- `gobuster` und `wfuzz` wurden zur Verzeichnis- und Subdomain-Enumeration eingesetzt.
- Ein entscheidender Fund war ein Kommentar im Quellcode der Webseite, der auf einen versteckten Pfad (`/74221/`) hinwies. Dieser Pfad führte zu einer Login-Seite.

### 3. Initial Access
- Ein Brute-Force-Angriff auf das Login-Formular unter `/74221/` mit `hydra` und gängigen Wortlisten war erfolgreich und lieferte die Zugangsdaten `admin:qwerty`.
- Nach dem Login wurde im Dashboard ein JSON Web Token (JWT) entdeckt.
- Der für die HS256-Signatur des JWT verwendete Secret-Key (`jwtsecret123`) konnte mittels `sjwt.py` und einer Wortliste geknackt werden.
- Mit dem geknackten Secret-Key wurde der JWT-Payload (Änderung der Rolle von `user` zu `admin`) mit `jwt_tool.py` manipuliert und neu signiert.
- Das manipulierte Admin-Token gewährte Zugriff auf eine zuvor gesperrte Dateiupload-Funktion.
- Eine PHP-Webshell (getarnt als `shell.php.jpg`) wurde hochgeladen. Da die direkte Ausführung verhindert wurde, wurde eine `.htaccess`-Datei hochgeladen, um den Server anzuweisen, `.jpg`-Dateien als PHP zu interpretieren.
- Dies ermöglichte Remote Code Execution (RCE) als Benutzer `www-data`. Eine Reverse Shell wurde etabliert.

### 4. Privilege Escalation
Der Weg zu Root-Rechten erfolgte über mehrere Stufen:
- **Als `www-data`:**
    - Auffinden von Hinweisen in Form von Notizen und Einträgen in `/etc/passwd`. Ein Kryptorätsel (basierend auf "A Tale of Two Cities" und einer XOR-ähnlichen Operation) wurde mit einem Python-Skript gelöst, um das Passwort `Y0U_5M4SH3D_17_8UDDY` zu erhalten.
- **Zu Benutzer `pentester`:**
    - Mit dem Passwort `Y0U_5M4SH3D_17_8UDDY` wurde via `su pentester` zum Benutzer `pentester` gewechselt.
- **Zu Benutzer `xiix`:**
    - Der Dienst auf Port `8989` (eine Python-Backdoor, die als `xiix` lief) war mit demselben Passwort (`Y0U_5M4SH3D_17_8UDDY`) zugänglich und gewährte eine Shell als `xiix`.
    - Im Home-Verzeichnis von `xiix` befand sich ein `guess_game`. Ein Bash-Skript wurde erstellt, um das Spiel zu lösen, was das Passwort `superxiix` offenbarte.
    - `sudo -l` für `xiix` zeigte, dass `/bin/whoami` mit der Option `env_keep+=LD_PRELOAD` ohne Passwort ausgeführt werden konnte.
- **Zu `root`:**
    - Die `LD_PRELOAD`-Schwachstelle wurde ausgenutzt: Eine C-Datei, die eine Shell startet und `setuid(0)/setgid(0)` aufruft, wurde als Shared Object (`/tmp/shell.so`) kompiliert.
    - Durch Ausführen von `sudo LD_PRELOAD=/tmp/shell.so /bin/whoami` wurde eine Root-Shell erlangt.

## Gefundene Flags

- **User Flag (`/home/pentester/user.txt`):** `Flag{c4f9375f9834b4e7f0a528cc65c055702bf5f24a}`
- **Root Flag (`/root/root.txt`):** `Flag{7ca62df5c884cd9a5e5e9602fe01b39f9ebd8c6f}`

## Verwendete Tools (Auswahl)

- `arp-scan` (Netzwerk-Scanner)
- `nmap` (Netzwerk-Mapper)
- `nikto` (Webserver-Scanner)
- `gobuster` (Verzeichnis-/Datei-Bruteforcer)
- `curl` (Datenübertragungstool)
- `hydra` (Passwort-Cracker)
- `git` (Versionskontrollsystem)
- `sjwt.py` (JWT Secret Cracker)
- `jwt_tool.py` (JWT Analyse und Manipulation)
- `nc` (Netcat - für Reverse Shells)
- `find` (Dateisuche)
- `ps` (Prozessstatus)
- `Python` (Skripting, HTTP-Server)
- `sudo` (Rechteausweitung)
- `gcc` (GNU Compiler Collection)
- Standard Linux-Kommandos

---
*Dieser Bericht wurde zu Lern- und Dokumentationszwecken im Rahmen der HackMyVM-Challenge "Tryharder" erstellt.*
