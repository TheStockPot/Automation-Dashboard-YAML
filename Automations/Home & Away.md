# Home & Away

A pair of Home Assistant automations that prepare the house for being empty when everyone leaves, and restore it when someone's on their way back — covering blinds, climate, lighting, and a robot vacuum, with an eye on solar export so the air conditioner isn't left running on grid power while nobody's home.

## How It Works

**Away** triggers when either person's `person` entity changes from `home` to `not_home`, but only proceeds if both people are confirmed `not_home` — so it doesn't fire on the first person leaving if the other is still home. It takes a snapshot of the current state of the blinds and the main air conditioning unit, saving it as a scene to restore later. It then turns off the lights across most of the house (by area, rather than listing individual lights), starts the robot vacuum, and closes a set of blinds. If solar export is already below a threshold (indicating no excess power being produced) at this point, it turns the air conditioner off immediately. Otherwise, it waits — for up to 12 hours — for either someone to arrive home/nearly home, or for export to drop below the threshold. Once one of those happens, it checks which: if export has dropped, it turns the AC off; if a person has arrived first, it stops without changing the AC, leaving that for the Home automation to handle via the restored scene.

**Home** triggers when either person's `person` entity changes from `not_home` to a custom "Nearly Home" zone state. It restores the snapshot scene taken when everyone left (covering blinds and AC), then checks whether it's dark — if so, the relevant blinds are immediately closed again, overriding whatever position the restored scene set them to. From there it pauses and docks the robot vacuum, and turns on the porch light.

## Dependencies

- `person` entities for each household member, with a custom "Nearly Home" zone configured in addition to the default home/away states
- Motorised blinds exposed as `cover` entities
- A main air conditioning unit exposed as a `climate` entity
- Lights grouped by Home Assistant **Area** (bedroom, dining room, garden, living room, nursery, office, or your own equivalent areas)
- A robot vacuum exposed as a `vacuum` entity, supporting start, pause, and return-to-base
- A solar inverter/battery system exposing a grid export sensor (in this case a Solix X1 system), used to decide whether to leave the AC running while away based on whether excess solar is being produced

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `person.person_one` / `person.person_two` | Your household members' `person` entities. |
| `Nearly Home` | The name of your custom "almost home" zone, set up under **Settings → Areas, Labels & Zones → Zones**. |
| `cover.blind_1` / `cover.blind_2` / etc. | Your blind/cover entities to snapshot and close when leaving. |
| `climate.main_ac_unit` | Your main/overall air conditioning unit entity. |
| Area IDs (`bedroom`, `dining_room`, `garden`, `living_room`, `nursery`, `office`) | Replace with your own Home Assistant area IDs, found under **Settings → Areas, Labels & Zones → Areas**. |
| `vacuum.your_vacuum` | Your robot vacuum entity. |
| `sensor.solar_grid_export` | Your solar/battery system's grid export sensor, used to determine whether excess solar is currently being produced. |
| `sensor.outdoor_lux` | A light level sensor used to decide whether to close the blinds again on arrival, regardless of the restored scene's blind positions. |
| `light.porch_light` | The light turned on when arriving home. |

> **Note:** The Away automation only proceeds once **both** people are confirmed away, to avoid one person leaving for an errand triggering a full "everyone's out" routine. The wait for someone returning (or export dropping) has a 12 hour timeout with `continue_on_timeout: false` — meaning if neither condition is met within 12 hours, the automation will stop with an error rather than continuing on. Adjust this timeout to suit how long the house is realistically left empty.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the **Away** YAML below, replacing all placeholders with your own entity IDs
3. Repeat for the **Home** YAML

### Away

```yaml
alias: Away
description: ""
triggers:
  - trigger: state
    entity_id:
      - person.person_one
    from:
      - home
    to:
      - not_home
  - trigger: state
    entity_id:
      - person.person_two
    from:
      - home
    to:
      - not_home
    for:
      hours: 0
      minutes: 0
      seconds: 0
conditions:
  - condition: and
    conditions:
      - condition: state
        entity_id: person.person_one
        state:
          - not_home
      - condition: state
        entity_id: person.person_two
        state:
          - not_home
actions:
  - action: scene.create
    metadata: {}
    data:
      scene_id: away
      snapshot_entities:
        - cover.blind_1
        - cover.blind_2
        - cover.blind_3
        - cover.blind_4
        - climate.main_ac_unit
  - action: light.turn_off
    metadata: {}
    target:
      area_id:
        - bedroom
        - dining_room
        - garden
        - living_room
        - nursery
        - office
    data: {}
  - action: vacuum.start
    metadata: {}
    target:
      entity_id: vacuum.your_vacuum
    data: {}
  - action: cover.close_cover
    metadata: {}
    data: {}
    target:
      entity_id:
        - cover.blind_1
        - cover.blind_2
        - cover.blind_3
        - cover.blind_4
  - if:
      - condition: numeric_state
        entity_id: sensor.solar_grid_export
        below: 0.1
    then:
      - action: climate.turn_off
        metadata: {}
        target:
          entity_id: climate.main_ac_unit
        data: {}
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - person.person_one
          - person.person_two
        to:
          - home
          - Nearly Home
      - trigger: numeric_state
        entity_id:
          - sensor.solar_grid_export
        below: 0.1
    timeout:
      hours: 12
    continue_on_timeout: false
  - choose:
      - conditions:
          - condition: numeric_state
            entity_id: sensor.solar_grid_export
            below: 0.1
        sequence:
          - action: climate.turn_off
            metadata: {}
            target:
              entity_id: climate.main_ac_unit
            data: {}
      - conditions:
          - condition: or
            conditions:
              - condition: state
                entity_id: person.person_one
                state:
                  - home
                  - Nearly Home
              - condition: state
                entity_id: person.person_two
                state:
                  - home
                  - Nearly Home
        sequence:
          - stop: ""
mode: single
```

### Home

```yaml
alias: Home
description: Automation that restores last 'away' state and returns vacuum to dock
triggers:
  - trigger: state
    entity_id:
      - person.person_one
    for:
      hours: 0
      minutes: 0
      seconds: 0
    from:
      - not_home
    to:
      - Nearly Home
  - trigger: state
    entity_id:
      - person.person_two
    for:
      hours: 0
      minutes: 0
      seconds: 0
    from:
      - not_home
    to:
      - Nearly Home
conditions: []
actions:
  - action: scene.turn_on
    metadata: {}
    target:
      entity_id: scene.away
    data:
      transition: 5
  - if:
      - condition: numeric_state
        entity_id: sensor.outdoor_lux
        below: 20
    then:
      - action: cover.close_cover
        metadata: {}
        target:
          entity_id:
            - cover.blind_1
            - cover.blind_2
            - cover.blind_3
            - cover.blind_4
        data: {}
    alias: If dark, put blinds down
  - action: vacuum.pause
    metadata: {}
    target:
      entity_id: vacuum.your_vacuum
    data: {}
  - action: vacuum.return_to_base
    metadata: {}
    target:
      entity_id: vacuum.your_vacuum
    data: {}
  - action: light.turn_on
    metadata: {}
    target:
      entity_id: light.porch_light
    data:
      transition: 10
mode: single
```
