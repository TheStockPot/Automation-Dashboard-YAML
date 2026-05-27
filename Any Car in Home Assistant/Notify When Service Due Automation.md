# Notify When Service Due

A Home Assistant automation that sends a push notification to household mobile devices when a vehicle service is approaching, with escalating alerts at 14, 7, and 1 day thresholds.

## Required Helpers & Template Sensors

This automation depends on two sensors that need to be set up manually before use.

### 1. KM Driven in Last 30 Days — Statistics Helper

This tracks how many kilometres your vehicle has travelled over the last 30 days, used to calculate an average daily distance.

1. Go to **Settings → Devices & Services → Helpers → Create Helper**
2. Choose **Statistics**
3. Configure it as follows:

| Field | Value |
|---|---|
| Name | `KM Change 30 Days` (or similar) |
| Entity | Your odometer sensor |
| Characteristic | `change` |
| Sampling size | Leave at default |
| Max age | `30 days` |

4. Note the entity ID it creates (e.g. `sensor.your_km_change_30_days`) — you'll need it in the template sensor below.

---

### 2. Days to Next Service — Template Sensor

This calculates the estimated number of days until service is due, based on your average daily km over the last 30 days.

1. Go to **Settings → Devices & Services → Helpers → Create Helper**
2. Choose **Template**
3. Choose **Template a sensor**
4. Paste the following into the **State template** field:

```jinja
{% set daily_km = states('sensor.your_km_change_30_days') | float(0) / 30 %}
{% set remaining = states('sensor.your_km_to_service_sensor') | float(0) %}
{% if daily_km > 0 %}
  {{ (remaining / daily_km) | round(0) | int }}
{% else %}
  0
{% endif %}
```

5. Set the **Unit of measurement** to `days`
6. Note the entity ID it creates (e.g. `sensor.your_days_to_service_sensor`) and use it in the automation below.

---

## Dependencies

This automation uses the **Mobile App** integration for push notifications. Each person you want to notify must have the [Home Assistant Companion App](https://companion.home-assistant.io/) installed and their device registered in HA.

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `sensor.your_days_to_service_sensor` | Sensor reporting the number of days until the next service is due. |
| `sensor.your_km_to_service_sensor` | Sensor reporting km remaining until next service. |
| `notify.mobile_app_your_phone` | Notify service for the first household member's phone. Found under **Developer Tools → Services**, search `notify.mobile_app_`. |
| `notify.mobile_app_second_phone` | Notify service for a second household member's phone. Remove this action block entirely if only notifying one person. |
| `YOUR VEHICLE NAME` | The vehicle name used in notification titles and messages (e.g. `My Car`). |

> **Note:** The three trigger thresholds are set to `15`, `8`, and `1` (i.e. alerts fire when days drop below 15, 8, and 1). Adjust these to suit your preferences.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the YAML below, replacing all placeholders with your own values

```yaml
alias: Notify When Service Due - YOUR VEHICLE NAME
description: Notify when service is due within 14 days
triggers:
  - trigger: numeric_state
    entity_id:
      - sensor.your_days_to_service_sensor
    below: 15
  - trigger: numeric_state
    entity_id:
      - sensor.your_days_to_service_sensor
    below: 8
  - trigger: numeric_state
    entity_id:
      - sensor.your_days_to_service_sensor
    below: 1
conditions: []
actions:
  - variables:
      days_remaining: "{{ states('sensor.your_days_to_service_sensor') | int(0) }}"
      message: |
        {% if days_remaining <= 0 %}
          ⚠️ YOUR VEHICLE NAME's service is due today!
          ({{ states('sensor.your_km_to_service_sensor') | round(0) }} km on the clock)
        {% elif days_remaining <= 7 %}
          YOUR VEHICLE NAME's service is due in {{ days_remaining }} days
          ({{ states('sensor.your_km_to_service_sensor') | round(0) }} km remaining).
        {% else %}
          YOUR VEHICLE NAME's service is coming up in {{ days_remaining }} days
          ({{ states('sensor.your_km_to_service_sensor') | round(0) }} km remaining).
        {% endif %}
      title: |
        {% if days_remaining <= 0 %}
          🔴 YOUR VEHICLE NAME Service Overdue
        {% elif days_remaining <= 7 %}
          🟠 YOUR VEHICLE NAME Service Due Soon
        {% else %}
          🟡 YOUR VEHICLE NAME Service Reminder
        {% endif %}
  - action: notify.mobile_app_your_phone
    data:
      title: "{{ title }}"
      message: "{{ message }}"
      data:
        notification_icon: mdi:wrench
        color: "#6dbf8a"
  - action: notify.mobile_app_second_phone  # Remove this block if only notifying one person
    data:
      title: "{{ title }}"
      message: "{{ message }}"
      data:
        notification_icon: mdi:wrench
        color: "#6dbf8a"
mode: single
```
