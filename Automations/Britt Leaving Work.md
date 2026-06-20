# Britt Leaving Work (And Bonus Smart Kettle)

A Home Assistant automation that gives a heads up when Britt leaves work, with an ETA based on live travel time, then automatically arms a Shelly-fitted kettle to boil by the time she gets home.

## How It Works

The automation triggers when Britt's `person` entity leaves a defined work zone. It immediately sends a notification with an ETA, calculated by adding a travel time sensor's current value (in minutes) to the current time. At the same time, it makes sure the kettle switch is off and ready to be armed.

From there, it waits (with a 30 minute timeout as a fallback) for Britt's `person` entity to report a "Nearly Home" zone state — a custom zone set a few minutes' drive from home, rather than the default "home" zone, so there's a window to act before she actually walks in. Once that triggers, a second notification confirms the kettle is on, and the kettle switch is turned on to start boiling.

## Dependencies

- A `zone` entity for the relevant workplace, and a custom "Nearly Home" zone (e.g. a few minutes' drive away) — both used as `person` entity zone triggers
- A travel time sensor reporting estimated minutes to home (in this case via the Waze Travel Time integration)
- The **Mobile App** integration for push notifications
- A smart switch fitted to the kettle (in this case a Shelly), exposed as a `switch` entity

## Configuration

Replace the following placeholders in the YAML before use:

| Placeholder | Description |
|---|---|
| `zone.work` | The work zone to trigger on leaving. |
| `notify.mobile_app_your_phone` | Notify service for the phone receiving the alerts. Found under **Developer Tools → Services**, search `notify.mobile_app_`. |
| `sensor.travel_time_to_home` | Your travel time sensor, reporting estimated minutes to home. |
| `switch.kettle` | The smart switch fitted to your kettle. |
| `Nearly Home` | The name of your custom "almost home" zone, set up under **Settings → Areas, Labels & Zones → Zones**. |

> **Note:** The kettle is switched off immediately when work is left (readying it to be armed) and switched on once the "Nearly Home" zone is reached, rather than at a fixed distance/time from home. The wait for "Nearly Home" has a 30 minute timeout, after which the automation continues anyway and arms the kettle regardless — so it won't wait indefinitely if the zone trigger doesn't fire for some reason.

## Usage

1. In Home Assistant, go to **Settings → Automations → Create Automation → Edit in YAML**
2. Paste the YAML below, replacing all placeholders with your own values

```yaml
alias: Britt Leaving Work
description: ""
triggers:
  - trigger: zone
    entity_id: person.britt
    zone: zone.work
    event: leave
    alias: When Britt leaves work zone
conditions: []
actions:
  - action: notify.mobile_app_your_phone
    metadata: {}
    data:
      message: >
        Britt has left work! She should be home around {{ (now() +
        timedelta(seconds=states('sensor.travel_time_to_home') | int *
        60)) | as_timestamp | timestamp_custom('%-I:%M %p') }}. Go fill the
        kettle and arm it.
      data:
        color: "#008080"
        visibility: public
        notification_icon: mdi:briefcase-outline
    alias: Notify Britt has left work
  - action: switch.turn_off
    metadata: {}
    target:
      entity_id: switch.kettle
    data: {}
    alias: Turn off kettle ready
  - wait_for_trigger:
      - trigger: state
        entity_id:
          - person.britt
        to:
          - Nearly Home
    continue_on_timeout: true
    timeout:
      hours: 0
      minutes: 30
      seconds: 0
      milliseconds: 0
  - action: notify.mobile_app_your_phone
    metadata: {}
    data:
      message: Britt's about to get home - the kettle is on.
      data:
        color: "#008080"
        visibility: public
        notification_icon: mdi:kettle
      title: Britt's nearly home!
    alias: Notify Britt is nearly home
  - action: switch.turn_on
    metadata: {}
    target:
      entity_id: switch.kettle
    data: {}
    alias: Turn on kettle
mode: single
```
