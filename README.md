# TeslaMate Discovery
If you're a fan of the very excellent [TeslaMate][tm] and use it with [Home Assistant][ha], then this project is for you!

The TeslaMate + Home Assistant [integration documentation][tmha] shows a very long, manual configuration of each of the entities that TeslaMate sends messages for.  While this works quite well (I used it for years) it has some shortcomings, the biggest of which is that they aren't all collected under a single [Home Assistant Device][had].  Unfortunately, devices cannot be created using purely manual end-user configuration, they can only be added with [MQTT Discovery][hamd] or a dedicated integration.

This application replaces most, if not all, of the documented TeslaMate + Home Assistant integration.  In a single command, it synthesizes and publishes the MQTT Discovery configuration for commonly used entities converted to desired units and a `device_tracker` with precise coordinates that can also determine if the car is home (any geofence with "Home" in it).

[ha]: https://www.home-assistant.io
[had]: https://developers.home-assistant.io/docs/device_registry_index/
[hamd]: https://www.home-assistant.io/docs/mqtt/discovery/
[tm]: https://github.com/adriankumpf/teslamate
[tmha]: https://docs.teslamate.org/docs/integrations/home_assistant

# Pre-requisites
The application assumes that you have a healthy TeslaMate installation sending messages to a [Mosquitto MQTT broker][mos] and that Home Assistant can see those messages.  The only other requirement is to have an account that can read messages from the `/teslamate/#` topic tree and send messages to the `/homeassistant/#` topic tree.  This is often done by configuring `/share/mosquitto/accesscontrollist` as in the following example:

```plain
user homeassistant
topic readwrite homeassistant/#
topic read teslamate/#

user teslamate
topic write teslamate/#

user teslamate-discovery
topic write homeassistant/#
topic read teslamate/#
```

[mos]: https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md

# Installation
To install the application, navigate to the [Releases][r] page for the project and download the appropriate Asset for the platform you'll be running on (e.g. `teslamate-discovery_2.0.2_Darwin_all.tar.gz` for macOS).  Unzip the package and within it you'll find the `teslamate-discovery` binary that you'll run.

[r]: https://github.com/nebhale/teslamate-discovery/releases

## Kubernetes
A ready-to-use `CronJob` manifest is provided in [`k8s/deployment.yaml`](./k8s/deployment.yaml).  Update the `Secret` with your MQTT credentials and adjust the environment variables before applying:

```sh
kubectl apply -f k8s/deployment.yaml
```

The CronJob runs hourly by default.  Trigger it manually with:

```sh
kubectl create job teslamate-discovery-manual --from=cronjob/teslamate-discovery -n <namespace>
```

# Usage
Common usage might look like:

```plain
$ teslamate-discovery \
    --mqtt-host <HOST> \
    --mqtt-username <USERNAME> \
    --mqtt-password <PASSWORD>
```

## Usage Options
```plain
Usage:
  teslamate-discovery [flags]

Flags:
      --ha-discovery-prefix string   home assistant discovery message prefix (default "homeassistant")
      --help                         help for teslamate-discovery
  -h, --mqtt-host string             mqtt broker host (default "127.0.0.1")
  -P, --mqtt-password string         mqtt broker password
  -p, --mqtt-port int                mqtt broker port (default 8883)
  -s, --mqtt-scheme string           mqtt broker scheme (default "ssl")
  -u, --mqtt-username string         mqtt broker username
      --range-type string            range type ["estimated", "ideal", "rated"] (default "rated")
      --tm-prefix string             teslamate message prefix (default "teslamate")
      --units-distance string        distance units ["imperial", "metric"] (default "imperial")
      --units-pressure string        pressure units ["imperial", "metric"] (default "imperial")
  -v, --version                      version for teslamate-discovery
```

Environment variable equivalents: `MQTT_SCHEME`, `MQTT_HOST`, `MQTT_PORT`, `MQTT_USERNAME`, `MQTT_PASSWORD`, `HA_DISCOVERY_PREFIX`, `TM_PREFIX`, `UNITS_DISTANCE`, `UNITS_PRESSURE`, `UNITS_RANGE_TYPE`.

# Discovered Entities

All entities are grouped under a single Home Assistant device per vehicle. Entity IDs follow the pattern `<domain>.<vehicle_name>_<entity_name>` (e.g. `sensor.bluey_battery`).

## Charging

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `sensor.<v>_charge_current_request` | `charge_current_request` | Current the car has requested from the charger | A |
| `sensor.<v>_charge_current_request_max` | `charge_current_request_max` | Maximum current the car can request | A |
| `sensor.<v>_energy_added` | `charge_energy_added` | Energy added in the current or last charging session | kWh |
| `sensor.<v>_limit` | `charge_limit_soc` | Configured charge limit | % |
| `sensor.<v>_charger_current` | `charger_actual_current` | Actual current being delivered by the charger | A |
| `binary_sensor.<v>_charging` | `state` | Whether the car is actively charging | on/off |
| `binary_sensor.<v>_plug` | `plugged_in` | Whether the charge cable is connected | on/off |
| `sensor.<v>_charger_phases` | `charger_phases` | Number of AC phases used (only published while charging) | — |
| `sensor.<v>_charger_power` | `charger_power` | Power being delivered by the charger | kW |
| `sensor.<v>_charger_voltage` | `charger_voltage` | Voltage at the charger | V |
| `sensor.<v>_scheduled_start_time` | `scheduled_charging_start_time` | When scheduled charging is set to begin (only published when set) | timestamp |
| `sensor.<v>_time_to_charged` | `time_to_full_charge` | Estimated time until fully charged | h |
| `sensor.<v>_charging_state` | `charging_state` | Charger state: `Disconnected`, `Charging`, `Stopped`, `Complete`, `NoPower` | — |

## Climate

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `sensor.<v>_inside_temp` | `inside_temp` | Cabin temperature | °C |
| `binary_sensor.<v>_climate` | `is_climate_on` | Whether climate control is running | on/off |
| `binary_sensor.<v>_preconditioning` | `is_preconditioning` | Whether the car is preconditioning | on/off |
| `sensor.<v>_outside_temp` | `outside_temp` | Outside air temperature | °C |
| `sensor.<v>_climate_keeper_mode` | `climate_keeper_mode` | Active climate keeper mode: `off`, `dog`, `camp`, `keep` | — |

## Location

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `sensor.<v>_active_route_destination` | `active_route_destination` | Destination name of the active navigation route | — |
| `sensor.<v>_active_route_latitude` | `active_route_latitude` | Destination latitude of the active route | ° |
| `sensor.<v>_active_route_longitude` | `active_route_longitude` | Destination longitude of the active route | ° |
| `sensor.<v>_elevation` | `elevation` | Current altitude | m or ft (units-distance) |
| `sensor.<v>_geofence` | `geofence` | Name of the active TeslaMate geofence, if any | — |
| `sensor.<v>_heading` | `heading` | Compass bearing the car is pointing | ° |
| `sensor.<v>_latitude` | `latitude` | GPS latitude | ° |
| `sensor.<v>_longitude` | `longitude` | GPS longitude | ° |
| `device_tracker.<v>` | `location` | GPS device tracker; reports `home` when a geofence containing "Home" is active | — |
| `sensor.<v>_power` | `power` | Instantaneous power draw (negative when regenerating) | kW |
| `sensor.<v>_speed` | `speed` | Current speed | km/h or mph (units-distance) |

## State & Security

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `sensor.<v>_display_name` | `display_name` | Vehicle name as set in the Tesla app | — |
| `sensor.<v>_exterior_color` | `exterior_color` | Paint colour code | — |
| `sensor.<v>_spoiler_type` | `spoiler_type` | Spoiler type | — |
| `sensor.<v>_wheel_type` | `wheel_type` | Wheel/tyre package code (e.g. `Pinwheel18CapKit`, `Crossflow19`) | — |
| `sensor.<v>_state` | `state` | Car state: `online`, `offline`, `asleep`, `charging`, `driving`, etc. | — |
| `sensor.<v>_last_seen` | `since` | Timestamp of the last state change | timestamp |
| `sensor.<v>_shift_state` | `shift_state` | Gear selector position: `P`, `D`, `R`, `N` | — |
| `binary_sensor.<v>_locked` | `locked` | Whether the car doors are locked | on/off |
| `binary_sensor.<v>_sentry_mode` | `sentry_mode` | Whether Sentry Mode is active | on/off |
| `binary_sensor.<v>_occupied` | `is_user_present` | Whether a driver is detected in the car | on/off |
| `binary_sensor.<v>_health` | `healthy` | Whether the TeslaMate logger is healthy (problem class: on = unhealthy) | on/off |
| `binary_sensor.<v>_update` | `update_available` | Whether a software update is available | on/off |
| `sensor.<v>_version` | `version` | Installed firmware version | — |
| `sensor.<v>_center_display` | `center_display_state` | Center screen state: Off, Standby, Charging, On, Big Charging, Ready to Unlock, Sentry Mode, Dog Mode, Media | — |

## Doors, Windows & Openings

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `binary_sensor.<v>_charge_port` | `charge_port_door_open` | Whether the charge port door is open | on/off |
| `binary_sensor.<v>_doors` | `doors_open` | Whether any door is open | on/off |
| `binary_sensor.<v>_door_driver_front` | `driver_front_door_open` | Driver front door | on/off |
| `binary_sensor.<v>_door_driver_rear` | `driver_rear_door_open` | Driver rear door | on/off |
| `binary_sensor.<v>_door_passenger_front` | `passenger_front_door_open` | Passenger front door | on/off |
| `binary_sensor.<v>_door_passenger_rear` | `passenger_rear_door_open` | Passenger rear door | on/off |
| `binary_sensor.<v>_frunk` | `frunk_open` | Front trunk (frunk) | on/off |
| `binary_sensor.<v>_trunk` | `trunk_open` | Rear trunk | on/off |
| `binary_sensor.<v>_windows` | `windows_open` | Whether any window is open | on/off |

## Battery & Range

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `sensor.<v>_battery` | `battery_level` | State of charge | % |
| `sensor.<v>_usable_battery` | `usable_battery_level` | Usable state of charge (excludes buffer) | % |
| `sensor.<v>_range` | `rated_battery_range_km` / `ideal_battery_range_km` / `est_battery_range_km` | Projected range; which topic is used depends on `--range-type` | km or mi (units-distance) |
| `sensor.<v>_odometer` | `odometer` | Total distance driven | km or mi (units-distance) |

## Tyre Pressure

| Entity | MQTT topic | Description | Unit |
|--------|-----------|-------------|------|
| `sensor.<v>_tire_pressure_front_left` | `tpms_pressure_fl` | Front-left tyre pressure | bar or psi (units-pressure) |
| `sensor.<v>_tire_pressure_front_right` | `tpms_pressure_fr` | Front-right tyre pressure | bar or psi (units-pressure) |
| `sensor.<v>_tire_pressure_rear_left` | `tpms_pressure_rl` | Rear-left tyre pressure | bar or psi (units-pressure) |
| `sensor.<v>_tire_pressure_rear_right` | `tpms_pressure_rr` | Rear-right tyre pressure | bar or psi (units-pressure) |
| `binary_sensor.<v>_tire_soft_front_left` | `tpms_soft_warning_fl` | Front-left soft tyre warning | on/off |
| `binary_sensor.<v>_tire_soft_front_right` | `tpms_soft_warning_fr` | Front-right soft tyre warning | on/off |
| `binary_sensor.<v>_tire_soft_rear_left` | `tpms_soft_warning_rl` | Rear-left soft tyre warning | on/off |
| `binary_sensor.<v>_tire_soft_rear_right` | `tpms_soft_warning_rr` | Rear-right soft tyre warning | on/off |

## Topics Published by TeslaMate but Not Discovered

The following MQTT topics are published by TeslaMate but do not currently have a corresponding discovery entity:

| MQTT topic | Description |
|-----------|-------------|
| `active_route` | JSON blob with destination, energy remaining, ETA, and miles to arrival for an active navigation route |

## License
Apache License v2.0: see [LICENSE](./LICENSE) for details.
