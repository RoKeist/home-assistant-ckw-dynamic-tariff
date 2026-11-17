# CKW Dynamic Tariff Dashboard Cards

These examples use a REST sensor with the entity ID `sensor.ckw_tariff_data`.
If Home Assistant creates a different entity ID, replace it in both cards.

Create a dashboard section with two columns and add two **Manual** cards to it.
Paste each complete YAML block into the corresponding card's YAML editor. On
narrow screens, Home Assistant may stack the cards vertically.

## Today's prices

```yaml
type: markdown
content: |
  {% set date_dt = now() %}
  {% set date_key = date_dt.strftime('%Y-%m-%d') %}
  {% set prices = state_attr('sensor.ckw_tariff_data', 'prices') or [] %}
  {% set ns = namespace(rows=[]) %}
  {% for p in prices %}
    {% if p.start_timestamp[:10] == date_key %}
      {% set interval = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = (p.integrated[0].value * 100) | round(2) %}
      {% set ns.rows = ns.rows + ['| ' ~ interval ~ ' | ' ~ rp|string ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW prices today · {{ date_dt.strftime('%d.%m.%Y') }}

  | Time | Price |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

## Tomorrow's prices

```yaml
type: markdown
content: |
  {% set date_dt = now() + timedelta(days=1) %}
  {% set date_key = date_dt.strftime('%Y-%m-%d') %}
  {% set prices = state_attr('sensor.ckw_tariff_data', 'prices') or [] %}
  {% set ns = namespace(rows=[]) %}
  {% for p in prices %}
    {% if p.start_timestamp[:10] == date_key %}
      {% set interval = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = (p.integrated[0].value * 100) | round(2) %}
      {% set ns.rows = ns.rows + ['| ' ~ interval ~ ' | ' ~ rp|string ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW prices tomorrow · {{ date_dt.strftime('%d.%m.%Y') }}

  | Time | Price |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

Each full day contains 96 quarter-hour values. CKW's next-day prices are
published around noon; the tomorrow card remains empty until those values are
available and the REST sensor has refreshed.
