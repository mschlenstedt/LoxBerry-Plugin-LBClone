# lbclone

Einen LoxBerry klonen, ohne ein Image der ganzen SD-Karte oder Platte zu ziehen.

`lbclone create` erfasst einen laufenden LoxBerry in einem Archiv. `lbclone apply`
spielt dieses Archiv auf ein frisches DietPi – als Ersatz für die alte Box (neue
Hardware, neue SD-Karte) oder als zweite Box.

Das Betriebssystem wird dabei nicht kopiert. `apply` installiert LoxBerry mit dem
offiziellen Installer in genau der Version der Quellbox neu, installiert die Pakete
der Quellbox nach und spielt danach Konfiguration und Daten zurück. Plugins mit
Autoupdate installiert es anschließend über den normalen LoxBerry-Mechanismus neu.
Das Ergebnis ist ein System, das so aufgebaut ist wie bei einer Neuinstallation von
Hand, aber den Zustand der Quellbox hat.

lbclone ist eine einzige Bash-Datei. Es gibt nichts zu installieren.

## Voraussetzungen

- Quelle und Ziel: DietPi mit **Debian 13 (trixie)** oder neuer und **LoxBerry 4**
  oder neuer.
- Ziel: ein **frisches DietPi** mit derselben Debian-Version (Codename) wie die Quelle,
  noch ohne LoxBerry. Eine abweichende Debian-Subversion (z. B. 13.4 → 13.7) ist
  erlaubt und wird nur als Warnung gemeldet.
- Die Hardware darf eine andere sein, auch die Architektur – siehe
  [Andere Hardware](#andere-hardware).
- Beide Kommandos laufen als `root`.
- Das Ziel braucht Internet: Installer, Pakete und Plugins werden heruntergeladen.
- Fehlende Werkzeuge (`jq`, 7-Zip) installiert lbclone selbst per apt nach.

## Herunterladen

```sh
wget https://raw.githubusercontent.com/mschlenstedt/LoxBerry-Plugin-LBClone/main/lbclone
# oder
curl -fsSLO https://raw.githubusercontent.com/mschlenstedt/LoxBerry-Plugin-LBClone/main/lbclone
```

Ausgeführt wird die Datei mit `bash lbclone …` – sie muss nicht ausführbar sein.

## Clone erstellen (auf der Quellbox)

```sh
bash lbclone create --output /media/usb/clones
```

Das Ergebnis ist ein Archiv `lbclone-loxberry-<version>-<datum>.7z`. Darin liegen
zwei Dateien:

| Datei | Inhalt |
|---|---|
| `lbclone` | dieses Skript, damit das Archiv alles für die Wiederherstellung enthält |
| `loxberry.tar` | der eigentliche Clone |

Die Dienste der Box laufen während der Sicherung weiter. Mosquitto schreibt seine
Datenbank vorher auf die Platte, MariaDB wird per Dump gesichert.

Das Zielverzeichnis darf nicht unter `/tmp` liegen (tmpfs, beim Neustart leer).

### Verschlüsseln

Ein Clone enthält Zugangsdaten, Passwort-Hashes und Schlüssel. Sobald er den LoxBerry
verlässt, sollte er verschlüsselt sein. Dafür nutzt lbclone die AES256-Verschlüsselung
von 7-Zip und verschlüsselt das **ganze Archiv**:

```sh
bash lbclone create --output /media/usb/clones --encrypt              # fragt das Passwort ab
bash lbclone create --output /media/usb/clones --password 'geheim'    # Passwort direkt
```

- Nur mit den Formaten `7z` (Standard) und `zip`, nicht mit `gz`.
- Bei `7z` sind auch die Dateinamen im Archiv verschlüsselt.
- Beim Entpacken fragt 7-Zip nach dem Passwort – auf dem LoxBerry mit `7z x` ebenso wie
  unter Windows. Danach liegen `lbclone` und `loxberry.tar` unverschlüsselt vor, und
  `bash lbclone apply` braucht kein Passwort mehr.
- Ein mit `--password` übergebenes Passwort steht in der Shell-History und ist während
  des Laufs in der Prozessliste sichtbar. `--encrypt` fragt es stattdessen verdeckt ab.

### Optionen für `create`

| Option | Wirkung |
|---|---|
| `--output <verzeichnis>` | Zielverzeichnis für das Archiv. Pflicht. |
| `--compress 7z\|zip\|gz` | Archivformat, Standard `7z`. `7z` und `zip` erzeugt 7-Zip, `gz` ein `.tar.gz`. Alle drei lassen sich auch unter Windows entpacken. |
| `--password <passwort>` | Verschlüsselt das ganze Archiv mit 7-Zip (AES256). Nur mit `7z` und `zip`. |
| `--encrypt` | Wie `--password`, fragt das Passwort aber verdeckt am Terminal ab. |
| `--exclude-plugin-data <folder>:<pfad>` | Lässt einen Pfad unter `data/plugins/<folder>/` weg, z. B. große Aufzeichnungen. Mehrfach verwendbar. |
| `--no-logs` | Lässt die Logs weg. |
| `--stop-service <name>` | Hält diesen Dienst für die Dauer der Sicherung an. Mehrfach verwendbar. |
| `--config <datei>` | Voreinstellungen aus einer Datei. Standard: `$LBHOMEDIR/config/system/lbclone.cfg`, falls vorhanden. Vorlage: `bash lbclone config-example`. |
| `--dry-run` | Zeigt Umfang und geschätzte Größe, schreibt nichts. |

## Clone einspielen (auf dem frischen DietPi)

1. Archiv auf das Ziel kopieren und dort entpacken – oder unter Windows entpacken
   und die beiden Dateien kopieren. Der Ordner darf **nicht** unter `/tmp` oder
   `/opt` liegen, z. B. `/root`:

   ```sh
   cd /root
   7z x lbclone-loxberry-4.0.0.15-20260925-115213.7z
   ```

2. Einspielen:

   ```sh
   bash lbclone apply
   ```

   Ohne `--input` nimmt lbclone das `loxberry.tar`, das neben ihm liegt. Mit `--input`
   geht auch das ganze Archiv direkt; ist es verschlüsselt, fragt lbclone nach dem
   Passwort (oder `--password`).

Zuerst prüft lbclone, ob Ziel und Clone zusammenpassen (Debian-Version,
DietPi, kein LoxBerry installiert, freier Platz, Installer und Release
auf GitHub vorhanden). Passt etwas nicht, bricht es ab, bevor es irgendetwas ändert.

Danach läuft der Restore selbständig durch und startet die Box am Ende einmal neu.
Jeder Schritt wird im Klartext mit Fortschritt angezeigt, z. B.
`Schritt 5 von 12: Pakete der Quellbox installieren`. Die Installation dauert auf
einem Raspberry bis zu einer Stunde.

**Die SSH-Verbindung darf abreißen.** lbclone löst sich nach der Prüfung von der
Sitzung und läuft im Hintergrund weiter. Nach dem erneuten Einloggen zeigt
`bash lbclone apply` wieder den Fortschritt. `Strg+C` beendet nur die Anzeige,
nicht den Restore.

Bricht ein Schritt mit einem Fehler ab, setzt `bash lbclone apply --resume` nach der
Behebung beim offenen Schritt fort.

Das Protokoll liegt während des Restores in `/var/log/lbclone/apply.log`, danach in
`$LBHOMEDIR/log/system/lbclone/`. Am Ende steht ein Bericht über die Plugins und
alle Warnungen.

### Ablauf

| Schritt | Was passiert |
|---|---|
| 1 | LoxBerry mit dem offiziellen Installer in der Version der Quellbox installieren |
| 2 | LoxBerry-Dienste anhalten |
| 3 | Paketquellen der Quellbox übernehmen, Paketlisten aktualisieren |
| 4 | Fehlende DietPi-Software nachinstallieren |
| 5 | Pakete der Quellbox installieren |
| 6 | Benutzer, Gruppen und Passwörter übernehmen |
| 7 | Dateien aus dem Clone zurückspielen: `/etc`, LoxBerry mit Plugins und Daten, `/usr/local`, Teile von `/var` |
| 8 | System anpassen: Hostname, neue LoxBerry-ID, Zeitzone, MQTT-Passwörter, Dateirechte |
| 9 | Dienste wieder starten |
| 10 | MariaDB-Datenbanken einspielen |
| 11 | Plugins neu installieren |
| 12 | Bericht ausgeben, Log ablegen, aufräumen, Neustart |

### Optionen für `apply`

| Option | Wirkung |
|---|---|
| `--input <clone>` | Das ganze Archiv (`.7z`, `.zip`, `.tar.gz`) oder das entpackte `loxberry.tar`. |
| `--password <passwort>` | Passwort für ein verschlüsseltes Archiv, das mit `--input` direkt übergeben wird. Ohne Angabe fragt lbclone nach. Es wird nirgends gespeichert. |
| `--hostname <name>` | Neuer Hostname. Macht den Clone zur zweiten Box: die SSH-Host-Keys werden neu erzeugt. Achtung: der Hostname ist Präfix der Topics des MQTT-Gateways. |
| `--lbname <name>` | Neuer LoxBerry-Name, unabhängig vom Hostnamen. |
| `--update-core` | Nach dem Einspielen und vor der Plugin-Neuinstallation LoxBerry auf das neueste Release aktualisieren (für ältere Clones). |
| `--force-plugin-reinstall` | Auch Plugins mit Autoupdate-Stufe 1 oder 2 neu installieren. |
| `--no-plugin-reinstall` | Kein Plugin neu installieren, nur den Zustand zurückspielen. |
| `--skip-plugin <folder>` | Dieses Plugin nicht neu installieren. Mehrfach verwendbar. |
| `--skip-install` | LoxBerry nicht installieren; setzt einen bereits installierten LoxBerry in derselben Version voraus. |
| `--allow-core-mismatch` | Nur mit `--skip-install`: erlaubt eine abweichende LoxBerry-Version und spielt dann nur Konfiguration und Daten zurück. Nicht erprobt. |
| `--github-auth <user:token>` | Wird an den Installer durchgereicht, gegen das Rate-Limit der GitHub-API. |
| `--no-reboot` | Am Ende nicht neu starten. |
| `--dry-run` | Nur prüfen und den Plan zeigen, nichts ändern. |
| `--resume` | Einen abgebrochenen Restore fortsetzen. |

## Andere Hardware

LoxBerry ist hardwareunabhängig. Ein Clone lässt sich deshalb auch auf andere Hardware
einspielen, z. B. vom Raspberry Pi 4 auf einen Pi 5 oder von einer VM auf einen Mini-PC.

- Was zur Hardware gehört, kommt vom Installer der neuen Box: die Hardware-Merker von
  LoxBerry (`config/system/is_*.cfg`), Kernelmodule und udev-Regeln (`/etc/modprobe.d`,
  `/etc/modules-load.d`, `/etc/udev/rules.d`). Hardwaregebundene Pakete wie Kernel,
  Bootloader und Firmware werden nicht übertragen.
- Unter `/usr/local` überschreibt lbclone nichts, was der Installer dort schon angelegt
  hat.

**Architekturwechsel** (z. B. Raspberry Pi mit `arm64` ↔ PC mit `amd64`) ist möglich,
aber ein Risiko – lbclone warnt davor deutlich. Programme und Bibliotheken aus dem Clone
passen dann nicht zur neuen Box. lbclone spielt sie nicht ein (beim Erstellen notiert es,
welche Dateien das sind) und installiert stattdessen **alle** Plugins mit
Download-Adresse neu, unabhängig von ihrer Autoupdate-Einstellung. Plugins ohne
Download-Adresse oder ohne Unterstützung für die neue Architektur funktionieren danach
vermutlich nicht; der Bericht nennt sie. Ein Architekturwechsel braucht einen Clone, der
mit der aktuellen Version von lbclone erstellt wurde.

## Weitere Kommandos

| Kommando | Wirkung |
|---|---|
| `bash lbclone info [--input <clone>]` | Zeigt, was im Clone steckt: Box, Versionen, Installer, Plugins und was beim Einspielen mit ihnen passiert. |
| `bash lbclone verify [--input <clone>]` | Prüft das Archiv vollständig und die Prüfsummen. |
| `bash lbclone config-example` | Vorlage für `lbclone.cfg` ausgeben. |

Alle Kommandos kennen `-v` bzw. `--verbose` für eine sehr ausführliche Ausgabe.

## Plugins

Die Dateien aller Plugins kommen mit dem Clone zurück. Neu installiert wird ein
Plugin je nach seiner Autoupdate-Stufe in der Plugin-Verwaltung:

| Autoupdate | Beim Einspielen |
|---|---|
| 3 oder 4 | wird neu installiert – in derselben Version, oder neuer, falls es eine gibt |
| 1 oder 2 | bleibt im zurückgespielten Stand; mit `--force-plugin-reinstall` neu installiert |
| 0 (aus) | bleibt im zurückgespielten Stand |

Schlägt die Neuinstallation fehl (Download nicht möglich, Installation mit Fehler),
bleibt das Plugin im zurückgespielten Stand und läuft weiter. Der Bericht am Ende
nennt es; es kann danach über die WebUI erneut installiert werden. lbclone schreibt
nie selbst in die Plugin-Datenbank.

## Was nicht geklont wird

- das Betriebssystem selbst – es kommt vom frischen DietPi
- Maschinen-ID, Netzwerkkonfiguration, `fstab` und die Benutzerdatenbank der neuen
  Box (Benutzer und Passwörter werden gezielt übernommen)
- die LoxBerry-ID – der Clone bekommt eine neue, damit Quelle und Clone
  unterscheidbar bleiben

## Exitcodes

| Code | Bedeutung |
|---|---|
| 0 | alles in Ordnung |
| 1 | fertig, aber mit Warnungen (siehe Bericht) |
| 2 | Voraussetzung nicht erfüllt – nichts geändert |
| 3 | Abbruch während des Laufs |

## Lizenz

Apache License 2.0 – siehe [LICENSE](LICENSE).
