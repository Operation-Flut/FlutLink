# FlutLink v1.5.0

Major-Release: **Entfernung des mobilen KMP-Clients** und **Autostart-Verhaltens-Fix**.

## Neu

- **Desktop-only**: Der mobile Kotlin-Multiplatform-Client (`kmp/`) wurde vollständig entfernt (Issue #523). FlutLink ist nun ein reiner Desktop-Client (Tauri v2).
- **FlutCloud Server-App 1.3.0**: Entfernt iOS/AltStore-Endpunkte (`IosController.php`, `AltStoreSourceService.php`), da der mobile Client entfällt.

## Behoben

- **Autostart-Verhalten** (Issue #494): Die App startet nun **nur noch beim OS-Login minimiert im Tray**; jeder manuelle Start öffnet das Fenster normal. Zuvor wurde das Fenster bei aktivierter Autostart-Einstellung **immer** versteckt.
- **Sicherheitslücken** geschlossen:
  - `source-map-js` 1.2.2 (CVE-2026-93749, DoS via malformed source maps)
  - `rustls` 0.23.43 (RUSTSEC-2026-0285 / GHSA-2mjx-qc3c-rqvc, TLS 1.3 handshake flaw)

## Verbessert

- **Tauri-Plugin-Abgleich**: `tauri-plugin-dialog` auf 2.8.1 aktualisiert (Version-Mismatch mit NPM-Paket behoben).
- **Abhängigkeiten**: Diverse Rust- und npm-Abhängigkeiten via Dependabot aktualisiert.
- **Dokumentation**: CLI-Flags (`--autostart`) und Autostart-Verhalten in `docs/*/tray-and-cli.md` und `features.md` dokumentiert (EN/DE).

## Hinweise

- Die CI-Installer sind derzeit nicht signiert. macOS Gatekeeper und Windows SmartScreen können Warnungen anzeigen.
- Der Zwei-Wege-Sync ist Desktop-only.
- Die Nextcloud-Server-App (`flutcloud-app.zip`, v1.3.0) muss auf dem Server aktualisiert werden (entfernt AltStore/iOS-Endpunkte).