# How I'm calculating EV Efficiency and Electricity Costs
By taking the daily statistics from the Bluelink integration to create some template sensors.  Then combined with a rest api sensor that pulls the current costs of electricity for my area, I can do some efficiency and cost calculations.

## Sensor.yaml entry
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
    {% if value_json is defined and value_json.price is defined %}
      {{ value_json.price | float }}
    {% else %}
      0
    {% endif %}
```
## Templates.yaml entries
```yaml

# ------------------------
# EV Daily Stats - Energy
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

# ------------------------
# EV Driving Costs Calculations
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
## Dashboard Cards
### EV Efficiency Stats
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
      {% for date, data in
      states['sensor.2023_ioniq_6_daily_driving_stats'].attributes.items() %}
          {% if date is defined and date is string and date | regex_match('^\\d{4}-\\d{2}-\\d{2}$') %}
            {{ date }}
            {% if data is mapping %}
                Total Consumed: {{ data.total_consumed }}
                Engine Consumption: {{ data.engine_consumption }}
                Climate Consumption: {{ data.climate_consumption }}
                Onboard Electronics Consumption: {{ data.onboard_electronics_consumption }}
                Battery Care Consumption: : {{ data.battery_care_consumption }}
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
### EV Costs
```yaml
type: entities
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
title: EV Costs
```
