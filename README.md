# 🔥 Heating Curve Optimizer

Deze Home Assistant custom integratie past automatisch en voorspellend de stooklijn-offset van je warmtepomp aan op basis van:

- **Dynamische stroomprijzen** (via Nordpool)
- **Zonneproductie-verwachting** (via Solcast)
- **Warmteverlies van de woning** (oppervlak, energielabel)
- **Buitentemperatuur en -voorspelling**
- **Actueel verbruik van de warmtepomp (via DSMR)**
- **Rekenkundige COP-modellen of actuele COP op basis van vermogen**

Het doel is om de **aanvoertemperatuur slim te verhogen of verlagen**, afhankelijk van kosten, efficiëntie en warmteverlies — met behoud van comfort.

---

## 📦 Functionaliteit

- Simuleert binnentemperatuur over de ingestelde horizon (bijv. 4 uur)
- Berekent netto warmtevraag (verlies - zoninstraling)
- Voert COP-berekening uit (geschat of werkelijk)
- Optimaliseert de stooklijn-offset op basis van verwachte kosten
- Publiceert offset in een sensor met toekomstvoorspelling als attributes

---

## ⚙️ Vereiste sensoren

### 🟠 Solcast
- `sensor.solcast_pv_forecast_forecast_today`
  - Verwachte opbrengst van vandaag in kWh
- `sensor.solcast_pv_forecast_forecast_tomorrow`
  - Verwachte opbrengst van morgen in kWh
  - Wordt gebruikt om verwachte zonnewinst per m² te schatten

### 🟠 Nordpool
- `sensor.nordpool_kwh_nl_eur_0_10`
  - Uurprijs voor afname (€/kWh)
  - Type: float, geüpdatet dagelijks
  - Gebruik: kostenvoorspelling bij warmtevraag

### 🟠 Buitentemperatuur
- `sensor.outdoor_temperature`
  - Actuele buitentemperatuur in °C
  - Kan ook van een andere sensor of weather entity komen

### 🟠 Binnentemperatuur
- `sensor.indoor_temperature`
  - Actuele binnentemperatuur in °C
  - Wordt gebruikt voor berekening van warmteverlies

### 🟠 Warmtepompverbruik
- `sensor.power_consumption`
  - Actueel opgenomen vermogen (W) van de warmtepomp
  - Gebruikt om actuele COP te schatten

---

## 🧾 Vereiste configuratie

Plaats deze configuratie in je `configuration.yaml` of configureer via toekomstige UI-configuratie:

```yaml
heating_curve_optimizer:
  area_m2: 120                  # Woonoppervlakte in vierkante meters
  energy_label: B              # Energielabel: A t/m G (voor warmteverlies)
  horizon_hours: 4             # Aantal uur vooruit voorspellen en optimaliseren
```

---

## 📥 Invoer (intern gebruikt)

| Naam                 | Bron                       | Beschrijving                                         |
|----------------------|----------------------------|------------------------------------------------------|
| `area_m2`            | YAML                       | Oppervlakte woning in m²                             |
| `energy_label`       | YAML                       | Wordt omgezet naar U-waarde per m² per Kelvin       |
| `outdoor_temperature`| sensor                     | Actuele buitentemperatuur                           |
| `solar_forecast`     | Solcast (1 of meer) | Totale dagopbrengst → verdeeld over forecast-horizon |
| `price_forecast`     | Nordpool                   | Prijs per uur over horizon                           |
| `power_consumption`  | DSMR of vermogenssensor    | Actuele warmtepompverbruik (optioneel)              |
| `indoor_temperature` | sensor                     | Actuele binnentemperatuur                            |

---

## 📤 Output

### `sensor.heating_curve_offset`
- Huidige geadviseerde offset (bijv. +1.5 °C)
- Attributes:
  - `future_offsets`: lijst met voorspelde offsets komende uren
  - `indoor_temperature_forecast`: voorspelde binnentemperatuur
  - (toekomstig) `predicted_savings`: verwachte energiekostenbesparing

### `sensor.hourly_heat_loss`
- Berekend warmteverlies van de woning in kW

### `sensor.hourly_net_heat_demand`
- Netto warmtevraag na aftrek van zonwinst in kW

### `sensor.current_net_consumption`
- Actuele netto stroomafname (verbruik min productie) in kW

---

## 🧮 Berekening

### Warmteverlies (per uur)

\[
Q_{verlies} = A \cdot U \cdot (T_{binnen} - T_{buiten})
\]

- U-waarde wordt bepaald uit energielabel:
  - A: 0.6, B: 0.8, ..., G: 1.8 W/m²K

### Zonwinst

\[
Q_{zon} = \text{zoninstraling} \cdot A \cdot \eta
\]

- Geschatte opbrengst per uur op basis van Solcast-data
- Efficiëntiefactor \( \eta \approx 0.15 \)

### Netto warmtevraag

\[
Q_{netto} = Q_{verlies} - Q_{zon}
\]

### COP (Coefficient of Performance)

1. **Fallback-model** (als geen werkelijk verbruik beschikbaar):

\[
COP = 6 - 0.1 \cdot (T_{aanvoer} - T_{buiten})
\]

2. **Werkelijk** (als warmtepompvermogen bekend):

\[
COP = \frac{Q_{netto}}{P_{verbruik}}
\]

### Kosten

\[
Kosten = \frac{Q_{netto}}{COP} \cdot \text{prijs}(t)
\]

De offset wordt gekozen die over de hele horizon de **laagste totale kostprijs** oplevert, binnen comfortgrenzen.

---

## 💡 Voorbeeldgebruik

```yaml
- alias: "Stel offset in op warmtepomp"
  trigger:
    - platform: state
      entity_id: sensor.heating_curve_offset
  action:
    - service: number.set_value
      target:
        entity_id: number.heat_curve_offset
      data:
        value: "{{ states('sensor.heating_curve_offset') | float }}"
```

---

## 🧪 Dashboard

Voeg de volgende sensoren toe aan je Lovelace-dashboard:

- `sensor.heating_curve_offset`
- `sensor.power_consumption`
- `sensor.outdoor_temperature`
- `sensor.nordpool_kwh_nl_eur_0_10`
- `sensor.solcast_pv_forecast_forecast_today`
- `sensor.hourly_heat_loss`
- `sensor.hourly_net_heat_demand`
- `sensor.current_net_consumption`

Gebruik een kaarttype zoals **entities**, **sensor graph**, of **custom:apexcharts-card** om toekomstige waarden te tonen.

---

## 📥 Installatie

1. Pak de ZIP uit in je `config/custom_components/` map.
2. Herstart Home Assistant.
3. Voeg `heating_curve_optimizer` toe via YAML.
4. Herstart opnieuw en controleer de sensoren.
5. Koppel aan je automatiseringen of visualiseer de uitkomsten.

---

## 📞 Ondersteuning

- DSMR P1 sensor: [DSMR Slimme Meter integratie](https://www.home-assistant.io/integrations/dsmr/)
- Solcast PV Forecast: [Solcast integratie](https://github.com/solcast/solcast-ha)
- Nordpool: [nordpool integratie](https://github.com/custom-components/nordpool)

---

## 📊 Uitleg output sensoren

### `sensor.heating_curve_offset`
Deze sensor toont de geadviseerde offset voor de stooklijn. De waarde wordt
berekend door voor de komende uren de verwachte warmtevraag, COP en
elektriciteitsprijs te combineren. Het algoritme kiest de offset die de laagste
totale kosten oplevert binnen de ingestelde grenzen.

### `sensor.hourly_heat_loss`
Geeft het geschatte warmteverlies per uur in kilowatt weer. De berekening
gebruikt de oppervlakte van de woning en het energielabel om een U-waarde te
bepalen. Deze U-waarde wordt vermenigvuldigd met het temperatuurverschil tussen
binnen en buiten.

### `sensor.hourly_net_heat_demand`
Dit is het verschil tussen het warmteverlies en de zonnewinst. Negatieve waarden
worden op nul gezet. Deze sensor gebruikt direct de opgegeven Solcast sensoren.
Zo zie je hoeveel netto warmte er per uur nodig is om de binnentemperatuur op peil te houden.


Gemaakt voor maximale efficiëntie, flexibiliteit en inzicht!