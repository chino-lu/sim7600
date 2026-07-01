# SIM7600 ESPHome External Component

An [ESPHome](https://esphome.io/) external component that adds support for the **SIMCom SIM7600** series 4G/LTE modules (SMS, voice calls, USSD, and mobile data) to any ESP32/ESP8266 device via UART and AT commands.

## Features

- 📩 **SMS** — send SMS messages and receive incoming SMS as an event (sender + message)
- 📞 **Voice calls** — dial out, and get notified on incoming calls, call connect, and call disconnect
- 🔢 **USSD** — send USSD codes (e.g. prepaid balance checks) and receive the response
- 🌐 **Mobile data** — connect/disconnect the module's PPP/data connection
- 📶 **Diagnostics** — optional RSSI (signal strength) and network status sensors, plus a "registered on network" binary sensor

## Requirements

- A SIM7600-based module (e.g. SIM7600E, SIM7600X, SIM7600G-H) wired to your ESP32/ESP8266 via UART (TX/RX)
- A **PIN-free SIM card**. If the SIM has a PIN, ESPHome logs will show `Error code 3` on `AT+CPIN=1` (or similar), and the module's status LED will stay solid instead of blinking.
- [ESPHome](https://esphome.io/) installed (Home Assistant Add-on, CLI, or standalone)

## Installation

Reference this repository directly as an external component in your ESPHome YAML — no manual file copying required:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/chino-lu/sim7600
    components: [sim7600]
```

Alternatively, clone/download this repository and reference it locally:

```yaml
external_components:
  - source:
      type: local
      path: components   # folder containing this repo's files, e.g. esphome/components/sim7600
```

## Wiring example

| ESP32 pin | SIM7600 pin |
|-----------|-------------|
| TX (e.g. GPIO18) | RX |
| RX (e.g. GPIO33) | TX |
| GND | GND |
| 5V/VUSB | VCC |

Use whichever free GPIOs you like for TX/RX — just match them in the `uart:` section below.

## Configuration example

```yaml
uart:
  id: sim7600_uart
  tx_pin: GPIO18
  rx_pin: GPIO33
  baud_rate: 115200

sim7600:
  id: modem
  uart_id: sim7600_uart
  on_sms_received:
    - lambda: |-
        ESP_LOGI("sim7600", "SMS from %s: %s", sender.c_str(), message.c_str());
  on_incoming_call:
    - lambda: |-
        ESP_LOGI("sim7600", "Incoming call from %s", caller_id.c_str());
  on_call_connected:
    - logger.log: "Call connected"
  on_call_disconnected:
    - logger.log: "Call disconnected"
  on_ussd_received:
    - lambda: |-
        ESP_LOGI("sim7600", "USSD reply: %s", ussd.c_str());

sensor:
  - platform: sim7600
    sim7600_id: modem
    rssi:
      name: "Modem RSSI"
    network:
      name: "Network status"

binary_sensor:
  - platform: sim7600
    sim7600_id: modem
    registered:
      name: "Modem registered"
```

### Actions

The component exposes the following actions for use in automations, buttons, or Home Assistant services:

```yaml
# Send an SMS
- sim7600.send_sms:
    id: modem
    recipient: "+33123456789"
    message: "Hello from ESPHome!"

# Dial a number
- sim7600.dial:
    id: modem
    recipient: "+33123456789"

# Send a USSD code
- sim7600.send_ussd:
    id: modem
    ussd: "*123#"

# Open / close the mobile data connection
- sim7600.connect:
    id: modem
- sim7600.disconnect:
    id: modem
```

### Exposing actions as Home Assistant services

You can expose these actions directly as Home Assistant services via the ESPHome `api:` block:

```yaml
api:
  services:
    - service: send_sms
      variables:
        recipient: string
        message: string
      then:
        - sim7600.send_sms:
            id: modem
            recipient: !lambda 'return recipient;'
            message: !lambda 'return message;'
```

## Triggers

| Trigger | Provides | Description |
|---|---|---|
| `on_sms_received` | `sender`, `message` | Fired when a new SMS arrives |
| `on_incoming_call` | `caller_id` | Fired when the modem detects an incoming call |
| `on_call_connected` | – | Fired when a call is answered/connected |
| `on_call_disconnected` | – | Fired when a call ends |
| `on_ussd_received` | `ussd` | Fired when a USSD response is received |

## Troubleshooting

- **Steady green LED / `Error code 3`**: your SIM card has a PIN. Remove it (e.g. with a phone) before use.
- **No response from the module**: double-check TX/RX are not swapped and that the baud rate matches your module (commonly `115200`).
- **SMS sending fails / times out**: ensure the SIM has SMS credit/plan and adequate signal (check the RSSI sensor).

## Credits

Component developed by [chino-lu](https://github.com/chino-lu).

## License

See the repository for license details.
