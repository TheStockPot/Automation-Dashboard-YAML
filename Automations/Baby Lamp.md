# Nursery Lamp

A pair of Home Assistant automations that bring a lamp on at a low, baby-safe brightness when the nursery door opens, boost it for a nappy change, and turn it off again once you leave.

## How It Works

There are two separate automations working together:

**Nursery Light** triggers when the nursery door sensor opens, but only runs if it's dark in the room — checked via a binary sensor exposed by the nursery camera's night vision state (so it only fires when night vision indicates low light). The lamp turns on at **15% brightness** with a 4 second transition — enough to see by without waking the baby. The automation then waits for the separate wipe container sensor to open, which signals a nappy change is happening, and boosts the lamp to **50% brightness** so you've got enough light to actually see what you're doing. Once the container closes again, signalling the change is done, the automation waits 5 seconds and then drops the lamp back down to 15% — keeping the room dim while you're still in there, rather than jumping straight back to off.

**Nursery Lamp Off** is a separate automation that simply turns the lamp off (with the same 4 second transition) whenever the nursery door sensor goes from open to closed.

These are kept as two separate automations rather than one, so the lamp still turns off automatically when you leave the room — even if it was switched on manually rather than by the door trigger.

## Dependencies

- A door sensor on the nursery door, exposed as a `binary_sensor` in Home Assistant ([here's how I 3D printed mine into the door frame](https://youtu.be/XVaGANL2T7o))
- A `binary_sensor` indicating darkness/night — in my case this comes from the nursery camera's night vision state, but any "is it dark" binary sensor will work (e.g. a lux sensor with a threshold, or `sun.sun` below horizon)
- A `binary_sensor` on the wipe container, used to detect when a nappy change starts and ends
- A dimmable smart bulb or lamp (`light` entity) in the nursery

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `binary_sensor.nursery_door` | Your door sensor entity for the nursery door. |
| `binary_sensor.is_dark` | A binary sensor indicating it's dark/night. Sourced from a camera's night vision state, a lux sensor, or similar. |
| `binary_sensor.wipe_container` | Your wipe container (or similar) sensor. Used to detect the start and end of a nappy change. |
| `light.nursery_lamp` | The light entity for your nursery lamp. |

> **Note:** Brightness levels are set to 15% for general visibility and 50% for nappy changes. Adjust these `brightness_pct` values to suit your own lamp and room.

## Usage

> **Before you use this:** the YAML below assumes `binary_sensor.wipe_container` reports `on` when physically open and `off` when closed. Confirm this against your own sensor's actual behaviour in **Developer Tools → States** before relying on it — if it's reversed, swap the `to:` values in the two `wait_for_trigger` blocks.

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the **Nursery Light** YAML below, replacing all placeholders with your own entity IDs
3. Repeat for the **Nursery Lamp Off** YAML

### Nursery Light

```yaml
alias: Nursery Light
description: ""
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.nursery_door
    to: "on"
conditions:
  - condition: state
    entity_id: binary_sensor.is_dark
    state: "on"
    alias: Check camera sees darkness
actions:
  - action: light.turn_on
    metadata: {}
    data:
      brightness_pct: 15
      transition: 4
    target:
      entity_id: light.nursery_lamp
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - binary_sensor.wipe_container
        to: "on"
    alias: Wait for wipe container to be opened
  - action: light.turn_on
    metadata: {}
    data:
      brightness_pct: 50
    target:
      entity_id: light.nursery_lamp
    alias: Increase brightness
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - binary_sensor.wipe_container
        to: "off"
    alias: Wait for wipe container to be closed
  - delay:
      hours: 0
      minutes: 0
      seconds: 5
      milliseconds: 0
  - action: light.turn_on
    metadata: {}
    data:
      brightness_pct: 15
      transition: 4
    target:
      entity_id: light.nursery_lamp
    alias: Reduce light brightness
mode: restart
```

### Nursery Lamp Off

```yaml
alias: Nursery Lamp Off
description: ""
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.nursery_door
    from: "on"
    to: "off"
conditions: []
actions:
  - action: light.turn_off
    metadata: {}
    data:
      transition: 4
    target:
      entity_id: light.nursery_lamp
mode: single
```
