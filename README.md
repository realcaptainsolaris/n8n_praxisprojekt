# Praxisprojekt: Support-Automatisierung

Referenzprojekt für Kapitel 20, Stand 0.3. Drei zusammengehörige Workflows.
Das vorhandene SQL-Schema mit support_customers, support_articles und
support_requests wird unverändert verwendet. Voraussetzung: 01_setup_support.sql
wurde bereits ausgeführt (3 Kunden, 5 freigegebene Artikel).

## Aufbau

| Workflow | Aufgabe |
|---|---|
| 31 Support-Automatisierung | Eingang validieren, neue Anfrage registrieren, Teilprozesse aufrufen, Ergebnis speichern |
| 32 Supportanfrage analysieren | Kundenkontext laden, Klassifikation prüfen, freigegebene Wissensartikel auswählen |
| 33 Supportantwort vorbereiten | Entwurf erstellen, formal prüfen und Entscheidung zurückgeben |

Ein Request wird als ein Item verarbeitet. Kein Batch-Eingang.
32 und 33 besitzen je ein typisiertes Eingabefeld context (Object).
31 übergibt an 32 {request}; an 33 das vollständige Analyseergebnis.
Kein Teilworkflow schreibt in die Datenbank. 32 liest daraus, 31 schreibt.

## Import und Einrichtung

1. 32_Supportanfrage_analysieren.json importieren.
2. In 32 in „Kundenkontext laden“ und „Freigegebene Wissensartikel laden“
   das Postgres-Credential für n8n_training auswählen. In „Anfrage klassifizieren“
   das OpenAI-Credential auswählen. Speichern.
3. 33_Supportantwort_vorbereiten.json importieren. In „Antwortentwurf erstellen“
   das OpenAI-Credential auswählen. Speichern.
4. Die bereits importierte Fassung von 31 durch 31_Support-Automatisierung.json
   ersetzen oder den alten, ungenutzten Workflow entfernen und die neue Datei importieren.
   Nur eine Fassung von 31 verwenden.
5. In 31 bei „Anfrage eindeutig anlegen“ und „Supportergebnis speichern“
   das Postgres-Credential für n8n_training auswählen.
6. In „Supportanfrage analysieren aufrufen“ den Workflow „32 Supportanfrage analysieren“
   aus der Liste wählen. Das Feld context bleibt auf {{ { request: $json.request } }}.
7. In „Supportantwort vorbereiten aufrufen“ den Workflow „33 Supportantwort vorbereiten“
   wählen. Das Feld context bleibt auf {{ $json }}.
8. Beide Aufrufe müssen auf den Abschluss des Teilworkflows warten.
   Es wird jeweils einmal mit allen Items aufgerufen; wir übergeben genau ein Item.
9. Speichern. Zunächst über „Test manuell starten“ ausführen.

Die Dateien enthalten keine persönlichen Credential- oder Workflow-IDs.
Modell: gpt-4.1-mini wie im bereitgestellten Referenzexport.
Veröffentlichen des Webhooks ist für den manuellen Test nicht erforderlich.

## Test 1: Routineanfrage

Die vorbereiteten Daten in „Testanfrage bereitstellen“ verwenden:

```json
{
  "request_id": "SUP-TEST-003",
  "customer_id": "K001",
  "subject": "Passwort vergessen",
  "message": "Wie kann ich mein Passwort zurücksetzen?"
}
```

Erwartung bei passender KI-Klassifikation und gültigem Entwurf:
decision=auto, status=ready_for_auto_reply, source_article_ids=["KB001"],
analysis.draft_mode=approved_template. draft ist der freigegebene KB001-Text,
keine freie Modellformulierung. Es wird nichts versendet.
Falls das Modell zusätzliche Prüfsignale meldet, ist needs_review der konservative
Fallback. technical_error weist auf einen Aufruf- oder Formatfehler hin.

Dieselbe Anfrage-ID erneut ausführen: status=duplicate, processed=false.
Keine weitere KI-Verarbeitung und keine zweite Datenbankzeile.
Ein bereits vorhandener Datensatz wird nicht überschrieben, unabhängig von seinem Status.

## Test 2: Manuelle Prüfung

Im Code-Node „Testanfrage bereitstellen“ das Objekt ersetzen:

```javascript
return [{json: {
  request_id: 'SUP-TEST-005',
  customer_id: 'K002',
  subject: 'Erstattung für Rechnung',
  message: 'Meine Rechnung R-2026-100 wurde doppelt berechnet. Bitte erstatten Sie den Betrag.'
}}];
```

Erwartung bei gültiger KI-Verarbeitung: decision=review, status=needs_review.
Bei passender Klassifikation wird KB003 als Quelle verwendet und ein Entwurf
gespeichert. Keine automatische Erstattungszusage und keine Zahlung.
Dieser Fall kann grundsätzlich nicht den exakt definierten Auto-Fall erfüllen.

## Ergebnisse unabhängig prüfen

Im Compose-Verzeichnis auf dem Trainingsserver:

```bash
docker compose exec -T postgres-training psql -U n8n_training -d n8n_training <<'SQL'
SELECT request_id, customer_id, status, decision, reason_codes,
       source_article_ids, analysis->>'draft_mode' AS draft_mode
FROM support_requests
WHERE request_id IN ('SUP-TEST-003', 'SUP-TEST-005')
ORDER BY request_id;
SQL
```

Für neue Tests andere IDs verwenden. Ein abgebrochener Lauf wird bei gleichem
request_id nicht automatisch wiederaufgenommen. Vor einem gezielten neuen Versuch
den bestehenden Befund prüfen; keine pauschale Löschung aller Anfragen nötig.

## Optionaler Webhook

31 enthält zusätzlich einen POST-Webhook mit Pfad training-support.
Vor Verwendung ein Header-Auth-Credential konfigurieren. Nur die konkrete
Test-URL oder Produktions-URL aus der eigenen n8n-Oberfläche verwenden.
Der Body entspricht dem Objekt aus Test 1. Keine Arrays.

Header-Auth authentifiziert den aufrufenden Dienst, nicht den Kunden.
Eine mitgelieferte customer_id ist kein Identitätsnachweis. Produktiv muss der
Absenderdienst den Kundenbezug vertrauenswürdig bestimmen.
Es werden nur ausgewählte Body-Felder gespeichert, keine Authentifizierungsheader.
Der Webhook wartet auf das Ergebnis. HTTP 200 allein ist keine fachliche Freigabe;
status und decision im Ergebnis prüfen. Für längere Prozesse asynchron erweitern.

## Grenzen und Fehlerverhalten

- Freie KI-Entwürfe werden immer zur manuellen Prüfung gespeichert.
- ready_for_auto_reply bedeutet vorbereitet, nicht versendet; answered wird nicht gesetzt.
- Keine Kontenänderung, kein E-Mail-Versand, kein Freigabe-UI, kein automatisches Retry.
- KI-Fehler und ungültige KI-Ausgaben liefern technical_error, sofern der Fehler
  im jeweiligen KI-Aufruf bzw. dessen Ausgabeprüfung aufgefangen wird.
- Unbekannte oder inaktive Kunden sowie fehlendes Wissen führen zu needs_review.
- Ungültige Eingaben werden ohne Speicherung zurückgewiesen.
- DB-, Aufruf- und unerwartete Laufzeitfehler stoppen die Execution. Ein bereits
  angelegter Datensatz kann processing bleiben. Kein Watchdog/Wiederanlauf enthalten.
- Die Wissenssuche ist eine begrenzte Auswahl nach Kategorie und Artikel-IDs.
  Keine Embeddings, keine semantische Suche. Quellen-IDs beweisen keine inhaltliche
  Korrektheit eines generierten Entwurfs. Promptregeln sind keine Sicherheitsgrenze.
- Kundenstammdaten werden nicht an das Modell übergeben. Anfrage und ausgewählte
  Wissensartikel werden für die KI-Verarbeitung an den konfigurierten Dienst gesendet.
- Alle drei Definitionen gemeinsam versionieren. Nach dem Import die beiden
  lokalen Teilworkflow-Zuordnungen prüfen.

## Validierungsstand

Die Grundlage sind die importierten und praktisch geprüften Exporte.
Routinefall: ready_for_auto_reply / auto mit KB001 und approved_template.
Abrechnungsfall: needs_review / review mit KB003. Auch der Duplikatpfad wurde beobachtet.
Für die Download-Fassung wurden Credential-Verweise, lokale Workflow-Zuordnungen
und Instanzmetadaten entfernt; die Startanfrage ist wieder der Routinefall.
Die Teilworkflow-Zuordnungen müssen nach jedem Neuimport gesetzt werden.
Nicht als live geprüft gelten insbesondere der optionale Webhook, externe Ausfälle
und die vollständige Menge möglicher Modellantworten. Eine neue Installation
muss mit ihren eigenen Credentials geprüft werden.
