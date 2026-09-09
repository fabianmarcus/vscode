---
name: firebase-functions-architect
description: "Use when: Firebase Functions entwerfen, strukturieren, versionieren oder refaktorieren; HTTPS/callable/scheduled/background Functions anlegen; Firebase Hosting Rollbacks trotz Functions-Deployments absichern; Funktionsnamen, Ordnerstruktur, API-Versionen oder Exportmuster für serverlose Firebase-Anwendungen festlegen."
user-invocable: false
---

# Firebase Functions Architekt

Entwirft Architektur und Design von Firebase Functions für versionierbare, skalierbare und wartbare serverlose Anwendungen.

## Ziel

Das hauptsächliche Ziel des Skills ist, Firebase Functions übersichtlich und versioniert zu generieren. Jedes Functions Deployment soll weiterhin einen Rollback auf ein vorheriges Firebase Hosting Release ermöglichen. Um dies zu erreichen, wird ein versioniertes Namensschema für Funktionen verwendet und die Dateien werden in einer auf Versionierung optimierten Ordnerstruktur angeordnet.

## Wann verwenden

- Du legst neue Firebase Functions an.
- Du refaktorierst bestehende Firebase Functions in eine versionierbare Struktur.
- Du planst HTTP-, callable-, scheduled-, Pub/Sub-, Firestore-, Auth- oder Storage-Trigger.
- Du musst vermeiden, dass ein Hosting-Rollback gegen eine inkompatible Functions-Version läuft.
- Du definierst ein Namens-, Export- oder Ordnerschema für Firebase Functions.

## Architekturprinzipien

1. **Versionierte öffentliche Oberfläche**

- Jede öffentlich erreichbare Function enthält die API-Version im exportierten Function-Namen.
- Bevorzugtes Schema: `v<major>_<domain><Action>`, z. B. `v1_contactSubmit`.
- Jede Änderung am deploybaren Function-Code erzeugt eine neue Function-Version, statt die bestehende Version umzubauen.

2. **Stabile Rollback-Fähigkeit**

- Eine bereits veröffentlichte Function-Version bleibt verfügbar, solange ein Firebase Hosting Release darauf verweisen kann.
- Veröffentlichte Function-Versionen werden grundsätzlich nicht mehr verändert, damit ein Frontend-Rollback wieder auf den alten Server-Code zurückfallen kann.
- Alte Versionen werden erst entfernt, wenn kein aktives oder rollbackfähiges Hosting Release sie mehr benötigt.
- Neue Hosting-Releases dürfen erst auf neue Function-Versionen umgestellt werden, wenn diese deploybar und getestet sind.

3. **Dünner Trigger, klare Fachlogik**

- Trigger-Dateien enthalten nur Request-/Event-Annahme, Validierung, Auth-Kontext und Response-Mapping.
- Fachlogik liegt in versionierten Service- oder Handler-Modulen.
- Provider-spezifische Firebase APIs werden nicht unnötig tief in Fachlogik verteilt.

4. **Explizite Kompatibilität**

- Kompatibilität entscheidet nicht darüber, ob eine neue Function-Version erstellt wird.
- Jede Änderung am deploybaren Function-Code erzeugt standardmäßig eine neue Function-Version; Breaking Changes beschreiben nur, ob alte Clients die neue Version gefahrlos nutzen könnten.
- Geänderte Pflichtfelder, Statuscodes, Fehlercodes oder Response-Strukturen gelten als Breaking Change.
- Auch non-breaking Bugfixes, Refactorings oder Runtime-Fixes an deploybarem Function-Code erzeugen eine neue Version, wenn Hosting-Rollbacks alte Server-Versionen wiederherstellen können sollen.

5. **Hotfixes sind dokumentierte Ausnahmen**

- Ein Hotfix liegt nur vor, wenn eine bereits veröffentlichte Function-Version produktiv fehlerhaft ist und aktive Clients nicht rechtzeitig auf eine neue Function-Version wechseln können.
- Hotfixes dürfen den öffentlichen Vertrag der alten Function-Version nicht erweitern oder fachlich neu ausrichten.
- Vor einem Hotfix muss dokumentiert werden, warum eine neue Function-Version nicht ausreicht und welches Rollback-Risiko bewusst akzeptiert wird.

6. **Region, Runtime und Security bewusst setzen**

- Region wird nach Priorität gewählt: Frankfurt (`europe-west3`) => Deutschland => Europa => USA.
- Eine niedrigere Priorität darf nur gewählt werden, wenn keine höher priorisierte Region für den konkreten Function-Typ, Dienst oder Projektkontext verfügbar ist.
- Runtime-Optionen, CORS, Auth und App-Check werden pro Function bewusst entschieden.
- Secrets werden über Firebase Secret Manager oder vorhandene Projektkonventionen eingebunden, nicht hart codiert.
- Öffentliche HTTP-Endpunkte bekommen ein explizites CORS- und Rate-Limit-Konzept, sofern das Projekt dafür Infrastruktur hat.

## Empfohlene Ordnerstruktur

```text
functions/
  src/
    index.ts
    contactSubmit/
      v1_contactSubmit/
        v1_contactSubmit.ts
        ContactSubmitRequest.ts
        validateContactSubmitRequest.ts
      v2_contactSubmit/
        v2_contactSubmit.ts
        ContactSubmitRequest.ts
        validateContactSubmitRequest.ts
    newsletterSubscribe/
      v1_newsletterSubscribe/
        v1_newsletterSubscribe.ts
        NewsletterSubscribeRequest.ts
        normalizeNewsletterEmail.ts
    shared/
      types/
      errors/
      firebase/
      validation/
```

Jede Function-Familie liegt in einem unversionierten Oberordner, z. B. `contactSubmit/`. Jede deploybare Server Function liegt darunter in einem eigenen Versionsordner, der exakt wie der Function-Export heißt. Neue Server Functions starten mit Version 1, also mit dem Präfix `v1_`. Wenn sich nur eine Function ändert, wird nur für diese Function ein neuer Versionsordner mit neuem Export angelegt, z. B. `contactSubmit/v2_contactSubmit/`. Unveränderte Function-Familien wie `newsletterSubscribe/` bleiben unangetastet und benötigen kein künstliches Versions-Update.

Der Versionsordner einer Server Function darf Typdefinitionen, Validierung, Mapper oder lokale Hilfsfunktionen enthalten. Der zentrale Export dieses Ordners bleibt aber genau eine deploybare Server Function. Gemeinsam genutzte Bausteine liegen unter `shared/`, müssen aber rückwärtskompatibel bleiben, weil Änderungen daran mehrere Function-Versionen beeinflussen können.

## Exportmuster

`functions/src/index.ts` exportiert nur stabile, deploybare Function-Namen:

```ts
export { v1_contactSubmit } from "./contactSubmit/v1_contactSubmit/v1_contactSubmit";
export { v1_newsletterSubscribe } from "./newsletterSubscribe/v1_newsletterSubscribe/v1_newsletterSubscribe";
export { v2_contactSubmit } from "./contactSubmit/v2_contactSubmit/v2_contactSubmit";
```

Die Exporte bilden den deploybaren Vertrag. Es gibt keine globalen Versionsordner, weil eine neue Version immer nur die einzelne geänderte Server Function betrifft.

## Beispiel: Einzelne Function weiterentwickeln

Ausgangszustand:

```text
functions/src/
  index.ts
  contactSubmit/
    v1_contactSubmit/
      v1_contactSubmit.ts
  newsletterSubscribe/
    v1_newsletterSubscribe/
      v1_newsletterSubscribe.ts
  shared/
```

Wenn sich nur `v1_contactSubmit` ändert, entsteht zusätzlich `v2_contactSubmit`. Das gilt auch für non-breaking Änderungen wie einen Runtime-Fix, wenn ein Frontend-Rollback wieder auf die unveränderte `v1_contactSubmit` zurückfallen können soll. Die Newsletter-Function bleibt unverändert:

```text
functions/src/
  index.ts
  contactSubmit/
    v1_contactSubmit/
      v1_contactSubmit.ts
    v2_contactSubmit/
      v2_contactSubmit.ts
  newsletterSubscribe/
    v1_newsletterSubscribe/
      v1_newsletterSubscribe.ts
  shared/
```

`functions/src/index.ts` exportiert danach beide Kontakt-Versionen und weiterhin die unveränderte Newsletter-Version:

```ts
export { v1_contactSubmit } from "./contactSubmit/v1_contactSubmit/v1_contactSubmit";
export { v2_contactSubmit } from "./contactSubmit/v2_contactSubmit/v2_contactSubmit";
export { v1_newsletterSubscribe } from "./newsletterSubscribe/v1_newsletterSubscribe/v1_newsletterSubscribe";
```

## Vorgehen

1. **Projektkontext prüfen**

- Firebase Functions SDK-Version feststellen (`firebase-functions/v1` oder `firebase-functions/v2`).
- Bestehende Ordnerstruktur, Exportnamen und Deployment-Konventionen prüfen.
- Vorhandene Tests, Emulator-Konfiguration und Lint-/Build-Befehle identifizieren.

2. **Function-Vertrag bestimmen**

- Trigger-Art festlegen: HTTP, callable, schedule, Pub/Sub, Firestore, Auth oder Storage.
- Version und Domain bestimmen.
- Eingabe, Ausgabe, Fehlerfälle, Auth-Anforderungen und Client-Kompatibilität dokumentieren.

3. **Versionierte Datei anlegen oder refaktorieren**

- Neue Functions immer als `v1_`-Function anlegen.
- Jede Function-Familie unter `src/<functionFamily>/` ablegen, z. B. `src/contactSubmit/`.
- Jede deploybare Server Function unter `src/<functionFamily>/<functionName>/` ablegen, z. B. `src/contactSubmit/v1_contactSubmit/`.
- Bei Änderungen am deploybaren Function-Code nur für die betroffene Function eine neue Version anlegen, z. B. `src/contactSubmit/v2_contactSubmit/`.
- Schema/Validierung und Fachlogik aus dem Trigger herausziehen, wenn der Code sonst zu breit wird.

4. **Kompatibilität absichern**

- Bestehende Function-Versionen nicht überschreiben, wenn Clients oder Hosting-Releases sie noch nutzen.
- Änderungen als neue Function mit neuer Major-Version implementieren, ohne unveränderte Functions mitzuversionieren.
- Änderungen an `shared/` nur vornehmen, wenn sie für alle importierenden Function-Versionen kompatibel sind; sonst Logik in den betroffenen Versionsordner kopieren oder versioniert kapseln.
- Hotfixes an alten Function-Versionen nur als bewusst dokumentierte Ausnahme durchführen, weil sie exakte Rollbacks auf den alten Server-Code verhindern.
- Alte Versionen nur mit klarer Deprecation- oder Cleanup-Entscheidung entfernen.

5. **Hotfix-Entscheidung prüfen**

- Standardentscheidung bleibt immer: neue Function-Version erstellen.
- Einen Hotfix nur vorschlagen, wenn die alte Version aktiv produktiv fehlerhaft ist und bestehende Clients weiter exakt diese Version aufrufen müssen.
- Kein Hotfix liegt vor bei normaler Weiterentwicklung, Refactoring, Performance-Optimierung oder Fehlerbehebung, die als neue Version ausgeliefert werden kann.
- Wenn ein Hotfix nötig ist, die Ausnahme im Antwortformat unter **Kompatibilität** benennen und das akzeptierte Rollback-Risiko erklären.

6. **Validieren**

- Mindestens Typecheck oder Build für Functions ausführen, wenn im Projekt vorhanden.
- Falls Tests existieren, die betroffene Function-Version gezielt testen.
- Bei HTTP/callable Functions nach Möglichkeit Emulator- oder Integrationstest ergänzen.

## Namensregeln

- Function-Export: `v<major>_<domain><Action>`.
- Function-Familienordner: unversionierter fachlicher Name, z. B. `contactSubmit/`.
- Versionsordner der Server Function: exakt wie der Exportname, z. B. `v1_contactSubmit/`.
- Datei der Function: exakt wie der Exportname, z. B. `v1_contactSubmit.ts`.
- Schema-Datei: `<functionName>.schema.ts`.
- Service-Datei: `<functionName>.service.ts`.
- Interne Hilfsfunktionen enthalten keine Versionsnummer, wenn sie nicht Teil des deploybaren Vertrags sind.

## Design-Checkliste

- Ist der exportierte Function-Name versioniert?
- Liegt die Server Function unter einem unversionierten Function-Familienordner?
- Heißt der Versionsordner exakt wie der Function-Export?
- Bleiben alte Function-Versionen deploybar?
- Bleiben unveränderte Functions ohne unnötiges Versions-Update erhalten?
- Ist klar, ob die Änderung breaking oder non-breaking ist?
- Bleibt veröffentlichter Function-Code unverändert, damit Frontend-Rollbacks alte Server-Versionen wieder nutzen können?
- Sind Änderungen an `shared/` für alle importierenden Function-Versionen rückwärtskompatibel?
- Falls eine alte Function-Version geändert wird: Ist der Hotfix zwingend nötig, dokumentiert und gegenüber einer neuen Function-Version begründet?
- Ist der Trigger schlank und die Fachlogik testbar?
- Folgt die Region der Priorität Frankfurt (`europe-west3`) => Deutschland => Europa => USA oder ist die Abweichung begründet?
- Sind Auth, CORS, App-Check, Secrets und Runtime-Optionen bewusst gesetzt?
- Gibt es einen klaren Pfad für Hosting-Rollbacks?
- Sind Build, Lint oder Tests für die betroffene Function ausführbar?

## Anti-Patterns

- Einen bestehenden Function-Export mit inkompatiblem Verhalten überschreiben.
- Einen bestehenden Function-Export mit einem Bugfix oder Refactoring überschreiben, obwohl Hosting-Rollbacks alte Server-Versionen benötigen.
- Unversionierte öffentliche HTTP- oder callable Functions anlegen.
- Mehrere API-Versionen in einer einzelnen Trigger-Datei vermischen.
- Hosting-Code direkt an nicht versionierte Function-Namen koppeln.
- Secrets, API-Keys oder Projekt-IDs hart codieren.
- Alte Versionen löschen, bevor Rollback-Fenster und aktive Clients geklärt sind.

## Antwortformat

Wenn der Skill für Planung oder Review verwendet wird, antworte strukturiert mit:

- **Architekturentscheidung**: vorgeschlagene Version, Trigger-Art und Exportname.
- **Namensschema**: Versionen werden im Exportnamen als Präfix geschrieben, z. B. `v1_`.
- **Dateistruktur**: relevante Dateien und Verantwortlichkeiten.
- **Kompatibilität**: Einschätzung zu Breaking Changes und Rollback-Fähigkeit.
- **Hotfix-Ausnahme**: nur ausfüllen, wenn eine veröffentlichte Function-Version absichtlich verändert werden soll.
- **Validierung**: konkrete Checks oder Tests, die ausgeführt werden sollen.

Wenn Code geändert wird, halte die Änderung klein, folge der vorhandenen Repository-Struktur und erkläre abschließend, welche Function-Versionen neu, geändert oder unverändert geblieben sind.
