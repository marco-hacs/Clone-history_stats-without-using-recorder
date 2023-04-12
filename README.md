# Clone history_stats without using recorder 

The main problem I encounter in history_stats is having to record the entity_id used in the recorder. This constrains should you want to keep 
data for a long time to record for many days, bringing a large utlization of resources, with the risk of the db becoming corrupted and losing everything.
With this alternative I tried to replicate a history_stats sensor *without using the recorder*.

As a first step I create a sensor that increments the time at the time of utlization, based on the last_changed and the state of the entity_id (in the example switch.luce_studio).

``` 
template:
  - sensor:
      - name: "increment_time"
        unit_of_measurement: min 
        icon: mdi:history
        state: >-
            {%if is_state('switch.luce_studio', 'on') and as_timestamp(states.switch.luce_studio.last_changed) <= as_timestamp(now())%}
              {{((as_timestamp(now()) - as_timestamp(states.switch.luce_studio.last_changed))/3600)|round(6)}}
            {% else %}
              0
            {% endif %}

``` 
This sensor is successively passed to utility_meter, so that it can track for time customizable by the platform itself (in the example yearly)

``` 
utility_meter:
  increase_time_year:
    source: sensor.increment_time
    cycle: yearly

``` 
We now convert the sensor to a more readable value.
``` 
template:
  - sensor:
      - name: "time_use_year"
        icon: mdi:history
        state: >-
            {% set hours = states('sensor.increase_time_year') | float(0) %}
            {% set minutes = ((hours % 1) * 60) | int(0) %}
            {% set hours = (hours - (hours % 1)) | int(0) %}
             {% set day = ((hours |int(0) /24))|int(0) %}
              {% if day|int(0) >0 %}
                {{day}}d {{(hours|int(0))-(day*24)}}h {{minutes}}m
              {% elif hours|int(0) >0 %}
                {{hours}}h {{minutes}}m
              {% else %}
                {{minutes}}min
              {% endif %}
``` 
**Finally we have our sensor.time_use_year to use.**

Another aspect not to be underestimated is the ability to *reset* the sensor (which is not possible with history_stats).
``` 
script:
  reset_time:
    sequence:
      - service: utility_meter.calibrate
        data:
          value: "0"
        target: 
          entity_id:
            - sensor.increase_time_year
``` 

https://community.home-assistant.io/t/clone-history-stats-without-using-recorder/508119

## Since version 2023.4 it is possible to create macros to avoid code repetition.

- Create a new folder in config named custom_templates.
- Inside it create a .jinja file. In the example history_stats_custom.jinja and copy the following templates

``` 

# usage time in sec to be passed to utility_meter

{% macro time_on(entity_id) %}
{%if is_state(entity_id, 'on') and (as_timestamp(states[entity_id].last_changed)) <= as_timestamp(now())%}
  {{ (as_timestamp(now()) - as_timestamp(states[entity_id].last_changed))/3600}}
{% else %} 0 {% endif %}
{% endmacro %}

# converts utility_meter time to a friendlier way

{% macro history(entity_id) %}
{% set hours = states(entity_id) |float(0) %}
{% set minutes = ((hours % 1) * 60) |int(0) %}
{% set hours = (hours - (hours % 1)) |int(0) %}
 {% set day = ((hours |int(0) /24))|int(0) %}
  {% if day|int(0) >0 %}
    {{day}}d {{(hours|int(0))-(day*24)}}h {{minutes}}m
  {% elif hours|int(0) >0 %}
    {{hours}}h {{minutes}}m
  {% else %}
    {{minutes}}min
  {% endif %}
{% endmacro %}

``` 
-At this point you can reuse with each entity the following code having to rewrite it.

As a first step I create a sensor that increments the time at the time of utlization, based on the last_changed and the state of the entity_id (in the example switch.luce_studio).

``` 
template:
  - sensor:
      - name: "increment_time"
        unit_of_measurement: min 
        icon: mdi:history
        state: >-
          {% from 'history_stats_custom.jinja' import time_on %}
          {{time_on('switch.luce_studio')}}
``` 
This sensor is successively passed to utility_meter, so that it can track for time customizable by the platform itself (in the example yearly)

``` 
utility_meter:
  increase_time_year:
    sorgente: sensor.increment_time
    ciclo: yearly

```
We now convert the sensor to a more readable value.
``` 
template:
  - sensor:
      - name: "time_use_year"
        icon: mdi:history
        state: >-
          {% from 'history_stats_custom.jinja' import history %}
          {{ history(states('sensor.increase_time_year')) }}
``` 