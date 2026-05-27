# Vehicle Dashboard Card

A Home Assistant Lovelace card displaying a vehicle summary panel with fuel level, range, service interval, and fault code alerting.

![Card preview showing vehicle name, image, fuel bar, service bar, range, and next service stats](Alfred%20Card.png)

## Dependencies

Install all of the following via [HACS](https://hacs.xyz) before using this card:

| Custom Card | Repository |
|---|---|
| `vertical-stack-in-card` | https://github.com/ofekashery/vertical-stack-in-card |
| `mushroom` (title + template cards) | https://github.com/piitaya/lovelace-mushroom |
| `bar-card` | https://github.com/custom-cards/bar-card |
| `card-mod` | https://github.com/thomasloven/lovelace-card-mod |

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `VEHICLE_NAME` | Display name for your vehicle (e.g. `My Car`) |
| `VEHICLE_DESCRIPTION` | Subtitle / descriptor (e.g. `2020 Toyota Corolla`) |
| `YOUR_IMAGE_MEDIA_ID` | UUID of your vehicle image uploaded via **Settings → Media**. Replace in both `media_content_id` and the `thumbnail` path. |
| `sensor.your_fault_distance_sensor` | Sensor reporting distance driven since a fault was detected. Banner shows when value > 0. |
| `sensor.your_fuel_level_sensor` | Sensor reporting fuel tank level (typically 0–100). Also used for the last-updated timestamp. |
| `sensor.your_km_to_service_sensor` | Sensor reporting km remaining until next service. |
| `sensor.your_remaining_range_sensor` | Sensor reporting estimated remaining driving range in km. |

> **Note:** Update the `max` value on the service `bar-card` to match your vehicle's service interval. The default is `12000` km.

## Usage

1. Install all dependencies via HACS
2. In Home Assistant, go to your dashboard → **Edit** → **Add Card** → **Manual**
3. Paste the YAML below, replacing all placeholders with your own values

```yaml
type: custom:vertical-stack-in-card
cards:
  - type: custom:mushroom-title-card
    title: VEHICLE_NAME
    subtitle: VEHICLE_DESCRIPTION
    alignment: center
    card_mod:
      style: |
        ha-card {
          padding-bottom: 0 !important;
        }
        .title {
          font-size: 2em !important;
        }

  - type: picture
    image:
      media_content_id: media-source://image_upload/YOUR_IMAGE_MEDIA_ID
      media_content_type: image/png
      metadata:
        title: vehicle.png
        thumbnail: /api/image/serve/YOUR_IMAGE_MEDIA_ID/256x256
        media_class: image
        navigateIds:
          - {}
          - media_content_type: app
            media_content_id: media-source://image_upload
    card_mod:
      style: |
        ha-card {
          padding: 10px !important;
          box-sizing: border-box;
          border: none !important;
          box-shadow: none !important;
          border-radius: 0 !important;
        }

  - type: custom:mushroom-template-card
    primary: ⚠️ Fault Code Reported
    secondary: >
      {{ states('sensor.your_fault_distance_sensor') | int(0) }} km since fault
      detected
    icon: ""
    layout: vertical
    tap_action:
      action: none
    card_mod:
      style: |
        ha-card {
          background: {% if states('sensor.your_fault_distance_sensor') | int(0) > 0 %}
            #c0392b
          {% else %}
            transparent
          {% endif %} !important;
          border-radius: 12px !important;
          border: none !important;
          box-shadow: none !important;
          margin: 0 10px 16px 10px !important;
          {% if states('sensor.your_fault_distance_sensor') | int(0) == 0 %}
          display: none !important;
          {% endif %}
        }
        .primary {
          color: white !important;
          font-weight: bold !important;
          font-size: 1.1em !important;
        }
        .secondary {
          color: rgba(255,255,255,0.85) !important;
        }

  - type: custom:bar-card
    entities:
      - entity: sensor.your_fuel_level_sensor
        icon: mdi:fuel
        color: "#4d9db3"
    positions:
      name: "off"
      icon: inside
      value: "off"
    align: center
    height: 40
    card_mod:
      style: |
        ha-card {
          margin-bottom: -12px !important;
        }
        bar-card-row {
          border-radius: 15px !important;
          overflow: hidden !important;
        }
        bar-card-background {
          border-radius: 15px !important;
        }
        bar-card-currentbar {
          border-radius: 15px !important;
        }

  - type: custom:bar-card
    entities:
      - entity: sensor.your_km_to_service_sensor
        icon: mdi:wrench
        color: "#6dbf8a"
    max: 12000  # Set to your vehicle's full service interval in km
    positions:
      name: "off"
      icon: inside
      value: "off"
    align: center
    height: 40
    card_mod:
      style: |
        bar-card-row {
          border-radius: 15px !important;
          overflow: hidden !important;
        }
        bar-card-background {
          border-radius: 15px !important;
        }
        bar-card-currentbar {
          border-radius: 15px !important;
        }

  - square: false
    type: grid
    columns: 2
    cards:
      - type: custom:mushroom-template-card
        primary: Range
        secondary: |
          {{ states('sensor.your_remaining_range_sensor') | float(0) | round(0) }} km
        icon: mdi:gas-station
        vertical: true
        features_position: bottom
        card_mod:
          style: |
            ha-card {
              border: none !important;
              box-shadow: none !important;
            }
      - type: custom:mushroom-template-card
        primary: Next Service
        secondary: >
          in {{ states('sensor.your_km_to_service_sensor') | float(0) | round(0)
          }} km
        icon: mdi:calendar-clock
        layout: vertical
        fill_container: true
        card_mod:
          style: |
            ha-card {
              border: none !important;
              box-shadow: none !important;
            }

  - type: custom:mushroom-template-card
    primary: ""
    secondary: >-
      Last updated: {{ states.sensor.your_fuel_level_sensor.last_updated |
      as_local | relative_time }} ago
    icon: ""
    layout: vertical
    card_mod:
      style: |
        ha-card {
          --card-secondary-font-size: 11px;
          opacity: 0.5;
        }
```
