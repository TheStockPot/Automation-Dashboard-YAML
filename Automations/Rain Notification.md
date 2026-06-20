# Rain Alert

A Home Assistant automation that notifies whoever's home as soon as rain starts, as a prompt to check whether there's washing on the line. If nobody's home when it starts raining, it waits for someone to arrive and lets them know it rained while they were out.

## How It Works

The automation triggers when a rain sensor reports a "moist" state, held for 10 seconds (to avoid false triggers from a single stray drop). From there, a `choose` block checks who's currently home:

- If both people are home, both get notified.
- If only one is home, only that person gets notified.
- If neither is home, the automation waits for either person to arrive (tracked individually via a `wait_for_trigger` with named trigger IDs), waits a further 10 minutes once someone does, then checks which specific person triggered the wait and sends a "it rained while you were out" notification to that person only.

## Dependencies

- A rain sensor exposed as a `binary_sensor` with a `moist`/dry-type device trigger (in this case from a local weather station integration's piezo rain gauge)
- The **Mobile App** integration for push notifications, with each household member's device registered in Home Assistant
- `person` entities for each household member, with home/away tracking configured

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `binary_sensor.rain_sensor` | Your rain sensor entity. |
| `person.person_one` / `person.person_two` | Your household members' `person` entities. |
| `notify.mobile_app_person_one` / `notify.mobile_app_person_two` | Notify services for each household member's phone. Found under **Developer Tools → Services**, search `notify.mobile_app_`. |

> **Note:** The rain trigger requires the sensor to hold its "moist" state for 10 seconds before firing, to filter out brief or false readings. The "while you were out" notification waits 10 minutes after someone arrives home before sending, giving them a moment to settle in rather than greeting them at the door.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the YAML below, replacing all placeholders with your own values

```yaml
alias: Rain Alert
description: Sends notification when rain starts
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.rain_sensor
    to: "on"
    for:
      hours: 0
      minutes: 0
      seconds: 10
conditions: []
actions:
  - choose:
      - conditions:
          - condition: state
            entity_id: person.person_one
            state:
              - home
          - condition: state
            entity_id: person.person_two
            state:
              - home
        sequence:
          - action: notify.mobile_app_person_one
            data:
              title: It's started raining
              message: Is there washing on the line?
              data:
                tag: rain_start
                notification_icon: mdi:weather-rainy
          - action: notify.mobile_app_person_two
            data:
              title: It's started raining
              message: Is there washing on the line?
              data:
                tag: rain_start
                notification_icon: mdi:weather-rainy
        alias: If both home
      - conditions:
          - condition: state
            entity_id: person.person_two
            state:
              - not_home
          - condition: state
            entity_id: person.person_one
            state:
              - home
        sequence:
          - action: notify.mobile_app_person_one
            data:
              title: It's started raining
              message: Is there washing on the line?
              data:
                tag: rain_start
                notification_icon: mdi:weather-rainy
        alias: If person one home
      - conditions:
          - condition: state
            entity_id: person.person_two
            state:
              - home
          - condition: state
            entity_id: person.person_one
            state:
              - not_home
        sequence:
          - action: notify.mobile_app_person_two
            data:
              title: It's started raining
              message: Is there washing on the line?
              data:
                tag: rain_start
                notification_icon: mdi:weather-rainy
        alias: If person two home
      - conditions:
          - condition: state
            entity_id: person.person_two
            state:
              - not_home
          - condition: state
            entity_id: person.person_one
            state:
              - not_home
        sequence:
          - wait_for_trigger:
              - id: person_one_arrives
                entity_id: person.person_one
                to: home
                trigger: state
              - id: person_two_arrives
                entity_id: person.person_two
                to: home
                trigger: state
          - delay:
              hours: 0
              minutes: 10
              seconds: 0
              milliseconds: 0
          - choose:
              - conditions:
                  - condition: template
                    value_template: "{{ wait.trigger and wait.trigger.id == 'person_one_arrives' }}"
                sequence:
                  - action: notify.mobile_app_person_one
                    data:
                      title: Rain!
                      message: >-
                        It rained while you were out - washing on the line might
                        be wet!
                      data:
                        notification_icon: mdi:cloud-alert
                        color: "#2196F3"
              - conditions:
                  - condition: template
                    value_template: "{{ wait.trigger and wait.trigger.id == 'person_two_arrives' }}"
                sequence:
                  - action: notify.mobile_app_person_two
                    data:
                      title: Rain!
                      message: >-
                        It rained while you were out - washing on the line might
                        be wet!
                      data:
                        notification_icon: mdi:cloud-alert
                        color: "#2196F3"
        alias: If nobody home
mode: single
```
