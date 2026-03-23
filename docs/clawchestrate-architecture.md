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
- Einzelne delegierte Aufgaben laufen nicht als neue externe Agenten, sondern als `DelegationSessions` mit konkreten `DelegationRuns` innerhalb eines bestehenden `ExternalProjectAgent`.
- `ClawChestrate` muss daher persistent speichern:
  - welche `ExternalProjectAgents` ein Projekt besitzt
  - welche `DelegationSessions` in diesen Agenten existieren
  - welche konkrete Aufgabe zu welcher `DelegationSession` gehört
  - welche `DelegationRuns` in diesen Sessions existieren
  - über welche externe Session der inhaltliche Output gelesen werden kann
  - über welchen externen Run Lifecycle, Terminalität und Completion verfolgt werden
- Der aktive Projektkontext einer Lead-Session wird bei projektgebundenen Runs aus vier Quellen zusammengesetzt:
  - Session-Metadaten
  - Project Registry
  - Project Workspace
  - Delegation Registry
- Die `Delegation Registry` ist projektgebundener Verwaltungszustand und speichert die Zuordnung zwischen Projekt, externem Agenten, `DelegationSession`, `DelegationRun`, delegierter Aufgabe und Ergebnisrückführung.
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
- `status`
- `activeRunId`
- `lastRunId`
- `createdAt`
- `updatedAt`

Bedeutung:
- `delegationSessionId`: interne ClawChestrate-ID dieser fachlichen Delegationssession
- `projectId`: Zuordnung zum Projekt
- `leadSessionKey`: Lead-Session, die diese Delegation ausgelöst hat
- `externalProjectAgentId`: externer Projektagent, in dem diese Delegation läuft
- `providerSessionRef`: externe Session-Referenz beim Provider, über die Transcript und History gelesen werden können
- `assignmentSummary`: kurze Beschreibung der delegierten Aufgabe
- `status`: fachlicher Zustand der Delegation auf Session-Ebene, z. B. `open`, `waiting_on_run`, `waiting_on_input`, `completed`, `abandoned`
- `activeRunId`: Verweis auf den aktuell laufenden `DelegationRun`, falls vorhanden
- `lastRunId`: Verweis auf den zuletzt relevanten `DelegationRun`
- `createdAt` / `updatedAt`: Zeitstempel für Verwaltung und Recovery

- `DelegationSession` ist die fachliche Delegationseinheit.
- Sie hält Projekt-, Lead-, Agent- und Session-Zuordnung.
- Sie ist nicht die primäre Ebene für Terminalität, technischen Laufstatus oder konkrete Ergebniszuordnung.

#### `DelegationRun`
- `delegationRunId`
- `delegationSessionId`
- `projectId`
- `providerRunRef`
- `runStatus`
- `resultStatus`
- `resultRef`
- `startedAt`
- `updatedAt`
- `completedAt`
- `completionHandled`

Bedeutung:
- `delegationRunId`: interne ClawChestrate-ID dieses konkreten Delegationsruns
- `delegationSessionId`: Zuordnung zur fachlichen `DelegationSession`
- `projectId`: Zuordnung zum Projekt
- `providerRunRef`: externe Run-Referenz beim Provider
- `runStatus`: technischer Laufstatus dieses konkreten externen Runs, z. B. `queued`, `running`, `completed`, `failed`, `cancelled`, `timeout`
- `resultStatus`: fachlicher Ergebnisstatus dieses konkreten Runs, z. B. `completed`, `partial`, `failed`, `cancelled`, `timeout`, `needs_input`
- `resultRef`: Verweis auf die intern übernommene Ergebnisablage, nicht auf den vollständigen externen Rohoutput
- `startedAt` / `updatedAt` / `completedAt`: Zeitverlauf des konkreten Runs
- `completionHandled`: markiert, ob der terminale Zustand dieses Runs intern bereits verarbeitet wurde

- `DelegationRun` ist die technische Ausführungseinheit.
- Auf dieser Ebene liegen:
  - Terminalität
  - Completion-Erkennung
  - technischer Laufstatus
  - fachlicher Ergebnisstatus
  - Ergebnisreferenz
  - Retry- oder Folgeausführungen innerhalb derselben externen Session

Für v1 trennt die Delegation Registry bewusst zwischen fachlicher Delegationssession und konkretem technischem Delegationsrun.
Die Registry speichert nur Status, Zuordnung und Referenzen.
Große Kontexte, vollständige Ergebnisinhalte und Artefakte bleiben weiterhin außerhalb der Registry, insbesondere im Projekt-Workspace und in den Ergebnisablagen.

### OpenClaw-Adapter und Completion-Erkennung

- Für `OpenClaw` als erstes externes Zielsystem benötigt `ClawChestrate` einen eigenen `OpenClawAdapter`.
- Der Adapter ist die Übersetzungsschicht zwischen dem internen `ClawChestrate`-Modell und dem externen `OpenClaw`-Modell.
- Der Adapter ist verantwortlich für:
  - externe Projektagenten anlegen oder wiederfinden
  - `DelegationSessions` in diesen Agenten anlegen oder wiederfinden
  - konkrete externe `DelegationRuns` starten
  - aktive externe Runs beobachten
  - Terminalität und Completion erkennen
  - den inhaltlichen Output aus der zugehörigen externen Session lesen
  - Ergebnisse in das interne `ClawChestrate`-Format normalisieren
  - interne Completion-Signale für die Lead-Session auslösen

- Der OpenClaw-Worker meldet sein Ergebnis nicht direkt an die `ClawChestrate`-Lead-Session.
- Stattdessen schreibt oder hinterlässt der Worker sein strukturiertes Ergebnis in seiner eigenen OpenClaw-Session.
- Der `OpenClawAdapter` erkennt den terminalen Zustand des konkreten externen Runs und übersetzt ihn in den internen `ClawChestrate`-Rückfluss.

### Push-first mit Polling-Fallback

- `ClawChestrate` verwendet für OpenClaw-Delegationen ein `Push-first mit Polling-Fallback`-Modell.
- Push ist der bevorzugte Weg, Polling ist das Sicherheitsnetz.
- Der Adapter beobachtet nur aktive oder offene `DelegationRuns` samt ihrer zugehörigen externen Sessions, nicht pauschal alle externen Sessions.

- OpenClaw bietet reale Signalwege, die der Adapter nutzen kann, insbesondere:
  - `sessions.subscribe`
  - `sessions.messages.subscribe`
  - Session-/Transcript-Updates über die vorhandenen Gateway-/History-Pfade
- Der Adapter kann diese Signalwege verwenden, um OpenClaw-Sessions gezielt auf neue Nachrichten, neue Abschlussantworten und terminale Zustände zu beobachten.

- Für jeden aktiven oder offenen `DelegationRun` gilt:
  - bevorzugt wird auf verwertbare OpenClaw-Run-, Session- oder Nachrichtenupdates reagiert
  - wenn diese Signale nicht ausreichen oder nicht zuverlässig eintreffen, wird derselbe Run kontrolliert per Polling überprüft

- Polling dient in v1 als verlässlicher Fallback und muss unabhängig von Push immer funktionieren.
- Push und Polling aktualisieren immer denselben Delegationsdatensatz und dürfen keine getrennten Wahrheiten erzeugen.

### Completion-Erkennung und Rückfluss

- Für externe Delegationen unterscheidet `ClawChestrate` sauber zwischen:
  - der externen Worker-Session
  - einem konkreten externen Run in dieser Session
- Der Begriff `terminal` bezieht sich in v1 auf den konkreten externen Run, nicht auf die gesamte Worker-Session.
- Terminal bedeutet: Dieser konkrete externe Run ist beendet und läuft nicht weiter.
- Terminal bedeutet nicht automatisch, dass das Ergebnis erfolgreich oder vollständig ist.

### Bedeutung von Push, Polling und Updates
- Push-Signale zeigen an, dass es neue Information über eine offene externe Delegation gibt.
- Diese neue Information kann z. B. sein:
  - neue Nachricht in der beobachteten Worker-Session
  - neuer Assistant-Output
  - neuer Transcript-Eintrag
  - Statusänderung des externen Runs
- Polling dient als Fallback und prüft offene oder aktive `DelegationRuns` gezielt auf neue Status- oder Ergebnisinformationen, falls kein verlässliches Push-Signal ankommt.
- Push und Polling dienen beide der Beobachtung offener externer Delegationen.
- Sie bedeuten nicht automatisch, dass ein externer Run bereits terminal ist.

### Nicht terminale Zustände
- Nicht terminal bedeutet: Der konkrete externe Run ist noch nicht beendet.
- Nicht terminale Zustände in v1 sind insbesondere:
  - `queued`
  - `running`
- Nicht terminale Updates werden nicht ignoriert.
- Sie führen dazu, dass der bekannte Zustand des `DelegationRun` aktualisiert und die zugehörige Delegation weiter beobachtet wird.
- Nicht terminale Updates lösen jedoch kein internes Completion-Signal für die Lead-Session aus.

### Terminale Zustände
- Terminal bedeutet: Der konkrete externe Run ist beendet.
- Terminale Zustände in v1 sind insbesondere:
  - `completed`
  - `failed`
  - `cancelled`
  - `timeout`
- Bloße Inaktivität oder fehlender neuer Output reichen nicht aus, um Terminalität anzunehmen.

### Bedeutung terminaler Zustände
- Ein terminaler Zustand beendet nur die technische Ausführung des konkreten externen Runs.
- Die fachliche Bedeutung für das Projekt wird erst danach durch `ClawChestrate` bewertet.
- Ein terminaler externer Run kann daher z. B. bedeuten:
  - brauchbares Ergebnis liegt vor
  - Teilergebnis liegt vor
  - Fehlerfall
  - Timeout
  - Abbruch
  - Rückfrage oder fehlende Information statt fertigem Ergebnis

### Verwertbares finales Ergebnis
- Wenn ein externer Run terminal wird, liest der Adapter das zugehörige Endergebnis aus.
- Ein verwertbares finales Ergebnis ist in v1 nur dann gegeben, wenn daraus mindestens ableitbar ist:
  - finaler Ergebnisstatus
  - kurze Summary
  - Issues / Probleme, falls vorhanden
  - Artefakte oder eine leere Artefaktliste
- Ein terminaler technischer Zustand ohne brauchbares Endergebnis ist ein problematischer Endzustand und muss als solcher behandelt werden.

### Einmalverarbeitung
- Damit terminale Zustände nicht mehrfach verarbeitet werden, hält `ClawChestrate` am `DelegationRun` fest, ob ein terminaler Run bereits intern übernommen wurde, z. B. über ein Feld wie `completionHandled`.
- Push und Polling dürfen denselben terminalen Zustand erkennen, aber nicht doppelt weiterverarbeiten.

### Rückfluss
- Sobald ein externer Run terminal ist und noch nicht intern verarbeitet wurde:
  1. wird die passende `DelegationSession` aktualisiert
  2. wird das Endergebnis gelesen und normalisiert
  3. wird die Delegation Registry aktualisiert
  4. wird ein internes Completion-Signal für die zugehörige Lead-Session ausgelöst
- Dieses interne Completion-Signal bedeutet nicht, dass der Fall bereits fachlich gelöst ist.
- Es bedeutet nur, dass ein konkreter externer Run beendet wurde und nun durch die Lead-Session behandelt werden muss.

### Reaktion der Lead-Session auf terminale externe Runs

- Sobald ein externer Run terminal wird, löst `ClawChestrate` einen normalen Folge-Run in der zuständigen Lead-Session aus.
- Dieser Folge-Run bewertet die fachliche Bedeutung des terminalen externen Runs für das Projekt und entscheidet über die weitere Projektführung.

- Die Lead-Session unterscheidet in v1 mindestens folgende Reaktionsarten:

#### 1. Ergebnis übernehmen
- Der externe Run hat ein brauchbares Ergebnis geliefert.
- Die Lead-Session übernimmt das Ergebnis in den Projektzustand und arbeitet normal weiter.

#### 2. Ergebnis teilweise übernehmen und Restarbeit neu planen
- Der externe Run hat verwertbare Teilresultate geliefert, aber nicht alles erreicht.
- Die Lead-Session übernimmt die brauchbaren Teile und plant die verbleibende Arbeit neu.

#### 3. Fehlerfall behandeln
- Der externe Run ist fehlgeschlagen oder hat kein brauchbares Ergebnis geliefert.
- Die Lead-Session behandelt dies als Replan-, Retry- oder Blocker-Fall.

#### 4. Rückfrage oder fehlende Information behandeln
- Der externe Run ist technisch terminal, liefert fachlich aber eine Rückfrage oder einen Klärungsbedarf statt eines fertigen Ergebnisses.
- Die Lead-Session übernimmt diesen Klärungsbedarf und entscheidet, ob sie ihn selbst auflösen kann oder ob Nutzerinput nötig ist.

#### 5. Delegation bewusst beenden oder verwerfen
- Der externe Run wurde abgebrochen oder seine Ergebnisse sollen nicht weiterverwendet werden.
- Die Lead-Session markiert die Delegation entsprechend und entscheidet über den weiteren Projektpfad.

### Grundregel
- Der terminale Zustand eines externen Runs ist nur das Ende der technischen Ausführung.
- Erst der Folge-Run der Lead-Session bestimmt die fachliche Bedeutung für das Projekt.
- Technischer Endzustand und fachliche Projektreaktion sind daher getrennte Ebenen.

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

## Projektkontext im Run

- Projektdateien und Projektzustände sind nicht automatisch vollständig Teil jedes Lead-Runs.
- Für projektgebundene Sessions muss der Projektkontext vor jedem relevanten Run gezielt in den Run-Kontext assembliert werden.

- In v1 erfolgt dieser Projektkontext-Aufbau möglichst eng an bestehenden OpenClaw-Kontext- und Promptaufbau-Mechanismen.
- Dabei wird kein eigener generischer Run-Typ-Mechanismus für ClawChestrate eingeführt.

- Der projektgebundene Run-Kontext wird in v1 aus folgenden Quellen zusammengesetzt:
  - Session-Metadaten
  - Project Registry
  - Project Workspace
  - Delegation Registry

### Kernkontext projektgebundener Lead-Runs
- Für normale projektgebundene Lead-Runs gehören in v1 grundsätzlich zum Kernkontext:
  - `PROJECT.md`
  - `summaries/current.md`
  - `DELEGATION.md`

- Dabei gilt:
  - `PROJECT.md` liefert die grundlegende Projektorientierung und den stabilen Projektrahmen
  - `summaries/current.md` liefert den aktuellen operativen Projektzustand
  - `DELEGATION.md` liefert die projektspezifische Delegationsstrategie

- `PROJECT.md` und `summaries/current.md` bilden gemeinsam den Kern der Projektorientierung.
- Für die konkrete nächste Handlung ist `summaries/current.md` in der Regel das wichtigere operative Dokument, während `PROJECT.md` den stabilen Rahmen sichert.

### Selektiver Zusatzkontext
- Nicht der gesamte Projektkontext wird pauschal vollständig geladen.
- Selektiv ergänzt werden nur bei Bedarf:
  - projektbezogenes `memory/`
  - relevante `artifacts/`
  - kompakte operative Zustände aus Project Registry und Delegation Registry

- Registry-Zustände werden dabei nicht als Rohdump übernommen, sondern in kompakter operativer Form.
- Ziel ist ein assembliertes, run-spezifisches Projektbild statt eines ungefilterten Dateidumps.

### Architektur-Richtung
- Für v1 soll dieser Projektkontext-Aufbau möglichst nah an OpenClaw bleiben und auf bestehenden Prompt-/Kontextaufbau-Mechanismen aufsetzen.
- Perspektivisch kann dieser Mechanismus später in eine stärkere Context-Engine-Integration überführt werden, sobald die ClawChestrate-Grundstruktur stabil läuft.

## Memory- und Compaction-Mapping auf OpenClaw

- ClawChestrate ersetzt in v1 nicht das bestehende Memory- und Compaction-System von OpenClaw.
- Stattdessen nutzt ClawChestrate OpenClaw als Basisschicht und erweitert es um projektspezifische Memory- und Rehydrationsmechanismen.

### OpenClaw-Basisschicht
- Als agentenweite Basisschicht übernimmt ClawChestrate die bestehenden OpenClaw-Mechanismen:
  - `MEMORY.md` als kuratiertes agentenweites Langzeitmemory
  - `memory/YYYY-MM-DD.md` als laufendes agentisches Tages- und Verlaufsmemory
  - `memory_search`
  - `memory_get`
  - Session-Compaction
  - pre-compaction memory flush
  - Session-Pruning für Tool-Result-Bloat

### ClawChestrate-Projektschicht
- Über diese OpenClaw-Basis legt ClawChestrate eine projektspezifische Schicht:
  - `projects/<project-id>/summaries/current.md` als projektbezogene Rehydrationssummary
  - `projects/<project-id>/memory/` als projektbezogenes Verlaufsmemory
- Diese Projektschicht ergänzt OpenClaw, ersetzt es aber nicht.

### Agentenweites vs. projektbezogenes Memory
- Agentenweites, projektübergreifend wiederverwendbares Wissen gehört in die OpenClaw-Basisdateien des Agenten, insbesondere in `MEMORY.md`.
- Laufende agentische Notizen und Tagesverlauf gehören in `memory/YYYY-MM-DD.md`.
- Projektspezifischer Verlauf, projektspezifische Learnings und projektinterne Zwischenerkenntnisse gehören in `projects/<project-id>/memory/`.
- Der aktuelle projektbezogene Arbeits- und Übergabezustand gehört in `projects/<project-id>/summaries/current.md`.

### Unterschied zwischen Projekt-Summary und Projekt-Memory
- `projects/<project-id>/summaries/current.md` hält den aktuellen verdichteten Projektzustand.
- Es beantwortet vor allem:
  - wo das Projekt gerade steht
  - was aktuell wichtig ist
  - was der Fokus ist
  - was als Nächstes zu tun ist
- `projects/<project-id>/memory/` hält den ausführlicheren Projektverlauf.
- Es beantwortet vor allem:
  - was im Projekt passiert ist
  - welche Zwischenerkenntnisse und Begründungen festgehalten werden sollen
  - was später im Projekt noch nachvollziehbar bleiben muss

### Verhältnis zu Compaction
- OpenClaw-Compaction bleibt in v1 die Basisschicht für die Verdichtung des laufenden Session-Kontexts.
- ClawChestrate führt darüber hinaus projektbezogene Verdichtung durch, insbesondere über `summaries/current.md` und projektbezogenes `memory/`.
- Damit werden Session-Compaction und projektbezogene Rehydration bewusst getrennt behandelt.

### Pre-compaction memory flush
- OpenClaw bietet bereits einen pre-compaction memory flush, also einen stillen Turn vor einer Compaction, um dauerhafte Informationen ins Memory zu schreiben.
- ClawChestrate soll diesen Mechanismus nicht ignorieren, sondern später projektspezifisch erweitern.
- Für projektgebundene Sessions bedeutet das perspektivisch:
  - vor Compaction kann nicht nur agentenweites Memory aktualisiert werden
  - sondern zusätzlich projektbezogenes `memory/` und `summaries/current.md`

### Session-Pruning
- OpenClaw-Session-Pruning bleibt in v1 die Basisschicht zur Reduktion alter Tool-Result-Inhalte im unmittelbaren Modellkontext.
- ClawChestrate führt dafür in v1 kein separates paralleles Pruning-System ein.
- Dadurch bleibt die Architektur nah an OpenClaw und vermeidet doppelte Kontextreduktionslogik.

### Grundregel
- OpenClaw stellt die agentweite Memory- und Compaction-Basis bereit.
- ClawChestrate ergänzt darüber eine projektspezifische Memory- und Rehydrationsschicht.
- Das ClawChestrate-Memory-Modell baut damit auf OpenClaw auf, statt es in v1 neu zu ersetzen.

## Abbildung der OpenClaw-Run-Ebene in der Delegation Registry

- ClawChestrate übernimmt für externe OpenClaw-Delegationen die Trennung zwischen externer Session und konkretem externem Run.
- In OpenClaw ist die Session der längerlebige Arbeits- und Transcript-Kontext.
- Ein Run ist eine konkrete Ausführung innerhalb dieses Session-Kontexts.
- Pro externer Session gibt es in OpenClaw immer nur einen aktiven Run gleichzeitig.
- Dieselbe externe Session kann jedoch über die Zeit mehrere Runs nacheinander haben.
- Terminalität, Lifecycle und `agent.wait` beziehen sich daher auf den konkreten externen Run, nicht auf die Session als Ganzes.

### Drei Ebenen externer Delegation
- `ExternalProjectAgent`
  - der langlebige externe Spezialagent eines Projekts
- `DelegationSession`
  - die fachliche Delegationseinheit in einer externen Session
- `DelegationRun`
  - der konkrete externe Run innerhalb dieser Session

### Rolle von `DelegationSession`
- `DelegationSession` ist die fachliche Delegationseinheit.
- Sie hält insbesondere:
  - Projektzuordnung
  - Lead-Session-Zuordnung
  - externen Agenten
  - externe Session-Referenz
  - Aufgabenbeschreibung
  - Verweis auf den aktiven oder letzten relevanten Run
- `DelegationSession` ist nicht die primäre technische Ebene für Terminalität oder Ergebniszuordnung.

### Rolle von `DelegationRun`
- `DelegationRun` ist die technische Ausführungseinheit.
- Er hält insbesondere:
  - Referenz auf die zugehörige `DelegationSession`
  - externe Run-Referenz
  - technischen Laufstatus
  - fachlichen Ergebnisstatus
  - Ergebnisreferenz
  - Zeitstempel
  - Completion-Verarbeitungsstatus
- Auf Ebene des `DelegationRun` liegen:
  - Terminalität
  - Completion-Erkennung
  - Ergebniszuordnung
  - Retry-Abbildung einzelner Ausführungen

### Bedeutung von Session-Ref und Run-Ref
- In OpenClaw liefert der Run-Lifecycle nur den technischen Zustand eines konkreten Runs.
- `agent.wait` liefert für einen `runId` insbesondere den terminalen Status, aber nicht den vollständigen inhaltlichen Output.
- Der eigentliche inhaltliche Output liegt in der zugehörigen externen Session bzw. ihrem Transcript.
- Für ClawChestrate bedeutet das:
  - `providerRunRef` dient zur Beobachtung von Lifecycle, Terminalität und Completion des konkreten externen Runs
  - `providerSessionRef` dient zum Auslesen des eigentlichen inhaltlichen Outputs aus der externen Session

### Push, Polling und Output-Übernahme
- Push- und Lifecycle-Signale dienen in v1 primär dazu, Statusänderungen und terminale Zustände externer Runs zu erkennen.
- Transcript- und Message-Events können dabei bereits inhaltliche Ausschnitte des externen Outputs mitliefern.
- Die verlässliche finale Ergebnisübernahme erfolgt jedoch nicht allein aus Push-Events.
- Stattdessen liest der `OpenClawAdapter` nach einem terminalen externen Run den relevanten Output gezielt aus der zugehörigen externen Session, insbesondere über Session-History- und Transcript-Flächen.
- Push ist damit das bevorzugte Signal für Veränderung und Terminalität.
- Die Session-History ist die verlässliche Quelle für die finale inhaltliche Ergebnisübernahme.

### Grundregel
- Terminalität bezieht sich in ClawChestrate auf den konkreten `DelegationRun`, nicht auf die `DelegationSession` als Ganzes.
- Wenn dieselbe delegierte Aufgabe in derselben externen Session erneut ausgeführt wird, bleibt die `DelegationSession` bestehen, aber es entsteht ein neuer `DelegationRun`.
- Dadurch bleiben Session-Kontext, technische Ausführung, Ergebnisübernahme und fachliche Delegationsbewertung sauber getrennt.

## Noch offen
- Über welche konkrete OpenClaw-Fläche der `OpenClawAdapter` externe Arbeit in v1 startet und wieder aufnimmt, insbesondere die Abgrenzung zwischen `agent`, `sessions_send`, `sessions_spawn` und ggf. ACP-Spawns
- Über welchen konkreten OpenClaw-Mechanismus der Projektkontext in v1 in den Lead-Run eingebracht wird, insbesondere die Abgrenzung zwischen `before_prompt_build`, `agent:bootstrap` und späterer Context-Engine-Integration
- Wie stark ClawChestrate in v1 OpenClaws bestehende Retrieval- und Memory-Funktionen für Projekt-Memory aktiv nutzt, insbesondere `memory_search`, `memory_get` und ggf. Session-/Transcript-Retrieval