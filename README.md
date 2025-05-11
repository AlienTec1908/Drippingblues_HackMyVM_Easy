# Drippingblues - HackMyVM (Easy)

![Drippingblues.png](Drippingblues.png)

## Übersicht

*   **VM:** Drippingblues
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Drippingblues)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 15. April 2023
*   **Original-Writeup:** https://alientec1908.github.io/Drippingblues_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser "Easy"-Challenge war es, Root-Zugriff auf der Maschine "Drippingblues" zu erlangen. Die Enumeration deckte einen FTP-Server mit anonymem Zugriff und einen Apache-Webserver auf. Vom FTP-Server wurde eine passwortgeschützte ZIP-Datei (`respectmydrip.zip`) heruntergeladen. Das Passwort (`072528035`) wurde mit `zip2john` und `john` geknackt. Die ZIP-Datei enthielt einen Hinweis ("just focus on 'drip'") und eine weitere passwortgeschützte Datei (`secret.zip`), die nicht geknackt werden konnte. Auf dem Webserver enthielt die `robots.txt` Hinweise auf `/dripisreal.txt` und `/etc/dripispowerful.html`. Eine LFI-Schwachstelle wurde im GET-Parameter `drip` der `index.php` identifiziert. Durch Ausnutzung dieser LFI (`?drip=/etc/dripispowerful.html`) wurde das SSH-Passwort (`imdrippinbiatch`) für den Benutzer `thugger` gefunden. Nach dem SSH-Login als `thugger` wurde die User-Flag gelesen. Die Privilegieneskalation zu Root erfolgte (laut Hinweis im Originaltext) über eine Methode aus GTFOBins, wobei der genaue ausgenutzte SUID-Befehl nicht im Log dokumentiert wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `ftp`
*   `unzip`
*   `zip2john`
*   `john` (John the Ripper)
*   `gobuster`
*   `wfuzz`
*   `ssh`
*   `find`
*   Standard Linux-Befehle (`cat`, `echo`, `md5sum`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Drippingblues" gliederte sich in folgende Phasen:

1.  **Reconnaissance & FTP Enumeration:**
    *   IP-Findung mittels `arp-scan` (Ziel: `192.168.2.114`, Hostname `drippingblues` oder `cve.hmv`).
    *   `nmap`-Scan identifizierte FTP (21/tcp, vsftpd 3.0.3, anonymer Login erlaubt), SSH (22/tcp) und Apache (80/tcp). `robots.txt` auf Port 80 verwies auf `/dripisreal.txt` und `/etc/dripispowerful.html`.
    *   Vom anonymen FTP-Server wurde die Datei `respectmydrip.zip` heruntergeladen.
    *   Die ZIP-Datei war passwortgeschützt. Mit `zip2john` und `john` wurde das Passwort zu `072528035` geknackt.
    *   Die entpackte Datei enthielt `respectmydrip.txt` (Hinweis: "just focus on 'drip'") und eine weitere passwortgeschützte `secret.zip` (Knackversuch erfolglos).

2.  **Web Enumeration & Information Disclosure (LFI):**
    *   `gobuster` auf Port 80 fand `index.php` und `robots.txt`.
    *   Der Inhalt von `/dripisreal.txt` (gefunden via `robots.txt`) gab einen Hinweis auf MD5-Hashing, der zu einem falschen SSH-Passwort führte.
    *   `wfuzz` wurde verwendet, um GET-Parameter auf `index.php` zu fuzzten, wobei der Parameter `drip` als potenziell anfällig für LFI identifiziert wurde.
    *   Mittels LFI (`http://192.168.2.114/index.php?drip=/etc/passwd`) wurde der Benutzer `thugger` bestätigt.
    *   Mittels LFI (`http://192.168.2.114/index.php?drip=/etc/dripispowerful.html`) wurde das Klartext-Passwort `imdrippinbiatch` gefunden.

3.  **Initial Access (als `thugger` via SSH):**
    *   Ein SSH-Login als `thugger` mit dem Passwort `imdrippinbiatch` war erfolgreich.

4.  **Privilege Escalation (von `thugger` zu `root` - Methode nicht explizit dokumentiert):**
    *   Als `thugger` wurde die User-Flag gelesen.
    *   `sudo -l` zeigte keine direkten sudo-Rechte.
    *   Eine Suche nach SUID-Dateien wurde durchgeführt.
    *   Der Originalbericht erwähnt, dass ein Polkit-Exploit (CVE-2021-3560) fehlschlug und die Eskalation stattdessen über eine Methode aus GTFOBins erfolgte. Der spezifische Befehl oder das ausgenutzte SUID-Binary wurde nicht im Log dokumentiert.
    *   Anschließend wurde die Root-Flag gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Anonymer FTP-Zugriff mit passwortgeschützten Dateien:** Obwohl der direkte Inhalt nicht sofort zugänglich war, lieferte die ZIP-Datei einen Hinweis.
*   **Schwaches ZIP-Passwort:** Das Passwort `072528035` konnte leicht geknackt werden.
*   **Information Disclosure in `robots.txt`:** Enthielt Pfade zu sensiblen Informationen.
*   **Local File Inclusion (LFI):** Im `drip`-Parameter der `index.php` ermöglichte das Auslesen von Dateien, einschließlich einer Datei mit einem Klartext-Passwort.
*   **Klartext-Passwort in Datei:** Das SSH-Passwort für `thugger` war in `/etc/dripispowerful.html` gespeichert.
*   **SUID Binary Exploit (vermutet via GTFOBins):** Der genaue Mechanismus ist nicht dokumentiert, aber die Eskalation erfolgte über eine SUID-Fehlkonfiguration.

## Flags

*   **User Flag (`/home/thugger/user.txt`):** `5C50FC503A2ABE93B4C5EE3425496521`
*   **Root Flag (`/root/root.txt` - Pfad nicht explizit im Log, aber Standard):** `5C42D6BB0EE9CE4CB7E7349652C45C4A`

## Tags

`HackMyVM`, `Drippingblues`, `Easy`, `FTP`, `ZIP Cracking`, `John the Ripper`, `Web`, `Apache`, `robots.txt`, `LFI`, `Information Disclosure`, `SSH`, `SUID Binary`, `GTFOBins`, `Privilege Escalation`, `Linux`
