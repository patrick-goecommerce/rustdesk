# Design: go Remote – Custom RustDesk Client

**Datum:** 2026-02-26
**Status:** Freigegeben

---

## Ziel

Eigener White-Label Remote-Support-Client "go Remote" für Kundensupport,
basierend auf einem Fork des Open-Source RustDesk-Projekts.
Kunden laden den Installer von der eigenen Website herunter und erhalten
automatisch Zugang zum eigenen Server – ohne manuelle Server-Konfiguration.

---

## Branding

| Parameter       | Wert                                         |
|-----------------|----------------------------------------------|
| App-Name        | `go Remote`                                  |
| Hintergrund     | `#121415`                                    |
| Buttons/Akzent  | `#F06800`                                    |
| Schriftfarbe    | `#FFFFFF`                                    |
| Logo-Quelle     | `logo.png` (500×210px, RGBA)                 |
| Icon Windows    | `res/icon.ico` (16–256px, bereits erstellt)  |
| Icon macOS      | `res/AppIcon.iconset/` (PNGs, bereits erstellt) |

---

## Server

| Parameter    | Wert                                          |
|--------------|-----------------------------------------------|
| Server-URL   | `goremote.go-ecommerce.de`                    |
| hbbs Port    | `21116` (TCP + UDP)                           |
| hbbr Port    | `21117`                                       |
| RS_PUB_KEY   | `BgxcHHWnIrhCy2ObBqKR7mg8REpGgrU3ZexkI6GKzA8=` |
| Server-Typ   | Standard Open Source (rustdesk/rustdesk-server:latest) |

Der RS_PUB_KEY ist der öffentliche Ed25519-Schlüssel des hbbs-Servers.
Er wird in den Client eingebaut, damit dieser ausschließlich mit dem
eigenen Server kommuniziert (verhindert Man-in-the-Middle).

---

## Architektur

```
GitHub Repo (Fork von rustdesk/rustdesk)
├── Branding-Änderungen (Icons, Farben, App-Name, Server-Config)
├── .github/workflows/
│   ├── build-windows.yml   → .exe NSIS Installer
│   └── build-macos.yml     → .dmg
└── GitHub Release
    ├── goremote-setup.exe  → Download für Windows-Kunden
    └── goremote.dmg        → Download für macOS-Kunden
```

**Build-Trigger:** `git tag v1.x.x` → GitHub Actions baut automatisch
beide Plattformen und legt die Installer als Release-Assets ab.

---

## Zu ändernde Dateien

### 1. Server-Konfiguration
- **`src/config.rs`** – `RENDEZVOUS_SERVER` und `RS_PUB_KEY` hardcoden

### 2. App-Name
- **`Cargo.toml`** – `name = "goremote"`
- **`flutter/pubspec.yaml`** – `name: goremote`, `description: go Remote`
- **`flutter/windows/runner/CMakeLists.txt`** – `set(BINARY_NAME "goremote")`
- **`flutter/macos/Runner/Info.plist`** – `CFBundleName`, `CFBundleDisplayName`

### 3. Icons
- **`res/icon.ico`** – Windows App-Icon (bereits erstellt)
- **`res/AppIcon.iconset/`** – macOS Icon-PNGs (bereits erstellt)
- **`flutter/assets/`** – Logo für Splash/About-Screen
- **`flutter/windows/runner/resources/app_icon.ico`** – Windows Runner-Icon
- **`flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/`** – macOS Assets

### 4. Farben / Theme (Flutter)
- **`flutter/lib/`** – Theme-Definitionen:
  - `primaryColor: Color(0xFF121415)`
  - `accentColor / buttonColor: Color(0xFFF06800)`
  - `textColor: Color(0xFFFFFFFF)`

### 5. CI/CD (neu)
- **`.github/workflows/build-windows.yml`** – Windows Build + NSIS Installer
- **`.github/workflows/build-macos.yml`** – macOS Build + DMG

---

## Plattformen

| Plattform | Output          | Runner                  |
|-----------|-----------------|-------------------------|
| Windows   | NSIS `.exe`     | `windows-latest`        |
| macOS     | `.dmg`          | `macos-latest`          |

---

## Nutzungsmodus

- **Unattended**: Client läuft als Windows-Dienst / macOS LaunchDaemon im
  Hintergrund, Autostart bei Systemstart
- **Interaktiv**: Kunde kann Client öffnen, ID + Passwort einsehen und
  für einmalige Sessions freigeben

Beides wird von RustDesk out of the box unterstützt.

---

## Nicht im Scope

- Mobile Apps (Android/iOS)
- Web-Client
- RustDesk Pro Features (Adressbuch, Web-Konsole)
- Automatisches Update-System (v1 – manuelle Installer-Downloads)
