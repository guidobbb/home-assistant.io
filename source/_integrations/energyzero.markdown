---
title: EnergyZero
description: Instructions on how to integrate EnergyZero within Home Assistant.
ha_category:
  - Energy
ha_release: 2023.2
ha_iot_class: Cloud Polling
ha_config_flow: true
ha_codeowners:
  - '@klaasnicolaas'
ha_domain: energyzero
ha_platforms:
  - diagnostics
  - sensor
ha_integration_type: service
---

The **EnergyZero** {% term integration %} integrates the [EnergyZero](https://www.energyzero.nl/) API platform with Home Assistant.

The integration makes it possible to retrieve the dynamic energy/gas prices
from EnergyZero to gain insight into the price trend of the day and
to adjust your consumption accordingly.

Partners who are a reseller from EnergyZero:

- [ANWB Energie](https://www.anwb.nl/huis/energie/anwb-energie)
- [Energie van Ons](https://www.energie.vanons.org)
- [GroeneStroomLokaal](https://www.groenestroomlokaal.nl)
- [Mijndomein Energie](https://www.mijndomein.nl/energie)
- [SamSam](https://www.samsam.nu)
- [ZonderGas](https://www.zondergas.nu)

{% include integrations/config_flow.md %}

## Use cases

With the [energy dashboard](/energy) you can use the `current hour` price entity to calculate how much the electricity or gas has cost each hour based on the prices from EnergyZero. Or use one of the actions in combination with a [template sensor](#prices-sensor-with-response-data) to show the prices for the next 24 hours in a chart on your dashboard.

## Data updates

The integration will poll the EnergyZero API every 10 minutes to update the data in Home Assistant.

## Known limitations

The prices retrieved via the API are bare prices including VAT, however an energy company also charges other rates such as **energy tax** and **purchase costs**. The integration has no configuration option to add these values, but you could create a [template sensor](#all-in-price-sensor) for this.

## Sensors

The EnergyZero integration creates several sensor entities for both gas and electricity prices.

### Energy market price

Every day around **14:00 UTC time**, the new prices are published for the following day.

- The `current` and `next hour` electricity market price
- Average electricity price of the day
- Lowest energy price
- Highest energy price
- Time of day when the price is highest
- Time of day when the price is at its lowest
- Percentage of the current price compared to the maximum price

### Gas market price

For the dynamic gas prices, only entities are created that display the
`current` and `next hour` price because the price is always fixed for
24 hours; new prices are published every morning at **05:00 UTC time**.

{% include integrations/actions.md %}

## Templates

Create template sensors to display the prices in a chart or to calculate the all-in hour price.

### Prices sensor with response data

To use the response data from the actions, you can create a template sensor that updates every hour, today and after 2pm also tomorrow.

```yaml
template:
  - trigger:
      - platform: time_pattern
        minutes: "5"
      - platform: homeassistant
        event: start
    action:
      - action: energyzero.get_energy_prices
        data:
          config_entry: 1b4a46c6cba0677bbfb5a8c53e8618b0
          incl_vat: true
          start: "{{ today_at('00:00') }}"
          end: "{{ today_at('23:59') + timedelta(days=1) }}"
        response_variable: ezero_prices
    
    sensor:
      # De 48-hour sensor for the graph
      - name: Energy Prices 2 Days
        unique_id: energy_prices_2_days_combined
        icon: mdi:currency-eur
        state: "{{ now().strftime('%Y-%m-%d') }}"
        attributes:
          prices: >-
            {% set ns = namespace(list=[]) %}
            {% if ezero_prices.prices is defined and ezero_prices.prices | length > 0 %}
              {% for p in ezero_prices.prices %}
                {% set ns.list = ns.list + [{'timestamp': p.timestamp, 'price': p.price}] %}
              {% endfor %}
            {% endif %}
            {{ ns.list | to_json }}
```

### Dashboard card

This is an example for the Dashboard,  used with the HACS apexcharts-card
darkgreen= cheapest, green= cheap, yellow= expensive,  red= most expensive 

```yaml
type: custom:apexcharts-card
experimental:
  color_threshold: true
graph_span: 48h
span:
  start: day
header:
  show: true
  title: powerprices today and tomorrow (incl. VAT)
now:
  show: true
  label: now
series:
  - entity: sensor.energy_prices_2_days
    name: Price
    type: column
    unit: €/kWh
    float_precision: 3
    data_generator: |
      if (!entity.attributes.prices) return [];
      let rawData = entity.attributes.prices;
      if (typeof rawData === 'string') {
        try { rawData = JSON.parse(rawData); } catch (e) { return []; }
      }
      if (!Array.isArray(rawData)) return [];
      return rawData.map(p => [
        new Date(p.timestamp).getTime(),
        parseFloat(p.price)
      ]);
    color_threshold:
      - value: -99
        color: "#1D9E75"
      - value: 0.1
        color: "#639922"
      - value: 0.2
        color: "#EF9F27"
      - value: 0.3
        color: "#E24B4A"
yaxis:
  - decimals: 3
    apex_config:
      forceNiceScale: true
apex_config:
  plotOptions:
    bar:
      columnWidth: 95%
  xaxis:
    type: datetime
    tickAmount: 12
    labels:
      datetimeUTC: false
      format: dd HH:mm
  tooltip:
    x:
      format: dd-MM HH:mm
```

### All-in price sensor
To calculate the all-in hour price, you can create a template sensor that calculates the price based on the current price, energy tax, and purchase costs.

```yaml
template:
  - sensor:
      - name: EnergyZero all-in current price
        unique_id: allin_current_price
        icon: mdi:cash
        unit_of_measurement: "€/kWh"
        state_class: measurement
        state: >
          {% set energy_tax = PUT_HERE_THE_PRICE %}
          {% set purch_costs = PUT_HERE_THE_PRICE %}
          {% set current_price = states('sensor.energyzero_today_energy_current_hour_price') | float(0) %}
          {{ (current_price + energy_tax + purch_costs) | round(2) }}
```

## Removing the integration

This integration follows standard integration removal steps. If you also use the template sensors, you need to remove them manually.

{% include integrations/remove_device_service.md %}
