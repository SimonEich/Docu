# Raspberry Pi Projekt: Befehlsreferenz & Spickzettel

Dieses Dokument enthält alle wichtigen Befehle für die Verwaltung deines Raspberry Pi, Docker-Container und die Entwicklung deines Mexiko-Projekts.

---

## 1. Docker & Container Management

Diese Befehle steuern deine Apps auf dem Pi.

| **Befehl** | **Erklärung** |
| --- | --- |
| `docker ps` | Zeigt alle aktuell laufenden Container an. |
| `docker ps -a` | Zeigt alle Container an (auch gestoppte). |
| `docker logs -f [name]` | Zeigt die Live-Ausgabe (Logs) eines Containers (wichtig zur Fehlersuche). |
| `docker stop [name]` | Stoppt einen laufenden Container. |
| `docker start [name]` | Startet einen gestoppten Container wieder. |
| `docker restart [name]` | Startet einen Container neu. |
| `docker rm [name]` | Löscht einen gestoppten Container permanent. |
| `docker run -d --name [name] [image]` | Startet ein neues Image als Container im Hintergrund. |

---

## 2. Docker Build & Deployment (MacBook)

Befehle zum Vorbereiten und Hochladen der Software auf dem MacBook Air.

| **Befehl** | **Erklärung** |
| --- | --- |
| `docker login ghcr.io -u [Username]` | Anmeldung bei der GitHub Container Registry (erfordert Token). |
| `docker buildx build --platform linux/arm64 -t ghcr.io/[user]/[repo]:latest --push .` | Baut das Image für die Pi-Architektur und lädt es hoch. |
| `docker image ls` | Listet lokal gespeicherte Images auf. |
| `docker image prune` | Bereinigt ungenutzte Images (spart Speicherplatz auf dem Mac). |

---

## 3. System-Konfiguration & SSH

Verwaltung des Pi ohne Bildschirm (Headless).

| **Befehl** | **Erklärung** |
| --- | --- |
| `ssh pi@mexiko-pi.local` | Verbindet dich mit dem Pi über den Netzwerknamen. |
| `sudo raspi-config` | Das grafische Konfigurationsmenü (WLAN, Boot-Optionen, Hostname). |
| `sudo reboot` | Startet den Pi neu. |
| `sudo shutdown -h now` | Fährt den Pi sicher herunter. |
| `hostname -I` | Zeigt die aktuelle IP-Adresse im Netzwerk. |
| `vcgencmd measure_temp` | Zeigt die aktuelle CPU-Temperatur an. |

---

## 4. Linux Terminal Basics

Dateiverwaltung direkt auf dem System.

| **Befehl** | **Erklärung** |
| --- | --- |
| `ls -la` | Zeigt alle Dateien im Ordner inkl. versteckter Dateien. |
| `cd [pfad]` | Wechselt das Verzeichnis. |
| `nano [datei]` | Einfacher Texteditor zum Bearbeiten von Skripten. |
| `cat [datei]` | Gibt den Inhalt einer Datei im Terminal aus. |
| `df -h` | Überprüft den freien Speicherplatz auf der SD-Karte. |

---

## 5. Tipps für die Reise (Mexiko)

- **Netzwerk:** Nutze die App **Fing** auf dem Handy, um die IP des Pi zu finden, falls `.local` nicht funktioniert.
- **Powerbank:** Schalte das Display aus und nutze den Console-Mode (`raspi-config`), um die Laufzeit zu verdoppeln.
- **Hotspot:** Trage die Zugangsdaten deines Handy-Hotspots in die `/etc/wpa_supplicant/wpa_supplicant.conf` ein als Backup zum Hotel-WLAN.
