---
title: "Fair-Value Pricing Methodik"
tags: [pricing, fair-value, competitor, sportstech, methodology]
status: active
created: 2026-09-30
---

# Fair-Value Pricing Methodik

> **Quelle:** Implementiert in `REPOS/monitoring-hub/fair-value.html` (Dashboard: https://ali-sportstech.github.io/monitoring-hub/fair-value.html)
> **Zweck:** Feature-basierter Konkurrenzvergleich → objektive Preis-Empfehlung pro Produkt.
> **Kernprinzip:** Jedes Feature hat einen EUR-Wert. Summe = Fair-Value des Produkts. Delta = VK-Preis − Fair-Value → Handlungsempfehlung.

---

## 1. Feature-Value-Modell (Weights)

Jedes Feature in der Gewichtungs-Tabelle `WEIGHTS` hat einen Typ und einen EUR-Wert:

### 1.1 Komposit-Features

| Feature | Typ | Formel | Wert |
|---------|-----|--------|------|
| **Smart Display** (Zoll) | `display_composite` | 250 € Basis + 12 € pro Zoll über 10" | Smart 21,5" → 250 + 11,5×12 = **388 €** |
| **Reines Touch-Display** | `display_composite` | 12 € pro Zoll über 10" | Kein Smart-Basis-Aufschlag |
| **Kein Display** | — | 0 € | 0 € |

### 1.2 Lineare (numerische) Features

| Feature | Typ | Wert pro Einheit | Beispiel |
|---------|-----|-----------------|----------|
| **Motorleistung (CHP)** | `numeric` | 50 € pro CHP | 3,5 CHP → **175 €** |
| **Max. Nutzergewicht** | `numeric` | 1 € pro kg | 150 kg → **150 €** |
| **Top-Speed** | `numeric` | 20 € pro km/h | 20 km/h → **400 €** |
| **Lauffläche (cm²)** | `numeric` | 0,05 € pro cm² über 1.200 cm² Basis | 1.500 cm² → 150 € |
| **Produktmasse** | `numeric` | 0,5 € pro kg | 95 kg → **47,50 €** |

> **Achtung:** Motor wird in **CHP (Dauerleistung)** bewertet, nicht in Peak-PS. Anzeige erfolgt in PS.

### 1.3 Stufenmodell (stepped)

| Feature | Stufe | Wert |
|---------|-------|------|
| **Steigung (incline %)** | 0 % | 0 € |
| | 0,1 – 5 % | 100 € |
| | 5,1 – 10 % | 150 € |
| | 10,1 – 15 % | 200 € |
| | 15,1 – 20 % | 300 € |
| | 20,1 – 999 % | 400 € |

> 12 % Steigung → **200 €** (Stufe 10,1–15 %)

### 1.4 Boolesche Features

| Feature | Wert |
|---------|------|
| **Bluetooth** | 25 € (Commodity, auf 90%+ aller Laufbänder) |
| **Klappbar** | 40 € |
| **Tablet-Halterung** | 25 € |
| **LED-Feedback (Farben)** | 60 € |
| **Induktionsladefläche** | 35 € |
| **WLAN** | 30 € |
| **Integrierte Lautsprecher** | 25 € |
| **Pulszonen-Feedback** | 30 € |
| **Premium-Material** (Hevea/Holz) | 120 € |

### 1.5 Abo-Abzug (subscription)

| Feature | Wert |
|---------|------|
| **Abo-Jahresgebühr** | **−** Jahresgebühr (Monatspreis × 12) |

> Ein Abo zählt als Minus-Punkt für die Seite, die das Abo zahlt. Kein Abo = 0 € (kein Vorteil, kein Nachteil).

---

## 2. Scoring-Formel

```
Fair-Value = Σ (Feature-Wert_i)   für alle Features im WEIGHTS-Modell
Delta      = VK-Preis − Fair-Value
```

**Beispiel (vereinfacht, sTread Pro 21,5"):**
```
Smart Display 21,5"      → 250 + 11,5 × 12 = 388 €
Motor 3,5 CHP            → 3,5 × 50 = 175 €
Max. Gewicht 150 kg      → 150 × 1 = 150 €
Top-Speed 20 km/h        → 20 × 20 = 400 €
Steigung 12 %            → 200 € (Stufe)
Lauffläche 1.500 cm²     → 0,05 × (1500−1200) = 15 €
Klappbar                 → 40 €
Bluetooth                → 25 €
WLAN                     → 30 €
Pulszonen                → 30 €
Produktmasse 95 kg       → 95 × 0,5 = 47,50 €
─────────────────────────────────────────────
Fair-Value ≈ 1.480 € (vor Abo-Abzug)
```

---

## 3. Drei Mechanismen (Dashboard-Tabs)

### 3.1 💰 Preis-Engine (Tab 2)

**Input:** Alle Produkte im Set mit VK-Preis.

**Logik:**
```
Delta = VK-Preis − Fair-Value

Delta > +50 €   → PREIS SENKEN   (zu teuer, Volumen-Hebel)
Delta < −50 €   → PREIS ERHÖHEN  (unterbewertet, Specs rechtfertigen mehr)
|Delta| ≤ 50 €  → FAIR           (keine Aktion)
```

**Output pro Produkt:**
- Fair-Value (€)
- Delta (€, farbcodiert: rot = zu hoch, grün = zu niedrig, gelb = fair)
- Top-5 treibende Features (Feature-Wert > 50 €, sortiert nach |Wert|)
- Empfehlung: senken / erhöhen / fair

### 3.2 📈 Spec-Gap → V2 Pipeline (Tab 3)

**Input:** Unsere Produkte (sportstech) vs. beste Konkurrenz je Spec.

**Geprüfte Specs:** Display (Zoll), Motor (CHP), Max. Gewicht (kg), Top-Speed (km/h), Steigung (%)

**Logik:**
```
Für jede Spec:
  ourBest  = höchster Wert unter unseren Produkten
  bestComp = höchster Wert unter Konkurrenz
  gap = bestComp − ourBest   (nur wenn bestComp > ourBest)
  gapPct = gap / ourBest × 100

  gapPct ≥ 20 %  → PRIORITÄT    (rot)
  gapPct ≥ 10 %  → sekundär     (gelb)
  gapPct < 10 %  → minor        (grau)
```

**Output:** Priorisierte Verbesserungsliste. Sammelt sich über Zeit → V2-Featureset.

### 3.3 🚀 Pre-Release Simulator (Tab 4)

**Input (manuelle Eingabe):**
- Produktname
- VK-Preis (Pflicht)
- Display (Zoll)
- Motor (CHP)
- Steigung (%)
- Max. Gewicht (kg)
- Top-Speed (km/h)
- Klappbar (ja/nein)
- Bluetooth (ja/nein)
- Produktmasse (kg)
- Abo-Jahresgebühr (€)

**Logik:** Gleiche `scoreProduct()`-Formel wie Preis-Engine.

**Output:**
```
Fair-Value (€)
Delta (€, VK-Preis − Fair-Value)
Urteil:
  Delta > +50 €  → PREIS ZU HOCH   (Empfehlung: um X € senken, Ziel: Fair-Value)
  Delta < −50 €  → PREIS ZU NIEDRIG (Empfehlung: um X € erhöhen)
  |Delta| ≤ 50 € → HALTBAR          (Preis ist im Fair-Value-Bereich)

+ Top-5 nächstgelegene Konkurrenten (nach Preis-Distanz) mit ihren Deltas
```

---

## 4. Datenquellen

| Quelle | Feld |
|--------|------|
| Shop-PDP (sportstech.de, hammer.de, fitshop.de) | Preis, Specs, Bilder |
| `PRODUCTS` JSON im Dashboard | Alle geparsten Produkte mit Specs |
| `WEIGHTS` JSON im Dashboard | Feature-Wert-Modell |

**Parseehefelder pro Produkt:**
```json
{
  "name": "string",
  "shop": "sportstech|hammer|fitshop|...",
  "price": "number|null",
  "display_inch": "number|null",
  "speed_kmh_max": "number|null",
  "incline_info": "string|null",
  "walking_surface_cm": "string|null",
  "max_user_weight_kg": "number|null",
  "motor_power_ps": "number|null",
  "motor_chp": "number|null",
  "weight_kg": "number|null",
  "bluetooth": "bool|null",
  "material": "string|null",
  "foldable": "bool|null",
  "footprint_cm": "string|null",
  "tablet_holder": "bool|null",
  "program_count": "number|null"
}
```

---

## 5. Grenzen & Annahmen

1. **Kein Marktwert-Vergleich:** Fair-Value ist ein Feature-Score, kein statistischer Marktpreis. Die 50 €-Toleranzbandbreite kompensiert das.
2. **CHP ≠ PS:** Bewertung erfolgt auf Dauer-CHP. Peak-PS wird nur angezeigt.
3. **Abo als Minus:** Peloton-ähnliche Abo-Modelle penalisieren den Preis. Ohne Abo = neutral.
4. **Keine Brand-Equity:** Premium-Branding (Peloton, NordicTrack) fließt nicht ein — das ist bewusst, da es nicht quantifizierbar ist.
5. **Lauffläche-Basis:** 1.200 cm² ist Basis (0 € Aufschlag). Nur darüber zählt.
6. **Bluetooth als Commodity:** 25 € — auf 90%+ der Laufbänder vorhanden, daher niedrig gewichtet.

---

## 6. Integration in Grok

Um die Methodik in ein anderes Tool (z.B. Grok) zu übernehmen, sind diese Bausteine nötig:

1. **WEIGHTS-Konfiguration** (JSON, oben in §1) — 1:1 übernehmbar
2. **PRODUCTS-Dataset** (JSON, oben in §4) — übernehmbar, muss bei neuen Produkten ergänzt werden
3. **scoreProduct()-Funktion** (Pseudocode):
   ```python
   def score_product(spec, weights):
       total = 0
       for key, w in weights.items():
           if w["type"] == "display_composite":
               inches = spec.get("display_inch", 0)
               val = w["base_smart"] + max(0, inches - 10) * w["per_inch_over_10"]
           elif w["type"] == "numeric":
               if key == "motor_power_ps":
                   v = spec.get("motor_chp") or spec.get("motor_power_ps") or 0
               else:
                   v = spec.get(key, 0)
               val = v * (w.get("value_per_unit") or 0)
           elif w["type"] == "stepped":
               if key == "incline_pct":
                   import re
                   m = re.search(r'(\d+[.,]?\d*)\s*%', spec.get("incline_info", ""))
                   v = float(m.group(1).replace(",", ".")) if m else 0
               else:
                   v = spec.get(key, 0)
               val = 0
               for s in w.get("steps", []):
                   if v >= s["min"] and v <= s["max"]:
                       val = s["value"]
                       break
           elif w["type"] == "bool":
               val = (w.get("value") or 0) if spec.get(key) else 0
           elif w["type"] == "subscription":
               val = -(spec.get("subscription_annual") or 0)
           total += val
       return total
   ```
4. **Delta-Logik** (oben in §3.1) — einfache if/else
5. **Spec-Gap-Tracker** (oben in §3.2) — max() pro Spec pro Shop-Gruppe

---

## 7. Änderungshistorie

| Datum | Änderung |
|-------|----------|
| 2026-09-29 | Mechanismen 1–3 als Tabs im Dashboard (PR #1, merged) |
| 2026-09-29 | F75-Titel-Fix: "sTread Pro" → "F75 Profi-Laufband" |
| 2026-09-30 | Diese Methodik-Dokumentation erstellt |
