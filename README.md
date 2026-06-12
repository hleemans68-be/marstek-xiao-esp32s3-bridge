# Marstek Venus E V2 — Modbus TCP Bridge with XIAO ESP32-S3

An ESPHome configuration that turns a **Seeed XIAO ESP32-S3** with the **Seeed RS485 Breakout Board for XIAO** into a transparent Modbus RTU ↔ TCP bridge for the **Marstek Venus E (V2)** home battery.

It is built for use with the Home Assistant integration
[ViperRNMC/marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus),
but any Modbus TCP client (e.g. evcc) can talk to the battery through it — the bridge supports multiple simultaneous TCP clients.

> **Status: working.** Tested with a Marstek Venus E V2, reading and writing via the Home Assistant integration.

## Hardware

| Part | Notes |
|---|---|
| [Seeed XIAO ESP32-S3](https://www.seeedstudio.com/XIAO-ESP32S3-p-5627.html) | Any XIAO ESP32-S3 variant works |
| [Seeed RS485 Breakout Board for XIAO](https://www.seeedstudio.com/RS485-Breakout-Board-for-XIAO-p-6306.html) | TP8485E transceiver, product 113991354 |
| 4-wire cable to the battery's RS485/Modbus connector | A, B, GND, optionally 5V |

## Pin mapping (the non-obvious part)

The breakout board wires the TP8485E to the XIAO's **D4/D5** pads — *not* to the
XIAO's default hardware UART pins (D6/D7). On the **ESP32-S3** specifically, the
GPIO numbers behind D4/D5 differ from the XIAO ESP32-C3 used in Seeed's wiki
examples, which trips a lot of people up.

| Function | XIAO pad | ESP32-S3 GPIO |
|---|---|---|
| UART TX → transceiver DI | D4 | **GPIO5** |
| UART RX ← transceiver RO | D5 | **GPIO6** |
| Direction enable (DE + /RE shared) | D2 | **GPIO3** |

The board does **not** auto-switch between transmit and receive — the direction
pin must be driven. The bridge component handles this via `de_pin`/`re_pin`
(same GPIO → shared-pin mode is enabled automatically).

## Board switches

| Switch | Position | Why |
|---|---|---|
| IN / OUT | **IN** | The 5V terminal acts as a power *input* |
| ON / NC | **ON** | 120 Ω termination — the bridge sits at the end of a point-to-point bus |

⚠️ Never set the switch to OUT while an external 5V source is wired to the 5V
terminal — two supplies would fight each other.

## Wiring

Battery RS485/Modbus connector → breakout board terminals:

```
A   → A
B   → B
GND → GND
5V  → 5V    (optional: powers the XIAO from the battery)
```

The Marstek Venus speaks Modbus RTU at **115200 baud, 8N1, unit ID 1**.

### Powering options

- **From the battery's 5V** (as wired above): works fine. IN/OUT switch must be on IN.
- **From USB-C**: also fine — then connect only A/B/GND to the battery.
- **Never both at the same time.** Unplug the battery connector before plugging
  in USB for a wired reflash. After the first flash, use OTA updates instead.

## Installation

1. Copy `marstek-xiao-esp32s3-bridge.yaml` into your ESPHome setup.
2. Add to your ESPHome `secrets.yaml`:
   ```yaml
   wifi_ssid: "YourSSID"
   wifi_password: "YourWifiPassword"
   ota_password: "PickSomethingRandom"
   ap_password: "PickSomethingRandom"
   ```
3. Adjust the `manual_ip` block to your network (or delete it and make a DHCP
   reservation in your router — the HA integration connects by IP, so the
   address must not change).
4. Flash the XIAO **via USB on your desk first** (battery disconnected).
5. Wire it to the battery and power it up.

## Home Assistant setup

1. Install [marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus) via HACS.
2. Add the integration: **host** = the bridge's IP, **port** = `502`, **unit ID** = `1`.
3. Select the Venus E V2 device type when prompted.

## Troubleshooting

- **Integration connects but all reads time out** (the "RTU Timeouts" sensor
  keeps climbing): swap `tx_pin` and `rx_pin` in the YAML and reflash. This is
  by far the most common failure mode with these boards.
- **Device reboot-loops when powered from the battery's 5V**: power it from a
  separate USB-C supply instead (A/B/GND only to the battery).
- **No connection at all**: check the static IP / DHCP reservation, and verify
  port 502 is reachable (`nc -zv <ip> 502`).

## Credits

- Bridge component: [rosenrot00/esphome_modbus_bridge](https://github.com/rosenrot00/esphome_modbus_bridge)
- Home Assistant integration: [ViperRNMC/marstek_venus_modbus](https://github.com/ViperRNMC/marstek_venus_modbus)
- ESP32-S3 pin mapping confirmed by [devMobile's RS485 test harness write-up](https://blog.devmobile.co.nz/2025/11/22/seeedstudio-xiao-esp32-s3-rs-485-test-harness-a/)

## License

MIT — see [LICENSE](LICENSE).
