# Smart Funte Standing Desk: ESP32 + ESPHome + Home Assistant + Apple Home

[繁體中文](README.md) | **English**

Connect a **Funte standing desk** (Jiecang **JCB35M12C** controller) to **Home Assistant** via an ESP32, then wrap it as a template cover so you can control sit/stand and set height from **Apple Home** using your iPhone or Siri.

> **No need to disassemble the desk or splice the original handset cable.** Use only the spare RJ12 port on the control box.

![Architecture](docs/images/architecture.svg)

## What You Get

![Demo](docs/images/demo.gif)

- A full Mushroom control panel on Home Assistant (up / down / stop / step / saved positions / height slider)
- Apple Home shows the desk as a "shade" — vertical slider that maps to actual height
- Siri: "Hey Siri, set Standing Desk to 80%" → desk moves to the corresponding height
- Real-time height reporting, ready for automations (e.g., "stand up after sitting too long")

## Hardware (BOM)

| Item | Spec / Notes | Qty |
|------|--------------|-----|
| Standing Desk | Funte (with **JCB35M12C** control box; has a spare RJ12 port) | 1 |
| Dev Board | ESP-WROOM-32 30-pin DevKit | 1 |
| Micro USB Cable | For powering the ESP32 | 1 |
| Dupont Wires | Male-to-female, ~3 (GND / RX / TX) | 3 |
| RJ12 Cable | Depends on the green terminal block adapter you receive | 1 |
| Logic Level Shifter *(optional)* | TXS0108E / BSS138 module, 3.3V ↔ 5V | 1 |

> **About the level shifter**: ESP32 GPIO is 3.3V, the controller's UART is 5V TTL. **You should technically use one**, but direct connection works in practice (most community builds do this). Long-term stability is at your own risk. This project currently has no level shifter and runs fine.

## Wiring

### Spare RJ12 Pinout (from [Rocka84 README](https://github.com/Rocka84/esphome_components/tree/master/components/jiecang_desk_controller))

| Pin | Function |
|-----|----------|
| 1 | NC |
| 2 | GND |
| 3 | Controller TX (sends height packets) |
| 4 | VCC +5V |
| 5 | Controller RX (receives commands) |
| 6 | NC |

### Green Terminal Block ↔ ESP32

The green terminal block adapter that comes with the controller is **numbered from the right side when viewed from the screw face (metal side)**. Counting the wrong way will cost you hours of debugging (speaking from experience).

![Wiring Diagram](docs/images/wiring-diagram.svg)

| Green Terminal | RJ12 Pin | ESP32 Pin (silkscreen) |
|----------------|----------|------------------------|
| 2 | Pin 2 GND | **GND** |
| 3 | Pin 3 Controller TX | **RX2** (GPIO16) |
| 5 | Pin 5 Controller RX | **TX2** (GPIO17) |
| 4 (VCC 5V) | Pin 4 | **Do not connect** |

Power the ESP32 from its Micro USB port (any 5V/500mA+ source: computer, USB charger, or power bank).

### Real-world Wiring Reference

Overall setup:

![Overall Wiring](docs/images/wiring-overview.jpg)

Green terminal block close-up (screw face — the direction used for pin numbering):

![Green Terminal Close-up](docs/images/green-terminal-closeup.jpg)

## ESPHome Firmware

Install the **ESPHome Builder** add-on in Home Assistant, create a new device, and paste in the contents of [esphome/desk.yaml](esphome/desk.yaml) (keep the Builder-generated `api: encryption: key:` and `ota: password:` lines).

Key configuration:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/Rocka84/esphome_components/
    components: [ jiecang_desk_controller ]

uart:
  tx_pin: GPIO17   # Silkscreen TX2 → RJ12 Pin 5 (Controller RX)
  rx_pin: GPIO16   # Silkscreen RX2 → RJ12 Pin 3 (Controller TX)
  baud_rate: 9600

jiecang_desk_controller:
  id: my_desk
  sensors:
    height:
      name: Height
    # ... other sensor / button / number
```

> Important: **Keep all entity names in English**. CJK characters get reduced to all-underscore IDs by ESPHome's ID generator, causing duplicate-ID compile errors. You can still set Chinese friendly names later in Home Assistant — the IDs don't have to be Chinese.

First flash requires USB: in ESPHome Builder, click INSTALL → Manual download → `.bin` → open [web.esphome.io](https://web.esphome.io) in Chrome → connect ESP32 via USB → upload the bin. OTA wireless flashing works after the first boot.

## Home Assistant Integration

Once the ESP32 is online, Home Assistant auto-discovers it. Click "Add". The following entities will appear:

- `sensor.standing_desk_height` (current height, cm)
- `sensor.standing_desk_height_pct` (percent)
- `sensor.standing_desk_height_min` / `_max` (factory limits)
- `sensor.standing_desk_position_1~4` (saved positions)
- `number.standing_desk_set_height` (set height)
- `button.standing_desk_move_up` / `_down` / `_stop`
- `button.standing_desk_step_up` / `_down` (~14 mm per press)
- `button.standing_desk_goto_position_1~4`
- `button.standing_desk_save_position`

### Prerequisite: A binary_sensor to detect sit / stand

The popup dashboard switches icon and color depending on whether the desk is raised. You need a template binary sensor that checks "is the desk above its minimum height". Add this to the `template:` block in `configuration.yaml` (you can put it in the same block as the cover):

```yaml
template:
  - binary_sensor:
      - name: Standing Desk Men
        unique_id: standing_desk_men
        state: "{{ states('sensor.standing_desk_height') | float(0) > 70.6 }}"
```

After restarting HA, `binary_sensor.standing_desk_men` becomes available (on = standing, off = sitting).

> Skip this if you only use [panel.yaml](homeassistant/dashboard/panel.yaml) and not the popup version.

### Mushroom Dashboard

Install [Mushroom](https://github.com/piitaya/lovelace-mushroom) via HACS first.

Two versions are provided:

- [homeassistant/dashboard/panel.yaml](homeassistant/dashboard/panel.yaml) — Full panel (always visible)
- [homeassistant/dashboard/panel-popup.yaml](homeassistant/dashboard/panel-popup.yaml) — A single button that opens a popup (requires [Browser Mod](https://github.com/thomasloven/hass-browser_mod), [button-card](https://github.com/custom-cards/button-card), and the binary_sensor above)

The popup:

![HA Dashboard Popup](docs/images/ha-dashboard-popup.png)

## Apple Home Integration (via Template Cover)

The trick: HomeKit doesn't have a "desk" device class, but the **shade / cover UI is a vertical slider** — perfect for desk height. Wrap the desk as `cover.standing_desk` where 0% = 70.6 cm (sitting) and 100% = 119 cm (standing), then expose it via the HomeKit Bridge.

Merge the `template:` block from [homeassistant/configuration.yaml.example](homeassistant/configuration.yaml.example) into your `configuration.yaml`:

```yaml
template:
  - cover:
      - name: Standing Desk
        unique_id: standing_desk_cover
        device_class: shade
        position: >-
          {% set h = states('sensor.standing_desk_height') | float(0) %}
          {{ ((h - 70.6) / (119 - 70.6) * 100) | round(0) }}
        open_cover:
          - action: button.press
            target:
              entity_id: button.standing_desk_move_up
        close_cover:
          - action: button.press
            target:
              entity_id: button.standing_desk_move_down
        stop_cover:
          - action: button.press
            target:
              entity_id: button.standing_desk_stop
        set_cover_position:
          - action: number.set_value
            target:
              entity_id: number.standing_desk_set_height
            data:
              value: "{{ position / 100 * (119 - 70.6) + 70.6 }}"
```

Restart HA → add the **HomeKit Bridge** integration → check `cover.standing_desk` → scan the QR code from the iPhone Home app to pair.

In Apple Home, the desk appears as a "shade" tile:

- Tap → full-screen vertical slider
- Drag to 50% → desk moves to 94.8 cm
- "Hey Siri, open Standing Desk" → raises to max
- "Hey Siri, close Standing Desk" → lowers to min

<table>
  <tr>
    <td><img src="docs/images/apple-home-1.png" alt="Apple Home control view"></td>
    <td><img src="docs/images/apple-home-2.png" alt="Apple Home settings view"></td>
  </tr>
  <tr>
    <td align="center"><sub>Shade-style vertical slider</sub></td>
    <td align="center"><sub>Accessory settings / automations</sub></td>
  </tr>
</table>

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| HA shows height but buttons do nothing | Counted "slots 3/5" from the wrong side of the green terminal | Number from the **right side of the screw face** |
| Compile error "Duplicate sensor entity" | CJK entity names collapse to all-underscore IDs | Use English `name`, rename to Chinese as friendly name in HA |
| Height reads fine but OTA fails after boot | UART logger conflicts with jiecang's UART | Use UART2 (GPIO16/17), not GPIO1/3 |
| Mushroom + Browser Mod popup doesn't open | mushroom-template-card sometimes drops `fire-dom-event` | Use `tile` or `custom:button-card` as the outer card instead |
| Apple Home slider does nothing | `number.set_value` not triggering | Verify the actual entity_id is `number.standing_desk_set_height` |

## Credits & Sources

Standing on the shoulders of giants:

- **[Rocka84/esphome_components](https://github.com/Rocka84/esphome_components)** — `jiecang_desk_controller` ESPHome component; the core dependency of this project
- **[phord/Jarvis](https://github.com/phord/Jarvis)** — Original reverse engineering of the Jiecang UART protocol
- **[pimp-my-desk/desk-control](https://gitlab.com/pimp-my-desk/desk-control)** — Collected information on various Jiecang controllers
- **[ESPHome](https://esphome.io/)** — The firmware framework and HA integration
- **[Home Assistant](https://www.home-assistant.io/)** — Template Cover / HomeKit Bridge documentation
- **[Mushroom](https://github.com/piitaya/lovelace-mushroom)** — Lovelace card style
- **[Browser Mod](https://github.com/thomasloven/hass-browser_mod)** — HA popup solution
- **[custom-cards/button-card](https://github.com/custom-cards/button-card)** — Dynamic icons and colors for dashboard cards
