# Dashboard-Karten für den dynamischen CKW-Tarif

Diese Beispiele verwenden den REST-Sensor mit der Entitäts-ID `sensor.ckw_tarifdaten`.
Falls Home Assistant eine andere Entitäts-ID erstellt, ersetze sie in beiden Karten.

Erstelle einen Dashboard-Abschnitt mit zwei Spalten und füge zwei **Manuelle**
Karten hinzu. Füge jeden vollständigen YAML-Block in den YAML-Editor der
entsprechenden Karte ein. Auf schmalen Bildschirmen kann Home Assistant die
Karten untereinander anzeigen.

## Preise von heute

```yaml
type: markdown
content: |
  {% set date_dt = now() %}
  {% set date_key = date_dt.strftime('%Y-%m-%d') %}
  {% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
  {% set ns = namespace(rows=[]) %}
  {% for p in prices %}
    {% if p.start_timestamp[:10] == date_key %}
      {% set interval = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = (p.integrated[0].value * 100) | round(2) %}
      {% set ns.rows = ns.rows + ['| ' ~ interval ~ ' | ' ~ rp|string ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW-Preise heute · {{ date_dt.strftime('%d.%m.%Y') }}

  | Zeit | Preis |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

## Preise von morgen

```yaml
type: markdown
content: |
  {% set date_dt = now() + timedelta(days=1) %}
  {% set date_key = date_dt.strftime('%Y-%m-%d') %}
  {% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
  {% set ns = namespace(rows=[]) %}
  {% for p in prices %}
    {% if p.start_timestamp[:10] == date_key %}
      {% set interval = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = (p.integrated[0].value * 100) | round(2) %}
      {% set ns.rows = ns.rows + ['| ' ~ interval ~ ' | ' ~ rp|string ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW-Preise morgen · {{ date_dt.strftime('%d.%m.%Y') }}

  | Zeit | Preis |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

Ein vollständiger Tag umfasst 96 Viertelstundenwerte. CKW veröffentlicht die
Preise für morgen ungefähr zur Mittagszeit. Die morgige Karte bleibt leer, bis
diese Daten verfügbar sind und der REST-Sensor aktualisiert wurde.
