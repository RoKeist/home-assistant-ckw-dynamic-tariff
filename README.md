# CKW Dynamic Tariff for Home Assistant

Community examples for reading CKW's public dynamic tariff API in Home Assistant, displaying today's and tomorrow's prices, and using the current price in automations.

This is an independent community project. It is not affiliated with, endorsed by, or supported by CKW or the Home Assistant project.

## What this provides

- A Home Assistant REST sensor that fetches CKW tariff data.
- Two Markdown dashboard cards: today's prices and tomorrow's prices.
- A template sensor for the price of the current 15-minute interval.
- An automation example that triggers when the price falls below a threshold.

CKW publishes prices as 15-minute values. A complete day contains 96 values. CKW's tariff profile for the following day is available from around 12:00 noon. The API's `integrated` value is in CHF/kWh and is not necessarily the complete all-in amount on a bill: taxes, meter charges, concession fees, and demand charges may be excluded.

## Requirements

- A Home Assistant installation with the built-in [RESTful integration](https://www.home-assistant.io/integrations/rest/).
- A CKW tariff and tariff name that match your account. `home_dynamic` is for annual consumption below 50 MWh; `business_dynamic` is for consumption above 50 MWh.
- A file editor or another way to edit `configuration.yaml`.

The examples below use `sensor.ckw_tarifdaten` as the REST sensor entity ID. Home Assistant should create the entity `sensor.ckw_tarifdaten` from the name above. If it creates a different ID in your installation, replace it in the templates and cards below.

## 1. Fetch CKW prices

Add this under the top-level `rest:` key in `configuration.yaml`. If you already have a `rest:` section, add the list item under it instead of creating a duplicate top-level key.

```yaml
rest:
  - resource: https://e-ckw-public-data.de-c1.eu1.cloudhub.io/api/v1/netzinformationen/energie/dynamische-preise
    scan_interval: 21600
    params:
      tariff_name: home_dynamic
      tariff_type: integrated
      start_timestamp: "{{ today_at('00:00').isoformat() }}"
      end_timestamp: "{{ (today_at('00:00') + timedelta(days=2)).isoformat() }}"
    sensor:
      - name: "CKW Tarifdaten"
        unique_id: ckw_tarifdaten
        value_template: "OK"
        json_attributes:
          - publication_timestamp
          - prices
```

The request covers today and tomorrow, from local midnight today to local midnight the day after tomorrow. The timestamps are generated using Home Assistant's configured timezone. The example polls every six hours; adjust `scan_interval` if needed.

After editing the file, validate the Home Assistant configuration and restart or reload the relevant configuration. Check the sensor under **Developer tools → States**. Its state is `OK`; the actual tariff data is in the `publication_timestamp` and `prices` attributes.

## 2. Dashboard cards for today and tomorrow

Create a dashboard section with two columns (or a section wide enough for two cards). Add two **Manual** cards to that section, one in each column. Paste each complete YAML block into its card's YAML editor.

On narrower screens Home Assistant may stack the cards vertically.

### Today's prices

```yaml
type: markdown
content: |
  {% set tag_dt = now() %}
  {% set tag = tag_dt.strftime('%Y-%m-%d') %}
  {% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
  {% set ns = namespace(rows=[]) %}
  {% for p in prices %}
    {% if p.start_timestamp[:10] == tag %}
      {% set interval = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = (p.integrated[0].value * 100) | round(2) %}
      {% set ns.rows = ns.rows + ['| ' ~ interval ~ ' | ' ~ rp|string ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW prices today · {{ tag_dt.strftime('%d.%m.%Y') }}

  | Time | Price |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

### Tomorrow's prices

```yaml
type: markdown
content: |
  {% set tag_dt = now() + timedelta(days=1) %}
  {% set tag = tag_dt.strftime('%Y-%m-%d') %}
  {% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
  {% set ns = namespace(rows=[]) %}
  {% for p in prices %}
    {% if p.start_timestamp[:10] == tag %}
      {% set interval = p.start_timestamp[11:16] ~ '–' ~ p.end_timestamp[11:16] %}
      {% set rp = (p.integrated[0].value * 100) | round(2) %}
      {% set ns.rows = ns.rows + ['| ' ~ interval ~ ' | ' ~ rp|string ~ ' Rp./kWh |'] %}
    {% endif %}
  {% endfor %}
  # CKW prices tomorrow · {{ tag_dt.strftime('%d.%m.%Y') }}

  | Time | Price |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

The cards convert the API value from CHF/kWh to Rp./kWh by multiplying by 100. The tomorrow card will be empty until CKW publishes the next day's profile and Home Assistant fetches it.

## 3. Current price template sensor

Create a template sensor helper from **Settings → Devices & services → Helpers → Create helper → Template → Sensor**. Use `CKW Current Price` as the name and enter this in the state template field:

```jinja
{% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set now_ts = as_timestamp(now()) %}
{% set current = namespace(value=none) %}
{% for p in prices %}
  {% set start = as_timestamp(p.start_timestamp, 0) %}
  {% set end = as_timestamp(p.end_timestamp, 0) %}
  {% if start <= now_ts < end and p.integrated is defined and p.integrated | count > 0 %}
    {% set current.value = p.integrated[0].value %}
  {% endif %}
{% endfor %}
{{ current.value }}
```

Set the unit to `CHF/kWh`. Leave the device class empty. If the helper offers a state class, choose `measurement`. Save the helper and check the entity ID Home Assistant created. The examples below use `sensor.ckw_current_price`; replace it with the actual ID if different.

## 4. Automation example

This example triggers when the current price crosses below 0.20 CHF/kWh (20 Rp./kWh). Add your own action under `action:`.

```yaml
alias: CKW price below 20 Rp per kWh
triggers:
  - trigger: numeric_state
    entity_id: sensor.ckw_current_price
    below: 0.20
actions:
  # Add the action you want to run here.
  - action: persistent_notification.create
    data:
      title: CKW price is low
      message: The current CKW price is below 20 Rp./kWh.
mode: single
```

A numeric-state trigger fires when the entity crosses the threshold. If you need an action once in every qualifying 15-minute interval, use a periodic time trigger and a separate price condition instead.

## Troubleshooting

- **The REST sensor state says `OK`:** This is expected. The prices are sensor attributes, not its state.
- **No entity appears in the States page:** Use the entity filter in Home Assistant, not the browser's find-in-page feature. Search by the sensor's friendly name or `ckw`.
- **Tomorrow's card is empty:** Check the last `start_timestamp` in the `prices` attribute and the `publication_timestamp`. CKW publishes the next day's prices around noon.
- **Only today's prices appear:** Confirm the request includes both `start_timestamp` and `end_timestamp` as shown above, then refresh the sensor after the next-day profile is published.
- **The Markdown table renders as plain text:** Paste the complete block, including `type: markdown` and `content: |`, into the card's YAML editor.
- **Cards are stacked:** Put both cards in the same two-column section and check the dashboard at a wider screen size. Home Assistant may use one column on narrow displays.
- **The automation does not trigger:** Check the current-price sensor's actual entity ID and confirm its state is numeric, for example `0.1583`.

## References

- [CKW dynamic tariff technical information and API](https://www.ckw.ch/energie/strom/dynamischer-tarif-fuer-privatkunden-und-unternehmen/dynamischer-tarif-technische-voraussetzungen)
- [Home Assistant RESTful integration](https://www.home-assistant.io/integrations/rest/)
- [Home Assistant RESTful Sensor](https://www.home-assistant.io/integrations/sensor.rest/)

## Contributing

Issues and pull requests with corrections, tested improvements, or compatibility notes are welcome. Please remove personal information, tokens, IP addresses, and private Home Assistant entity names before sharing configuration or logs.
