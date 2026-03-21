# ClawChestrate Architekturstand 1

## Summary
`ClawChestrate` soll die `openclaw`-Architektur möglichst eng übernehmen und nicht neu erfinden.  
Die Grundform bleibt daher: `Gateway -> Agenten -> Workspaces -> Sessions -> Runs`.  
Der Unterschied zu `openclaw` liegt nicht in einer völlig anderen Runtime, sondern darin, dass `ClawChestrate`-Agenten Projekte orchestrieren und Arbeit primär an externe Agenten delegieren.

## Bisher festgezurrt
- `ClawChestrate` übernimmt die `openclaw`-Grundarchitektur mit Gateway, Sessions, Runs und Agent-Workspaces.
- Ein `ClawChestrate`-Agent ist die langlebige fachliche Einheit und besitzt einen eigenen Workspace.
- Projekte liegen im Workspace des zuständigen Agents, nicht in einem separaten globalen Projekt-Dateiraum.
- Ein Agent kann mehrere Projekte seines Fachbereichs verwalten.
- Ein Projekt ist langlebig; Sessions sind austauschbare Arbeitskontexte.
- Pro Projekt gibt es in v1 maximal eine aktive `Lead-Session`.
- In v1 gibt es keine `Helper-Sessions`; nur `Lead-Session` und `External Runs`.
- Sessions bleiben `openclaw`-artig Session-Store-/Gateway-Objekte und werden nicht in den Workspace verlegt.
- Die Projektbindung erfolgt über minimale zusätzliche Session-Metadaten:
  - `projectId`
  - `projectRole`
  - `projectSessionState`
- Projektanlage erfolgt nicht automatisch, sondern nur explizit:
  - Nutzer sagt, dass daraus ein Projekt werden soll
  - oder Agent schlägt es vor und Nutzer bestätigt
- Projektstart entsteht aus einer bestehenden normalen Session; dieselbe Session bleibt erhalten und wird nach Projektanlage zur `Lead-Session`.
- Runs bleiben zunächst wie in `openclaw` sessiongebunden; spätere Runs der projektgebundenen Session laufen damit faktisch im Projektkontext.

## Projektmodell
- Jedes Projekt besitzt zwei Repräsentationen:
  - einen Projektordner im Agent-Workspace für Inhalte, Artefakte und Rehydration
  - einen offiziellen Eintrag in einer zentralen `Project Registry` für Verwaltung und Systemzustand
- Der Projektordner ist die Arbeits- und Inhaltsseite des Projekts.
- Die `Project Registry` ist die zentrale Verwaltungsinstanz für Projekte.
- Die Registry speichert keine vollständigen Arbeitsdokumente, sondern offizielle Projektmetadaten wie:
  - `projectId`
  - `agentId`
  - Titel/Name
  - Status
  - aktive Lead-Session
  - Zeitstempel und Verwaltungszustand
- Die Registry ist nötig, damit das System Projekte zuverlässig finden, auflisten, sperren und wiederaufnehmen kann.

## Projektstart-Flow
- Nutzer spricht zunächst normal mit einem bestehenden `ClawChestrate`-Agenten in einer Session.
- In dieser Session wird das Vorhaben gemeinsam geschärft und geplant.
- Nach expliziter Entscheidung wird aus dem besprochenen Session-Kontext ein offizielles Projekt erzeugt.
- Die Projektanlage läuft in einem normalen neuen Run derselben Session.
- Der Projektstart besteht intern aus drei Kernoperationen:
  - `ensureProjectWorkspace(...)`
  - `createProjectRecord(...)`
  - `bindSessionToProject(...)`
- `ensureProjectWorkspace(...)` erzeugt die minimale Projektstruktur im Agent-Workspace.
- `createProjectRecord(...)` erzeugt den offiziellen Eintrag in der `Project Registry`.
- `bindSessionToProject(...)` bindet die bestehende Session als aktive Lead-Session an dieses Projekt.
- Die Session bleibt bestehen und erhält danach:
  - `projectId`
  - `projectRole = lead`
  - `projectSessionState = active`

## Workspace- und Gateway-Richtung
- Projektordner liegen direkt im Agent-Workspace unter `projects/<project-id>/`.
- Minimaler Projekt-Workspace in v1:
  - `PROJECT.md`
  - `DELEGATION.md`
  - `summaries/current.md`
  - `memory/`
  - `artifacts/`
- Projekte sollen am Gateway einen eigenen Namespace erhalten, analog zu `sessions.*`, `cron.*`, `agents.*`.
- Minimal vorgesehene Projektmethoden in v1:
  - `projects.create_from_session`
  - `projects.get`
  - `projects.list`
  - `projects.update`
- `projects.create_from_session` nutzt die bestehende Session als inhaltliche Quelle und legt daraus das offizielle Projekt an.

## Memory-Modell
- `ClawChestrate` nutzt in v1 mehrere Memory-Ebenen:
  - aktiver Session-Kontext
  - `summaries/current.md` als kompakte Projekt-Rehydration
  - projektbezogenes `memory/` als Verlaufsmemory
  - agentenweites Langzeitmemory für projektübergreifendes Lernen
  - `artifacts/` als Beleg- und Ergebnisablage externer Runs
- Nicht alles wird in den aktiven Kontext geladen; stattdessen wird Wissen zwischen diesen Ebenen verdichtet verteilt.
- `summaries/current.md` hält den aktuellen kompakten Projektstand für neue oder fortgesetzte Lead-Sessions.
- Projekt-`memory/` hält den ausführlicheren Verlauf, Zwischenerkenntnisse und projektinterne Learnings.
- Agentenweites Langzeitmemory speichert nur verallgemeinerbares Wissen, das zukünftigen Projekten desselben Agents hilft.
- Artefakte bleiben primär Roh- und Ergebnisbelege und werden nur selektiv in Summary oder Memory überführt.

## Memory-Compaction in v1
- Verdichtung erfolgt nach jedem **relevanten Run**, also wenn der Projektzustand inhaltlich verändert wurde.
- Verdichtung erfolgt zusätzlich verpflichtend bei **Session-Ende** oder **Handover**.
- Verdichtung kann zusätzlich bei **Context-Druck** ausgelöst werden, wenn der aktive Session-Kontext zu groß wird.
- Relevante Änderungen für die Summary sind insbesondere:
  - neue Entscheidungen
  - neuer aktiver Fokus
  - neue oder gelöste Blocker
  - wichtige Ergebnisse externer Agentenläufe
  - erkennbare Projektstatusänderungen
- Triviale oder rein klärende Runs ohne echten Zustandsgewinn müssen nicht automatisch eine Summary-Verdichtung auslösen.

## Externe Agenten, Delegation und Ergebnisrückführung
- Ein Projekt kann mehrere `External Project Agents` besitzen, typischerweise rollenbasiert, z. B. `coding` oder `research`.
- Ein `External Project Agent` ist ein langlebiger externer Spezialagent für genau ein Projekt und eine klar definierte Rolle.
- Einzelne delegierte Aufgaben laufen nicht als neue externe Agenten, sondern als eigene `Delegation Sessions` innerhalb eines bestehenden `External Project Agent`.
- `ClawChestrate` muss daher persistent speichern:
  - welche `External Project Agents` ein Projekt besitzt
  - welche `Delegation Sessions` in diesen Agenten existieren
  - welche Aufgabe an welche externe Session gebunden ist
  - über welche externe Session das Ergebnis zurückgeholt werden kann
- Der aktive Projektkontext einer Lead-Session wird bei projektgebundenen Runs aus vier Quellen zusammengesetzt:
  - Session-Metadaten
  - Project Registry
  - Project Workspace
  - Delegation Registry
- Die `Delegation Registry` ist projektgebundener Verwaltungszustand und speichert die Zuordnung zwischen Projekt, externem Agenten, externer Session, delegierter Aufgabe und Ergebnisrückführung.
- Externe Agentenläufe liefern in v1 ein strukturiertes Rückgabeformat.
- Minimales Rückgabeformat in v1:
  - `status`
  - `summary`
  - `artifacts`
  - `issues`
  - optional `nextStepHints`
- Finale Ergebnisstatus in v1:
  - `completed`
  - `partial`
  - `failed`
  - `cancelled`
  - `timeout`
- Zusätzlich zum finalen Ergebnisstatus existiert ein technischer Laufstatus für externe Agentenläufe:
  - `queued`
  - `running`
  - `completed`
  - `failed`
  - `cancelled`
  - `timeout`
- Externe Agentensessions liefern Ergebnisse nicht direkt als unstrukturierte Chatantwort an die Lead-Session zurück.
- Stattdessen wird das Ergebnis zunächst persistent an der zugehörigen Delegation gespeichert.
- Nach Abschluss einer externen Delegation erzeugt das System ein internes Completion-Signal für die Lead-Session des Projekts.
- Dieses Completion-Signal triggert keinen Sondermodus, sondern weckt die Lead-Session für einen normalen Folge-Run.
- Im Folge-Run übernimmt die Lead-Session das Ergebnis kontrolliert:
  - relevante Artefakte werden übernommen
  - Projekt-Memory und `summaries/current.md` werden aktualisiert
  - der nächste Projektschritt wird geplant

### Minimale Feldstruktur der Delegation Registry

#### `ExternalProjectAgent`
- `externalProjectAgentId`
- `projectId`
- `role`
- `provider`
- `providerAgentRef`
- `status`
- `createdAt`
- `updatedAt`

Bedeutung:
- `externalProjectAgentId`: interne ClawChestrate-ID dieses externen Projektagenten
- `projectId`: Zuordnung zum Projekt
- `role`: z. B. `coding`, `research`
- `provider`: externer Agentenprovider, z. B. `openclaw`
- `providerAgentRef`: externe Referenz/ID dieses Agenten beim Provider
- `status`: Verwaltungsstatus des externen Projektagenten, z. B. `active`, `paused`, `archived`
- `createdAt` / `updatedAt`: Zeitstempel für Verwaltung und Recovery

#### `DelegationSession`
- `delegationSessionId`
- `projectId`
- `leadSessionKey`
- `externalProjectAgentId`
- `providerSessionRef`
- `assignmentSummary`
- `runStatus`
- `resultStatus`
- `resultRef`
- `startedAt`
- `updatedAt`
- `completedAt`

Bedeutung:
- `delegationSessionId`: interne ClawChestrate-ID dieser Delegationssession
- `projectId`: Zuordnung zum Projekt
- `leadSessionKey`: interne Lead-Session, die diese Delegation ausgelöst hat
- `externalProjectAgentId`: externer Projektagent, in dem diese Session läuft
- `providerSessionRef`: externe Session-ID / externer `sessionKey`, über die das Ergebnis wiedergefunden wird
- `assignmentSummary`: kurze Beschreibung der delegierten Aufgabe
- `runStatus`: technischer Laufstatus der Delegationssession
- `resultStatus`: fachlicher Ergebnisstatus, falls die Session bereits beendet wurde
- `resultRef`: Verweis auf gespeichertes Ergebnis oder übernommene Artefakte
- `startedAt`, `updatedAt`, `completedAt`: Zeitverlauf und Abschlussstatus der Delegation

Für v1 speichert die Delegation Registry bewusst nur den stabilen Verwaltungs- und Zuordnungszustand.
Große Kontexte, vollständige Ergebnisinhalte und Artefakte bleiben außerhalb der Registry, insbesondere im Projekt-Workspace und in den Ergebnisablagen.

### OpenClaw-Adapter und Completion-Erkennung

- Für `OpenClaw` als erstes externes Zielsystem benötigt `ClawChestrate` einen eigenen `OpenClawAdapter`.
- Der Adapter ist die Übersetzungsschicht zwischen dem internen `ClawChestrate`-Modell und dem externen `OpenClaw`-Modell.
- Der Adapter ist verantwortlich für:
  - externe Projektagenten anlegen oder wiederfinden
  - Delegation Sessions in diesen Agenten starten
  - externe Sessions beobachten
  - Completion erkennen
  - Ergebnisse in das interne `ClawChestrate`-Format normalisieren
  - interne Completion-Signale für die Lead-Session auslösen

- Der OpenClaw-Worker meldet sein Ergebnis nicht direkt an die `ClawChestrate`-Lead-Session.
- Stattdessen schreibt oder hinterlässt der Worker sein strukturiertes Ergebnis in seiner eigenen OpenClaw-Session.
- Der `OpenClawAdapter` erkennt den Abschluss dieser Session und übersetzt ihn in den internen `ClawChestrate`-Rückfluss.

### Push-first mit Polling-Fallback

- `ClawChestrate` verwendet für OpenClaw-Delegationen ein `Push-first mit Polling-Fallback`-Modell.
- Push ist der bevorzugte Weg, Polling ist das Sicherheitsnetz.
- Der Adapter beobachtet nur offene `DelegationSessions`, nicht pauschal alle externen Sessions.

- OpenClaw bietet reale Signalwege, die der Adapter nutzen kann, insbesondere:
  - `sessions.subscribe`
  - `sessions.messages.subscribe`
  - Session-/Transcript-Updates über die vorhandenen Gateway-/History-Pfade
- Der Adapter kann diese Signalwege verwenden, um OpenClaw-Sessions gezielt auf neue Nachrichten, neue Abschlussantworten und terminale Zustände zu beobachten.

- Für jede offene `DelegationSession` gilt:
  - bevorzugt wird auf verwertbare OpenClaw-Session- oder Nachrichtenupdates reagiert
  - wenn diese Signale nicht ausreichen oder nicht zuverlässig eintreffen, wird dieselbe Delegation kontrolliert per Polling überprüft

- Polling dient in v1 als verlässlicher Fallback und muss unabhängig von Push immer funktionieren.
- Push und Polling aktualisieren immer denselben Delegationsdatensatz und dürfen keine getrennten Wahrheiten erzeugen.

### Completion-Erkennung und Rückfluss

- Eine externe Delegation gilt erst dann als abgeschlossen, wenn:
  - ein terminaler technischer Zustand erkannt wurde
  - eine verwertbare strukturierte Abschlussantwort oder ein verwertbares Endergebnis vorliegt
  - das Ergebnis noch nicht intern verarbeitet wurde
- Der Adapter verarbeitet Completion nicht mehrfach.
- Dazu wird an der `DelegationSession` festgehalten, ob Completion bereits übernommen wurde, z. B. über ein Feld wie `completionHandled`.

- Nach erkannter Completion passiert folgender Rückfluss:
  1. die passende `DelegationSession` wird aktualisiert
  2. das Ergebnis wird normalisiert und persistent gespeichert oder referenziert
  3. ein internes Completion-Signal wird für die zugehörige Lead-Session ausgelöst
  4. die Lead-Session wird dadurch für einen normalen Folge-Run geweckt
  5. im Folge-Run übernimmt die Lead-Session das Ergebnis kontrolliert in Projektzustand, `artifacts/`, Projekt-`memory/` und `summaries/current.md`

- Das interne Completion-Signal ist nicht selbst schon die Ergebnisverarbeitung.
- Es ist nur der Auslöser dafür, dass die Lead-Session den nächsten normalen Orchestrierungsrun startet und das Ergebnis übernimmt.

### Architekturregel für OpenClaw-Delegationen

- Der `OpenClawAdapter` ist ein Client auf OpenClaw-Gateway-, Session- und Nachrichten-Signale.
- Externe Ergebnisse werden nicht direkt als freie Chatantwort in die Lead-Session übernommen.
- Stattdessen werden sie über den Adapter:
  - der richtigen `DelegationSession` zugeordnet
  - in die Delegation Registry geschrieben
  - und dann kontrolliert an die Lead-Session zurückgeführt

## Grenze zwischen Registry-/Systemwahrheit und Workspace-Dokumenten

- `ClawChestrate` trennt strikt zwischen operativer Systemwahrheit und agentischer Arbeitsdokumentation.
- Registry-/Systemwahrheit enthält alle Zustände, auf die das System präzise schalten, sperren, zuordnen, recovern und überwachen können muss.
- Workspace-Dokumente enthalten alle Inhalte, die für Verständnis, Rehydration, Nachvollziehbarkeit und agentische Projektarbeit gedacht sind.

- In Registry-/Systemwahrheit gehören insbesondere:
  - Project Registry mit offizieller Projektidentität, Status, Agent-Zuordnung und aktiver Lead-Session
  - Session Store mit projektbezogenen Session-Metadaten
  - Delegation Registry mit `ExternalProjectAgents`, `DelegationSessions`, Status, Referenzen und Ergebniszuordnung

- In den Workspace gehören insbesondere:
  - `PROJECT.md`
  - `DELEGATION.md`
  - `summaries/current.md`
  - projektbezogenes `memory/`
  - `artifacts/`

- Grundregel:
  - Wenn ein Zustand für Maschinenlogik, Eindeutigkeit oder Recovery nötig ist, gehört er in Registry bzw. Session Store.
  - Wenn ein Zustand für Verständnis, Rehydration, Nachvollziehbarkeit oder agentische Arbeit gedacht ist, gehört er in den Workspace.

- Dieselbe fachliche Realität kann in beiden Ebenen vorkommen, aber mit unterschiedlicher Funktion:
  - Registry speichert die knappe operative Wahrheit
  - Workspace speichert die erklärende und arbeitsbezogene Form

## Exklusivitäts- und Handover-Mechanik für Lead-Sessions

- Pro `projectId` darf es in v1 genau eine Session geben mit:
  - `projectRole = lead`
  - `projectSessionState = active`
- Diese Session ist die einzige offizielle steuernde Session des Projekts.

- Lead-Sessions kennen in v1 drei Zustände:
  - `active`
  - `paused`
  - `closed`

- Bedeutung der Zustände:
  - `active`: führt das Projekt aktuell und darf offiziellen Projektzustand fortschreiben
  - `paused`: ist historisch weiter vorhanden, führt das Projekt aber nicht mehr aktiv
  - `closed`: ist beendet und bleibt nur als Historie erhalten

- Ein Handover ist ein expliziter Führungswechsel von einer bisherigen Lead-Session zu einer neuen Lead-Session.
- Eine neue Lead-Session darf nicht automatisch aktiv werden, solange bereits eine andere Lead-Session für dasselbe Projekt `active` ist.

- Der Handover läuft in v1 kontrolliert ab:
  1. eine neue Session soll das Projekt übernehmen
  2. das System prüft, ob bereits eine aktive Lead-Session existiert
  3. die bisherige Lead-Session muss explizit auf `paused` oder `closed` gesetzt werden
  4. der Projektzustand wird vor Übergabe ausreichend verdichtet, insbesondere über `summaries/current.md`
  5. erst danach wird die neue Lead-Session auf `active` gesetzt und als offizielle Projektführung registriert

- Alte und neue Lead-Session dürfen nie gleichzeitig `active` sein.
- Nur die aktuell `active` Lead-Session darf:
  - offiziellen Projektzustand fortschreiben
  - neue Delegationen als Projektführung starten
  - `summaries/current.md` als kanonischen Projektstand aktualisieren

- Die Exklusivität der aktiven Lead-Session wird in v1 über den Session Store und die Projektzuordnung abgesichert.
- Der Projekteintrag hält zusätzlich fest, welche Session aktuell die offizielle Lead-Session des Projekts ist.
- Ein Wechsel der Lead-Session ist in v1 kein Normalfall für jeden kleinen Arbeitsschritt.
- Er ist vor allem sinnvoll bei:
  - Wiederaufnahme nach Session-Ende
  - zu groß oder unhandlich gewordenem Session-Kontext
  - bewusstem Phasenwechsel im Projekt
  - technischer oder organisatorischer Notwendigkeit
- Ziel des Handovers ist nicht häufiges Umschalten, sondern kontrollierte Projektkontinuität trotz austauschbarer Sessions.

## Mindestklarheit für `projects.create_from_session`

- `projects.create_from_session` darf in v1 nur erfolgreich sein, wenn die bestehende Session bereits genug Klarheit für ein brauchbares offizielles Projekt liefert.
- Ziel ist es, zu frühe oder inhaltlich leere Projektanlagen zu vermeiden.

- Mindestklarheit in v1 bedeutet:
  - es gibt ein verständliches Ziel oder eine verständliche Mission des Vorhabens
  - es gibt einen grob erkennbaren Scope des Projekts
  - es gibt einen ersten sinnvollen Arbeitsfokus nach Projektstart
  - der bisherige Session-Kontext reicht aus, um `PROJECT.md`, `summaries/current.md` und mindestens eine erste sinnvolle Projektinitialisierung zu befüllen

- Nicht erforderlich für die Projektanlage in v1 sind:
  - vollständige Tasklisten
  - endgültige Architekturentscheidungen
  - vollständige Risikoanalysen
  - vollständig ausformulierte Delegationsstrategien

- Wenn diese Mindestklarheit noch nicht gegeben ist, soll `projects.create_from_session` das Projekt nicht stillschweigend anlegen.
- Stattdessen soll der Projektstart zurückgestellt und die fehlende Klarheit im Gespräch weiter geschärft werden.

- Die Projektanlage bleibt in v1 ein expliziter Übergang und keine automatische Ableitung aus jeder beliebigen Unterhaltung.

### Rolle von Doku und Runtime-Policy

- Die Mindestklarheit ist zunächst eine Produkt- und Architekturregel und wird im Architekturstand dokumentiert.
- Die Architektur-Doku ist dabei die menschliche und konzeptionelle Wahrheit, nicht direkt der vollständige Run-Kontext.
- Für die Laufzeit wird aus dieser Regel eine kompakte operative Prüfrichtlinie abgeleitet.
- Diese operative Prüfrichtlinie wird im `projects.create_from_session`-Flow verwendet, sehr wahrscheinlich als flow-spezifischer Prompt- oder Policy-Baustein.
- Der eigentliche Projekt-Erstellungsrun bewertet den vorhandenen Session-Kontext gegen diese festgelegten Mindestkriterien.
- Dabei wird nicht frei nach Bauchgefühl entschieden, sondern anhand der dokumentierten Kriterien:
  - Ziel/Mission
  - grober Scope
  - erster Arbeitsfokus
  - ausreichende Grundlage für die Initialbefüllung der Projektdateien
- Wenn der Session-Kontext die Mindestkriterien nicht erfüllt, wird `projects.create_from_session` nicht erfolgreich abgeschlossen, sondern der fehlende Klärungsbedarf im Gespräch zurückgespiegelt.

- Beziehung der Ebenen:
  - Architektur-Doku definiert, was gelten soll
  - Runtime-Policy setzt diese Regel operativ für die Laufzeit um
  - der konkrete Run nutzt Session-Kontext plus Runtime-Policy zur Entscheidung

## Initiale Übernahme aus dem Session-Kontext

- Die Initialbefüllung von `PROJECT.md`, `DELEGATION.md` und `summaries/current.md` wird in v1 aus der Funktion dieser Dateien abgeleitet.
- Dabei gilt:
  - `PROJECT.md` beschreibt die stabile Definition des konkreten Projekts
  - `DELEGATION.md` beschreibt die projektspezifische Delegationsstrategie des konkreten Projekts
  - `summaries/current.md` beschreibt den aktuellen verdichteten Projektzustand
- Die Initialbefüllung darf daher nicht die allgemeine ClawChestrate-Architekturdoku in diese Dateien kopieren.
- Sie darf nur den projektbezogenen Inhalt aus dem Session-Kontext übernehmen, der für die jeweilige Datei funktional passend ist.

### `PROJECT.md`
- `PROJECT.md` ist das stabile Briefing des konkreten Projekts.
- Initial übernommen werden nur bereits ausreichend klare, stabile Projektinformationen:
  - `title`
  - `mission`
  - `scope`
  - `constraints`, falls besprochen
  - `success criteria`, falls besprochen
- Nicht in `PROJECT.md` gehören:
  - allgemeine ClawChestrate-Architekturregeln
  - aktueller Arbeitszustand
  - laufende Delegationen
  - Chatverlauf
  - vollständige Detailplanung

### `DELEGATION.md`
- `DELEGATION.md` ist keine Statusdatei und keine Registry für laufende Delegationen.
- Sie beschreibt die projektspezifische Delegationsstrategie des konkreten Projekts.
- Diese Strategie muss beim Projektstart nicht vollständig vom Nutzer vorgegeben werden.
- Stattdessen leitet der ClawChestrate-Agent aus dem Session-Kontext eine erste sinnvolle Delegationsstrategie für dieses konkrete Projekt ab.
- Explizite Nutzerpräferenzen gehen dabei vor.

- `DELEGATION.md` hält fest:
  - welche externen Rollen dieses Projekt typischerweise nutzen soll, falls bereits ableitbar
  - welche Arten von Arbeit typischerweise delegiert werden sollen
  - welche Arten von Arbeit die Lead-Session selbst behalten soll
  - welche Ergebnisformen von externen Agenten in diesem Projekt erwartet werden
  - welche projektspezifischen Delegationshinweise oder Vorsichtsregeln gelten, falls bereits klar

- Nicht in `DELEGATION.md` gehören:
  - aktive externe Agenteninstanzen
  - laufende externe Sessions
  - konkrete Delegationsstatus
  - Run-Historie
  - Ergebnisarchive
  - allgemeine Systemregeln, die nicht projektspezifisch sind

- Wenn aus der Session noch keine klare projektspezifische Delegationsstrategie ableitbar ist, darf `DELEGATION.md` in v1 bewusst knapp starten und später fortgeschrieben werden.
- `DELEGATION.md` verändert sich, wenn sich die Delegationsstrategie des Projekts weiterentwickelt, nicht bei jedem laufenden Statuswechsel einzelner Delegationen.

### `summaries/current.md`
- `summaries/current.md` ist die aktuelle rehydrierbare Zustandsverdichtung des Projekts.
- Beim Projektstart werden aus dem Session-Kontext übernommen:
  - kompakte Mission
  - aktueller Startzustand / `current state`
  - erster `active focus`
  - bereits getroffene wichtige Entscheidungen, falls vorhanden
  - bekannte Blocker / Risiken, falls vorhanden
  - erste `suggested next steps`
- Nicht in `summaries/current.md` gehören:
  - die vollständige stabile Projektdefinition
  - die komplette Historie
  - lange Begründungsprosa
  - vollständige Artefaktinhalte

### Grundregel der Initialbefüllung
- `PROJECT.md` erhält nur die stabile Projektdefinition.
- `DELEGATION.md` erhält nur die bereits klare oder sinnvoll ableitbare projektspezifische Delegationsstrategie.
- `summaries/current.md` erhält nur den aktuellen verdichteten Startzustand.
- Die Initialbefüllung ist immer eine Verdichtung des Session-Kontexts und niemals ein Chat-Dump.

## Noch offen
- Konkrete terminale Erkennungsregeln, mit denen der `OpenClawAdapter` eine externe Session als abgeschlossen bewertet