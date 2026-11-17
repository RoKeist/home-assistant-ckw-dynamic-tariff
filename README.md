# CKW dynamischer Tarif für Home Assistant

Community-Beispiele, um die öffentliche CKW-API für dynamische Tarife in Home Assistant abzufragen, die Preise für heute und morgen anzuzeigen und den aktuellen Preis in Automationen zu verwenden.

Dieses unabhängige Community-Projekt steht in keiner Verbindung zu CKW oder zum Home-Assistant-Projekt und wird von diesen weder unterstützt noch offiziell bestätigt.

## Inhalt

- Ein REST-Sensor für Home Assistant, der CKW-Tarifdaten abruft.
- Zwei Markdown-Karten fürs Dashboard: Preise von heute und morgen.
- Ein Template-Sensor für den Preis des aktuell gültigen 15-Minuten-Intervalls.
- Ein Automationsbeispiel, das bei Unterschreiten eines Preisgrenzwerts ausgelöst wird.

CKW veröffentlicht Viertelstundenwerte. Ein vollständiger Tag umfasst 96 Werte. Das Tarifprofil für den Folgetag ist ungefähr ab 12 Uhr mittags verfügbar. Der API-Wert `integrated` wird in CHF/kWh angegeben und entspricht nicht zwingend dem vollständigen Rechnungsbetrag: Steuern, Messentgelte, Konzessionsabgaben und Leistungspreise können darin fehlen.

## Voraussetzungen

- Eine Home-Assistant-Installation mit der integrierten [RESTful-Integration](https://www.home-assistant.io/integrations/rest/).
- Ein CKW-Tarif und ein Tarifname, die zu deinem Vertrag passen. `home_dynamic` gilt bei einem Jahresverbrauch unter 50 MWh; `business_dynamic` bei einem Jahresverbrauch über 50 MWh.
- Ein Datei-Editor oder eine andere Möglichkeit, `configuration.yaml` zu bearbeiten.

Die Beispiele verwenden `sensor.ckw_tarifdaten` als Entitäts-ID des REST-Sensors. Die Konfiguration unten verwendet den Namen „CKW Tarifdaten“. Falls Home Assistant bei dir eine andere ID erstellt, ersetze sie in den Vorlagen und Karten.

## 1. CKW-Preise abrufen

Füge diesen Abschnitt unter dem obersten Schlüssel `rest:` in `configuration.yaml` ein. Wenn bereits ein Abschnitt `rest:` existiert, ergänze dort den Listeneintrag, anstatt einen zweiten obersten `rest:`-Schlüssel anzulegen.

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

Die Abfrage umfasst heute und morgen: von heute um Mitternacht bis Mitternacht übermorgen. Die Zeitstempel verwenden die in Home Assistant eingestellte Zeitzone. Das Beispiel fragt die API alle sechs Stunden ab; passe `scan_interval` bei Bedarf an.

Prüfe nach dem Bearbeiten die Home-Assistant-Konfiguration und starte Home Assistant neu oder lade die betreffende Konfiguration neu. Kontrolliere den Sensor unter **Entwicklerwerkzeuge → Zustände**. Sein Zustand ist `OK`; die Tarifdaten stehen in den Attributen `publication_timestamp` und `prices`.

## 2. Dashboard-Karten für heute und morgen

Erstelle einen Dashboard-Abschnitt mit zwei Spalten (oder einen ausreichend breiten Abschnitt). Füge in jede Spalte eine **Manuelle** Karte ein. Füge den vollständigen YAML-Block jeweils im YAML-Editor der Karte ein.

Auf schmalen Bildschirmen kann Home Assistant die Karten untereinander anordnen.

### Preise von heute

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
  # CKW-Preise heute · {{ tag_dt.strftime('%d.%m.%Y') }}

  | Zeit | Preis |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

### Preise von morgen

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
  # CKW-Preise morgen · {{ tag_dt.strftime('%d.%m.%Y') }}

  | Zeit | Preis |
  |:---|---:|
  {{ ns.rows | join('\n  ') }}
```

Die Karten rechnen den API-Wert von CHF/kWh in Rp./kWh um, indem sie ihn mit 100 multiplizieren. Die Karte für morgen bleibt leer, bis CKW das Profil für den Folgetag veröffentlicht hat und Home Assistant die Daten abruft.

## 3. Template-Sensor für den aktuellen Preis

Erstelle einen Template-Sensor-Helfer unter **Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Template → Sensor**. Verwende als Namen `CKW aktueller Preis` und füge dieses Template ins Zustandsfeld ein:

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

Setze die Einheit auf `CHF/kWh`. Lasse die Geräteklasse leer. Falls der Helfer eine Zustandsklasse anbietet, wähle `measurement` (Messwert). Speichere den Helfer und prüfe die von Home Assistant erstellte Entitäts-ID. Die folgenden Beispiele verwenden `sensor.ckw_aktueller_preis`; ersetze sie durch die tatsächlich erstellte ID, falls sie abweicht.

## 4. Automationsbeispiel

Dieses Beispiel wird ausgelöst, wenn der aktuelle Preis die Schwelle von 0,20 CHF/kWh (20 Rp./kWh) nach unten überschreitet. Ergänze unter `actions:` die gewünschte Aktion.

```yaml
alias: CKW-Preis unter 20 Rp. pro kWh
triggers:
  - trigger: numeric_state
    entity_id: sensor.ckw_aktueller_preis
    below: 0.20
actions:
  # Ergänze hier die gewünschte Aktion.
  - action: persistent_notification.create
    data:
      title: CKW-Preis ist tief
      message: Der aktuelle CKW-Preis liegt unter 20 Rp./kWh.
mode: single
```

Ein numerischer Zustandsauslöser reagiert, wenn der Sensor die Schwelle überschreitet. Soll eine Aktion in jedem passenden 15-Minuten-Intervall ausgeführt werden, verwende stattdessen einen periodischen Zeitauslöser mit einer separaten Preisbedingung.

## Fehlerbehebung

- **Der REST-Sensor zeigt `OK`:** Das ist erwartungsgemäss. Die Preise sind Attribute des Sensors, nicht sein Zustand.
- **Die Entität erscheint nicht auf der Zuständeseite:** Verwende in Home Assistant den Entitätsfilter und nicht die Seitensuche des Browsers. Suche nach dem Anzeigenamen des Sensors oder nach `ckw`.
- **Die Karte für morgen ist leer:** Prüfe den letzten `start_timestamp` im Attribut `prices` sowie `publication_timestamp`. CKW veröffentlicht die Preise für den Folgetag ungefähr um die Mittagszeit.
- **Es erscheinen nur die heutigen Preise:** Kontrolliere, ob die Abfrage wie oben sowohl `start_timestamp` als auch `end_timestamp` enthält. Aktualisiere den Sensor erneut, nachdem CKW das nächste Tagesprofil veröffentlicht hat.
- **Die Markdown-Tabelle erscheint als normaler Text:** Füge den vollständigen Block einschliesslich `type: markdown` und `content: |` im YAML-Editor der Karte ein.
- **Die Karten stehen untereinander:** Ordne beide Karten demselben zweispaltigen Abschnitt zu und prüfe das Dashboard auf einem breiteren Bildschirm. Auf schmalen Displays kann Home Assistant auf eine Spalte wechseln.
- **Die Automation wird nicht ausgelöst:** Prüfe die tatsächliche Entitäts-ID des aktuellen Preis-Sensors und ob sein Zustand numerisch ist, zum Beispiel `0.1583`.

## Quellen

- [CKW: Technische Informationen zum dynamischen Tarif und zur API](https://www.ckw.ch/energie/strom/dynamischer-tarif-fuer-privatkunden-und-unternehmen/dynamischer-tarif-technische-voraussetzungen)
- [Home Assistant: RESTful-Integration](https://www.home-assistant.io/integrations/rest/)
- [Home Assistant: RESTful Sensor](https://www.home-assistant.io/integrations/sensor.rest/)

## Mitwirken

Fehlermeldungen und Pull Requests mit Korrekturen, getesteten Verbesserungen oder Hinweisen zur Kompatibilität sind willkommen. Entferne persönliche Angaben, Tokens, IP-Adressen und private Home-Assistant-Entitätsnamen, bevor du Konfigurationen oder Protokolle teilst.
