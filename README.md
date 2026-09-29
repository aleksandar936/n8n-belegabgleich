# Belegabgleich: Bestellung ↔ Auftragsbestätigung (KI-gestützt)

Automatischer Abgleich von **Bestellungen** und **Auftragsbestätigungen (AB)** im Einkauf.
Zwei PDFs rein, Prüfprotokoll raus – inklusive Feedback-Formular, über das Prüfer
Fehler melden können.

Gebaut mit [n8n](https://n8n.io) (Low-Code-Automatisierung) und GPT-4o.

> **In English:** n8n workflow that compares purchase orders with supplier order confirmations
> (two PDFs), extracts line items with GPT-4o, flags price/quantity deviations and emails a
> colour-coded audit report. Includes a feedback form that collects reviewer corrections.

![Workflow](screenshots/workflow.png)
![Feedback-Workflow](screenshots/feedback.png)

---

## Das Problem

Wer im Einkauf Bestellungen mit den Auftragsbestätigungen der Lieferanten abgleicht,
macht das oft von Hand: zwei PDFs nebeneinanderlegen, Position für Position Preise,
Mengen, Rabatte und Summen vergleichen. Das ist zeitraubend und fehleranfällig –
und gerade bei vielen Positionen rutschen Abweichungen durch, die später Geld kosten.

## Die Lösung

Ein Workflow, der den Abgleich übernimmt und ein farbcodiertes Prüfprotokoll per
E-Mail zurückschickt. Der Prüfer sieht auf einen Blick, wo Bestellung und AB
auseinanderlaufen – und kann per Klick Feedback geben, das gesammelt wird, um die
Prüfung gezielt zu verbessern.

---

## Wie es funktioniert

**Hauptworkflow – `belegabgleich-bestellung-ab.json`**

1. **Formular** – der Nutzer lädt Bestell-PDF und AB-PDF hoch, gibt eine E-Mail und
   optional eine Preistoleranz in Prozent an.
2. **PDF-Text extrahieren** – aus beiden Dokumenten wird der Rohtext gezogen.
3. **KI-Extraktion (GPT-4o)** – ein spezialisierter Prompt zieht pro Position
   Artikelnummer, Konfiguration, Rabatt %, Einzel- und Gesamtpreis sowie Menge als
   sauberes JSON heraus. Besonderheit: deutsche Einkaufs-PDFs „verkleben" Zahlen oft
   ohne Trennzeichen (z. B. `593,0031 Stück`). Der Prompt löst das über
   Plausibilitätsregeln statt über die Textposition – dadurch layout-unabhängig.
4. **Aufbereiten** – ein Code-Node korrigiert Mengen- und Preis-Zuordnung
   rechnerisch, unabhängig davon, in welcher Reihenfolge der Beleg die Werte listet.
5. **Vergleich** – Bestellung gegen AB: Abweichungen bei Preis, Menge und Summe.
   Artikelnummern werden auf ihren Modellkern reduziert und über Konfigurations-Tokens
   (Stoff, Farbe) unterschieden, damit gleiche Modelle nicht verwechselt werden.
   Preisabweichungen innerhalb der angegebenen Toleranz werden als „wahrscheinlich ok“
   markiert, Mengen werden immer exakt verglichen. Zusätzlich prüft eine Summenkontrolle
   je Beleg, ob die Positionen die genannte Nettosumme ergeben.
6. **HTML-Report** – ein Ampel-Prüfprotokoll (✅ identisch, 🔶 innerhalb der Preistoleranz,
   ℹ️ Hinweis aus der Summenkontrolle, 🔺 Abweichung) mit vorausgefülltem Feedback-Link.
7. **Versand** – der Report geht per E-Mail an die im Formular angegebene Adresse.

**Feedback-Loop – `belegabgleich-feedback.json`**

Der Report enthält einen Feedback-Button. Darüber meldet der Prüfer, welche Positionen
tatsächlich fehlerhaft waren, plus einen Kommentar. Das landet in einer n8n-Datentabelle.

Aktueller Stand: Das Feedback wird gesammelt und dient als Grundlage, um Prompt und
Vergleichslogik gezielt nachzuschärfen. Es fließt noch **nicht automatisch** in neue
Prüfungen ein (geplanter nächster Schritt: Kommentare pro Lieferant aus der Tabelle laden
und dem Extraktions-Prompt mitgeben).

---

## Eingesetzte Technik

- **n8n** – Workflow-Orchestrierung (selbst hostbar, DSGVO-freundlich)
- **GPT-4o** – strukturierte Extraktion aus unstrukturierten PDF-Belegen
- **JavaScript (Code-Nodes)** – Normalisierung, Vergleichslogik, HTML-Erzeugung
- **n8n Data Tables** – Speicher für den Feedback-Loop
- **Gmail-Node** – Versand des Prüfprotokolls

---

## Installation

1. Beide `.json`-Dateien aus `workflow/` in n8n importieren (*Workflows → Import from File*).
2. Eigene Credentials hinterlegen (OpenAI, Gmail) – die sind aus Datenschutzgründen
   **nicht** Teil dieses Exports.
3. Eine Data Table `Feedback` anlegen mit den Spalten `beleg`, `falsche_positionen`,
   `kommentar`, `name` (alle String) und im Feedback-Workflow auswählen.
4. Den Feedback-Workflow aktivieren, die Production-URL seines Formulars kopieren und im
   Node **HTML Report** bei `FEEDBACK_URL` eintragen.

---

## Bekannte Grenzen

- Die Konfigurations-Erkennung (Stoffe, Farben) ist auf Büromöbel-Belege abgestimmt.
  Für andere Branchen muss die Token-Liste im Node **Vergleich** angepasst werden.
- Der Prompt lässt die KI verklebte Zahlen über Plausibilität und Summe zuordnen. Das macht
  die Auslese robust, kann aber im Einzelfall einen echten Rechenfehler im Beleg „glätten“.
  Die Summenkontrolle im Code fängt einen Teil davon ab – der Report bleibt eine
  Prüfhilfe, keine Freigabe.

> Hinweis: Dieser Export ist bereinigt. E-Mail-Adressen, Instanz-URL und interne IDs
> sind durch Platzhalter ersetzt.
