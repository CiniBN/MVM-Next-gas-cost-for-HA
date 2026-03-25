# MVM Next gas cost for HA
Gázköltség számítása MVM Next szolgáltatónál Homa Assistant használata esetén

Tételezzük fel, hogy a gázóra impulzusait már számoljuk és már előállt a "virtuális gázóránk".
Tételezzük fel, hogy a gázóra 'sensor.gaz_meter' néven van jelen. Hozzunk létre egy template szenzort. Ez nálam egy template.yaml fájlban van, amit a configuration.yaml fájl include részén a 'template: !include templates.yaml' -tel tudjuk behívni a HA indulásakor.
Ha ez nem így van, akkor a configuration.yaml fájlba vegyük fel az alábbi sort:
template: !include templates.yaml

Hozunk létre egy template.yaml fájt és az alábbi kódot másoljuk bele.

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

Ez kiszámítja az elfogyasztott m3-ből az elfogyasztott energiát MJ-ban.
Hozzunk létre két közüzemi fogyasztásmérőt.
Az egyik a napi elfogyasztett gáz energiát fogja számolni. Névnek adjuk meg a 'Napi gázfogyasztás MJ'-t, forrás szenzornak adjuk meg a fenti 'sensor.gaz_energia_mj' szenzort, nullázási ciklust állítsuk Naponta-ra.
Az második a havi elfogyasztett gáz energiát fogja számolni. Névnek adjuk meg a 'Havi gázfogyasztás MJ'-t, forrás szenzornak adjuk meg a fenti 'sensor.gaz_energia_mj' szenzort, nullázási ciklust állítsuk Havonta-ra.

Ezek után a template.yaml fájba az előbbiek után másoljuk be az alábbi kódot:

            - name: "Napi gáz költség"
              unique_id: NGK
              unit_of_measurement: "Ft"
              device_class: monetary
              state_class: measurement
              state: >
                {% set limits = {
                  1: 12365, 2: 10421, 3: 8915, 4: 5145,
                  5: 1827, 6: 635, 7: 512, 8: 565,
                  9: 1109, 10: 3724, 11: 7490, 12: 10937
                } %}
                {% set napi = states('sensor.napi_gazfogyasztas_mj') | float(0) %}
                {% set havi = states('sensor.havi_gazfogyasztas_mj') | float(0) %}
                {% set limit = limits[now().month] %}
                {% set maradek = limit - (havi - napi) %}
                {% if maradek >= napi %}
                  {% set kedv = napi %}
                  {% set piaci = 0 %}
                {% elif maradek > 0 %}
                  {% set kedv = maradek %}
                  {% set piaci = napi - maradek %}
                {% else %}
                  {% set kedv = 0 %}
                  {% set piaci = napi %}
                {% endif %}
                {% set napok = 366 if now().year % 4 == 0 else 365 %}
                {% set alap = 11674 / napok %}
                {{ (kedv * 2.9 + piaci * 22.002 + alap) | round(2) }}

Ez fogja kiszámolni a napi gázfogyasztást forintban.
Figyelembe vesszük a havi kereteket, amely meghatározza, hogy ha a keret alatt maradunk, akkor rezsicsökkentett áron, ha azt túllépjük, akkor piaci áron kapjuk a gázt. Továbbá a havidíj elosztása is itt történik. Ezt úgy tudjuk megtenni, hogy a havi alapdíjat felszorozzuk egy évre, majd visszaosztjuk egy napra figyelembe véve a szökőévet is.

Sajnos az így kapott napi árat a HA Energy Dashboard-ja még mindig nem eszi meg, így ismételten létre kell hozzunk egy új segédentitást, azaz egy közüzemi fogyasztásmérőt.
Névnek adjuk meg a 'Napi gáz költség HUF', forrás szenzornak a 'sensor.napi_gaz_koltség'-et, nullázási ciklust állítsuk Naponta-ra.
Az így létrejövő szenzort már meg tudjuk adni az Energy Dashbord beállításaiban a 'A teljes költséget nyomon követő entitás használata' opciónál.

A fenti közüzemi fogyasztásmérőt természetesen havi nullázással is létre hozhatjuk, így a havi költség külön entitásként előáll. Ez nem alkalmas az Energy Dashboard-hoz!
