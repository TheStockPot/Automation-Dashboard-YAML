# Festoon Lights

A pair of Home Assistant automations that turn pergola/outdoor festoon lights on automatically when it's dark and you head outside, and turn them off again either when the blind closes for the evening or when sleep mode kicks in — covering both the "we're done for the night" case and the "we left the blind up overnight" case.

## How It Works

**Turn on Festoon Lights** triggers either when the back door opens, or when the living room blinds change from closed to opening/open — covering both ways you might head outside. It only proceeds if a lux sensor reads below a threshold, indicating it's dark enough for the lights to be worth turning on. From there it turns the festoon lights on (via a smart switch), then waits for sleep mode to be enabled before turning them off — acting as a safety net for nights where the blind is left up.

**Turn off Festoons** is a separate, simpler automation that turns the festoon lights off whenever the relevant blind changes from open to closed — covering the more common case of closing up for the evening without necessarily going to sleep yet.

Between the two, the lights get turned off either when you close the blind, or — if you forget and leave it up — automatically once sleep mode is enabled.

## Dependencies

- A door sensor exposed as a `binary_sensor` in Home Assistant
- Motorised blinds exposed as `cover` entities
- A lux/light level sensor (in this case from a local weather station integration) used to determine darkness
- A smart switch (in this case a Shelly, mounted inline on the festoon light power lead) exposed as a `switch` entity for turning the lights on
- The festoon lights also need to be exposed as a `light` entity for the off actions — in this setup, the same physical switch is represented as both a `switch` (used to turn on) and a `light` (used to turn off). If your switch is only exposed as a `switch` entity, use `switch.turn_off` instead of `light.turn_off` for the off actions below.
- An `input_boolean` helper used as a "sleep mode" flag (see the [Bedtime & Wakeup automation](#) for how this gets set)

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `binary_sensor.back_door` | Your back door sensor entity. |
| `cover.living_room_blinds` | The blind whose opening triggers the lights to turn on. |
| `sensor.outdoor_lux` | Your outdoor light level sensor. |
| `switch.festoon_lights` | The smart switch controlling your festoon lights (used to turn them on). |
| `light.festoon_lights` | The light entity representing the same festoon lights (used to turn them off). |
| `cover.back_door_blind` | The blind whose closing triggers the lights to turn off. |
| `input_boolean.sleep_mode` | Your sleep mode helper boolean. |

> **Note:** The darkness threshold is set to a lux value of `15`. Adjust this to suit your own sensor and how dark you want it before the lights kick in. Also note that `cover.living_room_blinds` (the "turn on" trigger) and `cover.back_door_blind` (the "turn off" trigger) are two different blinds in this setup — adjust to match your own layout if they're the same blind for you.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the **Turn on Festoon Lights** YAML below, replacing all placeholders with your own entity IDs
3. Repeat for the **Turn off Festoons** YAML

### Turn on Festoon Lights

```yaml
alias: Turn on Festoon Lights
description: ""
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.back_door
    to: "on"
  - trigger: state
    entity_id:
      - cover.living_room_blinds
    from:
      - closed
    to:
      - opening
      - open
conditions:
  - condition: numeric_state
    entity_id: sensor.outdoor_lux
    below: 15
actions:
  - action: switch.turn_on
    metadata: {}
    target:
      entity_id: switch.festoon_lights
    data: {}
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - input_boolean.sleep_mode
        to:
          - "on"
  - action: light.turn_off
    metadata: {}
    target:
      entity_id: light.festoon_lights
    data: {}
mode: restart
```

### Turn off Festoons

```yaml
alias: Turn off Festoons
description: ""
triggers:
  - trigger: state
    entity_id:
      - cover.back_door_blind
    from:
      - open
    to:
      - closed
conditions: []
actions:
  - action: light.turn_off
    metadata: {}
    target:
      entity_id: light.festoon_lights
    data: {}
mode: single
```
