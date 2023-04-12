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

Dalla versione 2023.4 è possibile creare macro per evitare ripetizioni di codice.

- Creare una nuova cartella in config dal nome custom_templates
- Al suo interno creare un file .jinja. Nell'esempio history_stats_custom.jinja e copiare i seguenti template

``` 

# tempo di utilizzo in sec da passare ad utility_meter

{% macro time_on(entity_id) %}
{%if is_state(entity_id, 'on') and (as_timestamp(states[entity_id].last_changed)) <= as_timestamp(now())%}
  {{ (as_timestamp(now()) - as_timestamp(states[entity_id].last_changed))/3600}}
{% else %} 0 {% endif %}
{% endmacro %}

# converte tempo di utility_meter in un modo più amichevole

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
- A questo punto è possibile riutilizzare con ogni entità il seguente codice doverlo riscrivere. 

Come primo passo ho creato un sensore che incrementa il tempo al momento dell'utilizzo, in base a last_changed e allo stato dell'entity_id (nell'esempio switch.luce_studio).

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
Questo sensore viene passato successivamente a utility_meter, in modo che possa tracciare per tempo personalizzabile dalla piattaforma stessa (nell'esempio annuale)

``` 
utility_meter:
  increase_time_year:
    sorgente: sensor.increment_time
    ciclo: annuale

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