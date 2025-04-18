# Rainwater Harvesting Management in Home Assistant

I have a rainwater collection system to provide "grey water" for flushing the toilets in my house.
The system supplied [Rainwater Harvesting UK](https://www.rainwaterharvesting.co.uk/rainwater-harvesting-gravity-feed-systems/) comprises of a 7,500l storage tank under the drive, a submersible pump and control panel, and a 40l header tank in the loft. The system supplied is basically a "fit-and-forget" in that it just works and if it isn't working it switches over to mains water. It does have LED status indicators on the control panel, but at the time of installation there was no remote monitoring capability. As a consequence when the pump failed after three years, I was unaware for some time that I was using expensive drinking water to flush my toilets.

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
The [TTL-RS485 interface](https://a.aliexpress.com/_EHIAcRK) came from AliExpress as did the [ESP32S WROOM](https://a.aliexpress.com/_EzcOL0M) and the enclosure.

I stumbled across [this discussion](https://community.home-assistant.io/t/water-level-sensor-qdy30a-modbus-rs485-with-esp32-s2-mini/698712/4) regarding a very similar application.
Since I was having a few problems porting my arduino mqtt code to ESP32, I decided to jump on the EspHome bus.

The yaml code is:
```
esphome:
  name: "esp32-water-level-sensor"
  friendly_name: ESP32 Water Level Sensor
  min_version: 2024.11.0
  name_add_mac_suffix: false

esp32:
  board: esp32dev
  framework:
    type: esp-idf

logger:
  level: DEBUG #VERY_VERBOSE
  baud_rate: 9600
  #baud_rate: 0

api:

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
  update_interval: 15s #60s

sensor:
# Reports how long the device has been powered (in days)
  - platform: uptime
    name: "Uptime Days"
    filters:
      - lambda : return (x / 86400.0) ;
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

The Fritzing sketch shows the logical connections: 

![image](https://github.com/user-attachments/assets/d7287d88-5745-48a2-b78c-34bd0d4b751c)

Which (forgive the soldering!) looks like this on a breadboard and assembles like this:
![2025-03-03 13 44 24](https://github.com/user-attachments/assets/d9707095-8c6e-4e6b-8a76-1e9f29e51792)
![2025-03-03 13 46 38](https://github.com/user-attachments/assets/6df39f71-0340-40f3-80d0-93aa304dcf99)
The electronics in my house are connected to the level sensor in the tank by approx 12m of 4-core cable and a junction box in the neck of the tank.
![2025-02-27 11 40 04](https://github.com/user-attachments/assets/f4d7414c-6e33-402f-bf08-966de9269ef5)

This creates a new device in Home Assistant:
![image](https://github.com/user-attachments/assets/16e215a6-09b7-4e6d-aa66-c957d62e8bb9)

The final piece of the jigsaw is to monitor local rainfall so that we know what is going into the tank.
This could be through a local connected sensor or else caould be from the recently launched [DEFRA check for flooding service] (https://check-for-flooding.service.gov.uk/river-and-sea-levels).
When you have determined your nearest monitoring station in the form Xnnnn, the data can be presented in Home Assistant by using a "scrape" and the code is:

```
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
What about cumulative rainfall? Is this reliable?
```
- trigger:
    - platform: time
      at: "11:00:01"
  sensor:
    - name: "Cumulative Rainfall"
      unique_id: cumulativerainfall
      unit_of_measurement: mm
      icon: mdi:sigma
      state: >
        {% if  (states('sensor.cumulative_rainfall') == 'unknown') %}
           {% set Cumulative_Rainfall = states('sensor.rainfall_last_24hr')|float %}
        {% else %}
          {% set Cumulative_Rainfall = states('sensor.rainfall_last_24hr')|float + states('sensor.cumulative_rainfall')|float %}
        {% endif %}
        {{Cumulative_Rainfall|round(1)}}
```

The resultant display in HA is:
![image](https://github.com/user-attachments/assets/8c9c36bf-9c66-452f-83e1-dc22eca99810)

ps you can also add flood risk:
```      
- resource: https://check-for-flooding.service.gov.uk/station/6038
  scan_interval: 21600
  sensor:
    - name: "Flood Height"
      unique_id: floodheight
      icon: mdi:home-flood
      select: "#main-content > div:nth-child(3) > div > div > dl > div:nth-child(1) > dd"
      value_template: '{{ value.split("m")[0] }}'
      state_class: measurement
      unit_of_measurement: m
    
    - name: "Flood Trend"
      unique_id: floodtrend
      icon: mdi:home-flood
      select: "#main-content > div:nth-child(3) > div > div > dl > div:nth-child(2) > dd"
      value_template: '{{ value|string }}'

    - name: "Flood State"
      unique_id: floodstate
      icon: mdi:home-flood
      select: "#main-content > div:nth-child(3) > div > div > dl > div:nth-child(3) > dd"
      value_template: '{{ value|string }}'
```
