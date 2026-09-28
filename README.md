# Ariston Genus ONE+ WiFi — eBUSd & Home Assistant Integration

This repository provides an optimized, reverse-engineered configuration for **eBUSd** and **Home Assistant** to interface with **Ariston Genus ONE+ WiFi** gas boilers, **Sensys HD** room interfaces, **Ariston WiFi Gateways**, and the **Ariston Multi-Function Controller (Kit Multifunzionale 3318636)** via the 2-wire Bridgenet eBUS protocol.

---

## 🌟 Features & What is Working

### 1. ♨️ Boiler Telemetry & Diagnostics (`ebusd boiler`)
* **Live Sensors & Gauges**:
  * Real-time water pressure (bar) with low-pressure warning threshold.
  * Heating flow water temperature and return (EWT) water temperature (°C).
  * Burner flame detection, flame power output (kW), and gas modulation level (%).
  * Combustion fan speed (RPM) and diverter valve state (DHW vs. Central Heating).
  * Current boiler operating status (`standby`, `heating`, `heating hot water`, `circulating`, etc.).
* **Runtime & Health Statistics**:
  * Total burner operating hours for Central Heating (CH) and Domestic Hot Water (DHW).
  * Pump operating hours and boiler total power-on lifetime.
  * Total ignition cycles, fan cycles, circulation cycles, and diverter valve switches.
  * Flame lift-off counter.
* **Power & Combustion Parameters**:
  * Max / Min heating power percentage limits, DHW max power, and slow ignition power percentage.
  * Pump modulation envelope (Min / Max PWM %) and overrun post-circulation timers.

---

### 2. 🎛️ Thermostat & Heating Controls (`ebusd energymgr`)
* **Room Temperature & Setpoints**:
  * Ambient room temperature reading from Sensys HD (°C).
  * Outdoor weather temperature sensor reading (°C).
  * Day / Comfort target room temperature setpoint (°C).
  * Night / Reduced target room temperature setpoint (°C).
  * Direct flow water setpoint and dynamic modulated flow target (°C).
  * Maximum heating flow limit (LWT setpoint).
* **Heating Operating Modes**:
  * **Winter / Summer Toggle**: Control heating and domestic hot water generation independently.
  * **AUTO / SRA Mode**: Enable/disable Ariston smart automatic thermoregulation.
  * **Schedule / Forced Mode**: Operating program schedule and manual temperature overrides.

---

### 3. 🚿 Domestic Hot Water (DHW)
* DHW comfort target setpoint adjustment ($36\ ^\circ\text{C} - 60\ ^\circ\text{C}$).
* DHW economy / standby setpoint ($10\ ^\circ\text{C}$).
* Fast pre-heating Comfort Mode (`off`, `time_based`, `always_on`).
* Thermal Cleansing / Anti-legionella mode switch.
* Master DHW production switch.

---

### 4. 🧩 Multi-Function Controller Implementation (Menu 11 / Kit Multifunzionale 3318636)
A dedicated `ebusd multifunc` device provides full bidirectional control of the Ariston 3-relay / 3-sensor multi-zone expansion board (eBUS slave address `fc` / master `f7`):

* **11.0.0 Function Selection (`function_selection`)**:
  * Dropdown selector in Home Assistant with all 6 supported modes:
    * `0` — *Not defined / Unassigned*
    * `1` — *3 Direct Zones (Relays OUT1/OUT2/OUT3 for zone pumps)*
    * `2` — *Error Reporting & Remote Reset Relay*
    * `3` — *Differential Thermostat (Solar / Storage Tank mode)*
    * `4` — *Thermostat Mode (Probe IN1 $\rightarrow$ Relay OUT1)*
    * `5` — *Timed Auxiliary Output (Controlled by Sensys timer program)*
* **11.0.1 – 11.0.4 Manual Relay Controls**:
  * `manual_mode`: Master manual test switch.
  * `out1_manual_control`, `out2_manual_control`, `out3_manual_control`: Direct relay toggle switches.
* **11.1.0 – 11.1.5 Sensors & Diagnostics**:
  * Probe temperature readings for `in1_temp`, `in2_temp`, and `in3_temp` ($-20\ ^\circ\text{C} \dots +180\ ^\circ\text{C}$).
  * Active relay status indicators for `out1_status`, `out2_status`, and `out3_status`.
* **11.2 & 11.3 Differential Thermostat & Setpoints**:
  * Differential turn ON/OFF $\Delta T$ thresholds ($0 \dots 30\ ^\circ\text{C}$).
  * Sensor input Max / Min safety temperature limits.
  * Thermostat target temperature and hysteresis adjustments.

---

### 5. 🌐 WiFi Gateway & System Clock (`ebusd gateway` & `ebusd broadcast`)
* Network date and time synchronization broadcast across all eBUS devices (`cast_date`).
* Ariston NET gateway WiFi state (`cloud_connected`, `wifi_connected`, etc.).

---

## 📁 Repository Structure

| File | Description |
|---|---|
| [`ariston.csv`](ariston.csv) | Cleaned, gas-boiler-optimized eBUSd message definition CSV (stripped of unused heat pump / hybrid leftovers). |
| [`_templates.csv`](_templates.csv) | Custom data type & dropdown enum templates (`mf_func`, `onoff`, `heat_thermoreg_types`, error tables). |
| [`mqtt-hassio.cfg`](mqtt-hassio.cfg) | Customized Home Assistant MQTT Discovery configuration with `select` dropdown support, switches, and `filter-seen = 2` active polling. |
| [`home_assistant_climate.yaml`](home_assistant_climate.yaml) | Native Home Assistant MQTT Climate (Thermostat) entity definition. |
| [`home_assistant_dashboard.yaml`](home_assistant_dashboard.yaml) | Complete Lovelace Dashboard configuration with thermostat card, pressure gauge, and history charts. |

---

## 🚀 Installation & Setup

### 1. eBUSd Configuration (Home Assistant Add-on)

1. Ensure the files are placed in your Home Assistant configuration directory under `/config/ebusd/` (or `/homeassistant/ebusd/`):
   ```text
   /homeassistant/ebusd
   ├── ariston/
   │   ├── _templates.csv
   │   └── ariston.csv
   └── mqtt-hassio.cfg
   ```
2. In **Settings $\rightarrow$ Add-ons $\rightarrow$ eBUSd $\rightarrow$ Configuration**, set the following options:
   * **Network / enhanced-protocol adapter (`--device`)**: `IP_OF_ADAPTER:PORT` (e.g., `10.10.0.7:3333`)
   * **Additional ebusd options**:
     * `--configpath=/config/ebusd/ariston`
     * `--mqttint=/config/ebusd/mqtt-hassio.cfg`
     * `--mqttvar=filter-direction=r|u|^w`
3. Restart the **eBUSd** add-on. The log will confirm:
   ```text
   found messages: 351 (0 conditional on 0 conditions, 53 poll, 80 update)
   ```

---

### 2. Add the Thermostat (Climate Entity)

Add the contents of [`home_assistant_climate.yaml`](home_assistant_climate.yaml) under the `mqtt:` section in your `/config/configuration.yaml`:

```yaml
mqtt:
  climate:
    - name: "Ariston Heating"
      unique_id: "ariston_genus_climate"
      icon: "mdi:radiator"
      modes:
        - "heat"
        - "off"
      mode_state_topic: "ebusd/energymgr/heating_status"
      mode_state_template: >
        {% if value_json.onoff.value == 'on' %}
          heat
        {% else %}
          off
        {% endif %}
      mode_command_topic: "ebusd/energymgr/heating_status/set"
      mode_command_template: >
        {% if value == 'heat' %}
          on
        {% else %}
          off
        {% endif %}
      current_temperature_topic: "ebusd/energymgr/z1_room_temp_f"
      current_temperature_template: "{{ value_json['0'].value | float(0) }}"
      temperature_state_topic: "ebusd/energymgr/z1_day_temp_e"
      temperature_state_template: "{{ value_json['0'].value | float(21.0) }}"
      temperature_command_topic: "ebusd/energymgr/z1_day_temp/set"
      min_temp: 10
      max_temp: 30
      temp_step: 0.5
      temperature_unit: "C"
      action_topic: "ebusd/boiler/flame_active"
      action_template: >
        {% if value_json.onoff.value == 'on' %}
          heating
        {% elif states('switch.ebusd_energymgr_heating_status_winter_mode') == 'on' %}
          idle
        {% else %}
          off
        {% endif %}
```

Reload MQTT entities under **Developer Tools $\rightarrow$ YAML $\rightarrow$ Reload Manually Configured MQTT Entities**.

---

### 3. Setup Dashboard

Use the code in [`home_assistant_dashboard.yaml`](home_assistant_dashboard.yaml) to add the full view with:
* **Zone 1 Thermostat card** (live room temperature dial, setpoint slider, and heat/off toggle).
* **Boiler water pressure gauge** with colored safety zones.
* **Boiler overview card** (status, flame state, diverter valve position, hot water switch).
* **Heating water temperatures history graph** (target flow vs. return EWT).
* **Room & outdoor temperatures graph**.
* **Burner flame power (kW), modulation (%), and fan speed (RPM) graph**.

---

## 🔒 Safety & Stability Notes

* **Read-Only Compatibility**: All sensors and statistics query data cleanly without disturbing boiler EEPROM configuration.
* **Strict Gas Boiler Scope**: Unused heat pump and chiller registers have been excluded to avoid bus collision errors and prevent ghost MQTT entities.
* **Auto-Polling Balance**: Polling priorities (`r1` for fast telemetry, `r2` for periodic statistics) ensure smooth real-time monitoring without saturating the eBUS baud rate ($2400\ \text{baud}$).

---

## 📄 License
Released under the [GPLv3 License](LICENSE). Based on research from the eBUSd community and Ariston Bridgenet reverse-engineering projects.
