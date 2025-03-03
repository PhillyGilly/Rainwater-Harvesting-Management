# Rainwater Harvesting Management in Home Assistant

I have a rainwater collection system to provide "grey water" for flushing the toilets in my house.
The system supplied [Rainwater Harvesting UK](https://www.rainwaterharvesting.co.uk/rainwater-harvesting-gravity-feed-systems/) comprises of a 7,500l storage tank under the drive, a submersible pump and control panel, and a 40l header tank in the loft. The system supplied is basically a "fit-and-forget" in that it just works and if it isn't working it switches over to mains water. It does have LED status indicators on the control panel, but at the time of installation there was no remote monitoring capability. As a consequence when the pump failed aftert three years, I was unaware for some time that I was using expensive drinking water to flush my toilets.

Initially I considered implementing a small system that would monitoring the voltages on the control panel LEDs and realay those by wifi and mqtt to my Home Assistant, I discounted that as unnecessarily invasive. My approach was to install a tasmotised Sonoff R2 power monitor on the supply to the pump. The Sonoff device works with HA by the Tasmota integration. Everything is pretty standard:

![image](https://github.com/user-attachments/assets/2f1de149-dd33-417e-b867-51ffc1ef204d)
The Power reading is used to detect when the pump is running and this sets the state of a template sensor "Rain Pump Status".  Changes in Rain Pump Status our counted by a helper counter "Rain Pump Counter" and a simple automation. The "Rain volume" and "Rain Pump Last Run" are aslo created by template sensors in homeassistant\template.yaml:
```
#################################################################
# rainwater                                                     #
#################################################################
- sensor:
    - name: "Rain Pump Status"
      unique_id: rainpumpstatus
      state: >
        {% set Rain_Pump_Status = 'Off' %}
        {% if states('sensor.rain_pump_energy_power')|float > 10 %}
          {% set Rain_Pump_Status = 'On' %}
        {% endif %}
        {{ Rain_Pump_Status }}

    - name: Rain Volume
      unique_id: rainvolume
      unit_of_measurement: l
      state: >
        {{ states('counter.rain_pump_count')|int * 40 }}
      icon: mdi:water-alert

    - name: Rain Pump Last Run
      unique_id: rainpumplastrun
      state: >
        {{ as_timestamp(states.sensor.rain_pump_status.last_changed)|timestamp_custom('%A %B %-d, %H:%M') }}
```
and in homeassistant\automations.yaml:
```
alias: 15. Rain Pump Count Increment
description: ""
mode: single
triggers:
  - entity_id:
      - sensor.rain_pump_status
    for:
      hours: 0
      minutes: 0
      seconds: 0
    from: "Off"
    to: "On"
    trigger: state
conditions: []
actions:
  - data: {}
    target:
      entity_id: counter.rain_pump_count
    action: counter.increment
```

This gave me a simple overview of my pump operation and status:

![image](https://github.com/user-attachments/assets/996b4e93-9c18-4a41-9cc9-92a99cc6c94b)

However I was still interested to know how much rainwater I had in my underground storage tank and initially was interested in the Rainwater Harvesting new undergound, [in tank monitoring system](https://www.rainwaterharvesting.co.uk/product/rain-acculevel-level-sensor-for-f-line-tanks/) Although this looks quite neat, it is a cloud based sytem accessed via a proprietary app and therefore cannot be integrated into Home Assistant (or any other system) which wasn't going to work for me.

So I decided to independently build my own tank level sensor based on the modbus version of the [QDY30A product](https://a.aliexpress.com/_EjQJvbW).
I chose this as wanted a waterpressure depth measurement, not an ultrasonic level probe, and I went for modbus because I had recent successful experience of an Arduino nano-33 based modbus interface to a Solaris solar inverter.
At the time Arduino products had become difficult to obtain and were also overkill for this application, so I decided to switch to an ESP32 product.
The QDY30A needs a 20V supply and the ESP32 needs 5V so I got this [power supply](https://www.ebay.co.uk/itm/295877465206?var=594084681451) from eBay.
The [TTL-RS485 interface](https://a.aliexpress.com/_EHIAcRK) came from AliExpress as did the [ESP32S WROOM](https://a.aliexpress.com/_EzcOL0M)

After assembling the hardware as below: 

![QDY30A level monitor_bb](https://github.com/user-attachments/assets/a1b994a4-3a81-4181-8b0d-a5d6919e8bb6)

![2025-01-30 17 16 27](https://github.com/user-attachments/assets/b723df25-0618-4a86-b751-4bb50e795e4c)


I stumbled across [this discussion](https://community.home-assistant.io/t/water-level-sensor-qdy30a-modbus-rs485-with-esp32-s2-mini/698712/4) regarding a very similar application.
Since I was having a few problems porting my arduino mqtt code to ESP32, I decided to jump on the EspHome bus.
My yaml code is:
```
esphome:
  name: "water-level-sensor"
  friendly_name: Wemos Mini D1 - Water Level Sensor
  min_version: 2024.11.0
  name_add_mac_suffix: false

esp8266:
  board: esp01_1m #Pinout: WemosMini1D.jpg

# Enable logging  
logger:
  level: DEBUG #VERY_VERBOSE
  baud_rate: 0

# Enable Home Assistant API
api:

# Allow Over-The-Air updates
ota:
- platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap: {}

captive_portal:

web_server:
  port: 80
  version: 2

time:
  - platform: homeassistant
    id: homeassistant_time

uart:
  tx_pin: GPIO1
  rx_pin: GPIO3
  baud_rate: 9600
  data_bits: 8
  stop_bits: 1
  parity: NONE
  #debug:

modbus:
  id: modbus1

modbus_controller:
- id: modbus_device
  address: 0x01   ## address of the Modbus slave device on the bus
  modbus_id: modbus1
  setup_priority: -10
  update_interval: 5s

sensor:
# Reports how long the device has been powered (in days)
  - platform: uptime
    name: Uptime
    filters:
      - lambda : return x / 86400.0 ) }
    unit_of_measurement: days
 
# from register 0x002 we know that the units are cm
# but from register 0x003 there is a decimal shift of one place 
  - platform: modbus_controller
    modbus_controller_id: modbus_device
    name: "Water level"
    id: modbus_water_level
    register_type: holding
    address: 0x0004
    unit_of_measurement: mm
    value_type: S_WORD
```
This creates two sensors which are shone in Home Assistant as:

![image](https://github.com/user-attachments/assets/0c1adee1-340c-4464-98e8-3765311c9c77)

The level sensor is presently installed in the header tank and the steps down on the history graph represent discrete toilet flushes.

# Next steps are:
to move it into the underground tank. I am going to extend the lead by soldering its connectors to a legth of four core cable.
The joints will be protected in a small junction box to be screwed to the side of the tank neck, with the extending cable being pulled back through the power/water duck into the house where the electrronics will be located in one or more small enclosures.
I would like to complete this before the end of spring 2025, so that I can monitor how water is used over the summer.

![2025-02-27 10 45 38](https://github.com/user-attachments/assets/34e587ce-ff85-4351-a611-93df8bead9cd)

![2025-02-27 11 40 15](https://github.com/user-attachments/assets/566e1dc8-b1cd-4732-bc1b-d38faee6233b)

![2025-03-03 13 44 24](https://github.com/user-attachments/assets/85ed4607-e059-443d-9786-0661f164204c)

![2025-03-03 13 46 38](https://github.com/user-attachments/assets/8aa2dcc8-9ef9-4b64-be4f-aa1e4a45b83d)


https://check-for-flooding.service.gov.uk/river-and-sea-levels

```
scrape: !include scrape.yaml

- resource: https://check-for-flooding.service.gov.uk/rainfall-station/Xnnnn
  scan_interval: 21600
  sensor:
    - name: "Rainfall last 24hr"
      unique_id: rainfalllast24hr
      icon: mdi:weather-rainy
      select: "#main-content > div:nth-child(3) > div > div > dl > div:nth-child(3) > dd"
      value_template: '{{ value.split("m")[0] }}'
      state_class: measurement
      unit_of_measurement: mm
```
