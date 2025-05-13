# Memories - HackMyVM (Medium)

![Memories.png](Memories.png)

## Übersicht

*   **VM:** Memories
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Memories)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 12. November 2022
*   **Original-Writeup:** https://alientec1908.github.io/Memories_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Memories" zu erlangen. Der initiale Zugriff erfolgte durch das Erraten von Standard-HTTP-Basic-Authentication-Credentials (`admin:admin`) für ein geschütztes Verzeichnis (`/memories`). Auf der zugänglichen Seite wurde der private SSH-Schlüssel für den Benutzer `laura` gefunden, was einen SSH-Login ermöglichte. Die erste Rechteausweitung zum Benutzer `lucy` gelang durch Ausnutzung einer unsicheren `sudo`-Regel, die `laura` erlaubte, `/usr/bin/whiptail` als `lucy` auszuführen, um Lucys privaten SSH-Schlüssel zu lesen. Die finale Eskalation zu Root erfolgte durch Ausnutzung einer weiteren unsicheren `sudo`-Regel, die `lucy` erlaubte, `/usr/bin/gcore` als Root auszuführen. Damit konnte ein Core-Dump eines Root-Prozesses (`/root/memories`) erstellt werden, aus dem das Root-Passwort im Klartext extrahiert wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi` / `nano`
*   `nmap`
*   `gobuster`
*   `nikto`
*   Burp Suite (impliziert für Basic Auth)
*   `base64`
*   `chmod`
*   `ssh`
*   `sudo`
*   `whiptail`
*   `gcore`
*   `strings`
*   `ps`
*   `grep`
*   `awk`
*   `xargs`
*   Bash Scripting (while loop)
*   Standard Linux-Befehle (`ls`, `cat`, `su`, `cd`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Memories" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.110) mit `arp-scan` identifiziert. Hostname `memories.hmv` in `/etc/hosts` eingetragen.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 7.9p1) und Port 80 (HTTP, Apache 2.4.38).
    *   `gobuster` fand das Verzeichnis `/memories`, das mit HTTP Basic Authentication (Status 401) geschützt war.

2.  **Initial Access (Basic Auth & SSH als `laura`):**
    *   Durch Testen von Standard-Credentials (`admin:admin`) wurde Zugriff auf das Verzeichnis `/memories` erlangt.
    *   Auf der Seite `/memories` wurde der Benutzername `laura` und ein zugehöriger privater SSH-Schlüssel im Klartext gefunden.
    *   Der SSH-Schlüssel wurde gespeichert und mit `ssh -i laurarsa laura@memories.hmv` wurde erfolgreich eine SSH-Verbindung als `laura` hergestellt.

3.  **Privilege Escalation (von `laura` zu `lucy` via `sudo whiptail`):**
    *   `sudo -l` als `laura` zeigte, dass der Befehl `/usr/bin/whiptail` als Benutzer `lucy` ohne Passwort ausgeführt werden durfte: `(lucy) NPASSWD: /usr/bin/whiptail`.
    *   Mittels `sudo -u lucy /usr/bin/whiptail --textbox /home/lucy/.ssh/id_rsa 40 80` wurde der private SSH-Schlüssel des Benutzers `lucy` ausgelesen.
    *   Der Schlüssel von `lucy` wurde gespeichert und mit `ssh -i lucyrsa lucy@memories.hmv` wurde erfolgreich eine SSH-Verbindung als `lucy` hergestellt.
    *   Die User-Flag (`imissingsomething`) wurde in `/home/lucy/user.txt` gefunden.

4.  **Privilege Escalation (von `lucy` zu `root` via `sudo gcore`):**
    *   `sudo -l` als `lucy` zeigte, dass der Befehl `/usr/bin/gcore` als jeder Benutzer (einschließlich Root) ohne Passwort ausgeführt werden durfte: `(ALL : ALL) NPASSWD: /usr/bin/gcore`.
    *   Ein `while true`-Loop wurde gestartet, um kontinuierlich zu versuchen, einen Core-Dump von einem Prozess namens "memories" (wahrscheinlich `/root/memories`) zu erstellen: `while true ; do ps -aux | grep memories | awk {'print $2'} | xargs sudo /usr/bin/gcore ; done`.
    *   Nachdem erfolgreich ein Core-Dump (z.B. `core.15355`) erstellt wurde, wurde dieser mit `strings core.15355` analysiert.
    *   In den Strings wurde das Root-Passwort im Klartext gefunden: `whataboutyourthinking`.
    *   Mit `su root` und dem Passwort `whataboutyourthinking` wurde Root-Zugriff erlangt.
    *   Die Root-Flag (`HMVtakingthisgames`) wurde in `/root/ro0t.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Schwache HTTP Basic Authentication Credentials:** Das Verzeichnis `/memories` war mit den Standard-Credentials `admin:admin` geschützt.
*   **Preisgabe privater SSH-Schlüssel:** Ein privater SSH-Schlüssel für den Benutzer `laura` wurde auf einer Webseite im Klartext offengelegt.
*   **Unsichere `sudo`-Regeln:**
    *   `laura` durfte `whiptail` als `lucy` ausführen. Da `whiptail --textbox <datei>` Dateiinhalte anzeigt, konnte dies zum Lesen von Lucys privatem SSH-Schlüssel missbraucht werden.
    *   `lucy` durfte `gcore` als `root` ausführen. Da `gcore` Speicherabbilder von Prozessen erstellt, konnte dies genutzt werden, um einen Dump eines Root-Prozesses zu erzeugen und daraus Klartext-Passwörter zu extrahieren.
*   **Klartextpasswörter im Speicher:** Das Root-Passwort war im Speicher eines laufenden Prozesses (`/root/memories`) vorhanden und konnte über einen Core-Dump ausgelesen werden.

## Flags

*   **User Flag (`/home/lucy/user.txt`):** `imissingsomething`
*   **Root Flag (`/root/ro0t.txt`):** `HMVtakingthisgames`

## Tags

`HackMyVM`, `Memories`, `Medium`, `HTTP Basic Auth`, `SSH Key Leak`, `sudo Exploit`, `whiptail Exploit`, `gcore Exploit`, `Memory Dump`, `Linux`, `Web`, `Privilege Escalation`, `Apache`
