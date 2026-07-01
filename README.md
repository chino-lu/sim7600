# Home assistant ESPHome Addon config

### Yaml code for module:
```
esphome:
  name: esphome-web-014ffc
  friendly_name: ES32Relay
esp32:
  board: esp32-s2-saola-1
  framework:
    type: arduino
    
external_components:
  - source:
      type: local
      path: components

# Enable logging
logger:


# Enable Home Assistant API
api:
  encryption:
    key: "XXX"
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
    - service: dial
      variables:
        recipient: string
      then:
        - sim7600.dial:
            id: modem
            recipient: !lambda 'return recipient;'
    - service: connect
      then:
        - sim7600.connect
    - service: disconnect
      then:
        - sim7600.disconnect
    - service: send_ussd
      variables:
        ussdCode: string
      then:
        - sim7600.send_ussd:
            ussd: !lambda 'return ussdCode;'


ota:
  - platform: esphome
    password: "pwdXXXota"




wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Esphome-Web-014Ffc"
    password: "pwdXXXHotspot"


captive_portal:

text_sensor:
  - platform: template
    id: sms_sender
    name: "Sms Sender"
  - platform: template
    id: sms_message
    name: "Sms Message"
  - platform: template
    id: caller_id_text_sensor
    name: "Caller ID"
  - platform: template
    id: ussd_message
    name: "Ussd Code"


switch:
  - platform: gpio
    name: "SIM800_PWKEY"
    pin: 4
    restore_mode: ALWAYS_OFF
  - platform: gpio
    name: "SIM800_RST"
    pin: 5
    restore_mode: ALWAYS_ON
    internal: true
  - platform: gpio
    name: "SIM800_POWER"
    pin: 6
    restore_mode: ALWAYS_ON
    internal: true
  - platform: gpio
    id: relay_1
    name: "PowerOn_Relay"
    pin: 16
    restore_mode: ALWAYS_ON
  - platform: restart
    name: "Sim800L Restart"


uart:
  tx_pin: GPIO18
  rx_pin: GPIO33
  baud_rate: 115200
  debug:
    direction: BOTH
    dummy_receiver: false
    after:
      delimiter: "\n"
    sequence:
      - lambda: UARTDebug::log_string(direction, bytes);


sim7600:
  id: modem
  on_sms_received:
    - lambda: |-
        id(sms_sender).publish_state(sender);
        id(sms_message).publish_state(message);
        if ( (id(sms_sender).state == "+33123456789") && ( (id(sms_message).state == "PowerOff") || (id(sms_message).state == "poweroff") ) ) {
          id(relay_1).turn_off();
        }
        if ( (id(sms_sender).state == "+33123456789") && ( (id(sms_message).state == "PowerOn") || (id(sms_message).state == "poweron") ) ) {
          id(relay_1).turn_on();
        }
  on_incoming_call:
    - lambda: |-
        id(caller_id_text_sensor).publish_state(caller_id);
  on_call_connected:
    - logger.log:
        format: Call connected
  on_call_disconnected:
    - logger.log:
        format: Call disconnected
  on_ussd_received:
    - lambda: |-
        id(ussd_message).publish_state(ussd);
 ```      

### Home Assistant folders structure:
Copy the git repo files from https://github.com/smillier/sim7600.git in esphome/components/sim7600 
![VSCode components](https://github.com/smillier/sim7600/blob/main/HomeAssistant_FilesLocation.png)

SIM Card on the sim7600 module must be PIN free. If there is a PIN code on your SIM card, the ESPHome logs will show Error code 3 when running the AT+CMFG=1 command. The sim7600 will also show a steady green led. When there is no PIN code on SIM, the green led will blink.

### sim7600 module wiring example 
| Wemos S2 Mini | SIM7600 | Relay board |
|---------------|---------|-------------|
| GPIO18 - TX   | R       |             |
| GPIO33 - RX   | T       |             |
| GPIO4         | K       |             |
| VUSB/5V       | V       | VCC         |
| GND/GND       | G       | GND         |
| GPIO16        |         | IN1         |


Special thanks to Chino-Lu for his work on the sim7600. All the code here is from his repo: https://github.com/chino-lu/sim7600.git
