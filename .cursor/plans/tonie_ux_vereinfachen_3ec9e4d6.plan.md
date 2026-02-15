---
name: Tonie UX vereinfachen
overview: "Umbau von \"WIP: Modell anlegen\" zu einem vollwertigen, nutzerfreundlichen Editor fuer `tonies.custom.json` mit Wiederverwendung der bestehenden FileBrowser-Logik."
todos:
  - id: backend-write-api
    content: Schreib-API per POST fuer `tonies.custom.json` ergaenzen (Full-Save) inkl. atomischem Schreiben und Reload.
    status: pending
  - id: backend-validation
    content: Serverseitige Validierung fuer Struktur, Pflichtfelder, Datentypen und Konflikte implementieren (inkl. Block bei dupliziertem audio_id+hash).
    status: pending
  - id: editor-list-crud
    content: UI-Editor fuer `tonies.custom.json` mit Liste, Hinzufuegen, Bearbeiten und Loeschen umsetzen.
    status: pending
  - id: required-defaults-only
    content: Nur fuer `model` Default/Auto-Vorschlag beim Oeffnen setzen (`custom-{maxId+1}`); `series` bleibt bewusst manuell, optionale Felder ohne Defaults.
    status: pending
  - id: image-tooltip
    content: Bild-Tooltip mit korrekter Nutzung ergaenzen (`/custom_img/...`, keine `/web/...` URL).
    status: pending
  - id: image-manager-api
    content: File-API fuer `custom_img` als eigener `special`-Root ergaenzen (Index, Upload, CreateDir, Move/Delete).
    status: pending
  - id: image-manager-ui
    content: Unabhaengige Bildverwaltung mit Ordnern, Multi-Upload und Sortierung im Bereich `Custom-Modelle verwalten` integrieren.
    status: pending
  - id: image-picker-in-form
    content: Im Model-Formular Bildauswahl ueber Picker/Autocomplete anbieten; Upload aus dem Formular setzt `pic` automatisch.
    status: pending
  - id: pre-save-checks
    content: Vor dem Speichern im UI Validierung und klare Fehleranzeige fuer JSON-Lesbarkeit und Feldfehler einbauen.
    status: pending
  - id: custom-json-backups
    content: Beim Speichern von `tonies.custom.json` Backups mit Timestamp-Namen erzeugen und auf feste Anzahl N=10 begrenzen.
    status: pending
isProject: false
---

# Vollwertiger Custom-Model-Editor

## Zielbild

- `WIP: Modell anlegen` wird zu einem benutzbaren Editor fuer `tonies.custom.json`.
- Menueeintrag/Label wird lokalisiert auf `Custom-Modelle verwalten`.
- Namen bleiben im bestehenden Modellschema (`series` im Custom-Model).
- Speichern ist sicher: validiert, atomisch geschrieben, danach Reload.
- Bilder koennen unabhaengig verwaltet werden und direkt im Editor komfortabel ausgewaehlt werden.

## Ist-Stand (relevante Stellen)

- Read-only APIs existieren bereits:
  - `[/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/src/handler_api.c](/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/src/handler_api.c)` mit `handleApiToniesCustomJson`, `handleApiToniesJsonSearch`, `handleApiToniesJsonReload`.
- Im Frontend ist der Dialog vorhanden, aber Save erzeugt nur JSON-Ausgabe:
  - `[/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/teddycloud_web/src/components/tonies/ToniesCustomJsonEditor.tsx](/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/teddycloud_web/src/components/tonies/ToniesCustomJsonEditor.tsx)`.
- Bildpfad fuer Modelle ist `pic` und wird als `picture` verwendet:
  - `[/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/src/toniesJson.c](/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/src/toniesJson.c)`.
- FileBrowser-APIs und UI-Funktionen fuer Upload/Multi-Upload/Ordner existieren bereits fuer `special=library`:
  - Backend Root-Auswahl in `queryPrepare` in `[/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/src/handler_api.c](/home/ditschi/Dokumente/DIY/TeddyCloud/teddycloud/src/handler_api.c)`.
  - Upload `POST /api/fileUpload`, Index `GET /api/fileIndexV2`, DirCreate `POST /api/dirCreate`.
  - Frontend-Module: `UploadFilesModal`, `CreateDirectoryModal`, `SelectFileFileBrowser`.

## Umsetzungsweg

### 1) Backend: Schreibfaehigkeit fuer `tonies.custom.json`

- Neuen Write-Endpunkt ergaenzen, bevorzugt:
  - `POST /api/toniesCustomJsonSet` (gesamte bereinigte Custom-Liste speichern).
- API-Methode folgt dem bestehenden Muster im Projekt (mutierende Endpunkte per `POST`).
- Speicherung atomisch per tmp+rename, danach `tonies_init`/Reload triggern.
- Fehlerantworten mit klarer Ursache (Validation statt generisches 500).

### 2) Validierung: technisch korrekt und lesbar

- Pflicht:
  - `series` muss manuell gesetzt sein (kein Default).
  - `model` muss gesetzt sein (Default erlaubt, editierbar).
- Struktur:
  - `audio_id` und `hash` konsistent (gleiche Anzahl).
  - `hash` nur 40-hex.
  - Arrays/Strings/Numerics sauber typisiert.
- Konflikte:
  - doppeltes `model` in custom verhindern.
  - doppeltes `audio_id+hash` (identisches Paar) blockieren.
  - Kollision mit base `tonies.json` als Warnung mit expliziter Bestaetigung (Override erlaubt).
- Beim Speichern zusaetzlich:
  - vor dem Ueberschreiben ein Backup der bisherigen `tonies.custom.json` erzeugen.
  - Dateiname timestamp-basiert im Format `<orig_name>.<YYYYMMDD-HHMMSS>.bak`.
  - Backups werden im gleichen Verzeichnis wie `tonies.custom.json` abgelegt.
  - Aufbewahrung auf feste Anzahl `N=10` begrenzen (aelteste Backups zuerst loeschen).

### 3) UI: vom WIP-Dialog zum Editor

- In einem Screen:
  - aktuelle Custom-Eintraege laden und anzeigen.
  - Eintrag anlegen/bearbeiten/loeschen.
  - explizite Kennzeichnung von Pflichtfeldern.
- Default-Strategie gemaess Vorgabe:
  - beim Oeffnen der Maske bekommt `model` einen Auto-Vorschlag auf Basis der bekannten Custom-IDs (`custom-{maxId+1}`), editierbar.
  - `series` bleibt leer und wird vom Nutzer gesetzt.
  - optionale Felder bleiben leer (keine impliziten Defaults).
  - bei `model`-Kollision Fehler anzeigen; User aendert den Wert manuell.
- Speichern-Button erst bei valider Eingabe aktiv oder mit klaren Feldfehlern.

### 4) Bildnutzung: Tooltip statt falscher URL-Muster

- Tooltip bei `pic` nur mit klaren, korrekten Beispielen:
  - Extern: `https://example.com/images/biene-maja.png`
  - Lokal: `/custom_img/images/custom-tonies/biene-maja-coin.png`

### 5) Bildverwaltung als integrierte Funktion

- Backend erweitern: `special=custom_img` in `queryPrepare` auf `/teddycloud/data/www/custom_img` (bzw. `internal.wwwdirfull/custom_img`) mappen.
- Vorhandene File-APIs unveraendert nutzen:
  - `GET /api/fileIndexV2?special=custom_img&path=...`
  - `POST /api/fileUpload?special=custom_img&path=...` (Multi-Upload bleibt moeglich)
  - `POST /api/dirCreate?special=custom_img`
  - `POST /api/fileMove` und `POST /api/fileDelete` fuer Aufraeumen/Sortieren (wie bei `library`).
- Unterordner muessen voll unterstuetzt sein (analog `library`):
  - Navigation ueber `path` in beliebige Tiefe
  - Upload/CreateDir/Move/Delete jeweils relativ zum aktuellen Unterordner.
- UI in `Custom-Modelle verwalten` als zwei gekoppelte Bereiche:
  - **Bildverwaltung** (unabhaengig): Ordnerbaum, Multi-Upload, Umbenennen/Verschieben/Loeschen.
  - **Modell-Editor**: `pic` Feld mit Picker + Autocomplete aus `/custom_img`.
- Umsetzung der Bildverwaltung als Reuse des bestehenden Library-Browsers:
  - gleiche Browser-Komponenten/Modals/Patterns (`SelectFileFileBrowser`, `UploadFilesModal`, `CreateDirectoryModal`)
  - nur Root/Spezialpfad auf `custom_img` umstellen (`special=custom_img`)
  - gleiches Interaktionsverhalten wie `library` (inkl. Move/Delete-Workflow).
- Embedded Flow im Formular:
  - Button **Bild hochladen** oeffnet Upload fuer aktuellen Zielordner.
  - Nach erfolgreichem Upload wird der neue Pfad automatisch in `pic` gesetzt.
  - Button **Bild auswaehlen** oeffnet Dateiauswahl auf `custom_img`, uebernimmt selektierten Pfad.
  - Autocomplete fuer `pic` verwendet denselben Such-/Filteransatz wie im Library-Kontext.
  - Bildvorschau als Modal-Preview per Klick auf Bilddatei (kein zusaetzliches Seitenpanel in v1).
- Bildfilter fuer Browser/Picker in v1:
  - `.png`, `.jpg`, `.jpeg`, `.webp`, `.gif`.

## Technischer Ablauf

```mermaid
flowchart TD
  editorUi[ToniesCustomJsonEditor]
  validateClient[ClientValidation]
  postApi[/api/toniesCustomJsonSet POST]
  validateServer[ServerValidation]
  writeFile[WriteToniesCustomJsonTmpRename]
  reload[toniesJsonReload]
  imageApi[/fileIndexV2 fileUpload dirCreate special=custom_img]
  imageManager[ImageManagerInCustomModels]
  searchAndCards[ToniesSearchAndCards]

  editorUi --> validateClient
  editorUi --> imageManager
  imageManager --> imageApi
  imageApi --> editorUi
  validateClient --> postApi
  postApi --> validateServer
  validateServer --> writeFile
  writeFile --> reload
  reload --> searchAndCards
```



## Abnahmekriterien

- Custom-Model-Editor zeigt bestehende Eintraege aus `tonies.custom.json` und erlaubt Add/Edit/Delete.
- Speichern schreibt tatsaechlich auf Datei und ueberlebt Neustart.
- Vor Save werden Feldfehler klar angezeigt; Server blockt ungueltige Daten.
- `series` ist Pflicht ohne Default; `model` hat Auto-Vorschlag, bleibt editierbar.
- Identisches `audio_id+hash` wird nicht gespeichert (harte Validierung).
- Base-Override wird nur nach Warnung + Bestaetigung gespeichert.
- `pic`-Tooltip erklaert korrekte Pfadangabe inkl. `/custom_img/...` Beispiel.
- In `Custom-Modelle verwalten` sind unabhaengige Bildverwaltung (inkl. Multi-Upload + Ordner anlegen) und direkte Bildauswahl im Formular verfuegbar.
- Upload im Formular setzt den hochgeladenen Bildpfad automatisch in `pic`.
- Beim Speichern von `tonies.custom.json` werden timestamp-basierte Backups angelegt und auf maximal 10 Dateien begrenzt.
- Backup-Format und Ablage sind fest: `<orig_name>.<YYYYMMDD-HHMMSS>.bak` neben `tonies.custom.json`.
- Image-Preview erfolgt per Modal-Ansicht; Bildfilter ist auf gaengige Formate begrenzt (`png/jpg/jpeg/webp/gif`).

## PR-Aufteilung

### PR 1: Backend Save + Validation + Backups

- Neuer mutierender Endpunkt im bestehenden Stil:
  - `POST /api/toniesCustomJsonSet` (Full-Save von `tonies.custom.json`).
- Request-Format fest:
  - `Content-Type: application/json`
  - Body = vollstaendiges JSON-Array der Custom-Modelle.
- Serverseitige Validierung:
  - `model` required und eindeutig in custom.
  - `audio_id/hash` strukturell valide.
  - identisches `audio_id+hash` blockieren.
  - Base-Override als warnfaehiger Fall kennzeichnen.
- Sicheres Schreiben:
  - timestamp-basiertes Backup (`N=10`).
  - atomisch via `.tmp` + rename.
  - `tonies` Reload nach erfolgreichem Save.

### PR 2: Editor von WIP zu produktiv

- `ToniesCustomJsonEditor` auf echten Arbeitsfluss umstellen:
  - bestehende custom Eintraege anzeigen.
  - Add/Edit/Delete in der UI (Speichern als Full-Save ueber PR-1-API).
- Formularregeln:
  - `model` required mit Default `custom-{maxId+1}`.
  - `series` required ohne Default.
  - Fehler/Warnings konsistent anzeigen.
  - Base-Override nur mit expliziter Bestaetigung.
- UX/Localization:
  - Menueeintrag auf `Custom-Modelle verwalten`.
  - Tooltips inkl. `pic` Beispiele.
- Save-Verhalten:
  - Speichern immer als Full-Save (kein Teil-CRUD in v1).
  - Bei Base-Override gilt: ohne explizite Bestaetigung kein Save.
  - Delete-Aktionen im Editor sind zunaechst lokal und werden erst mit `Speichern` persistent.

### PR 3: Bildverwaltung via Library-Reuse

- Backend:
  - `special=custom_img` in `queryPrepare` (Root `internal.wwwdirfull/custom_img`).
- Frontend-Reuse:
  - bestehende FileBrowser-/Modal-/Hook-Logik fuer `custom_img` parametrieren.
  - Unterordner, Multi-Upload, Move/Delete wie bei `library`.
- Formularintegration:
  - Bildauswahl/Autocomplete fuer `pic`.
  - Upload aus Formular setzt `pic` automatisch.
  - Bild-Preview als Modal.
  - Bei lokaler Auswahl/Upload wird ein Web-Pfad unter `/custom_img/...` in `pic` uebernommen.
  - Preview-Interaktion in v1: Klick auf Dateiname/Thumbnail (kein separates Preview-Action-Icon).
