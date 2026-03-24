# OpenClaw-Baseline für ClawChestrate

## Zweck

Diese Datei hält fest, auf welchem OpenClaw-Stand `ClawChestrate` in der Grundstruktur aufsetzt und welche Teile davon zunächst als Basisschicht übernommen werden.

Sie dokumentiert bewusst nur die aktuelle technische Ausgangsbasis.  
ClawChestrate-spezifische Erweiterungen werden weiterhin in [`clawchestrate-architecture.md`](/C:/Users/Jamah/Desktop/ClawChestrate/docs/clawchestrate-architecture.md) beschrieben und später ergänzt, sobald ihre Implementierung konkretisiert oder umgesetzt wird.

## Verwendeter OpenClaw-Stand

- `ClawChestrate` basiert in der aktuellen Grundstruktur auf `OpenClaw v2026.3.23`.
- Die übernommene Baseline stammt aus dem Release-Tag `v2026.3.23`.
- Diese Baseline dient als Runtime- und Workspace-Ausgangspunkt für `ClawChestrate`.

## Lokale Entwicklungsgrundlage

Für die Arbeit aus dem Source-Checkout gelten dieselben Grundvoraussetzungen wie für OpenClaw:

- Node.js `>= 22.16.0`
- `pnpm@10.23.0`
- Source-Build über den Workspace, nicht über einen separaten vereinfachten Mini-Startpfad

Für ein lokales Source-Setup bedeutet das typischerweise:

1. Node `22+` installieren
2. `corepack enable`
3. `corepack prepare pnpm@10.23.0 --activate`
4. im Projekt `pnpm install`
5. danach Start- oder Smoke-Checks wie `pnpm start` oder `pnpm gateway:dev`

Hinweis:
- OpenClaw empfiehlt für Windows grundsätzlich WSL2 als bevorzugte Entwicklungsumgebung.
- Für `ClawChestrate` ist diese Baseline-Datei zunächst nur die Dokumentation der übernommenen technischen Voraussetzung, nicht bereits eine vollständige eigene Installationsanleitung.

## Übernommene Basisschicht

`ClawChestrate` übernimmt in der Grundstruktur die OpenClaw-Basisschicht und erfindet diese Runtime nicht neu.

Dazu gehören insbesondere:

- Gateway-Grundstruktur
- Agent-/Runtime-Grundstruktur
- Session-Modell
- Run-Modell
- Workspace-Modell
- Bootstrap-/Prompt-Grundaufbau
- Tool-System
- Memory-/Compaction-/Pruning-Basis
- Root-Workspace-/Build-Struktur des OpenClaw-Source-Repos

Praktisch bedeutet das:

- `ClawChestrate` startet nicht mit einer leeren Eigenruntime.
- Stattdessen wird die OpenClaw-Basis als technischer Ausgangspunkt übernommen.
- Die ClawChestrate-spezifische Projektschicht wird darauf aufgesetzt.

## Was bewusst noch nicht Teil der Baseline ist

Nicht als eigentliche ClawChestrate-Basisschicht übernommen oder festgelegt sind an dieser Stelle insbesondere:

- OpenClaw-Produkt- und Onboarding-Dokumentation als eigene ClawChestrate-Doku
- Release-, CI- und Repo-Meta-Dateien als architektonischer Kern
- ClawChestrate-spezifische Projektlogik
- `Project Registry`
- `projects.*`-Gateway-Methoden
- Lead-Session-Projektbindung
- Projekt-Workspace-Logik unter `projects/<project-id>/`
- `OpenClawAdapter` für externe Worker
- Delegation Registry und Completion-/Wake-Logik

Diese Dinge gehören zur ClawChestrate-spezifischen Erweiterung der Baseline und werden getrennt fortgeschrieben.

## Verhältnis zur ClawChestrate-Architektur

Die OpenClaw-Baseline ist die technische Ausgangsschicht.  
Die ClawChestrate-Architektur beschreibt, wie darauf eine projektspezifische Orchestrierungsschicht aufgebaut wird.

Kurz gesagt:

- OpenClaw liefert die Basismaschine
- ClawChestrate ergänzt Projektführung, Projektkontext und externe Delegationslogik

## Offene Weiterentwicklung

Diese Baseline-Datei soll bewusst stabil bleiben und nur dann angepasst werden, wenn sich der tatsächliche OpenClaw-Basestand oder die übernommene technische Grundschicht ändert.

Spätere ClawChestrate-Erweiterungen werden nicht hier als Baseline beschrieben, sondern in der Architektur-Doku oder in ergänzenden Implementierungsdokumenten.
