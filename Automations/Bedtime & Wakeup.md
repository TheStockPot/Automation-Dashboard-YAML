# Bedtime & Wakeup

A pair of Home Assistant automations that handle the small, repetitive things we do every night before bed and every morning when we wake up — bedside lamps, AC zoning, blinds, and a "sleep mode" boolean used as a condition elsewhere in the house.

## How It Works

**Bed Time Mode** triggers when the Android TV (technically the TV's remote entity) is turned off, but only runs if it's between 8:00pm and 11:58pm — used as a rough signal that we're winding down for the night, while avoiding false triggers if the TV happens to switch off earlier in the day. From there it turns the bedside lamps on at full brightness with a slow 30 second transition, switches the master bedroom climate zone on, and turns the bathroom, office, and dining climate zones off — so we're not heating or cooling rooms nobody's in overnight. It then waits (up to 3 hours, after which it continues anyway) for the bedside lamps to be turned off as the signal that we're actually going to sleep. Once that happens, the blinds close, sleep mode is enabled, and after a 45 minute delay the main air conditioning unit is switched off entirely.

**Wakeup** triggers 15 minutes before Britt's phone alarm on weekdays, or at a flat 9:00am on weekends. A time window condition (4:00am–9:00am) prevents it from firing on any other alarms set outside that range. If the outdoor temperature is below 15°C, it turns on the master bedroom, bathroom, and office climate zones, to take the chill off before we're up. After a 15 minute delay (bringing us to the actual alarm time), it disables sleep mode, turns the bedside lamps on at full brightness with a slow 60 second transition, and opens the back door blind ready to let the dog in. After a further 10 minute delay, the bedroom downlights turn on — a backup in case we've slept through the alarm and lamps.

## Dependencies

- An Android TV (or similar) exposed as a `remote` entity in Home Assistant, used as the Bedtime trigger
- A sensor reporting the next alarm time from a phone (e.g. via the Home Assistant Companion App), exposed as a `sensor` with a datetime state — used for the weekday Wakeup trigger
- Ducted zone control, with each zone exposed as a `climate` entity. If your zone dampers don't expose individual climate entities, a `switch` entity per zone works just as well — swap the relevant `climate.turn_on`/`climate.turn_off` actions for `switch.turn_on`/`switch.turn_off` targeting your zone switches instead.
- An outdoor temperature sensor (in this case from a local weather station integration)
- A main air conditioning unit exposed as a `climate` entity, separate from the per-zone entities
- Motorised blinds exposed as `cover` entities
- An `input_boolean` helper used as a "sleep mode" flag, referenced elsewhere in the house as a condition

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `remote.your_tv` | Your TV/remote entity, used to detect when the TV is turned off as a "winding down" signal. |
| `light.bedside_lamp_1` / `light.bedside_lamp_2` | Your bedside lamp entities. |
| `climate.master_bed_thermostat` | The master bedroom climate zone entity. |
| `climate.bathroom_thermostat` | The bathroom zone climate entity. |
| `climate.office_thermostat` / `climate.dining_thermostat` | Office and dining zone climate entities, turned off overnight. |
| `cover.your_blind_1` / `cover.your_blind_2` / etc. | Your blind/cover entities to close at bedtime. |
| `input_boolean.sleep_mode` | Your sleep mode helper boolean. |
| `climate.main_ac_unit` | Your main/overall air conditioning unit entity, switched off after the 45 minute delay. |
| `sensor.your_phone_next_alarm` | A sensor exposing your phone's next alarm time, used for the weekday Wakeup trigger. |
| `sensor.outdoor_temperature` | Your outdoor temperature sensor. |
| `cover.back_door_blind` | The blind opened at wake time, e.g. to let a pet in. |
| `light.bedroom_downlights` | Bedroom downlight/ceiling light entity, used as the wake-up backup. |

> **Note:** Bedtime's TV-off trigger only fires between 8:00pm and 11:58pm. The wait for bedside lamps to turn off has a 3 hour timeout, after which the automation continues regardless — so it won't run forever if the lamps are left on. Wakeup's heater zone logic only fires below 15°C outdoor temperature, and the weekday/weekend trigger split means weekday wake time follows a phone alarm while weekends default to 9:00am.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the **Bed Time Mode** YAML below, replacing all placeholders with your own entity IDs
3. Repeat for the **Wakeup** YAML

### Bed Time Mode

```yaml
alias: Bed Time Mode
description: Configure house for sleep
triggers:
  - trigger: state
    entity_id:
      - remote.your_tv
    to: "off"
conditions:
  - condition: time
    after: "20:00:00"
    before: "23:58:00"
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
      - sat
      - sun
actions:
  - action: light.turn_on
    metadata: {}
    target:
      entity_id:
        - light.bedside_lamp_1
        - light.bedside_lamp_2
    data:
      transition: 30
      brightness_pct: 100
  - action: climate.turn_on
    metadata: {}
    target:
      entity_id: climate.master_bed_thermostat
    data: {}
  - action: climate.turn_off
    metadata: {}
    target:
      entity_id:
        - climate.bathroom_thermostat
        - climate.office_thermostat
        - climate.dining_thermostat
    data: {}
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - light.bedside_lamp_1
          - light.bedside_lamp_2
        from:
          - "on"
        to:
          - "off"
    timeout:
      hours: 3
      minutes: 0
      seconds: 0
      milliseconds: 0
    continue_on_timeout: true
  - action: cover.close_cover
    metadata: {}
    target:
      entity_id:
        - cover.your_blind_1
        - cover.your_blind_2
        - cover.your_blind_3
        - cover.your_blind_4
    data: {}
  - action: input_boolean.turn_on
    metadata: {}
    target:
      entity_id: input_boolean.sleep_mode
    data: {}
  - delay:
      hours: 0
      minutes: 45
      seconds: 0
      milliseconds: 0
  - action: climate.turn_off
    metadata: {}
    target:
      entity_id: climate.main_ac_unit
    data: {}
mode: single
```

### Wakeup

```yaml
alias: Wakeup
description: ""
triggers:
  - trigger: time
    at:
      entity_id: sensor.your_phone_next_alarm
      offset: "-00:15:00"
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
    alias: 15 minutes before alarm
  - trigger: time
    at: "09:00:00"
    weekday:
      - sun
      - sat
conditions:
  - condition: time
    after: "04:00:00"
    before: "09:00:00"
actions:
  - alias: Choose whether to enable heating
    choose:
      - conditions:
          - condition: numeric_state
            entity_id: sensor.outdoor_temperature
            below: 15
        sequence:
          - action: climate.turn_on
            metadata: {}
            target:
              entity_id:
                - climate.office_thermostat
                - climate.master_bed_thermostat
                - climate.bathroom_thermostat
            data: {}
  - delay:
      hours: 0
      minutes: 15
      seconds: 0
      milliseconds: 0
  - action: input_boolean.turn_off
    metadata: {}
    target:
      entity_id: input_boolean.sleep_mode
    data: {}
    alias: Turn off sleep mode
  - action: light.turn_on
    metadata: {}
    data:
      brightness_pct: 100
      transition: 60
    target:
      entity_id:
        - light.bedside_lamp_1
        - light.bedside_lamp_2
    alias: Turn on bedside lamps
  - action: cover.open_cover
    metadata: {}
    target:
      entity_id: cover.back_door_blind
    data: {}
    alias: Open back door blind
  - delay:
      hours: 0
      minutes: 10
      seconds: 0
      milliseconds: 0
  - action: light.turn_on
    metadata: {}
    data:
      transition: 30
    target:
      entity_id: light.bedroom_downlights
    alias: Turn on bedroom downlights
mode: single
```
