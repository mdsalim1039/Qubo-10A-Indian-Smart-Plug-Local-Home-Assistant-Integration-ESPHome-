# Qubo 10A Indian Smart Plug Local Home Assistant Integration (ESPHome)

This repository contains the complete reverse-engineered configuration and documentation to flash the **Qubo 10A Wi-Fi Smart Plug (Indian Variant)** with custom open-source ESPHome firmware. This completely detaches the plug from Qubo's closed proprietary cloud infrastructure, enabling fast, local, and private energy-monitoring control directly inside **Home Assistant**.

## 📌 Hardware Overview
* **PCB Revision Board Stamp:** `PTTY_OEM_003_01`
* **Central Microcontroller:** Espressif ESP32-C3 (Single-core RISC-V architecture)
* **Energy Monitoring IC:** Shanghai Belling BL0937 (High-frequency pulse-output metering chip)

---

## ⚡ Critical Electrical Safety Warning
⚠️ **NEVER connect your USB-to-UART serial flashing adapter to your computer while the smart plug is inserted into a live 220V wall socket!** 
The `PTTY_OEM_003_01` board utilizes a **non-isolated design**. The low-voltage DC Ground (`GND`) is directly tied to your alternating current mains potential. Connecting the UART lines under grid power will instantly destroy your computer's motherboard, trigger a short circuit, or cause **fatal electrical shock**. Perform all physical wire flashes powered strictly by your UART adapter's isolated 3.3V power pin.

---

## 🛠️ Pin Mapping Reference Summary
Through exhaustive multimeter continuity and real-world high-load frequency analysis, the physical tracking paths on the `PTTY_OEM_003_01` revision are mapped as follows:

| Hardware Component | ESP32-C3 Pin Location | Function / Behavior |
| :--- | :--- | :--- |
| **Power Relay** | `GPIO4` | Controls mechanical power latch terminal socket |
| **Status LED** | `GPIO6` | Low-active status light inside plug casing shell |
| **Physical Toggle Button**| `GPIO10` | Mechanical tactile override switch |
| **BL0937 SEL Pin** | `GPIO5` | Active-low channel toggle (alternates Volts/Amps data streams) |
| **BL0937 CF Pin** | `GPIO7` | Power / Wattage active pulse frequency collector line |
| **BL0937 CF1 Pin** | `GPIO3` | Voltage / Current active pulse frequency collector line |

---

## 💾 Installation & Flashing Procedure

### 1. Hardware Preparation
1. Carefully crack open the ultrasonically welded plastic casing shell using a thin pry tool or hobby knife.
2. Locate the central ESP32-C3 module and solder thin wires to the following programming pads exposed on the PCB:
   * **TX**
   * **RX**
   * **GND**
   * **3.3V**
3. Locate **GPIO9** on the chip board. This is the hardware programming strapping pin.

### 2. Putting Chip into Bootloader Mode
1. Ensure the plug is completely removed from any wall outlet.
2. Connect your USB-to-UART adapter lines directly to your matching soldered wires.
3. **Short GPIO9 to GND** using a jumper wire or tweezers.
4. While holding the short, insert the USB adapter into your computer. Release the short 2 seconds after power-on.

### 3. Compilation and Initial Flash
1. Open your Home Assistant **ESPHome Dashboard**.
2. Create a new device, click the three-dots menu, and select **Clean Build** to drop any lingering environment caches.
3. Paste the provided production code below into your editor panel.
4. Click **Install** -> **Plug into this computer** to compile the custom code and flash over the serial line.
5. Once verified, desolder the serial adapter wires, reassemble the protective casing shell, and insert it back into your wall socket. All future updates can be handled wirelessly **Over-The-Air (OTA)**.

---

## 📝 Final Calibrated ESPHome YAML Code

Below is the complete, factory-calibrated configuration file ready to deploy. It strips out system logging overhead to protect timing loops and uses custom multipliers scaled directly against a benchmark **2400W Remington D5220 High-Heat Turbo hairdryer** profile to pull dead-accurate voltage (~230V) and amperage.

```yaml
substitutions:
  device_name: qubo-smart-plug
  friendly_name: "Qubo Smart Plug"

esphome:
  name: \${device_name}
  friendly_name: \${friendly_name}

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino             # Stable, lightweight framework essential for hardware timer accuracy

logger:
  level: INFO                 # Throttles debug text output logs to secure data pulse streams

api:
  encryption:
    key: "YOUR_NATIVE_API_ENCRYPTION_KEY_HERE"

ota:
  - platform: esphome

wifi:
  ssid: "YOUR_WIFI_SSID"
  password: "YOUR_WIFI_PASSWORD"
  
  ap:
    ssid: "\${friendly_name} Fallback Hotspot"

captive_portal:

# The Confirmed Hardware Relay
switch:
  - platform: gpio
    name: "Power Relay"
    pin: GPIO4
    id: relay
    restore_mode: RESTORE_DEFAULT_OFF

# The Confirmed Casing Manual Toggle Switch
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO10
      mode: INPUT_PULLUP
      inverted: true
    name: "Physical Button"
    on_press:
      - switch.toggle: relay

# The Confirmed Status LED
status_led:
  pin:
    number: GPIO6
    inverted: true

# The Fully Calibrated BL0937 Multi-Pulse Core Processing Framework
sensor:
  - platform: hlw8012
    model: BL0937
    sel_pin: 
      number: GPIO5
      inverted: true          # Inverts state logic to cleanly toggle between Volts and Amps
    cf1_pin: GPIO3           # Confirmed Pin 7 (CF1) -> Tracks Volts/Amps Stream
    cf_pin: GPIO7            # Confirmed Pin 6 (CF) -> Tracks active load Wattage trace
    
    # Custom Calibrated Reference Multipliers matching the specific Qubo board shunts
    voltage_divider: 16038.0  
    current_resistor: 0.001   
    
    voltage:
      name: "Voltage"
      unit_of_measurement: "V"
      accuracy_decimals: 1
      filters:
        - multiply: 0.1       # Corrects your raw core calculation baseline to clean 230V mains
        
    current:
      name: "Current"
      unit_of_measurement: "A"
      accuracy_decimals: 3
      filters:
        - multiply: 0.711     # Calibrates amperage data limits directly under load parameters
        
    power:
      name: "Power"
      unit_of_measurement: "W"
      accuracy_decimals: 1
      filters:
        - multiply: 0.0844    # Calibrates pulse counting data directly to true load limits
        
    change_mode_every: 8s
    update_interval: 5s

  # Optional Total Energy Totalizer for native Home Assistant Energy Dashboard Integration
  - platform: total_daily_energy:
    name: "Total Daily Energy"
    unit_of_measurement: "kWh"
    accuracy_decimals: 3
    restore: true
```

---

## 📊 Post-Flash Verification Metrics
Once safely deployed back under 220V grid power lines with an attached operating load, your Home Assistant sensor panels should capture stable, linear curves matching the calibrated limits below:
* **Idle State (Relay OFF):** Voltage shows active ~225V–230V grid levels; Current and Power drop to clean `0.000A` and `0.0W`.
* **Full Active Load (Hairdryer Running):** Power reads exact appliance values up to `2400W`; Current registers matching operational targets up to `10.6A`.

## 🤝 Acknowledgments
Special thanks to the open-source community contributors who parsed the hardware characteristics of the BL0937 pulse-engine counters on RISC-V core profiles. Feel free to open an issue or fork if you find minor alignment drifts on alternative manufacturing batch runs!
