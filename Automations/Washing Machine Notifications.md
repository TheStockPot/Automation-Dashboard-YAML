# Washing Machine & Dryer Notifications

A pair of near-identical Home Assistant automations that notify whoever's home when the washing machine or dryer finishes a cycle. If nobody's home when it finishes, the notification waits until someone actually arrives — so you're not getting alerted about laundry you can't do anything about.

## How It Works

Each automation triggers when the appliance's status sensor changes to `end`. From there, a `choose` block checks who's currently home:

- If only one person is home, only that person gets notified.
- If both are home, both get notified.
- If neither is home, the automation falls through to the `default` case: it waits for either person to arrive (tracked individually via a `wait_for_trigger` with named trigger IDs), waits an extra minute once someone does, then checks which specific person triggered the wait and sends a "finished while you were out" notification to that person only — rather than blasting both phones regardless of who actually walked in.

The washing machine and dryer versions are functionally identical, differing only in the trigger entity, notification text, and icon.

## Dependencies

- An appliance integration that exposes a cycle status sensor with an `end` state when a cycle finishes (in this case, LG's ThinQ integration)
- The **Mobile App** integration for push notifications, with each household member's device registered in Home Assistant
- `person` entities for each household member, with home/away tracking configured (e.g. via device trackers or the Companion App's location features)

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `sensor.washing_machine_current_status` | Your washing machine's cycle status sensor. |
| `sensor.dryer_current_status` | Your dryer's cycle status sensor. |
| `person.person_one` / `person.person_two` | Your household members' `person` entities. |
| `notify.mobile_app_person_one` / `notify.mobile_app_person_two` | Notify services for each household member's phone. Found under **Developer Tools → Services**, search `notify.mobile_app_`. |

> **Note:** If you only want to notify one person, or have a single-person household, remove the relevant conditions/sequences from the `choose` block and the second `wait_for_trigger`/`choose` entries in the `default` case.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the **Washing Machine Notification** YAML below, replacing all placeholders with your own values
3. Repeat for the **Dryer Notification** YAML

### Washing Machine Notification

```yaml
alias: Washing Machine Notification
description: Notifies when washing machine is finished
triggers:
  - trigger: state
    entity_id: sensor.washing_machine_current_status
    to: end
conditions: []
actions:
  - choose:
      - conditions:
          - condition: state
            entity_id: person.person_one
            state: home
          - condition: state
            entity_id: person.person_two
            state: not_home
        sequence:
          - action: notify.mobile_app_person_one
            data:
              title: Washing Machine
              message: The washing machine has finished
              data:
                notification_icon: mdi:washing-machine
                color: "#2196F3"
      - conditions:
          - condition: state
            entity_id: person.person_one
            state: not_home
          - condition: state
            entity_id: person.person_two
            state: home
        sequence:
          - action: notify.mobile_app_person_two
            data:
              title: Washing Machine
              message: The washing machine has finished
              data:
                notification_icon: mdi:washing-machine
                color: "#2196F3"
      - conditions:
          - condition: state
            entity_id: person.person_one
            state: home
          - condition: state
            entity_id: person.person_two
            state: home
        sequence:
          - action: notify.mobile_app_person_one
            data:
              title: Washing Machine
              message: The washing machine has finished
              data:
                notification_icon: mdi:washing-machine
                color: "#2196F3"
          - action: notify.mobile_app_person_two
            data:
              title: Washing Machine
              message: The washing machine has finished
              data:
                notification_icon: mdi:washing-machine
                color: "#2196F3"
    default:
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
          minutes: 1
      - choose:
          - conditions:
              - condition: template
                value_template: "{{ wait.trigger and wait.trigger.id == 'person_one_arrives' }}"
            sequence:
              - action: notify.mobile_app_person_one
                data:
                  title: Washing Machine
                  message: The washing machine finished while you were out!
                  data:
                    notification_icon: mdi:washing-machine
                    color: "#2196F3"
          - conditions:
              - condition: template
                value_template: "{{ wait.trigger and wait.trigger.id == 'person_two_arrives' }}"
            sequence:
              - action: notify.mobile_app_person_two
                data:
                  title: Washing Machine
                  message: The washing machine finished while you were out!
                  data:
                    notification_icon: mdi:washing-machine
                    color: "#2196F3"
mode: single
```

### Dryer Notification

```yaml
alias: Dryer Notification
description: Notifies when Dryer is finished
triggers:
  - trigger: state
    entity_id: sensor.dryer_current_status
    to: end
conditions: []
actions:
  - choose:
      - conditions:
          - condition: state
            entity_id: person.person_one
            state: home
          - condition: state
            entity_id: person.person_two
            state: not_home
        sequence:
          - action: notify.mobile_app_person_one
            data:
              title: Dryer
              message: The dryer has finished
              data:
                notification_icon: mdi:tumble-dryer
                color: "#2196F3"
      - conditions:
          - condition: state
            entity_id: person.person_one
            state: not_home
          - condition: state
            entity_id: person.person_two
            state: home
        sequence:
          - action: notify.mobile_app_person_two
            data:
              title: Dryer
              message: The dryer has finished
              data:
                notification_icon: mdi:tumble-dryer
                color: "#2196F3"
      - conditions:
          - condition: state
            entity_id: person.person_one
            state: home
          - condition: state
            entity_id: person.person_two
            state: home
        sequence:
          - action: notify.mobile_app_person_one
            data:
              title: Dryer
              message: The dryer has finished
              data:
                notification_icon: mdi:tumble-dryer
                color: "#2196F3"
          - action: notify.mobile_app_person_two
            data:
              title: Dryer
              message: The dryer has finished
              data:
                notification_icon: mdi:tumble-dryer
                color: "#2196F3"
    default:
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
          minutes: 1
      - choose:
          - conditions:
              - condition: template
                value_template: "{{ wait.trigger and wait.trigger.id == 'person_one_arrives' }}"
            sequence:
              - action: notify.mobile_app_person_one
                data:
                  title: Dryer
                  message: The Dryer finished while you were out!
                  data:
                    notification_icon: mdi:tumble-dryer
                    color: "#2196F3"
          - conditions:
              - condition: template
                value_template: "{{ wait.trigger and wait.trigger.id == 'person_two_arrives' }}"
            sequence:
              - action: notify.mobile_app_person_two
                data:
                  title: Dryer
                  message: The Dryer finished while you were out!
                  data:
                    notification_icon: mdi:tumble-dryer
                    color: "#2196F3"
mode: single
```
