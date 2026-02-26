# go Remote – Custom RustDesk Client Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fork RustDesk, brand it as "go Remote" (Logo, Farben, App-Name) und hardcode den eigenen Server – Ergebnis sind automatisch gebaute Windows (.exe) + macOS (.dmg) Installer via GitHub Actions.

**Architecture:** Fork von rustdesk/rustdesk in dieses Repo; Branding-Änderungen direkt im Fork-Source (Rust + Flutter); GitHub Actions Workflows triggern bei `git tag v*` und erzeugen Release-Installer.

**Tech Stack:** Rust (cargo), Flutter (Dart), GitHub Actions, NSIS (Windows Installer), create-dmg (macOS), Python/Pillow (Icon-Konvertierung – bereits erledigt)

---

## Voraussetzungen

- GitHub Account mit Zugriff auf dieses Repo
- `gh` CLI installiert und eingeloggt (`gh auth login`)
- Git konfiguriert
- Für lokale Builds (optional): Rust toolchain + Flutter SDK

---

## Task 1: RustDesk Fork einrichten

**Ziel:** Den RustDesk-Quellcode in dieses Repo holen, Upstream-Remote setzen.

**Files:**
- Modify: `.git/config` (via git remote)

**Step 1: Fork via GitHub CLI**

```bash
gh repo fork rustdesk/rustdesk --clone=false
```

Das erzeugt `github.com/DEIN_USER/rustdesk` auf GitHub.

**Step 2: Aktuellen Inhalt sichern**

```bash
cd D:/repos/goremote
git stash  # Falls ungestagete Änderungen vorhanden
```

**Step 3: Fork als zweiten Remote hinzufügen und pullen**

```bash
# Ersetze DEIN_USER mit deinem GitHub-Nutzernamen
git remote add fork https://github.com/DEIN_USER/rustdesk.git
git remote add upstream https://github.com/rustdesk/rustdesk.git
git fetch fork
git checkout -b rustdesk-main fork/master
git checkout main
git merge rustdesk-main --allow-unrelated-histories -m "feat: merge RustDesk source as base"
```

**Step 4: Verifizieren**

```bash
ls src/ flutter/ Cargo.toml
```

Erwartete Ausgabe: `src/  flutter/  Cargo.toml` vorhanden.

**Step 5: Push**

```bash
git remote set-url origin https://github.com/DEIN_USER/rustdesk.git
git push origin main
```

**Step 6: Commit** (bereits via merge-commit)

---

## Task 2: Server-URL und Public Key hardcoden

**Ziel:** Client verbindet sich ausschließlich mit `goremote.go-ecommerce.de`.

**Files:**
- Modify: `src/config.rs`

**Step 1: Relevante Zeilen in config.rs finden**

```bash
grep -n "RENDEZVOUS_SERVER\|RS_PUB_KEY\|rendezvous_server\|custom_server" src/config.rs | head -30
```

**Step 2: Build-Env-Variablen-Ansatz prüfen**

```bash
grep -n "env!" src/config.rs | head -20
```

Falls `env!("RENDEZVOUS_SERVER", "")` vorhanden → Ansatz A (env var im Build).
Falls direkte String-Konstanten → Ansatz B (direkte Änderung).

**Step 3a: Ansatz A – Env-Variablen (bevorzugt für CI)**

In `src/config.rs` nach `RENDEZVOUS_SERVER` suchen. Falls es so aussieht:
```rust
pub const RENDEZVOUS_SERVER: &str = env!("RENDEZVOUS_SERVER", "");
pub const RS_PUB_KEY: &str = env!("RS_PUB_KEY", "");
```
→ Dann werden diese Werte über Cargo-Env-Vars gesetzt (siehe Task 7).

**Step 3b: Ansatz B – Direkt in config.rs hardcoden**

Falls die Konstanten leer initialisiert sind (`""`), ersetzen:
```rust
// Vorher:
pub const RENDEZVOUS_SERVER: &str = "";
pub const RS_PUB_KEY: &str = "";

// Nachher:
pub const RENDEZVOUS_SERVER: &str = "goremote.go-ecommerce.de";
pub const RS_PUB_KEY: &str = "BgxcHHWnIrhCy2ObBqKR7mg8REpGgrU3ZexkI6GKzA8=";
```

**Step 4: Falls RustDesk die Server-Config über eine eingebettete Datei löst**

```bash
# Prüfen ob es eine custom-server Konfiguration gibt
find . -name "*.toml" | xargs grep -l "rendezvous\|relay" 2>/dev/null | head -5
grep -r "custom_server\|CUSTOM_SERVER" src/ --include="*.rs" | head -10
```

Eine Datei `RustDesk2.toml` im Projektroot mit folgendem Inhalt wird vom NSIS-Installer mitgebündelt:
```toml
[options]
custom_rendezvous_server = "goremote.go-ecommerce.de"
key = "BgxcHHWnIrhCy2ObBqKR7mg8REpGgrU3ZexkI6GKzA8="
relay_server = "goremote.go-ecommerce.de"
api_server = ""
```

Datei anlegen:
```bash
cat > RustDesk2.toml << 'EOF'
[options]
custom_rendezvous_server = "goremote.go-ecommerce.de"
key = "BgxcHHWnIrhCy2ObBqKR7mg8REpGgrU3ZexkI6GKzA8="
relay_server = "goremote.go-ecommerce.de"
api_server = ""
EOF
```

**Step 5: Commit**

```bash
git add src/config.rs RustDesk2.toml
git commit -m "feat: hardcode go Remote server config"
```

---

## Task 3: App-Name in allen Manifesten setzen

**Ziel:** Überall wo "RustDesk" steht, "go Remote" einsetzen.

**Files:**
- Modify: `Cargo.toml`
- Modify: `flutter/pubspec.yaml`
- Modify: `flutter/windows/runner/CMakeLists.txt`
- Modify: `flutter/windows/runner/Runner.rc`
- Modify: `flutter/macos/Runner/Info.plist`

**Step 1: Cargo.toml**

```bash
grep -n "^name\|RustDesk\|rustdesk" Cargo.toml | head -10
```

Ersetze:
```toml
# Vorher:
name = "rustdesk"

# Nachher:
name = "goremote"
```

**Step 2: flutter/pubspec.yaml**

```bash
grep -n "^name:\|description:\|RustDesk\|rustdesk" flutter/pubspec.yaml | head -10
```

Ersetze:
```yaml
# Vorher:
name: rustdesk
description: RustDesk - Remote Desktop

# Nachher:
name: goremote
description: go Remote - Remote Support
```

**Step 3: Windows CMakeLists.txt**

```bash
grep -n "BINARY_NAME\|RustDesk\|rustdesk" flutter/windows/runner/CMakeLists.txt | head -10
```

Ersetze:
```cmake
# Vorher:
set(BINARY_NAME "rustdesk")

# Nachher:
set(BINARY_NAME "goremote")
```

**Step 4: Windows Runner.rc (Versioninfo + App-Name)**

```bash
grep -n "RustDesk\|rustdesk\|FileDescription\|ProductName" flutter/windows/runner/Runner.rc | head -20
```

Alle Vorkommen von `RustDesk` ersetzen:
```
FileDescription   → "go Remote"
ProductName       → "go Remote"
InternalName      → "goremote"
OriginalFilename  → "goremote.exe"
```

**Step 5: macOS Info.plist**

```bash
grep -n "RustDesk\|rustdesk\|CFBundle" flutter/macos/Runner/Info.plist | head -20
```

Folgende Keys setzen:
```xml
<key>CFBundleName</key>
<string>go Remote</string>
<key>CFBundleDisplayName</key>
<string>go Remote</string>
<key>CFBundleIdentifier</key>
<string>de.go-ecommerce.goremote</string>
<key>CFBundleExecutable</key>
<string>goremote</string>
```

**Step 6: Commit**

```bash
git add Cargo.toml flutter/pubspec.yaml flutter/windows/runner/CMakeLists.txt flutter/windows/runner/Runner.rc flutter/macos/Runner/Info.plist
git commit -m "feat: rename app to go Remote in all manifests"
```

---

## Task 4: Icons in Flutter-Projekt integrieren

**Ziel:** Das logo.png-basierte Icon überall in Flutter einsetzen (Windows Runner, macOS Assets, Flutter Launcher Icons).

**Files:**
- Modify: `flutter/windows/runner/resources/app_icon.ico`
- Modify: `flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/`
- Modify: `flutter/pubspec.yaml` (flutter_launcher_icons)

**Step 1: Windows Runner-Icon ersetzen**

```bash
cp res/icon.ico flutter/windows/runner/resources/app_icon.ico
```

**Step 2: macOS AppIcon.appiconset befüllen**

```bash
# Vorhandene Icons prüfen
ls flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/

# Contents.json prüfen um die erwarteten Dateinamen zu sehen
cat flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/Contents.json
```

PNGs aus `res/AppIcon.iconset/` in das appiconset kopieren. Typische Contents.json erwartet:
```
app_icon_16.png, app_icon_32.png, app_icon_64.png,
app_icon_128.png, app_icon_256.png, app_icon_512.png, app_icon_1024.png
```

```bash
cd res/AppIcon.iconset
for size in 16 32 64 128 256 512 1024; do
  cp icon_${size}x${size}.png ../../flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/app_icon_${size}.png
done
cd ../..
```

Dann `Contents.json` anpassen, falls die Dateinamen nicht übereinstimmen:
```bash
cat flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/Contents.json
# Dateinamen in Contents.json an die kopierten Dateien anpassen
```

**Step 3: Flutter Launcher Icons (für Android/iOS falls je benötigt, überspringbar)**

In `flutter/pubspec.yaml` prüfen ob `flutter_launcher_icons` bereits enthalten:
```bash
grep -n "flutter_launcher_icons\|flutter_icons" flutter/pubspec.yaml
```

Falls vorhanden, konfigurieren:
```yaml
flutter_launcher_icons:
  android: false
  ios: false
  windows:
    generate: true
    image_path: "../res/logo_512.png"
    icon_size: 256
  macos:
    generate: true
    image_path: "../res/logo_512.png"
```

Falls nicht vorhanden, füge `flutter_launcher_icons: ^0.13.0` zu `dev_dependencies` hinzu und führe aus:
```bash
cd flutter && dart run flutter_launcher_icons
```

**Step 4: Logo für Splash / About Screen**

```bash
# Logo in Flutter Assets kopieren
mkdir -p flutter/assets
cp logo.png flutter/assets/logo.png
cp res/logo_512.png flutter/assets/logo_512.png
```

In `flutter/pubspec.yaml` unter `flutter: assets:` eintragen:
```yaml
flutter:
  assets:
    - assets/logo.png
    - assets/logo_512.png
```

**Step 5: Commit**

```bash
git add flutter/windows/runner/resources/app_icon.ico flutter/macos/Runner/Assets.xcassets/ flutter/assets/ flutter/pubspec.yaml
git commit -m "feat: replace icons and add logo assets"
```

---

## Task 5: Flutter Theme – Farben anpassen

**Ziel:** App-weites Theme mit #121415 (Hintergrund), #F06800 (Akzent/Buttons), #FFFFFF (Text).

**Files:**
- Modify: `flutter/lib/` (Theme-Datei lokalisieren)

**Step 1: Theme-Datei finden**

```bash
grep -rn "ThemeData\|primaryColor\|MaterialApp\|darkTheme\|lightTheme" flutter/lib/ --include="*.dart" | head -20
grep -rn "primaryColor\|accentColor\|colorScheme" flutter/lib/ --include="*.dart" | head -20
```

Typischerweise in `flutter/lib/main.dart` oder `flutter/lib/common.dart`.

**Step 2: Farben als Konstanten in `flutter/lib/common.dart` eintragen**

```bash
grep -n "Color\|color" flutter/lib/common.dart | head -20
```

Füge am Anfang der Datei (nach Imports) ein:
```dart
// go Remote Brand Colors
const Color kBrandBackground = Color(0xFF121415);
const Color kBrandAccent = Color(0xFFF06800);
const Color kBrandText = Color(0xFFFFFFFF);
```

**Step 3: ThemeData anpassen**

Die `ThemeData`-Definition suchen und ergänzen:
```dart
ThemeData(
  brightness: Brightness.dark,
  scaffoldBackgroundColor: kBrandBackground,
  colorScheme: ColorScheme.dark(
    primary: kBrandAccent,
    secondary: kBrandAccent,
    background: kBrandBackground,
    surface: const Color(0xFF1E2124),
    onPrimary: kBrandText,
    onSecondary: kBrandText,
    onBackground: kBrandText,
    onSurface: kBrandText,
  ),
  elevatedButtonTheme: ElevatedButtonThemeData(
    style: ElevatedButton.styleFrom(
      backgroundColor: kBrandAccent,
      foregroundColor: kBrandText,
    ),
  ),
  textButtonTheme: TextButtonThemeData(
    style: TextButton.styleFrom(
      foregroundColor: kBrandAccent,
    ),
  ),
  appBarTheme: const AppBarTheme(
    backgroundColor: kBrandBackground,
    foregroundColor: kBrandText,
  ),
)
```

**Hinweis:** RustDesk hat eigene Theming-Logik. Nach dem Suchen ggf. mehrere Theme-Definitionen (light/dark) anpassen oder die eigene Farb-Logik überschreiben.

**Step 4: Verifizieren (falls Flutter lokal installiert)**

```bash
cd flutter && flutter analyze 2>&1 | head -30
```

**Step 5: Commit**

```bash
git add flutter/lib/
git commit -m "feat: apply go Remote brand colors to Flutter theme"
```

---

## Task 6: Logo im About-Screen / Splash anzeigen

**Ziel:** Das go Remote Logo im About-Dialog und ggf. Splash sichtbar machen.

**Files:**
- Modify: `flutter/lib/` (About-Screen / Splash-Widget suchen)

**Step 1: About-Screen finden**

```bash
grep -rn "about\|About\|logo\|Logo\|splash\|Splash" flutter/lib/ --include="*.dart" -l | head -10
grep -rn "RustDesk\|rustdesk" flutter/lib/ --include="*.dart" | grep -i "logo\|image\|asset" | head -10
```

**Step 2: Logo-Asset einbinden**

Im gefundenen About-Widget das bestehende Logo/Icon ersetzen:
```dart
// Vorher (typisch):
Image.asset('assets/logo.svg')
// oder
SvgPicture.asset('assets/logo.svg')

// Nachher:
Image.asset('assets/logo.png', width: 120, height: 120)
```

**Step 3: App-Name im About-Dialog**

```bash
grep -rn "RustDesk\|\"rustdesk\"\|'rustdesk'" flutter/lib/ --include="*.dart" | head -20
```

Alle Anzeige-Strings `RustDesk` → `go Remote` ersetzen.

**Step 4: Commit**

```bash
git add flutter/lib/
git commit -m "feat: replace RustDesk branding with go Remote in UI"
```

---

## Task 7: GitHub Actions – Windows Build

**Ziel:** Automatischer Windows `.exe` Installer-Build bei jedem Release-Tag.

**Files:**
- Create: `.github/workflows/build-windows.yml`
- (Prüfen ob RustDesk bereits Workflows hat)

**Step 1: Vorhandene Workflows prüfen**

```bash
ls .github/workflows/ 2>/dev/null || echo "Keine Workflows vorhanden"
```

Falls RustDesk-Workflows vorhanden → diese als Basis nutzen, nicht neu schreiben.

**Step 2: Windows-Build-Workflow anlegen**

```yaml
# .github/workflows/build-windows.yml
name: Build Windows Installer

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:

jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: stable
          targets: x86_64-pc-windows-msvc

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.x'
          channel: stable

      - name: Install dependencies
        run: |
          flutter pub get
        working-directory: flutter

      - name: Build (Cargo + Flutter)
        env:
          RENDEZVOUS_SERVER: goremote.go-ecommerce.de
          RS_PUB_KEY: BgxcHHWnIrhCy2ObBqKR7mg8REpGgrU3ZexkI6GKzA8=
        run: |
          cargo build --release

      - name: Build Flutter Windows
        run: |
          flutter build windows --release
        working-directory: flutter

      - name: Create NSIS Installer
        uses: joncloud/makensis-action@v4
        with:
          script-file: res/nsis/installer.nsi

      - name: Upload Installer
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: |
            target/release/goremote-setup.exe
```

**Hinweis:** RustDesk hat ein eigenes NSIS-Script in `res/`. Dieses muss auf den neuen App-Namen angepasst werden (Task 8).

**Step 3: Commit**

```bash
git add .github/workflows/build-windows.yml
git commit -m "ci: add Windows installer build workflow"
```

---

## Task 8: GitHub Actions – macOS Build

**Ziel:** Automatischer macOS `.dmg` Build bei jedem Release-Tag.

**Files:**
- Create: `.github/workflows/build-macos.yml`

**Step 1: macOS-Build-Workflow anlegen**

```yaml
# .github/workflows/build-macos.yml
name: Build macOS DMG

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:

jobs:
  build:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: stable
          targets: x86_64-apple-darwin aarch64-apple-darwin

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.x'
          channel: stable

      - name: Install dependencies
        run: flutter pub get
        working-directory: flutter

      - name: Generate AppIcon.icns
        run: |
          iconutil -c icns res/AppIcon.iconset -o flutter/macos/Runner/Assets.xcassets/AppIcon.appiconset/AppIcon.icns

      - name: Build (Cargo + Flutter)
        env:
          RENDEZVOUS_SERVER: goremote.go-ecommerce.de
          RS_PUB_KEY: BgxcHHWnIrhCy2ObBqKR7mg8REpGgrU3ZexkI6GKzA8=
        run: |
          cargo build --release

      - name: Build Flutter macOS
        run: flutter build macos --release
        working-directory: flutter

      - name: Create DMG
        run: |
          brew install create-dmg
          create-dmg \
            --volname "go Remote" \
            --background "res/logo_512.png" \
            --window-size 800 400 \
            --icon-size 128 \
            --icon "go Remote.app" 200 200 \
            --app-drop-link 600 200 \
            "goremote.dmg" \
            "flutter/build/macos/Build/Products/Release/go Remote.app"

      - name: Upload DMG
        uses: softprops/action-gh-release@v1
        if: startsWith(github.ref, 'refs/tags/')
        with:
          files: goremote.dmg
```

**Step 2: Commit**

```bash
git add .github/workflows/build-macos.yml
git commit -m "ci: add macOS DMG build workflow"
```

---

## Task 9: NSIS-Script für Windows-Installer anpassen

**Ziel:** Windows-Installer zeigt "go Remote" an, installiert in `C:\Program Files\go Remote`.

**Files:**
- Modify: `res/nsis/installer.nsi` (oder äquivalentes NSIS-Script im Repo)

**Step 1: NSIS-Script finden**

```bash
find . -name "*.nsi" -o -name "*.nsh" | grep -v ".git" | head -10
```

**Step 2: App-Name und Pfade ersetzen**

```bash
grep -n "RustDesk\|rustdesk\|OutFile\|InstallDir" res/nsis/installer.nsi | head -20
```

Ersetzen:
```nsi
; Vorher:
Name "RustDesk"
OutFile "rustdesk-setup.exe"
InstallDir "$PROGRAMFILES64\RustDesk"

; Nachher:
Name "go Remote"
OutFile "goremote-setup.exe"
InstallDir "$PROGRAMFILES64\go Remote"
```

Auch Shortcuts, Registry-Einträge und Uninstaller-Namen anpassen.

**Step 3: Commit**

```bash
git add res/nsis/
git commit -m "feat: rebrand NSIS installer as go Remote"
```

---

## Task 10: Erster Release-Tag setzen und Build testen

**Ziel:** GitHub Actions Build triggern und Installer verifizieren.

**Step 1: Alle Änderungen gepusht?**

```bash
git status
git push origin main
```

**Step 2: Release-Tag setzen**

```bash
git tag v1.0.0
git push origin v1.0.0
```

**Step 3: Build überwachen**

```bash
gh run list --limit 5
gh run watch
```

**Step 4: Bei Build-Fehler – Logs prüfen**

```bash
gh run view --log-failed
```

Häufige Probleme:
- Flutter-Version inkompatibel → in Workflow anpassen
- Native Libraries fehlen → RustDesk-Build-Dependencies nachinstallieren (vcpkg für Windows)
- NSIS-Script Pfade falsch → Step 9 nachbessern

**Step 5: Fertige Installer herunterladen und testen**

```bash
gh release download v1.0.0
```

- Windows: `goremote-setup.exe` installieren, App starten, Server-Verbindung prüfen
- macOS: `goremote.dmg` mounten, App starten, Server-Verbindung prüfen

---

## Task 11: Upstream-Updates einspielen (wiederkehrend)

**Ziel:** Neue RustDesk-Versionen in den Fork übernehmen ohne eigene Änderungen zu verlieren.

**Step 1: Upstream-Änderungen holen**

```bash
git fetch upstream
```

**Step 2: Auf neuem Branch rebasen**

```bash
git checkout -b update/rustdesk-upstream
git merge upstream/master
```

**Step 3: Konflikte auflösen**

Typische Konflikte in:
- `src/config.rs` (unsere Server-Config)
- `flutter/pubspec.yaml` (App-Name)
- `flutter/lib/` (Theme)

Nach Auflösung:
```bash
git add .
git commit -m "chore: merge upstream RustDesk updates"
git checkout main
git merge update/rustdesk-upstream
git push origin main
```

---

## Wichtige Hinweise

### Build-Komplexität
RustDesk ist ein großes Projekt mit nativen Abhängigkeiten. Der erste CI-Build wird wahrscheinlich Anpassungen brauchen. RustDesk pflegt eigene Build-Skripte – diese als Basis verwenden, nicht neu erfinden.

### RustDesk bestehende Workflows
Nach dem Fork unbedingt `.github/workflows/` inspizieren. RustDesk hat bereits komplexe Build-Workflows die man erweitern sollte, nicht ersetzen.

### Code-Signing (optional für v1)
Für vertrauenswürdige Installer (kein "Unknown Publisher"-Dialog):
- Windows: EV Code Signing Zertifikat + GitHub Secret `WINDOWS_SIGNING_CERT`
- macOS: Apple Developer ID + GitHub Secret `MACOS_SIGNING_CERT`
Dies für v2 einplanen.

### Server-Config Verifikation
Nach erstem Build: Client starten → Einstellungen → Server sollte `goremote.go-ecommerce.de` zeigen. Falls nicht, war Task 2 nicht wirksam und `RustDesk2.toml` Bündelung im Installer prüfen.
