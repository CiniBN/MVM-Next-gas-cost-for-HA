# MVM-Next-gas-cost-for-HA
Gázköltség számítása MVM Next szolgáltatónál Homa Assistant használata eseén

          - name: "Gáz energia MJ"
            default_entity_id: sensor.gaz_energia_mj
            unique_id: GEMJ
            unit_of_measurement: "MJ"
            device_class: energy
            state_class: total_increasing
            icon: mdi:fire
            state: >
              {% set gaz_m3 = states('sensor.gas_meter') | float(2) %}
              {% set korrekcios_tenyezo = 1.0184 %}
              {% set futoertek = 36 %}
              {{ (gaz_m3 * korrekcios_tenyezo * futoertek) | round(2) }}
