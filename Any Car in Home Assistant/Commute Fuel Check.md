# Morning Fuel Check Automation

A Home Assistant automation that checks whether a household member has enough fuel for their commute, triggering 20 minutes after their morning alarm. Sends a critical or low-fuel warning notification depending on how much range is available.

## How It Works

- Triggers when the time matches the next scheduled alarm on a specified phone
- Only runs on weekdays, between 4am and 8am
- Skips if there's already enough fuel for a full return commute plus a 30 km buffer
- After a 20-minute delay (giving time to get ready), sends one of two notifications:
  - **Critical** — not enough range to reach work safely
  - **Low** — enough to get there, but not enough to return home

## Required Helpers

### Commute Distance — Input Number Helper

This stores the one-way commute distance in km, used to calculate whether enough fuel is available.

1. Go to **Settings → Devices & Services → Helpers → Create Helper**
2. Choose **Number**
3. Configure it as follows:

| Field | Value |
|---|---|
| Name | `Commute Distance` (or similar) |
| Minimum | `0` |
| Maximum | `500` |
| Unit of measurement | `km` |

4. Note the entity ID it creates (e.g. `input_number.your_commute_distance`) — you'll need it in the automation below.

## Dependencies

This automation uses the **Mobile App** integration for push notifications. The person being notified must have the [Home Assistant Companion App](https://companion.home-assistant.io/) installed and their device registered in HA.

The alarm trigger relies on the **next alarm sensor** exposed by the Companion App on Android devices (e.g. `sensor.your_phone_next_alarm`). This sensor may not be available on all devices or iOS.

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `YOUR VEHICLE NAME` | The vehicle name used in notification messages (e.g. `My Car`). |
| `sensor.your_phone_next_alarm` | The next alarm sensor from the Companion App on the relevant phone. Found under **Developer Tools → States**, search for `next_alarm`. |
| `sensor.your_remaining_range_sensor` | Sensor reporting estimated remaining driving range in km. |
| `input_number.your_commute_distance` | The commute distance helper created above. |
| `notify.mobile_app_your_phone` | Notify service for the relevant person's phone. Found under **Developer Tools → Services**, search `notify.mobile_app_`. |

> **Note:** The 30 km buffer used throughout this automation provides a safety margin on top of the commute distance. Adjust this value to suit your comfort level.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the YAML below, replacing all placeholders with your own values

```yaml
alias: Morning Fuel Check - YOUR VEHICLE NAME
description: >-
  Notify if there isn't enough fuel for the commute, 20 minutes after the
  morning alarm
triggers:
  - trigger: template
    value_template: >
      {{ now().strftime('%H:%M') ==
      as_datetime(states('sensor.your_phone_next_alarm')).strftime('%H:%M') }}
conditions:
  - condition: time
    weekday:
      - mon
      - tue
      - wed
      - thu
      - fri
  - condition: template
    value_template: |
      {{ now().strftime('%H') | int >= 4 and now().strftime('%H') | int <= 8 }}
  - condition: template
    value_template: >
      {{ states('sensor.your_remaining_range_sensor') | float(0) <
      (states('input_number.your_commute_distance') | float(0) * 2) + 30 }}
actions:
  - delay:
      minutes: 20
  - variables:
      range: "{{ states('sensor.your_remaining_range_sensor') | float(0) }}"
      commute: "{{ states('input_number.your_commute_distance') | float(0) }}"
      range_at_work: "{{ (range - commute) | round(0) }}"
  - choose:
      - conditions:
          - condition: template
            value_template: |
              {{ range < commute + 30 }}
        sequence:
          - action: notify.mobile_app_your_phone
            data:
              title: ⛽ Fuel Critical - Fill Up Before Work
              message: >
                YOUR VEHICLE NAME doesn't have enough fuel to get you to work
                safely. You'll need to fill up on the way!
              data:
                notification_icon: mdi:gas-station
                color: "#d43f3f"
      - conditions:
          - condition: template
            value_template: |
              {{ range >= commute + 30 and range < (commute * 2) + 30 }}
        sequence:
          - action: notify.mobile_app_your_phone
            data:
              title: ⛽ Fuel Low - Fill Up Before Home
              message: >
                YOUR VEHICLE NAME will have approximately {{ range_at_work }} km
                of range when you get to work. Make sure you fill up before
                heading home!
              data:
                notification_icon: mdi:gas-station
                color: "#d4923f"
mode: restart
```
