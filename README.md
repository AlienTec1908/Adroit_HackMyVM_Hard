# Adroit - HackMyVM Lösungsweg

![Adroit VM Icon](Adroitpic.png)

Dieses Repository enthält einen Lösungsweg (Walkthrough) für die HackMyVM-Maschine "Adroit".

## Details zur Maschine & zum Writeup

*   **VM-Name:** Adroit
*   **VM-Autor:** DarkSpirit
*   **Plattform:** HackMyVM
*   **Schwierigkeitsgrad (laut Writeup):** Schwer (Hard)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Adroit](https://hackmyvm.eu/machines/machine.php?vm=Adroit)
*   **Autor des Writeups:** DarkSpirit
*   **Original-Link zum Writeup:** [https://alientec1908.github.io/Adroit_HackMyVM_Hard/](https://alientec1908.github.io/Adroit_HackMyVM_Hard/)
*   **Datum des Originalberichts:** 18. Oktober 2022

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `nmap`
*   `ftp`
*   `cat`
*   `jd-gui` (Java Decompiler)
*   `vi` / `nano`
*   `java` (zum Ausführen des Clients und der Shell)
*   `nc` (netcat)
*   `ssh`
*   `ls`
*   `sudo`
*   `wget`
*   `chmod`
*   Standard Linux-Befehle (`id`, `pwd`, `cd`, `grep`, `mv`)

## Zusammenfassung des Lösungswegs

Das Folgende ist eine gekürzte Version der Schritte, die unternommen wurden, um die Maschine zu kompromittieren, basierend auf dem bereitgestellten Writeup.

### 1. Reconnaissance (Aufklärung)

*   Die Ziel-IP `192.168.2.156` wurde mittels `arp-scan -l` identifiziert.
*   Ein `nmap`-Scan ergab offene Ports:
    *   **Port 21/tcp (FTP):** vsftpd 3.0.3. **Anonymer Login erlaubt**. Verzeichnis `pub` sichtbar.
    *   **Port 22/tcp (SSH):** OpenSSH 7.9p1 Debian 10.
    *   **Port 3000/tcp (ppp?):** Unbekannter Dienst.
    *   **Port 3306/tcp (MySQL):** MySQL (unauthorized).
    *   **Port 33060/tcp (mysqlx?):** MySQL X Protocol.

### 2. Initial Access als Benutzer `writer`

1.  **FTP Enumeration:**
    *   Anonymer FTP-Login auf Port 21 war erfolgreich.
    *   Im Verzeichnis `/pub` wurden die Dateien `adroitclient.jar`, `note.txt` und `structure.PNG` heruntergeladen.
2.  **Analyse der heruntergeladenen Dateien:**
    *   `note.txt` enthielt Hinweise auf eine "Java Socket App" und einen kryptischen Hinweis: "one 0 is not 0 but O".
    *   `adroitclient.jar` wurde mit `jd-gui` dekompiliert. Im Quellcode wurden gefunden:
        *   Hardcodierter Secret Key: `Sup3rS3cur3Dr0it` (später mit dem Hinweis zu `Sup3rS3cur3DrOit` korrigiert).
        *   Verbindung zu `adroit.local` auf Port `3000`.
        *   Hardcodierte Zugangsdaten für die Anwendung: `zeus` / `god.thunder.olympus`.
3.  **Interaktion mit der Java-Anwendung und SQL-Injection:**
    *   `adroit.local` wurde der IP `192.168.2.156` in der `/etc/hosts`-Datei des Angreifers zugeordnet.
    *   Der Java-Client (`adroitclient.jar`) wurde gestartet und ein Login mit `zeus:god.thunder.olympus` war erfolgreich.
    *   Die "get"-Funktion der Anwendung war anfällig für **SQL-Injection** im Feld "phrase identifier".
    *   Mittels SQL-Injection wurden Datenbanknamen (`adroit`), Tabellennamen (`users`, `ideas`) und Spaltennamen (`id`, `username`, `password`) ausgelesen.
    *   Die Daten aus der `users`-Tabelle wurden extrahiert: `ID: 1000, Username: writer, Password (Hash/Base64?): l4A+n+p+xSxDcYCl0mgxKr015+OEC3aOfdrWafSqwpY=`.
    *   Das Klartext-Passwort für `writer` wurde als **`just.write.my.ideas`** identifiziert (vermutlich durch Offline-Cracking des Hashes oder einen anderen, nicht explizit gezeigten Hinweis).
4.  **SSH-Login als `writer`:**
    *   Erfolgreicher SSH-Login auf `adroit.hmv` als Benutzer `writer` mit dem Passwort `just.write.my.ideas`.
    *   Die User-Flag wurde in `/home/writer/user.txt` gefunden.

### 3. Privilege Escalation zu `root`

1.  **Sudo-Rechte-Analyse:**
    *   Als Benutzer `writer` wurde `sudo -l` ausgeführt.
    *   Ergebnis: `(root) NOPASSWD: /usr/bin/java -jar /tmp/testingmyapp.jar`.
        *   Der Benutzer `writer` darf den Befehl `java -jar /tmp/testingmyapp.jar` als `root` ohne Passwort ausführen.
2.  **Erstellung einer bösartigen JAR-Datei:**
    *   Ein Shell-Skript (`generate-java-reverse-shell.sh`) wurde von GitHub heruntergeladen, um Java-Code für eine Reverse Shell zu generieren.
    *   Die IP-Adresse des Angreifers (`192.168.2.153`) und ein Listener-Port (`9001`) wurden im generierten Java-Code konfiguriert.
    *   Das Skript kompilierte den Java-Code zu einer JAR-Datei (z.B. `shell_*.jar`).
3.  **Ausnutzung der Sudo-Regel:**
    *   Die generierte JAR-Datei wurde auf dem Zielsystem `adroit` in `/tmp/testingmyapp.jar` umbenannt/kopiert und ausführbar gemacht.
    *   Ein Netcat-Listener wurde auf dem Angreifer-System auf Port `9001` gestartet (`nc -nlvp 9001`).
    *   Auf dem Zielsystem wurde der `sudo`-Befehl ausgeführt: `sudo -u root /usr/bin/java -jar /tmp/testingmyapp.jar`.
4.  **Root-Shell erhalten:**
    *   Die Java-Reverse-Shell wurde als `root` ausgeführt und verband sich zum Listener des Angreifers.
    *   `id` bestätigte `uid=0(root)`.
    *   Die Root-Flag wurde in `/root/root.txt` gefunden.

### 4. Flags

*   **User-Flag (`/home/writer/user.txt`):**
    ```
    61de3a25161dcb2b88b5119457690c3c
    ```
*   **Root-Flag (`/root/root.txt`):**
    ```
    017a030885f25af277dd891d0f151845
    ```

## Haftungsausschluss (Disclaimer)

Dieser Lösungsweg dient zu Bildungszwecken und zur Dokumentation der Lösung für die "Adroit" HackMyVM-Maschine. Die Informationen sollten nur in ethischen und legalen Kontexten verwendet werden, wie z.B. bei CTFs und autorisierten Penetrationstests.
