# CKW-Dashboard für Home Assistant

Diese Datei beschreibt das aktuelle CKW-Dashboard und enthält die YAML-Konfigurationen der dargestellten Karten. Die Beispiele verwenden die unten aufgeführten Entity-IDs. Wenn Home Assistant bei dir andere IDs vergeben hat, ersetze sie durch die tatsächlichen IDs deiner Entitäten.

## Dashboard-Aufbau

Das Dashboard ist in zwei Bereiche gegliedert:

### Strompreise

1. Aktueller CKW-Strompreis.
2. Abschnitt **Heute** mit den vier heutigen 2-Stunden-Zeitfenstern und der heutigen Preisempfehlung.
3. Abschnitt **Morgen** mit den vier morgigen 2-Stunden-Zeitfenstern und der morgigen Preisempfehlung.

### Dynamische Strompreise

1. Markdown-Tabelle mit allen 96 Viertelstundenpreisen von heute.
2. Markdown-Tabelle mit allen verfügbaren Preisen von morgen.
3. Button, der den REST-Sensor sofort erneut abfragt.

Für ein Layout wie im Screenshot platziere die Karten in einer breiten Dashboard-Ansicht: links den Bereich **Strompreise**, rechts **Dynamische Strompreise** mit den beiden Tabellen nebeneinander und dem Aktualisierungsbutton daneben. In der mobilen Ansicht können Karten untereinander angeordnet werden.

## Verwendete Entitäten

### CKW-Daten und aktueller Preis

| Anzeige | Entity-ID |
|---|---|
| CKW Tarifdaten (REST) | `sensor.ckw_tarifdaten` |
| CKW aktueller Strompreis | `sensor.ckw_aktueller_preis` |

### Zeitfenster heute

| Anzeige | Entity-ID |
|---|---|
| CKW günstigstes Zeitfenster | `sensor.ckw_guenstigstes_zeitfenster_von_2h` |
| CKW zweitgünstigstes Zeitfenster | `sensor.ckw_zweitguenstigstes_zeitfenster_von_2h` |
| CKW teuerstes Zeitfenster | `sensor.ckw_teuerstes_zeitfenster_von_2h` |
| CKW zweitteuerstes Zeitfenster | `sensor.ckw_zweitteuerstes_zeitfenster_von_2h` |
| CKW Preisempfehlung | `sensor.ckw_preisempfehlung` |

### Zeitfenster morgen

| Anzeige | Entity-ID |
|---|---|
| CKW günstigstes Zeitfenster morgen | `sensor.ckw_guenstigstes_zeitfenster_morgen_von_2h` |
| CKW zweitgünstigstes Zeitfenster morgen | `sensor.ckw_zweitguenstigstes_zeitfenster_morgen_von_2h` |
| CKW teuerstes Zeitfenster morgen | `sensor.ckw_teuerstes_zeitfenster_morgen_von_2h` |
| CKW zweitteuerstes Zeitfenster morgen | `sensor.ckw_zweitteuerstes_zeitfenster_morgen_von_2h` |
| CKW Preisempfehlung morgen | `sensor.ckw_preisempfehlung_morgen` |

## Karten für den Bereich «Strompreise»

### Aktueller Preis

```yaml
type: entity
entity: sensor.ckw_aktueller_preis
name: CKW aktueller Strompreis
```

### Vier Zeitfenster für heute

```yaml
type: entities
title: Heute
show_header_toggle: false
entities:
  - entity: sensor.ckw_guenstigstes_zeitfenster_von_2h
    name: CKW günstigstes Zeitfenster
  - entity: sensor.ckw_zweitguenstigstes_zeitfenster_von_2h
    name: CKW zweitgünstigstes Zeitfenster
  - entity: sensor.ckw_teuerstes_zeitfenster_von_2h
    name: CKW teuerstes Zeitfenster
  - entity: sensor.ckw_zweitteuerstes_zeitfenster_von_2h
    name: CKW zweitteuerstes Zeitfenster
```

### Preisempfehlung für heute

Die Karte liest die Zeitfenster direkt aus den vier Sensoren, sortiert sie chronologisch und zeigt die Hinweise farbig an.

```yaml
type: markdown
title: CKW Preisempfehlung
entity_id:
  - sensor.ckw_guenstigstes_zeitfenster_von_2h
  - sensor.ckw_zweitguenstigstes_zeitfenster_von_2h
  - sensor.ckw_teuerstes_zeitfenster_von_2h
  - sensor.ckw_zweitteuerstes_zeitfenster_von_2h
content: |
  {% set g = states('sensor.ckw_guenstigstes_zeitfenster_von_2h').split(' · ') %}
  {% set zg = states('sensor.ckw_zweitguenstigstes_zeitfenster_von_2h').split(' · ') %}
  {% set t = states('sensor.ckw_teuerstes_zeitfenster_von_2h').split(' · ') %}
  {% set zt = states('sensor.ckw_zweitteuerstes_zeitfenster_von_2h').split(' · ') %}
  {% if g | count >= 2 and zg | count >= 2 and t | count >= 2 and zt | count >= 2 %}
    {% set guenstig = [g[1], zg[1]] | sort %}
    {% set teuer = [t[1], zt[1]] | sort %}
  - **Günstig:** {{ guenstig | join(' und ') }}<br><font color="#43a047">Ideal für stromintensive Arbeiten</font>
  - **Teuer:** {{ teuer | join(' und ') }}<br><font color="#e53935">Verbrauch reduzieren</font>
  {% else %}
  Die Preisempfehlung für heute ist derzeit nicht verfügbar.
  {% endif %}
```

### Vier Zeitfenster für morgen

```yaml
type: entities
title: Morgen
show_header_toggle: false
entities:
  - entity: sensor.ckw_guenstigstes_zeitfenster_morgen_von_2h
    name: CKW günstigstes Zeitfenster morgen
  - entity: sensor.ckw_zweitguenstigstes_zeitfenster_morgen_von_2h
    name: CKW zweitgünstigstes Zeitfenster morgen
  - entity: sensor.ckw_teuerstes_zeitfenster_morgen_von_2h
    name: CKW teuerstes Zeitfenster morgen
  - entity: sensor.ckw_zweitteuerstes_zeitfenster_morgen_von_2h
    name: CKW zweitteuerstes Zeitfenster morgen
```

### Preisempfehlung für morgen

```yaml
type: markdown
title: CKW Preisempfehlung morgen
entity_id:
  - sensor.ckw_guenstigstes_zeitfenster_morgen_von_2h
  - sensor.ckw_zweitguenstigstes_zeitfenster_morgen_von_2h
  - sensor.ckw_teuerstes_zeitfenster_morgen_von_2h
  - sensor.ckw_zweitteuerstes_zeitfenster_morgen_von_2h
content: |
  {% set g = states('sensor.ckw_guenstigstes_zeitfenster_morgen_von_2h').split(' · ') %}
  {% set zg = states('sensor.ckw_zweitguenstigstes_zeitfenster_morgen_von_2h').split(' · ') %}
  {% set t = states('sensor.ckw_teuerstes_zeitfenster_morgen_von_2h').split(' · ') %}
  {% set zt = states('sensor.ckw_zweitteuerstes_zeitfenster_morgen_von_2h').split(' · ') %}
  {% if g | count >= 2 and zg | count >= 2 and t | count >= 2 and zt | count >= 2 %}
    {% set guenstig = [g[1], zg[1]] | sort %}
    {% set teuer = [t[1], zt[1]] | sort %}
  - **Günstig:** {{ guenstig | join(' und ') }}<br><font color="#43a047">Ideal für stromintensive Arbeiten</font>
  - **Teuer:** {{ teuer | join(' und ') }}<br><font color="#e53935">Verbrauch reduzieren</font>
  {% else %}
  Die Preisempfehlung für morgen ist derzeit nicht verfügbar.
  {% endif %}
```

## Karten für den Bereich «Dynamische Strompreise»

Die Tabellen lesen die `prices`-Daten direkt aus dem REST-Sensor. Preise werden mit zwei Dezimalstellen in Rp./kWh ausgegeben.

### CKW Preise heute

```yaml
type: markdown
content: |
  {% set datum = now() %}
  {% set tag = datum.strftime('%Y-%m-%d') %}
  {% set preise = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
  {% set ns = namespace(zeilen=[]) %}
  {% for p in preise %}
    {% if p.start_timestamp[:10] == tag and p.integrated is defined and p.integrated | count > 0 %}
      {% set zeit = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = '%.2f' | format((p.integrated[0].value | float(0)) * 100) %}
      {% set ns.zeilen = ns.zeilen + ['| ' ~ zeit ~ ' | ' ~ rp ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW Preise heute
  ## {{ datum.strftime('%d.%m.%Y') }}

  | Zeit | Preis |
  |:---|---:|
  {{ ns.zeilen | join('\n') }}
```

### CKW Preise morgen

```yaml
type: markdown
content: |
  {% set datum = now() + timedelta(days=1) %}
  {% set tag = datum.strftime('%Y-%m-%d') %}
  {% set preise = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
  {% set ns = namespace(zeilen=[]) %}
  {% for p in preise %}
    {% if p.start_timestamp[:10] == tag and p.integrated is defined and p.integrated | count > 0 %}
      {% set zeit = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = '%.2f' | format((p.integrated[0].value | float(0)) * 100) %}
      {% set ns.zeilen = ns.zeilen + ['| ' ~ zeit ~ ' | ' ~ rp ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW Preise morgen
  ## {{ datum.strftime('%d.%m.%Y') }}

  | Zeit | Preis |
  |:---|---:|
  {{ ns.zeilen | join('\n') }}
```

### CKW-Preise aktualisieren

Der Button löst eine sofortige Aktualisierung des REST-Sensors aus. Er ändert das normale Abrufintervall nicht.

```yaml
type: button
name: CKW-Preise aktualisieren
icon: mdi:refresh
show_state: false
tap_action:
  action: perform-action
  perform_action: homeassistant.update_entity
  data:
    entity_id: sensor.ckw_tarifdaten
```

## Karten im Dashboard anordnen

1. Dashboard bearbeiten und zwei breite Bereiche oder Spalten anlegen: **Strompreise** und **Dynamische Strompreise**.
2. Im Bereich **Strompreise** die Karten in dieser Reihenfolge hinzufügen: aktueller Preis, heutige Zeitfenster, heutige Empfehlung, morgige Zeitfenster, morgige Empfehlung.
3. Im Bereich **Dynamische Strompreise** die beiden Markdown-Tabellen nebeneinander platzieren und den Aktualisierungsbutton rechts daneben setzen.
4. Für jede Karte den vollständigen YAML-Block in den YAML-Editor der Karte einfügen.

Home Assistant kann bei kleinen Displays oder schmalen Abschnitten Karten untereinander stapeln. Die Reihenfolge bleibt dabei erhalten.

## Hinweise

- Die Zeitfenster-Sensoren müssen zuerst als Template-Sensor-Helfer eingerichtet sein. Die Entity-IDs in diesem Dokument entsprechen dem hier verwendeten Beispiel.
- CKW muss die Preise für morgen bereits veröffentlicht haben. Bis dahin können morgige Tabelle und Morgen-Sensoren leer oder nicht verfügbar sein.
- Die beiden Preisempfehlungskarten greifen auf die vier jeweiligen Zeitfenster-Sensoren zu. Wenn einer davon `unknown` oder `unavailable` ist, zeigt die Karte einen Verfügbarkeitshinweis.
- Der Button aktualisiert nur den REST-Sensor `sensor.ckw_tarifdaten`; die Preise werden nur sichtbar, wenn die API sie zurückliefert.
