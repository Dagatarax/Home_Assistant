Creates template for weather details saved in attributes, triggered from **Meteorologisk institutt (Met.no)** integration (native, not HACS). On Voice Assistant conversation trigger reads the hourly weather aloud with number of degress Celsius and % chance of rain for each phase of 1 day. No HACS or LLM/Ai needed.

Google Gemini (free tier) helped with this. Feel free to ask it for customization. If it nicked your ideas, I am sorry.

# **Prerequisites:**

* [ ] Voice Assistant (Voice PE) device with TTS and SST pipeline

* [ ] Split `configuration.yaml`, i.e. containing:
  `template: !include_dir_merge_list ./templates/`

* [ ] Integration **Meteorologisk institutt (Met.no)**

# Settings:

## Templates:

Create this file for Weather service:

`/homeassistant/templates/weather_forecast.yaml`

```
- trigger:
    - trigger: time_pattern
      minutes: "/30"
    - trigger: event
      event_type: homeassistant_started
    - trigger: event
      event_type: event_template_reloaded
  action:
    - action: weather.get_forecasts
      target:
        entity_id: weather.forecast_home
      data:
        type: daily
      response_variable: daily_data
    - action: weather.get_forecasts
      target:
        entity_id: weather.forecast_home
      data:
        type: hourly
      response_variable: hourly_data
  sensor:
    - name: "Daily Weather Forecast"
      unique_id: daily_weather_forecast
      state: "{{ states('weather.forecast_home') }}"
      attributes:
        forecast_data: >
          {{ daily_data['weather.forecast_home'].forecast[:3] if 'weather.forecast_home' in daily_data else [] }}

    - name: "Hourly Weather Forecast"
      unique_id: hourly_weather_forecast
      state: "{{ states('weather.forecast_home') }}"
      attributes:
        forecast_data: >
          {{ hourly_data['weather.forecast_home'].forecast[:24] if 'weather.forecast_home' in hourly_data else [] }}
```

**Check configuration** `/config/tools/yaml` and reload | restart

## Script:

```
sequence:
  - action: assist_satellite.announce
    metadata: {}
    target:
      device_id: '{{ target_satellite }}'
    data:
      message: >-
        {% set current_state = states('weather.forecast_home') %} {% set
        current_temp = state_attr('weather.forecast_home', 'temperature') %} {%
        set now_dt = now() %} {% set forecast =
        state_attr('sensor.hourly_weather_forecast', 'forecast_data') or [] %}

        {# Define period buckets #} {% set buckets = namespace(
          morning=[], 
          afternoon=[], 
          evening=[], 
          night=[]
        ) %}

        {% for entry in forecast %}
          {% set entry_time = entry.datetime | as_datetime | as_local %}
          {# Consider future forecast entries within the next 24 hours #}
          {% if entry_time > now_dt and entry_time <= now_dt + timedelta(hours=24) %}
            {% set hour = entry_time.hour %}
            {% set precip = entry.precipitation_probability | default(entry.precipitation) | default(0) %}
            {% set is_tomorrow = entry_time.day != now_dt.day %}
            {% set item = {'temp': entry.temperature, 'precip': precip, 'tomorrow': is_tomorrow} %}

            {% if hour >= 6 and hour < 12 %}
              {% set buckets.morning = buckets.morning + [item] %}
            {% elif hour >= 12 and hour < 17 %}
              {% set buckets.afternoon = buckets.afternoon + [item] %}
            {% elif hour >= 17 and hour < 21 %}
              {% set buckets.evening = buckets.evening + [item] %}
            {% else %}
              {% set buckets.night = buckets.night + [item] %}
            {% endif %}
          {% endif %}
        {% endfor %}

        Currently, it is {{ current_state }} and {{ current_temp }} degrees
        Celsius. Here is the upcoming forecast:

        {% set summary = namespace(parts=[]) %} {% set sections = [
          ('Morning', buckets.morning), 
          ('Afternoon', buckets.afternoon), 
          ('Evening', buckets.evening), 
          ('Night', buckets.night)
        ] %}

        {% for name, items in sections %}
          {% if items | count > 0 %}
            {% set max_temp = (items | map(attribute='temp') | list | max) | round %}
            {% set max_rain = (items | map(attribute='precip') | list | max) | round %}
            {% set prefix = "Tomorrow " ~ name if items[0].tomorrow else name %}
            {% set summary.parts = summary.parts + [prefix ~ ": high of " ~ max_temp ~ " degrees with a " ~ max_rain ~ " percent chance of rain."] %}
          {% endif %}
        {% endfor %}

        {{ summary.parts | join(' ') }}
      preannounce: false
alias: READ_Weather_hourly
description: >-
  Summarizes the upcoming 24-hour weather by Morning, Afternoon, Evening, and
  Night.

```

The algorithm is maybe a little off here, apologies. It "works" but might not give you precise calculations.

## Automation:

```
alias: Voice_Trigger-Read_Weather_Forecast_Hourly
description: ''
triggers:
  - trigger: conversation
    command:
      - Read weather
      - Tell me weather
      - Tell me forecast
      - Read forecast
      - weather
      - forecast
conditions: []
actions:
  - action: script.read_weather_hourly
    metadata: {}
    data:
      target_satellite: '{{ trigger.device_id }}'
mode: single

```

This passes argument to the script, so the script answers on the device triggered (handy if you have two or more).

# ENJOY
