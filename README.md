# Hyundai EV Dashboard for Home Assistant

A comprehensive guide to setting up a beautiful vehicle status dashboard for your Hyundai EV using Home Assistant.

<img width="506" height="713" alt="image" src="https://github.com/user-attachments/assets/654c340b-1f35-4158-8f87-89de30c0c361" />

## Overview

This guide combines two powerful tools to create a feature-rich dashboard for monitoring and controlling your Hyundai electric vehicle:

- **[Hyundai Kia Connect](https://github.com/Hyundai-Kia-Connect/kia_uvo)** - Integration for connecting your Hyundai/Kia vehicle to Home Assistant
- **[Vehicle Status Card](https://github.com/ngocjohn/vehicle-status-card)** - Beautiful, customizable Lovelace card for displaying vehicle information

## Credits

This project builds upon the excellent work of:
- [Hyundai-Kia-Connect](https://github.com/Hyundai-Kia-Connect/kia_uvo) by the Hyundai-Kia-Connect team
- [Vehicle Status Card](https://github.com/ngocjohn/vehicle-status-card) by ngocjohn

## Prerequisites

- Home Assistant installed and running
- HACS (Home Assistant Community Store) installed
- A compatible Hyundai or Kia vehicle with Bluelink/UVO services
- Active Bluelink/UVO subscription
- For EV Efficiency Stats and Electricity Costs see: evstats.md

## Installation

### Step 1: Install the Hyundai Kia Connect Integration

1. Open HACS in your Home Assistant instance
2. Click on "Integrations"
3. Click the "+" button in the bottom right
4. Search for "Hyundai Kia Connect"
5. Click "Download"
6. Restart Home Assistant

#### Configuration

1. Go to **Settings** → **Devices & Services**
2. Click **+ Add Integration**
3. Search for "Hyundai Kia Connect"
4. Enter your Bluelink/UVO credentials:
   - Username (email)
   - Password
   - Region (e.g., USA, Canada, Europe)
   - Brand (Hyundai or Kia)
5. Click **Submit**

Your vehicle should now appear as a device with multiple entities.

### Step 2: Install the Vehicle Status Card

1. Open HACS in your Home Assistant instance
2. Click on "Frontend"
3. Click the "+" button in the bottom right
4. Search for "Vehicle Status Card"
5. Click "Download"
6. Refresh your browser cache (Ctrl+F5 or Cmd+Shift+R)

## Configuration

### Basic Setup

1. Create a new Lovelace dashboard or edit an existing one
2. Add a new card
3. Select "Custom: Vehicle Status Card" from the card picker
4. Configure the card using the visual editor or YAML mode

### Example Configuration for Hyundai Ioniq 6

Below is a complete configuration example for a Hyundai Ioniq 6. Adjust the entity names to match your vehicle:

```yaml
type: custom:vehicle-status-card
name: Ioniq 6
images:
  - image: /local/images/ioniq6_clearbckgnd3.png
layout_config:
  button_grid:
    rows: 2
    columns: 2
    swipe: true
  images_swipe: {}
  section_order:
    - indicators
    - range_info
    - images
    - buttons
  range_info_config:
    layout: row
  hide_card_name: true
mini_map:
  device_tracker: device_tracker.2023_ioniq_6_location
  enable_popup: true
indicator_rows:
  - row_items:
      - entity: binary_sensor.2023_ioniq_6_locked
        type: entity
        show_name: true
        show_state: true
        show_icon: true
        name: Status
      - entity: binary_sensor.2023_ioniq_6_engine
        type: entity
        show_name: true
        show_state: true
        show_icon: true
        name: "Engine "
      - type: group
        show_name: true
        show_icon: true
        name: Doors
        icon: mdi:car-door
        items:
          - entity: binary_sensor.2023_ioniq_6_front_left_door
            name: Front Left
          - entity: binary_sensor.2023_ioniq_6_front_right_door
            name: Front Right
          - entity: binary_sensor.2023_ioniq_6_back_left_door
            name: Back Left
          - entity: binary_sensor.2023_ioniq_6_back_right_door
            name: Back Right
          - entity: binary_sensor.2023_ioniq_6_hood
            name: Hood
          - entity: binary_sensor.2023_ioniq_6_trunk
            name: Trunk
      - type: group
        show_name: true
        show_icon: true
        name: Climate
        icon: mdi:air-conditioner
        items:
          - entity: binary_sensor.2023_ioniq_6_air_conditioner
            name: A/C
          - entity: binary_sensor.2023_ioniq_6_defrost
            name: Defrost
          - entity: binary_sensor.2023_ioniq_6_back_window_heater
            name: Rear Defrost
          - entity: binary_sensor.2023_ioniq_6_steering_wheel_heater
            name: Steering Heat
          - entity: sensor.2023_ioniq_6_set_temperature
            name: Set Temp
      - entity: binary_sensor.2023_ioniq_6_ev_battery_plug
        type: entity
        show_name: true
        name: EV Battery Plug
range_info:
  - energy_level:
      entity: sensor.2023_ioniq_6_ev_battery_level
      tap_action:
        action: more-info
    range_level:
      entity: sensor.2023_ioniq_6_ev_range
      value_position: outside
    progress_color: "#1e88e5"
    color_blocks: true
    color_thresholds:
      - value: 0
        color: "#cde8b5"
      - value: 24
        color: "#b1dd8b"
      - value: 49
        color: "#96d35f"
      - value: 74
        color: "#76bb40"
      - value: 100
        color: "#669d34"
button_cards:
  - entity: sensor.2023_ioniq_6_odometer
    show_icon: true
    show_primary: true
    show_secondary: true
    layout: horizontal
  - entity: sensor.2023_ioniq_6_ev_range
    show_icon: true
    show_primary: true
    show_secondary: true
    layout: horizontal
```

### Customization Tips

#### Finding Your Entity Names

After installing the integration, find your vehicle's entities:

1. Go to **Settings** → **Devices & Services**
2. Find your vehicle under the Hyundai Kia Connect integration
3. Click on the device to see all available entities
4. Copy the entity IDs you want to display

#### Adding a Vehicle Image

1. Create a folder at `/config/www/images/` in your Home Assistant installation
2. Add a PNG image of your vehicle (transparent background works best)
3. Reference it in the configuration: `/local/images/your-vehicle.png`

#### Color Customization

The `color_thresholds` section allows you to customize the battery level colors:

```yaml
color_thresholds:
  - value: 0
    color: "#cde8b5"  # Light green for low battery
  - value: 25
    color: "#b1dd8b"
  - value: 50
    color: "#96d35f"
  - value: 75
    color: "#76bb40"
  - value: 100
    color: "#669d34"  # Dark green for full battery
```

#### Available Entities (Examples)

Common entities you might want to display:

**Status Sensors:**
- `binary_sensor.VEHICLE_locked`
- `binary_sensor.VEHICLE_engine`
- `sensor.VEHICLE_odometer`
- `sensor.VEHICLE_last_updated_at`

**EV Specific:**
- `sensor.VEHICLE_ev_battery_level`
- `sensor.VEHICLE_ev_range`
- `binary_sensor.VEHICLE_ev_battery_plug`
- `sensor.VEHICLE_ev_charging_power`
- `sensor.VEHICLE_estimated_charge_duration`

**Doors & Windows:**
- `binary_sensor.VEHICLE_front_left_door`
- `binary_sensor.VEHICLE_front_right_door`
- `binary_sensor.VEHICLE_back_left_door`
- `binary_sensor.VEHICLE_back_right_door`
- `binary_sensor.VEHICLE_hood`
- `binary_sensor.VEHICLE_trunk`

**Climate:**
- `binary_sensor.VEHICLE_air_conditioner`
- `binary_sensor.VEHICLE_defrost`
- `binary_sensor.VEHICLE_back_window_heater`
- `sensor.VEHICLE_set_temperature`

**Location:**
- `device_tracker.VEHICLE_location`

> **Note:** Replace `VEHICLE` with your actual vehicle's entity prefix (e.g., `2023_ioniq_6`)

## Troubleshooting

### Integration Not Working

1. Verify your Bluelink/UVO credentials are correct
2. Ensure your vehicle subscription is active
3. Check the Home Assistant logs for errors
4. Try reloading the integration

### Card Not Displaying

1. Clear your browser cache
2. Verify the card is installed in HACS
3. Check that entity IDs match your vehicle's entities
4. Look for JavaScript errors in browser console (F12)

### Entities Not Updating

- The integration typically polls every 30 minutes by default
- You can manually refresh using the integration's service calls
- Some data may only update when the vehicle is used

## Advanced Features

### Automations

Create automations based on vehicle status:

```yaml
automation:
  - alias: "Notify when EV charging complete"
    trigger:
      - platform: numeric_state
        entity_id: sensor.2023_ioniq_6_ev_battery_level
        above: 95
    condition:
      - condition: state
        entity_id: binary_sensor.2023_ioniq_6_ev_battery_plug
        state: "on"
    action:
      - service: notify.mobile_app
        data:
          message: "Your Ioniq 6 is fully charged!"
```

### Services

The integration provides services for remote control:

- `kia_uvo.update` - Force update vehicle status
- `kia_uvo.start_climate` - Start climate control
- `kia_uvo.stop_climate` - Stop climate control
- `kia_uvo.lock` - Lock vehicle
- `kia_uvo.unlock` - Unlock vehicle

## Support

- **Hyundai Kia Connect Issues**: [GitHub Issues](https://github.com/Hyundai-Kia-Connect/kia_uvo/issues)
- **Vehicle Status Card Issues**: [GitHub Issues](https://github.com/ngocjohn/vehicle-status-card/issues)
- **Home Assistant Community**: [Community Forum](https://community.home-assistant.io/)

## Contributing

Contributions are welcome! If you have improvements to this guide:

1. Fork the repository
2. Make your changes
3. Submit a pull request

## License

This guide is provided as-is for the Home Assistant community. The referenced projects maintain their own licenses:
- Hyundai Kia Connect: Check their repository for license details
- Vehicle Status Card: Check their repository for license details

---

**Disclaimer:** This is not an official Hyundai or Kia project. Use at your own risk. Always ensure your vehicle is secure and never rely solely on remote monitoring for safety-critical functions.
