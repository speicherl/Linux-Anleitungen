# TU Darmstadt AnyConnect VPN auf Fedora Linux (KDE Plasma)

Vollständige Anleitung zur automatisierten VPN-Anbindung unter Fedora KDE (Wayland). Diese Lösung nutzt einen sauberen Hintergrunddienst (`systemd`) mit nativer TOTP-Token-Generierung, globalem Tastenkürzel (`Super + V`), Desktop-Benachrichtigungen und einem reaktiven Status-Icon im Systemabschnitt der Taskleiste.

---

### 1. Benötigte Pakete installieren

Installiere OpenConnect, SELinux-Verwaltungswerkzeuge und die Bibliotheken für das Status-Tray:

```bash
sudo dnf install openconnect setroubleshoot-server checkpolicy python3-gobject libayatana-appindicator-gtk3 libnotify
```

---

### 2. Vorbereitung der Zugangsdaten

Halte folgende Zugangsdaten bereit:
* **TU-ID** (z. B. `ab12cdef`)
* **TU-Passwort**
* **TOTP-Secret** (der Base32-Schlüssel aus dem TU-ID-Portal / der Authenticator-App, z. B. `QD3CUOGBKFFD...`)

---

### 3. Passwort geschützt hinterlegen

Erstelle eine Datei, die ausschließlich von `root` gelesen werden kann:

```bash
sudo nano /etc/openconnect-tuda.passwd
```

Schreibe ausschließlich das Kennwort in die Datei (ohne voran- oder nachgestellte Leerzeichen). Speichern mit `Strg + O`, `Enter` und beenden mit `Strg + X`.

Berechtigungen einschränken:

```bash
sudo chmod 600 /etc/openconnect-tuda.passwd
```

---

### 4. Hintergrunddienst (`systemd`) konfigurieren

Erstelle die Service-Unit:

```bash
sudo nano /etc/systemd/system/vpn-tuda.service
```

Füge folgenden Inhalt ein. Ersetze `<DEINE_TU_ID>` und `<DEIN_BASE32_SECRET>` durch deine Werte:

```ini
[Unit]
Description=TU Darmstadt AnyConnect VPN
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
ExecStart=/bin/sh -c 'cat /etc/openconnect-tuda.passwd | /usr/sbin/openconnect -Q 10 --protocol=anyconnect -u <DEINE_TU_ID> --passwd-on-stdin --token-mode totp --token-secret "base32:<DEIN_BASE32_SECRET>" --useragent="AnyConnect" --no-external-auth --authgroup campus vpn.hrz.tu-darmstadt.de'
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

*Hinweis: Der Parameter `-Q 10` verhindert den Kernel-Beschleunigungszugriff auf `/dev/vhost-net`, wodurch SELinux-Blockaden vermieden werden.*

Lade die Konfiguration neu:

```bash
sudo systemctl daemon-reload
```

---

### 5. Passwortlose Dienst-Steuerung erlauben

Damit die Nutzerskripte den Dienst ohne Administrator-Passwortabfrage steuern können:

```bash
sudo tee /etc/sudoers.d/vpn-tuda << 'EOF'
%wheel ALL=(ALL) NOPASSWD: /usr/bin/systemctl start vpn-tuda, /usr/bin/systemctl stop vpn-tuda, /usr/bin/systemctl restart vpn-tuda, /usr/bin/systemctl status vpn-tuda
EOF
```

---

### 6. SELinux-Ausnahme einrichten (Fedora)

OpenConnect schreibt Server-Zertifikate in den lokalen Trust-Store, was SELinux im Hintergrunddienst standardmäßig blockiert. Starte den Dienst einmalig und generiere das Ausnahmemodul:

```bash
# Dienst anstarten, um den Logeintrag zu erzeugen
sudo systemctl start vpn-tuda

# Ausnahme-Modul aus dem Audit-Log erzeugen und laden
sudo ausearch -c 'openconnect' --raw | audit2allow -M my-openconnect
sudo semodule -i my-openconnect.pp

# Temporäre Dateien entfernen
rm -f my-openconnect.te my-openconnect.pp

# Dienst neu starten
sudo systemctl restart vpn-tuda
```

---

### 7. Steuerungsskript (Toggle mit Benachrichtigungen)

Erstelle das Umschalt-Skript:

```bash
mkdir -p ~/.local/bin
cat << 'EOF' > ~/.local/bin/tuda-vpn-toggle.sh
#!/usr/bin/env bash

# 1. Wenn openconnect aktiv läuft oder ein tun-Interface existiert -> Trennen
if pgrep -x openconnect > /dev/null || ip link show up | grep -qE "tun[0-9]+"; then
    sudo systemctl stop vpn-tuda
    notify-send -a "TU Darmstadt VPN" -i network-vpn-disconnected "TU Darmstadt VPN" "Disconnected."
    exit 0
fi

# 2. Ansonsten -> Verbinden
sudo systemctl start vpn-tuda

# Bis zu 10 Sekunden warten, bis das Interface aktiv ist
for _ in {1..10}; do
    if ip link show up | grep -qE "tun[0-9]+"; then
        notify-send -a "TU Darmstadt VPN" -i network-vpn "TU Darmstadt VPN" "Connected successfully!"
        exit 0
    fi
    sleep 1
done

notify-send -a "TU Darmstadt VPN" -i dialog-error "TU Darmstadt VPN" "Connection failed!"
exit 1
EOF

chmod +x ~/.local/bin/tuda-vpn-toggle.sh
```

---

### 8. Dynamisches Status-Icon für die Taskleiste (Wayland SNI)

Erstelle das Python-Tray-Skript:

```bash
cat << 'EOF' > ~/.local/bin/tuda-vpn-tray.py
#!/usr/bin/env bash
''''exec python3 "$0" "$@" #'''

import os
import subprocess
import gi

# Unterdrückt die libayatana Deprecation-Warnung im Terminal
gi.require_version('GLib', '2.0')
from gi.repository import GLib
GLib.log_set_handler("libayatana-appindicator", GLib.LogLevelFlags.LEVEL_WARNING, lambda *args: None)

try:
    gi.require_version('AyatanaAppIndicator3', '0.1')
    from gi.repository import AyatanaAppIndicator3 as AppIndicator
except (ValueError, ImportError):
    gi.require_version('AppIndicator3', '0.1')
    from gi.repository import AppIndicator3 as AppIndicator

gi.require_version('Gtk', '3.0')
from gi.repository import Gtk

# Icon-Definitionen
ICON_CONNECTED = "network-vpn"
ICON_DISCONNECTED = "security-low"

def check_active():
    p = subprocess.run(["pgrep", "-x", "openconnect"], stdout=subprocess.DEVNULL)
    return p.returncode == 0

class VPNTray:
    def __init__(self):
        self.current_state = None

        self.indicator = AppIndicator.Indicator.new(
            "vpn-tuda-tray",
            ICON_DISCONNECTED,
            AppIndicator.IndicatorCategory.APPLICATION_STATUS
        )
        self.indicator.set_status(AppIndicator.IndicatorStatus.ACTIVE)

        self.menu = Gtk.Menu()
        self.item_toggle = Gtk.MenuItem(label="Connect")
        self.item_toggle.connect("activate", self.toggle_vpn)
        self.menu.append(self.item_toggle)

        item_quit = Gtk.MenuItem(label="Quit")
        item_quit.connect("activate", Gtk.main_quit)
        self.menu.append(item_quit)

        self.menu.show_all()
        self.indicator.set_menu(self.menu)

        self.update_ui()
        GLib.timeout_add_seconds(1, self.update_ui)

    def toggle_vpn(self, _):
        toggle_script = os.path.expanduser("~/.local/bin/tuda-vpn-toggle.sh")
        subprocess.Popen([toggle_script])

    def update_ui(self):
        active = check_active()
        if active != self.current_state:
            self.current_state = active
            if active:
                self.indicator.set_icon_full(ICON_CONNECTED, "TU VPN Connected")
                self.item_toggle.set_label("Disconnect")
            else:
                self.indicator.set_icon_full(ICON_DISCONNECTED, "TU VPN Disconnected")
                self.item_toggle.set_label("Connect")
        return True

if __name__ == "__main__":
    Gtk.init([])
    app = VPNTray()
    Gtk.main()
EOF

chmod +x ~/.local/bin/tuda-vpn-tray.py
```

---

### 9. Autostart und Desktop-Starter anlegen

```bash
mkdir -p ~/.config/autostart ~/.local/share/applications

# Starter für KRunner / Startmenü
cat << EOF > ~/.local/share/applications/vpn-tuda-toggle.desktop
[Desktop Entry]
Type=Application
Name=TU VPN Toggle
Comment=Toggle TU Darmstadt VPN Connection
Exec=$HOME/.local/bin/tuda-vpn-toggle.sh
Icon=network-vpn
Categories=Network;
Terminal=false
StartupNotify=false
EOF

# Autostart für das Leisten-Icon
cat << EOF > ~/.config/autostart/vpn-tuda-tray.desktop
[Desktop Entry]
Type=Application
Name=TU VPN Tray
Comment=TU Darmstadt VPN System Tray Indicator
Exec=$HOME/.local/bin/tuda-vpn-tray.py
Terminal=false
X-KDE-autostart-phase=2
EOF

# Absolute Pfade auflösen und KDE-Menüdatenbank aktualisieren
sed -i "s|\$HOME|$HOME|g" ~/.local/share/applications/vpn-tuda-toggle.desktop ~/.config/autostart/vpn-tuda-tray.desktop
kbuildsycoca6 2>/dev/null || kbuildsycoca5 2>/dev/null || update-desktop-database ~/.local/share/applications

# Statusleisten-App starten
~/.local/bin/tuda-vpn-tray.py &
```

---

### 10. Globales Tastenkürzel (Super + V) einrichten (nicht getestet)

1. **Systemeinstellungen** $\rightarrow$ **Tastatur** $\rightarrow$ **Kurzbefehle** öffnen.
2. Unten auf **Neu hinzufügen** $\rightarrow$ **Befehl** klicken.
3. Name eintragen: `TU VPN Toggle`.
4. Befehl eintragen: `/home/<DEIN_NUTZERNAME>/.local/bin/tuda-vpn-toggle.sh` *(Nutzernamen anpassen)*.
5. Tastenkürzel `Super + V` zuweisen und auf **Anwenden** klicken.

---

### Anhang: Symbole anpassen und vorab ansehen

Unter KDE Plasma greift das Skript auf die Icons deines Breeze-Themes zu. Du kannst dir Symbole vorab ohne Skriptänderung in einem Dialogfenster ansehen:

```bash
kdialog --msgbox "Vorschau" --icon <ICON_NAME>
```

**Verfügbare Alternativen:**
* `kdialog --msgbox "Vorschau" --icon security-low` (Schloss mit Warnzeichen)
* `kdialog --msgbox "Vorschau" --icon network-vpn-no-route` (VPN-Symbol mit Kreuz/Sperre)
* `kdialog --msgbox "Vorschau" --icon process-stop` (Auffälliges rotes Stopp-Kreuz)
* `kdialog --msgbox "Vorschau" --icon emblem-unlocked` (Offenes Vorhängeschloss)
* `kdialog --msgbox "Vorschau" --icon dialog-cancel` (Subtiles Schließen-Kreuz)

**Icon im Skript ändern:**
Öffne `~/.local/bin/tuda-vpn-tray.py`, passe den String in `ICON_DISCONNECTED = "..."` an und starte den Prozess neu:

```bash
pkill -f tuda-vpn-tray.py && ~/.local/bin/tuda-vpn-tray.py &
```
