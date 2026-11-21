# Projektwissen – Spytify (Fork)

Diese Datei ist der Einstieg für jede Sitzung in diesem Ordner. Die ausführliche Entwickler-Doku des Originalprojekts steht in [CONTRIBUTING.md](CONTRIBUTING.md), der Datenschutz in [PRIVACY.md](PRIVACY.md); hier steht das Wichtigste auf Deutsch plus alles, was für diesen Fork gilt.

## Was das Projekt ist

Spytify nimmt den Ton **einer** Anwendung auf (z. B. einen Musik-Player), schneidet ihn sampelgenau in einzelne Titel und taggt sie mit Titel, Interpret, Album und Cover, die der Player meldet. Es ist ein Rekorder, kein Downloader: kein Streaming-Protokoll, keine DRM-Umgehung, kein Nachschlagen im Web.

- Plattformen: Linux, macOS ab 14.2, Windows 10 ab 2004.
- Sprache: Go (Modul `github.com/spytify/spytify`, Go 1.26), Oberfläche mit [Fyne](https://fyne.io). Helfer in Swift (macOS) und C++ (Windows).
- Lizenz: MIT.
- `v1` / `v1-dev` sind die alte C#-Version (EspionSpotify) – nicht anfassen, sie hat mit dem Go-Code nichts gemeinsam.

## Fork und Remotes

| Remote | Repository | Zweck |
| --- | --- | --- |
| `origin` | https://github.com/BerniPi/spytify | mein Fork, hierhin pushen |
| `upstream` | https://github.com/spytify/spytify | Original, nur holen |

- Aktualisieren: `git fetch upstream` und `git merge upstream/main` (auf `main`).
- Eigene Arbeit auf einem eigenen Zweig, nicht direkt auf `main`, damit `main` sauber dem Original folgen kann.
- Pull Requests ans Original gehen von einem Zweig des Forks gegen `spytify/spytify:main`.
- Die `PROJEKTWISSEN.md` gibt es nur im Fork. Bei einem PR ans Original darf sie nicht mit hinein (Zweig für den PR von `upstream/main` abzweigen).

## Aufbau

```text
Spytify-App ─┐                ┌── Aufnahme ◀── Tonstrom der Anwendung
spytify-cli ─┴─ JSON-RPC ──▶ spytifyd ◀── „Now playing“ ◀── Medienschnittstelle der Anwendung
                                  └──▶ ffmpeg ──▶ getaggte Dateien
```

- **Zwei Prozesse.** `spytifyd` nimmt auf, schneidet, kodiert und taggt, auch ohne offenes Fenster. Die Desktop-App und `spytify-cli` sind Clients über einen lokalen Socket (zeilenweise JSON-RPC 2.0).
- **Aufnahme eines Prozesses:** Linux über PipeWire (`pw-dump`, `pw-record`; PulseAudio: `parec`), macOS über `spytify-tap` (Swift, Core-Audio-Process-Tap), Windows über `spytify-tap.exe` (C++, WASAPI-Process-Loopback).
- **Now playing:** MPRIS (Linux), AppleScript (macOS), Windows-Mediensitzung (Windows).
- **Schnitt:** Die Titelwechsel-Meldung ist nur ungefähr pünktlich; aus der Titellänge folgt das erwartete Ende, die leiseste 20-ms-Stelle drumherum liefert das Sample.
- **Fertigstellen:** temporäres WAV → prüfen, trimmen, normalisieren → als `<name>.part` kodieren → taggen → umbenennen. Eine halbe Datei hat nie den endgültigen Namen.

### Wichtige Ordner

| Pfad | Inhalt |
| --- | --- |
| `cmd/spytifyd` | Dienst (Aufnahme, Schnitt, Kodierung) |
| `cmd/spytify-ui` | Desktop-App (Fyne, braucht cgo) |
| `cmd/spytify-cli` | Kommandozeilen-Client |
| `cmd/fakeplayer` | Test-Player für PipeWire/MPRIS |
| `internal/core` | Herzstück: Ringpuffer, Schnitt, Finisher, Ablage |
| `internal/capture`, `internal/meta` | Aufnahme bzw. Now-playing je Plattform |
| `internal/encode`, `internal/tag` | ffmpeg-Aufruf, Tags |
| `internal/settings` | alle Einstellungen, Tabelle in `table.go` |
| `internal/telemetry` | Sentry-Bugtracker |
| `internal/ui` | Oberfläche |
| `internal/tools` | Generatoren und Prüfwerkzeuge (`gendocs`, `genkeys`, `genschema`, `checkexe`, `gate1` …) |
| `pkg/proto` | öffentliches Protokoll, `schema.json` daraus generiert |
| `macos/`, `windows/` | native Capture-Helfer, FakePlayer, Ressourcen |
| `testdata/timelines` | Szenarien für den Schnitt-Test |
| `scripts/` | Build-, Signier-, Release- und CI-Skripte |

## Regeln, die man leicht übersieht

- **Einstellungen:** Jede Einstellung ist eine Zeile in `internal/settings/table.go`. Code liest sie nur über den generierten Schlüssel (`settings.RecorderTrimSilence` …), nie über einen String. Nach Änderung der Tabelle `go generate ./internal/settings` – CI prüft, dass `keys_gen.go` aktuell ist.
- **Protokoll:** Nach Änderungen an `pkg/proto` `go generate ./pkg/proto` – CI prüft `schema.json`.
- **Telemetrie:** Ändert sich, was Sentry gesendet oder entfernt wird (`internal/telemetry`), muss `PRIVACY.md` im selben PR angepasst werden.
- **ffmpeg:** Braucht Code ein neues ffmpeg-Element (Encoder, Filter, Format), kommt es in die Liste von `scripts/build-ffmpeg.sh`, mit einem Test, der es ausführt.
- **Windows-Exe:** `Spytify.exe` darf `opengl32.dll` nicht direkt linken (Mesa-Fallback) und keine Daten hinter der letzten Section bzw. keine Export-Tabelle haben – `checkexe` lässt den Build sonst scheitern (Virenscanner-Fehlalarme).
- **Fyne-Patch:** `go.mod` ersetzt Fyne durch `spytify/fyne-fix-6642` (Font-Cache-Fix). Entfernen, sobald ein Fyne-Release fyne-io/fyne#6642 enthält.
- **Website-Doku:** `v2/settings.md` und der Protokollteil auf `gh-pages` kommen aus dem Code (`go run ./internal/tools/gendocs -out <gh-pages-Checkout>`).

## Build

Auf diesem Rechner (Windows) sind derzeit **weder Go noch gcc/MinGW noch ffmpeg installiert** – vor dem ersten Build Go ≥ 1.26 installieren.

```sh
go build -o bin/ ./cmd/spytifyd ./cmd/spytify-cli ./cmd/spytify-ui   # Linux
scripts/build-linux.sh      # dist/*.tar.gz, .deb, .rpm
scripts/build-macos.sh      # bin/Spytify.app
scripts/build-windows.sh    # cross-kompiliert von Linux/macOS: bin/windows/*.exe
```

- `spytifyd` und `spytify-cli` sind reines Go (`CGO_ENABLED=0` geht) – das lässt sich auch unter Windows direkt bauen.
- Die Desktop-App braucht cgo. Windows-Builds sind im Projekt als Cross-Build von Linux/macOS mit MinGW-w64 vorgesehen; unter Windows selbst am einfachsten über WSL.
- Build-Variablen (Umgebung oder untracktes `build.env`): `SPYTIFY_DEFAULT_PROCESS` (Standard-Player), `SPYTIFY_SENTRY_DSN` (ohne: nichts wird gesendet). Eigene Builds senden also nie etwas an Sentry.
- Version kommt aus `scripts/version.sh` (letzter Tag + `-dev.<Commits>`). Tags ohne `v`, z. B. `2.0.2`.

## Tests

```sh
gofmt -l .                              # muss leer sein
go vet ./...
go test -race -timeout 20m ./...        # Unit-, Szenario-, UI- und End-to-End-Tests
```

- `internal/ui` braucht unter `-race` lange (CI gibt 40 Minuten); ohne `-timeout` bricht `go test` nach 10 Minuten ab.
- Unter Windows ohne cgo: `CGO_ENABLED=0 go test $(go list ./... | grep -v -e /internal/ui$ -e /cmd/spytify-ui)` – so macht es auch CI.
- Integrationstests mit echtem PipeWire, macOS-Capture, AppleScript, Sentry und `gate1` (echter Player): siehe Abschnitt „Tests“ in [CONTRIBUTING.md](CONTRIBUTING.md#tests).
- CI (`.github/workflows/ci.yml`): gofmt, vet, staticcheck, Generierungsprüfungen, Tests auf Linux/macOS/Windows, Cross-Builds, Windows-Build und Smoke-Tests der ausgelieferten Exes. Im Fork läuft CI nur, wenn Actions im Fork aktiviert sind.

## Stil des Originalprojekts

- Commit-Betreffs sind ganze englische Sätze, die das neue Verhalten beschreiben, nicht die Tätigkeit: „Show in folder asks the Windows shell in-process, with no explorer.exe (#905)“. Kein `fix:`/`feat:`-Präfix.
- Der Commit-Text erklärt Ursache und Wirkung in Prosa; `Fixes #<Nr>` für Issues.
- PRs werden gesquasht, die PR-Nummer steht in Klammern am Ende des Betreffs.
- Testnamen sind ebenfalls Sätze (`TestUIStaysIdleWhileRecording`, CI-Schritte wie „Without an OpenGL driver, the app draws its window with Mesa's“).
- Doku-Sprache ist knappes, schlichtes Englisch aus Nutzersicht.
