# EV Efficiency and Cost Tracking for Home Assistant

Track your electric vehicle's efficiency metrics (MPGe, mi/kWh) and electricity costs using Home Assistant's Bluelink integration combined with real-time electricity pricing data.

## Overview

This setup allows you to:
- Monitor daily driving efficiency (kWh/mi, mi/kWh, MPGe)
- Calculate daily electricity costs for EV charging
- Compare EV efficiency to ICE vehicle fuel economy
- Track efficiency trends over time and across different temperatures

---

## Prerequisites

### 1. Home Assistant Setup
- Bluelink integration installed and configured
- Your EV must report daily driving statistics

### 2. Electricity Price API
Sign up for a free API key at [EIA API](https://api.eia.gov) to get real-time electricity pricing for your area.

---

## Configuration

### Step 1: Add Electricity Price Sensor

Add this to your `sensor.yaml` file to fetch current electricity rates:

```yaml
# NC Residential Electricity Price via EIA v2 API
- platform: rest
  name: NC Residential Electricity Price Raw
  resource: "https://api.eia.gov/v2/electricity/retail-sales/data"
  method: GET

  params:
    api_key: !secret my_eia_api_key
    data[]: price
    facets[sectorid][]: RES
    facets[stateid][]: NC
    frequency: monthly
    length: 1

  headers:
    Accept: application/json
    User-Agent: HomeAssistant/2024.12

  scan_interval: 3600
  timeout: 30

  json_attributes_path: "$.response.data[0]"
  json_attributes:
    - period
    - price

  value_template: >
    {% if value_json.response is defined and 
          value_json.response.data is defined and 
          value_json.response.data | length > 0 %}
      {{ value_json.response.data[0].price | float }}
    {% else %}
      0
    {% endif %}
```

**Note:** Update the `facets[stateid][]` parameter to match your state.

---

### Step 2: Add Template Sensors

Add these templates to your `templates.yaml` file:

#### Energy Consumption Sensors

```yaml
# ------------------------
# EV Daily Energy Stats
# ------------------------
  - name: "EV Distance Today"
    unique_id: ev_distance_today
    state: "{{ state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'distance') | float(0) | round(1) }}"
    unit_of_measurement: "mi"
    icon: mdi:map-marker-distance

  - name: "EV Energy Consumed Today"
    unique_id: ev_energy_consumed_today
    state: "{{ (state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) / 1000) | round(2) }}"
    unit_of_measurement: "kWh"
    device_class: energy
    icon: mdi:lightning-bolt

  - name: "EV Energy Regenerated Today"
    unique_id: ev_energy_regenerated_today
    state: "{{ (state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'regenerated_energy') | float(0) / 1000) | round(2) }}"
    unit_of_measurement: "kWh"
    device_class: energy
    icon: mdi:battery-charging

  - name: "EV Climate Energy Today"
    unique_id: ev_climate_energy_today
    state: "{{ (state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'climate_consumption') | float(0) / 1000) | round(2) }}"
    unit_of_measurement: "kWh"
    device_class: energy
    icon: mdi:air-conditioner
```

#### Efficiency Metrics

```yaml
# ------------------------
# EV Efficiency Calculations
# ------------------------
  - name: "EV Efficiency Today"
    unique_id: ev_efficiency_today
    state: >
      {% set distance = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'distance') | float(0) %}
      {% set consumed = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) %}
      {% if distance > 0 %}
        {{ ((consumed / 1000) / distance) | round(2) }}
      {% else %}
        0
      {% endif %}
    unit_of_measurement: "kWh/mi"
    icon: mdi:gauge

  - name: "EV Miles per kWh Today"
    unique_id: ev_miles_per_kwh_today
    unit_of_measurement: "mi/kWh"
    device_class: energy
    state: >
      {% set stats = states('sensor.2023_ioniq_6_todays_daily_driving_stats') %}
      {% if stats != 'unknown' and stats != 'unavailable' %}
        {% set distance = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'distance') | float(0) %}
        {% set consumed = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) %}
        {% if consumed > 0 %}
          {{ (distance / (consumed / 1000)) | round(2) }}
        {% else %}
          0
        {% endif %}
      {% else %}
        0
      {% endif %}
    icon: mdi:gauge
  
  - name: "EV MPGe Today"
    unique_id: ev_mpge_today
    unit_of_measurement: "MPGe"
    state: >
      {% set stats = states('sensor.2023_ioniq_6_todays_daily_driving_stats') %}
      {% if stats != 'unknown' and stats != 'unavailable' %}
        {% set distance = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'distance') | float(0) %}
        {% set consumed = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) %}
        {% if consumed > 0 %}
          {{ ((distance / (consumed / 1000)) * 33.7) | round(1) }}
        {% else %}
          0
        {% endif %}
      {% else %}
        0
      {% endif %}
    icon: mdi:gas-station

  - name: "EV Net Efficiency Today"
    unique_id: ev_net_efficiency_today
    unit_of_measurement: "mi/kWh"
    state: >
      {% set stats = states('sensor.2023_ioniq_6_todays_daily_driving_stats') %}
      {% if stats != 'unknown' and stats != 'unavailable' %}
        {% set distance = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'distance') | float(0) %}
        {% set consumed = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) %}
        {% set regen = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'regenerated_energy') | float(0) %}
        {% set net = consumed - regen %}
        {% if net > 0 %}
          {{ (distance / (net / 1000)) | round(2) }}
        {% else %}
          0
        {% endif %}
      {% else %}
        0
      {% endif %}
    icon: mdi:leaf
```

#### Cost Calculations

```yaml
# ------------------------
# EV Driving Cost Calculations
# ------------------------
  - name: "EV Energy Cost Today"
    unique_id: ev_energy_cost_today
    state: >
      {% set consumed = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) %}
      {% set rate = states('sensor.nc_residential_electricity_price') | float(0.1512) %}
      {{ ((consumed / 1000) * rate) | round(2) }}
    unit_of_measurement: "$"
    icon: mdi:currency-usd

  - name: "EV Cost Per Mile Today"
    unique_id: ev_cost_per_mile_today
    state: >
      {% set distance = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'distance') | float(0) %}
      {% set consumed = state_attr('sensor.2023_ioniq_6_todays_daily_driving_stats', 'total_consumed') | float(0) %}
      {% set rate = states('sensor.nc_residential_electricity_price') | float(0.1512) %}
      {% if distance > 0 %}
        {{ (((consumed / 1000) * rate) / distance) | round(3) }}
      {% else %}
        0
      {% endif %}
    unit_of_measurement: "$/mi"
    icon: mdi:cash
```

**Important:** Replace `sensor.2023_ioniq_6_todays_daily_driving_stats` with your actual Bluelink sensor entity name.

---

## Dashboard Cards

### Efficiency Stats Card

This card displays all your efficiency metrics in one place:

```yaml
type: vertical-stack
cards:
  - type: entities
    entities:
      - entity: sensor.2023_ioniq_6_daily_driving_stats
        secondary_info: last-changed
        name: Daily Driving Stats
        
  - type: markdown
    content: >-
      {% for date, data in states['sensor.2023_ioniq_6_daily_driving_stats'].attributes.items() %}
          {% if date is defined and date is string and date | regex_match('^\\d{4}-\\d{2}-\\d{2}$') %}
            {{ date }}
            {% if data is mapping %}
                Total Consumed: {{ data.total_consumed }}
                Engine Consumption: {{ data.engine_consumption }}
                Climate Consumption: {{ data.climate_consumption }}
                Onboard Electronics Consumption: {{ data.onboard_electronics_consumption }}
                Battery Care Consumption: {{ data.battery_care_consumption }}
                Regenerated Energy: {{ data.regenerated_energy }}
                Distance: {{ data.distance }}
            {% endif %}
          {% endif %}
      {% endfor %}
    title: 2023 Ioniq 6 Daily Stats
    
  - type: entities
    title: Today's EV Energy Usage
    entities:
      - entity: sensor.ev_distance_today
      - entity: sensor.ev_energy_consumed_today
      - entity: sensor.ev_energy_regenerated_today
      - entity: sensor.ev_climate_energy_today
      - entity: sensor.ev_efficiency_today
      - entity: sensor.ev_miles_per_kwh_today
      - entity: sensor.ev_mpge_today
      - entity: sensor.ev_net_efficiency_today
```

### Cost Tracking Card

Track your daily EV driving costs and electricity rates:

```yaml
type: entities
title: EV Costs
entities:
  - type: section
    label: EV Related (Ioniq 6)
  - entity: sensor.ev_distance_today
  - entity: sensor.ev_energy_cost_today
    name: Cost of Driving the EV today
  - entity: sensor.ev_cost_per_mile_today
    
  - type: section
    label: Electricity
  - type: weblink
    name: NC Residential Electricity Rates
    url: >-
      https://www.eia.gov/electricity/data/browser/#/topic/7?agg=1,0&geo=00000004&endsec=8&freq=M&start=200101&end=202509&ctype=linechart&ltype=pin&rtype=s&maptype=0&rse=0&pin=
    icon: mdi:chart-areaspline-variant
  - entity: sensor.nc_residential_electricity_price
    icon: mdi:currency-usd
  - type: attribute
    entity: sensor.nc_residential_electricity_price
    attribute: period
    name: Last Updated (YYYY-MM)
    icon: mdi:calendar-multiselect
```

---

## Understanding the Metrics

| Metric | Description |
|--------|-------------|
| **kWh/mi** | Energy consumed per mile driven (lower is better) |
| **mi/kWh** | Miles driven per kilowatt-hour consumed (higher is better) |
| **MPGe** | Miles Per Gallon equivalent - converts kWh to gasoline equivalent (33.7 kWh = 1 gallon) |
| **Net Efficiency** | Miles per kWh accounting for regenerative braking |

---

## Troubleshooting

**Sensors showing 0 or unavailable:**
- Verify your Bluelink integration is working and reporting data
- Check that your EV sensor entity names match those in the templates
- Ensure you've driven the vehicle today for "today" sensors to populate

**Electricity price not updating:**
- Verify your EIA API key is valid and in `secrets.yaml`
- Check that the state code matches your location
- EIA data updates monthly, so prices won't change daily

---

## Customization

To adapt this for your vehicle:
1. Find your Bluelink sensor entity name in Home Assistant
2. Replace all instances of `sensor.2023_ioniq_6_todays_daily_driving_stats`
3. Update the electricity price sensor state code to your region
4. Adjust the default electricity rate fallback value (currently 0.1512)

---

## Contributing

Feel free to modify these templates for your needs. Consider tracking additional metrics like:
- Average efficiency over time (use statistics sensors)
- Efficiency by temperature
- Cost comparisons with different charging rates (home vs. public charging)
