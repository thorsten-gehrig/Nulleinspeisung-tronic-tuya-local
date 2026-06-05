# Home Assistant – Nulleinspeisung für Balkonkraftwerk / Solarspeicher

Diese Home-Assistant-Automation regelt die **Entladeleistung** eines Solarspeichers bzw. Balkonkraftwerks so, dass der **Netzbezug minimiert** und eine **Nulleinspeisung** bestmöglich eingehalten wird (konkret: Netzbezug 0-100W).

Enwickelt für ein Lidl / Tronic Speicher (B2500) - aber auch für andere nutzbar.

Die Logik arbeitet nur Tagsüber (nach Sonnenaufgang / vor Sonnenuntergang) oder wenn im SOC >5% vorhanden sind - so das bei leerer Batterie nachts keine unnötige Steuerung durchgeführt wird.

Die Logik basiert auf einem gemessenen Leistungswert (`powermeter`) und arbeitet mit:

- **Hysterese** zur Vermeidung unnötiger Umschaltungen
- **Step-Up / Step-Down** in mehreren Schritten auf einmal
- automatischer Umschaltung des **Betriebsmodus**
- einem **Mindestwert von 80 W** statt `0`, falls der Wechselrichter / Speicher `0` nicht akzeptiert

---

## Funktionen

- **Step-Down**, wenn der Bedarf unter die untere Schwelle fällt
- **Step-Up**, wenn der Bedarf über die obere Schwelle steigt
- **mehrere Schritte auf einmal**, wenn die Abweichung ein Vielfaches des Schrittwerts ist
- **Hysterese-Bereich** zwischen `80 W` und `120 W`
- Umschaltung des Betriebsmodus:
  - bei Minimalleistung → **„Laden zuerst“**
  - bei Leistung über `100 W` → **„Laden und Entladen“**
- schreibt bei Minimalbetrieb **80 W statt 0 W**, damit auch Geräte mit Mindestwert unterstützt werden

---

## Voraussetzungen

Als Vorrausetzung für den Tronic / Lidl speicher habe ich Tuya-Local mit der Anpassung von https://github.com/Maztah/tronic-speicher-tuya-local verwendet.
Der Netzbezug muss gemessen werden (Shelly oder Hichi o.ä.). Dieser wird mittels Helfer => statistik / Durchschnitt (Stichprobe 100, maximal-alter 5min) normalisiert als `sensor. lesekopf_2_min_mittelwert_stromzahler` verwendet.

Zusätzlich müssen folgende Entitäten in Home Assistant vorhanden sein und ggf. an deine Umgebung angepasst werden:

### Sensoren

- `sensor.lesekopf_2_min_mittelwert_stromzahler`
  - gemittelter Leistungswert vom Stromzähler
  - **positiv = Netzbezug**
  - **negativ = Einspeisung / Überschuss**
- `sensor.solarkraftwerk_batteriestand`
  - Batteriestand in Prozent

### Steuerbare Entitäten

- `number.solarkraftwerk_entladeleistung_slot_1`
  - Entladeleistung des Speichers / Wechselrichters
- `select.solarkraftwerk_betriebsmodus`
  - Betriebsmodus des Geräts
  - erwartete Optionen:
    - `Laden zuerst`
    - `Laden und Entladen`

---

## Logik im Überblick

### 1. Step-Down
Wenn `powermeter < 80`, wird die Entladeleistung reduziert.

- Die Anzahl der Schritte richtet sich nach der Abweichung zur unteren Schwelle.
- Beispiel bei `step_down: 50`:
  - `79 W` → 1 Schritt
  - `120 W` → 2 Schritte
  - `170 W` → 3 Schritte

Wenn das Ergebnis auf `<= 100 W` fallen würde:

- wird stattdessen **80 W** gesetzt
- und der Betriebsmodus auf **`Laden zuerst`** umgestellt
Somit wird die Entladeleistung auf 0 gesetzt.

### 2. Step-Up
Wenn `powermeter > 120`, wird die Entladeleistung erhöht.

- Die Anzahl der Schritte richtet sich nach der Abweichung zur oberen Schwelle.
- Beispiel bei `step_up: 100`:
  - `121 W` → 1 Schritt
  - `180 W` → 1 Schritt
  - `221 W` → 2 Schritte
  - `320 W` → 3 Schritte

Wenn das neue Ergebnis über `100 W` liegt:

- wird der Betriebsmodus auf **`Laden und Entladen`** gestellt

### 3. Hysterese
Im Bereich zwischen `80 W` und `120 W` passiert **keine Änderung**.

Dadurch werden unnötige Schaltvorgänge vermieden.

---

## Konfigurierbare Werte

Diese Variablen können direkt in der Automation angepasst werden:

```yaml
base: 10
min_preset: 80
max_preset: 800
step_up: 100
step_down: 50
lower_threshold: 80
upper_threshold: 120
```

### Bedeutung

- `base` → Rundung auf 10-W-Schritte
- `min_preset` → Minimalwert, der statt `0` geschrieben wird
- `max_preset` → Maximal zulässige Entladeleistung
- `step_up` → Erhöhung pro Schritt
- `step_down` → Reduzierung pro Schritt
- `lower_threshold` → Untere Hysterese-Schwelle
- `upper_threshold` → Obere Hysterese-Schwelle

---

## Beispiel-Automation

> **Hinweis:** Bitte die Entity-IDs an deine Installation anpassen.

```yaml
alias: Nulleinspeisung Stephen (Pro)
description: >
  powermeter = zusätzlicher Bedarf (Netzbezug)
  <80W -> Step-Down
  Wenn Step-Down <=100 ergeben würde:
    - Modus auf "Laden zuerst"
    - Wert 80 schreiben
  >120W -> Step-Up
  80-120W -> halten (Hysterese)
  Bei result >100 -> Modus "Laden und Entladen"

triggers:
  - trigger: time_pattern
    minutes: "/2"

conditions:
  - condition: or
    conditions:
      - condition: numeric_state
        entity_id: sensor.solarkraftwerk_batteriestand
        above: 5
      - condition: sun
        after: sunrise
        before: sunset

actions:
  - variables:
      base: 10
      min_preset: 80
      max_preset: 800
      step_up: 100
      step_down: 50
      lower_threshold: 80
      upper_threshold: 120

      powermeter: >
        {{ states('sensor.lesekopf_2_min_mittelwert_stromzahler_stephen') | int(0) }}

      actual_preset: >
        {{ states('number.solarkraftwerk_entladeleistung_slot_1') | int(0) }}

      actual_mode: >
        {{ states('select.solarkraftwerk_betriebsmodus') }}

      new_preset: >
        {% set result = actual_preset %}

        {% if powermeter < lower_threshold %}
          {% set delta_down = lower_threshold - powermeter %}
          {% set steps_down = ((delta_down - 1) // step_down) + 1 %}

          {% if actual_preset > 100 %}
            {% set raw = actual_preset - (steps_down * step_down) %}
            {% if raw <= 100 %}
              {% set result = min_preset %}
            {% else %}
              {% set result = (raw // base) * base %}
            {% endif %}
          {% else %}
            {% set result = min_preset %}
          {% endif %}

        {% elif powermeter > upper_threshold %}
          {% set delta_up = powermeter - upper_threshold %}
          {% set steps_up = ((delta_up - 1) // step_up) + 1 %}
          {% set raw = actual_preset + (steps_up * step_up) %}

          {% if raw > max_preset %}
            {% set result = max_preset %}
          {% else %}
            {% set result = (raw // base) * base %}
          {% endif %}
        {% endif %}

        {{ result }}

  - variables:
      target_mode: >
        {% if powermeter < lower_threshold and (new_preset | int) == min_preset %}
          Laden zuerst
        {% elif (new_preset | int) > 100 %}
          Laden und Entladen
        {% else %}

        {% endif %}

  - if:
      - condition: template
        value_template: >
          {{ target_mode | trim != '' and target_mode | trim != actual_mode }}
    then:
      - action: select.select_option
        target:
          entity_id: select.solarkraftwerk_betriebsmodus
        data:
          option: "{{ target_mode | trim }}"

  - if:
      - condition: template
        value_template: >
          {{ (new_preset | int) != actual_preset }}
    then:
      - action: number.set_value
        target:
          entity_id: number.solarkraftwerk_entladeleistung_slot_1
        data:
          value: "{{ new_preset | int }}"

mode: single
```

---

## Installation

1. In Home Assistant eine neue Automation anlegen.
2. Den YAML-Code einfügen.
3. Die **Entity-IDs** anpassen.
4. Prüfen, ob die Select-Optionen exakt so heißen:
   - `Laden zuerst`
   - `Laden und Entladen`
5. Automation speichern und testen.

---

## Testempfehlungen

Folgende Fälle gezielt prüfen:

- **starker Überschuss / negative Werte** → Preset muss sinken
- **leichter Netzbezug** → Preset halten
- **deutlicher Netzbezug** → Preset erhöhen
- Wechsel von Minimalbetrieb auf normalen Betrieb → Moduswechsel prüfen

Praktische Testwerte für `powermeter`:

- `-100`
- `50`
- `100`
- `150`
- `250`

---

## Hinweise

- Falls dein Powermeter-Vorzeichen anders ist, muss die Logik angepasst werden.
- Falls du im Dateimodus `automations.yaml` statt im UI-Editor arbeitest, kann sich das notwendige YAML-Schema unterscheiden.
- Diese Automation ist als Beispiel gedacht und sollte vor produktivem Einsatz ausgiebig getestet werden.

---

## Lizenz

- `GPL-3.0`

---

## Haftungsausschluss

Die Nutzung erfolgt auf eigene Verantwortung. Bitte teste die Automation zunächst in einer sicheren Umgebung und überprüfe insbesondere:

- die Vorzeichenlogik des Powermeters
- die zulässigen Werte deines Wechselrichters / Speichers
- die exakten Namen deiner Select-Optionen
