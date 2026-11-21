# CKW dynamischer Tarif in Home Assistant

Diese Anleitung zeigt, wie du CKW-Preise über die öffentliche API in Home Assistant abrufst, im Dashboard darstellst und die günstigsten sowie teuersten 2-Stunden-Zeitfenster ermittelst. Sie enthält Beispiele für heute und morgen, Preisempfehlungen, manuelles Aktualisieren und E-Mail-Benachrichtigungen.

Dieses Community-Beispiel ist unabhängig und steht in keiner Verbindung zu CKW oder dem Home-Assistant-Projekt. Es wird von diesen nicht offiziell unterstützt.

## Dashboard-Vorschau

Das folgende Bild zeigt das vollständige Dashboard mit Beispielwerten vom 21. und 22. November 2026. Die Preise, Zeitfenster und Datumsangaben ändern sich laufend und dienen nur zur Illustration.

![Vorschau des CKW-Home-Assistant-Dashboards](assets/dashboard-preview.png)

Links im Dashboard stehen der aktuelle Strompreis, die vier günstigsten und teuersten 2-Stunden-Fenster für heute und morgen sowie die farbigen Preisempfehlungen. Darunter zeigen Verlaufskarten den Preisverlauf. Rechts stehen die Viertelstundenpreise für heute und morgen in separaten Tabellen. Der Aktualisierungsbutton fragt die CKW-Daten bei Bedarf sofort erneut ab.

## Inhalt

- [Voraussetzungen und Datenformat](#voraussetzungen-und-datenformat)
- [1. CKW-REST-Sensor](#1-ckw-rest-sensor)
- [2. Aktuellen Strompreis als Helfer erstellen](#2-aktuellen-strompreis-als-helfer-erstellen)
- [3. Acht Zeitfenster-Sensoren erstellen](#3-acht-zeitfenster-sensoren-erstellen)
- [4. Preisempfehlung für heute und morgen](#4-preisempfehlung-für-heute-und-morgen)
- [5. Dashboard-Karten](#5-dashboard-karten)
- [6. Preise manuell aktualisieren](#6-preise-manuell-aktualisieren)
- [7. E-Mail-Benachrichtigungen](#7-e-mail-benachrichtigungen)
- [Fehlerbehebung](#fehlerbehebung)

## Voraussetzungen und Datenformat

- Home Assistant mit aktivierter RESTful-Integration.
- Eine CKW-Tarifart und ein passender Tarifname. In diesem Beispiel ist `home_dynamic` eingestellt. Prüfe bei CKW, welcher Tarifname zu deinem Vertrag passt.
- Die Entitäts-ID des REST-Sensors ist in den Beispielen `sensor.ckw_tarifdaten`. Falls deine Installation eine andere ID erzeugt, ersetze sie in den Vorlagen.

CKW-Preise kommen in 15-Minuten-Intervallen. Ein vollständiger Tag hat 96 Werte. Für ein 2-Stunden-Fenster werden acht aufeinanderfolgende Werte gemittelt. Die API-Angabe `integrated` liegt in CHF/kWh; für Rp./kWh wird sie mit 100 multipliziert. Die API-Werte müssen nicht sämtliche Bestandteile der späteren Stromrechnung enthalten.

## 1. CKW-REST-Sensor

Füge den Eintrag unter dem obersten Schlüssel `rest:` in `configuration.yaml` ein. Wenn du bereits einen `rest:`-Abschnitt hast, ergänze den Listeneintrag dort; lege keinen zweiten obersten Schlüssel `rest:` an.

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

`scan_interval: 21600` fragt alle sechs Stunden ab. Die Abfrage umfasst heute und morgen, gerechnet in der Home-Assistant-Zeitzone. Der Sensorzustand ist `OK`; die Tarifdaten stehen in den Attributen `publication_timestamp` und `prices`.

Nach der Änderung Konfiguration prüfen und Home Assistant neu starten oder die REST-Konfiguration neu laden. Kontrolliere unter **Entwicklerwerkzeuge → Zustände**, dass `sensor.ckw_tarifdaten` vorhanden ist und das Attribut `prices` Daten enthält. CKW stellt das Profil für den Folgetag üblicherweise ab etwa Mittag bereit; die Verfügbarkeit kann variieren.

## 2. Aktuellen Strompreis als Helfer erstellen

Erstelle unter **Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Template → Sensor** einen Sensor namens `CKW aktueller Preis`. Füge Folgendes in das Feld **Zustand** ein:

```jinja
{% set prices = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set now_ts = as_timestamp(now()) %}
{% set current = namespace(value=none) %}
{% for p in prices %}
  {% set start = as_timestamp(p.start_timestamp, 0) %}
  {% set end = as_timestamp(p.end_timestamp, 0) %}
  {% if start <= now_ts < end and p.integrated is defined and p.integrated | count > 0 %}
    {% set current.value = p.integrated[0].value | float(0) %}
  {% endif %}
{% endfor %}
{{ current.value if current.value is not none else 'unavailable' }}
```

Setze als Einheit `CHF/kWh`, Gerätekategorie leer und – sofern angeboten – Zustandsklasse `measurement`. In den Beispielen wird die Entität `sensor.ckw_aktueller_preis` genannt; falls Home Assistant eine andere ID vergibt, passe sie an.

## 3. Acht Zeitfenster-Sensoren erstellen

Erstelle acht Template-Sensor-Helfer. Für jeden Helfer verwendest du denselben Zustandscode weiter unten, aber passt die drei Einstellungen am Anfang entsprechend der Tabelle an. Der Code filtert erst auf den Zieldatumsbereich und sortiert danach. Die Sensoren für den zweiten Platz schliessen Zeitfenster aus, die das jeweils beste Zeitfenster überlappen.

| Helfername | `tage_voraus` | `teuerstes` | `zweites` |
|---|---:|---|---|
| CKW günstigstes Zeitfenster von 2h | `0` | `false` | `false` |
| CKW zweitgünstigstes Zeitfenster von 2h | `0` | `false` | `true` |
| CKW teuerstes Zeitfenster von 2h | `0` | `true` | `false` |
| CKW zweitteuerstes Zeitfenster von 2h | `0` | `true` | `true` |
| CKW günstigstes Zeitfenster morgen von 2h | `1` | `false` | `false` |
| CKW zweitgünstigstes Zeitfenster morgen von 2h | `1` | `false` | `true` |
| CKW teuerstes Zeitfenster morgen von 2h | `1` | `true` | `false` |
| CKW zweitteuerstes Zeitfenster morgen von 2h | `1` | `true` | `true` |

Für jeden Sensor: Code kopieren, die drei Parameterzeilen oben setzen, den Code im Feld **Zustand** des jeweiligen Template-Sensor-Helfers speichern. Nicht unter **Entwicklerwerkzeuge → Zustände → Zustand setzen** einfügen; dieses Werkzeug wertet Jinja nicht aus.

```jinja
{% set tage_voraus = 0 %}
{% set teuerstes = false %}
{% set zweites = false %}

{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set zieltag = (now() + timedelta(days=tage_voraus)).strftime('%Y-%m-%d') %}
{% set ns = namespace(fenster=[]) %}

{# Acht Viertelstundenpreise ergeben ein 2-Stunden-Fenster #}
{% for i in range(0, (p | count) - 7) %}
  {% set block = p[i:i+8] %}
  {% set werte = namespace(liste=[]) %}

  {% for q in block %}
    {% if q.integrated is defined and q.integrated | count > 0 %}
      {% set werte.liste = werte.liste + [q.integrated[0].value | float(0)] %}
    {% endif %}
  {% endfor %}

  {% if block | count == 8
        and werte.liste | count == 8
        and block[0].start_timestamp[:10] == zieltag
        and block[7].end_timestamp[:10] == zieltag %}
    {% set ns.fenster = ns.fenster + [{
      'start': i,
      'preis': (werte.liste | sum) / 8
    }] %}
  {% endif %}
{% endfor %}

{% if teuerstes %}
  {% set sortiert = ns.fenster | sort(attribute='preis', reverse=true) %}
{% else %}
  {% set sortiert = ns.fenster | sort(attribute='preis') %}
{% endif %}

{% set auswahl = namespace(fenster=none) %}

{% if sortiert | count > 0 %}
  {% if zweites %}
    {% set bestes = sortiert[0] %}
    {% for f in sortiert %}
      {% if auswahl.fenster is none
            and (f.start >= bestes.start + 8
                 or f.start + 8 <= bestes.start) %}
        {% set auswahl.fenster = f %}
      {% endif %}
    {% endfor %}
  {% else %}
    {% set auswahl.fenster = sortiert[0] %}
  {% endif %}
{% endif %}

{% if auswahl.fenster is not none %}
  {% set i = auswahl.fenster.start %}
  {{ p[i].start_timestamp[:10] }} · {{ p[i].start_timestamp[11:16] }}–{{ p[i+7].end_timestamp[11:16] }} · {{ '%.2f' | format(auswahl.fenster.preis * 100) }} Rp./kWh
{% else %}
Für den gewählten Tag wurden keine passenden CKW-Preise gefunden
{% endif %}
```

Die `sensor.`-IDs entstehen aus den Helfernamen. Home Assistant kann Umlaute umschreiben oder bei Namenskonflikten eine Endung hinzufügen. Prüfe die tatsächlichen IDs unter **Einstellungen → Geräte & Dienste → Entitäten** und verwende genau diese IDs in weiteren Vorlagen.

## 4. Preisempfehlung für heute und morgen

Für jede Empfehlung einen weiteren Template-Sensor-Helfer erstellen, zum Beispiel `CKW Preisempfehlung` und `CKW Preisempfehlung morgen`. Der folgende Code berechnet die vier Zeitfenster direkt aus `sensor.ckw_tarifdaten`; er ist dadurch nicht von den acht separaten Zeitfenster-Helfern abhängig.

Für die Empfehlung **heute** `tage_voraus` auf `0` setzen. Für die Empfehlung **morgen** auf `1` setzen.

```jinja
{% set tage_voraus = 0 %}
{% set p = state_attr('sensor.ckw_tarifdaten', 'prices') or [] %}
{% set zieltag = (now() + timedelta(days=tage_voraus)).strftime('%Y-%m-%d') %}
{% set ns = namespace(fenster=[]) %}

{% for i in range(0, (p | count) - 7) %}
  {% set block = p[i:i+8] %}
  {% set werte = namespace(liste=[]) %}

  {% for q in block %}
    {% if q.integrated is defined and q.integrated | count > 0 %}
      {% set werte.liste = werte.liste + [q.integrated[0].value | float(0)] %}
    {% endif %}
  {% endfor %}

  {% if block | count == 8
        and werte.liste | count == 8
        and block[0].start_timestamp[:10] == zieltag
        and block[7].end_timestamp[:10] == zieltag %}
    {% set ns.fenster = ns.fenster + [{
      'start': i,
      'preis': (werte.liste | sum) / 8
    }] %}
  {% endif %}
{% endfor %}

{% set billig = ns.fenster | sort(attribute='preis') %}
{% set teuer = ns.fenster | sort(attribute='preis', reverse=true) %}
{% set auswahl = namespace(billig_zwei=none, teuer_zwei=none) %}

{% if billig | count > 0 %}
  {% set bestes = billig[0] %}
  {% for f in billig %}
    {% if auswahl.billig_zwei is none
          and (f.start >= bestes.start + 8
               or f.start + 8 <= bestes.start) %}
      {% set auswahl.billig_zwei = f.start %}
    {% endif %}
  {% endfor %}
{% endif %}

{% if teuer | count > 0 %}
  {% set bestes = teuer[0] %}
  {% for f in teuer %}
    {% if auswahl.teuer_zwei is none
          and (f.start >= bestes.start + 8
               or f.start + 8 <= bestes.start) %}
      {% set auswahl.teuer_zwei = f.start %}
    {% endif %}
  {% endfor %}
{% endif %}

{% if billig | count > 0
      and auswahl.billig_zwei is not none
      and teuer | count > 0
      and auswahl.teuer_zwei is not none %}
  {% set b = [billig[0].start, auswahl.billig_zwei] | sort %}
  {% set t = [teuer[0].start, auswahl.teuer_zwei] | sort %}
{{ 'Morgen' if tage_voraus == 1 else 'Heute' }} ({{ zieltag }}): Günstig {{ p[b[0]].start_timestamp[11:16] }}–{{ p[b[0]+7].end_timestamp[11:16] }} und {{ p[b[1]].start_timestamp[11:16] }}–{{ p[b[1]+7].end_timestamp[11:16] }}. Teuer {{ p[t[0]].start_timestamp[11:16] }}–{{ p[t[0]+7].end_timestamp[11:16] }} und {{ p[t[1]].start_timestamp[11:16] }}–{{ p[t[1]+7].end_timestamp[11:16] }}.
{% else %}
Für {{ 'morgen' if tage_voraus == 1 else 'heute' }} konnten nicht genügend CKW-Preise für vier nicht überlappende Zeitfenster gefunden werden.
{% endif %}
```

Der Ausgabezustand ist kurz gehalten. Lange Texte können für Sensorzustände ungeeignet sein; die Markdown-Karte darunter übernimmt die Darstellung als lesbare Liste.

## 5. Dashboard-Karten

### Preise heute und morgen

Füge für jeden Tag eine **Manuelle → Markdown-Karte** hinzu. Die beiden Karten können in einem ausreichend breiten zweispaltigen Dashboard-Abschnitt nebeneinander platziert werden. Auf schmalen Bildschirmen kann Home Assistant sie untereinander anzeigen.

**Preise heute:**

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

**Preise morgen:**

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

### Preisempfehlungen heute und morgen

Erstelle je eine Markdown-Karte. Ändere zwischen den beiden Karten nur die Sensor-ID und den Titel. Die folgende Karte ist für morgen; für heute verwende `sensor.ckw_preisempfehlung` und den Titel `CKW Preisempfehlung`.

```yaml
type: markdown
title: CKW Preisempfehlung morgen
entity_id: sensor.ckw_preisempfehlung_morgen
content: |
  {% set text = states('sensor.ckw_preisempfehlung_morgen') %}
  {% set teile = text.split('. Teuer ') %}
  {% if teile | count > 1 %}
    {% set guenstig = teile[0].split('Günstig ')[1] | replace('.', '') | trim %}
    {% set teuer = teile[1] | replace('.', '') | trim %}
  - **Günstig:** {{ guenstig }}<br><font color="#43a047">Ideal für stromintensive Arbeiten</font>
  - **Teuer:** {{ teuer }}<br><font color="#e53935">Verbrauch reduzieren</font>
  {% else %}
  {{ text }}
  {% endif %}
```

Für die heutige Karte den Inhalt entsprechend anpassen:

```yaml
 type: markdown
 title: CKW Preisempfehlung
 entity_id: sensor.ckw_preisempfehlung
 content: |
   {% set text = states('sensor.ckw_preisempfehlung') %}
   {% set teile = text.split('. Teuer ') %}
   {% if teile | count > 1 %}
     {% set guenstig = teile[0].split('Günstig ')[1] | replace('.', '') | trim %}
     {% set teuer = teile[1] | replace('.', '') | trim %}
   - **Günstig:** {{ guenstig }}<br><font color="#43a047">Ideal für stromintensive Arbeiten</font>
   - **Teuer:** {{ teuer }}<br><font color="#e53935">Verbrauch reduzieren</font>
   {% else %}
   {{ text }}
   {% endif %}
```

### Manueller Aktualisierungsbutton

Füge eine Button-Karte hinzu. Ein Tippen fragt den REST-Sensor sofort erneut ab; das ändert nicht den regulären `scan_interval`.

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

## 6. Preise manuell aktualisieren

Alternativ unter **Entwicklerwerkzeuge → Aktionen** die Aktion **Entität aktualisieren** ausführen. Der YAML-Inhalt der Aktion ist:

```yaml
action: homeassistant.update_entity
data:
  entity_id: sensor.ckw_tarifdaten
```

Home Assistant sendet damit sofort eine neue Anfrage an die CKW-API. Das kann keine Preise abrufen, die CKW noch nicht veröffentlicht hat.

## 7. E-Mail-Benachrichtigungen

Richte zuerst die SMTP-Integration in Home Assistant ein und füge dort einen Empfänger hinzu. Ersetze in den Beispielen `notify.DEIN_EMAIL` durch die von Home Assistant angelegte Notify-Entity.

Die folgende Automation sendet eine E-Mail, wenn eines der vier **heutigen** Zeitfenster beginnt. Für morgen die vier Entity-IDs auf die entsprechenden `_morgen_`-Sensoren ändern. Home Assistant aktualisiert Template-Trigger, die `now()` enthalten, zu Beginn jeder Minute.

```yaml
alias: CKW – E-Mail zu den Preisfenstern
description: Benachrichtigt beim Beginn der heutigen günstigen und teuren Zeitfenster
triggers:
  - trigger: template
    id: guenstigstes
    value_template: >
      {% set s = states('sensor.ckw_guenstigstes_zeitfenster_von_2h').split(' · ') %}
      {% if s | count >= 2 %}
        {{ now().strftime('%Y-%m-%d %H:%M') == s[0] ~ ' ' ~ s[1].split('–')[0] }}
      {% else %} false {% endif %}
  - trigger: template
    id: zweitgünstigstes
    value_template: >
      {% set s = states('sensor.ckw_zweitguenstigstes_zeitfenster_von_2h').split(' · ') %}
      {% if s | count >= 2 %}
        {{ now().strftime('%Y-%m-%d %H:%M') == s[0] ~ ' ' ~ s[1].split('–')[0] }}
      {% else %} false {% endif %}
  - trigger: template
    id: teuerstes
    value_template: >
      {% set s = states('sensor.ckw_teuerstes_zeitfenster_von_2h').split(' · ') %}
      {% if s | count >= 2 %}
        {{ now().strftime('%Y-%m-%d %H:%M') == s[0] ~ ' ' ~ s[1].split('–')[0] }}
      {% else %} false {% endif %}
  - trigger: template
    id: zweitteuerstes
    value_template: >
      {% set s = states('sensor.ckw_zweitteuerstes_zeitfenster_von_2h').split(' · ') %}
      {% if s | count >= 2 %}
        {{ now().strftime('%Y-%m-%d %H:%M') == s[0] ~ ' ' ~ s[1].split('–')[0] }}
      {% else %} false {% endif %}
actions:
  - action: notify.send_message
    target:
      entity_id: notify.DEIN_EMAIL
    data:
      title: >
        {% if trigger.id == 'guenstigstes' %}Günstigstes Zeitfenster beginnt
        {% elif trigger.id == 'zweitgünstigstes' %}Zweitgünstiges Zeitfenster beginnt
        {% elif trigger.id == 'teuerstes' %}Teuerstes Zeitfenster beginnt
        {% else %}Zweitteuerstes Zeitfenster beginnt{% endif %}
      message: >
        {% if trigger.id == 'guenstigstes' %}
          {{ states('sensor.ckw_guenstigstes_zeitfenster_von_2h') }}
        {% elif trigger.id == 'zweitgünstigstes' %}
          {{ states('sensor.ckw_zweitguenstigstes_zeitfenster_von_2h') }}
        {% elif trigger.id == 'teuerstes' %}
          {{ states('sensor.ckw_teuerstes_zeitfenster_von_2h') }}
        {% else %}
          {{ states('sensor.ckw_zweitteuerstes_zeitfenster_von_2h') }}
        {% endif %}
mode: single
```

Eine Template-Automation reagiert zum Zeitpunkt, an dem die Vorlage von `false` auf `true` wechselt. Wenn Home Assistant zur Startzeit ausgeschaltet ist oder die Automation deaktiviert ist, wird die E-Mail nicht nachträglich versendet.

## Fehlerbehebung

- **REST-Sensor zeigt `OK`:** Erwartet. Die Preise sind Attribute des Sensors.
- **Morgenpreise fehlen:** Prüfe `publication_timestamp` und die Datumswerte in `prices`. CKW muss das morgige Profil bereits veröffentlicht haben. Danach den REST-Sensor manuell aktualisieren.
- **Ein Zeitfenster-Helfer zeigt `unknown`:** Prüfe, ob sein Zustandscode im Feld **Zustand** des Template-Helfers steht und ob `sensor.ckw_tarifdaten` Daten im Attribut `prices` hat.
- **Preisempfehlung meldet fehlende Daten:** Prüfe die Entitäts-ID des REST-Sensors und ob genügend Preise für den Zieltagesbereich vorhanden sind.
- **E-Mail kommt nicht an:** Prüfe SMTP-Integration, Empfänger-Notify-Entity und den Automationsverlauf.
- **Dashboard-Tabelle erscheint als Text:** Den vollständigen YAML-Block in den YAML-Editor einer Markdown-Karte einfügen.

## Dateien in diesem Repository

- [`configuration.yaml.example`](configuration.yaml.example): Beispiel für den REST-Sensor.
- [`dashboard.md`](dashboard.md): Beispielkarten für die Preistabellen.
- [`de/`](de/): deutsche Beispieldateien.

## Quellen

- [CKW: Technische Informationen zum dynamischen Tarif](https://www.ckw.ch/energie/strom/dynamischer-tarif-fuer-privatkunden-und-unternehmen/dynamischer-tarif-technische-voraussetzungen)
- [Home Assistant: RESTful-Integration](https://www.home-assistant.io/integrations/rest/)
- [Home Assistant: RESTful Sensor](https://www.home-assistant.io/integrations/sensor.rest/)
- [Home Assistant: Template-Integration](https://www.home-assistant.io/integrations/template/)
- [Home Assistant: SMTP-Integration](https://www.home-assistant.io/integrations/smtp/)

## Mitwirken

Pull Requests mit Korrekturen, getesteten Verbesserungen oder Kompatibilitätshinweisen sind willkommen. Entferne vor dem Veröffentlichen persönliche Angaben, E-Mail-Adressen, Zugangsdaten, Tokens, interne IP-Adressen und private Entitätsnamen aus Beispielen und Screenshots.
