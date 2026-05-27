# Notify Fault Code Automation

A Home Assistant automation that sends a push notification to household mobile devices when a vehicle ECU reports a fault code, including how many kilometres ago it was detected.

## How It Works

- Triggers when the vehicle's ECU status binary sensor turns on
- Only fires if the fault was detected more than 0 km ago (i.e. it's a persisted fault, not a brand new detection at 0 km)
- Notifies all configured household devices with the fault distance

## Dependencies

This automation uses the **Mobile App** integration for push notifications. Each person you want to notify must have the [Home Assistant Companion App](https://companion.home-assistant.io/) installed and their device registered in HA.

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `YOUR VEHICLE NAME` | The vehicle name used in the automation alias and notification messages (e.g. `My Car`). |
| `binary_sensor.your_ecu_status_sensor` | A binary sensor that turns `on` when the vehicle ECU reports a fault code. |
| `sensor.your_fault_distance_sensor` | A sensor reporting the distance driven since the fault was detected. |
| `notify.mobile_app_your_phone` | Notify service for the first household member's phone. Found under **Developer Tools → Services**, search `notify.mobile_app_`. |
| `notify.mobile_app_second_phone` | Notify service for a second household member's phone. Remove this action block entirely if only notifying one person. |

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the YAML below, replacing all placeholders with your own values

```yaml
alias: Notify Fault Code - YOUR VEHICLE NAME
description: Notify when YOUR VEHICLE NAME reports a fault code on each drive
triggers:
  - trigger: state
    entity_id:
      - binary_sensor.your_ecu_status_sensor
    to: "on"
conditions:
  - condition: numeric_state
    entity_id: sensor.your_fault_distance_sensor
    above: 0
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: ⚠️ Fault Code
      message: >
        YOUR VEHICLE NAME logged a fault code
        {{ states('sensor.your_fault_distance_sensor') }} km ago.
      data:
        notification_icon: mdi:car-wrench
        color: "#c0392b"
  - action: notify.mobile_app_second_phone  # Remove this block if only notifying one person
    data:
      title: ⚠️ Fault Code
      message: >
        YOUR VEHICLE NAME logged a fault code
        {{ states('sensor.your_fault_distance_sensor') }} km ago.
      data:
        notification_icon: mdi:car-wrench
        color: "#c0392b"
mode: single
```
