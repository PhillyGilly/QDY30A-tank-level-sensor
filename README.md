# Rainwater Harvesting Management in Home Assistant

I have a rainwater collection system to provide "grey water" for flushing the toilets in my house.
The system supplied [Rainwater Harvesting UK](https://www.rainwaterharvesting.co.uk/rainwater-harvesting-gravity-feed-systems/) comprises of a 7,500l storage tank under the drive, a submersible pump and control panel, and a 40l header tank in the loft. The system supplied is basically a "fit-and-forget" in that it just works and if it isn't working it switches over to mains water. It does have LED status indicators on the control panel, but at the time of installation there was no remote monitoring capability. As a consequence when the pump failed aftert three years, I was unaware for some time that I was using expensive drinking water to flush my toilets.

Initially I considered implementing a small system that would monitoring the voltages on the control panel LEDs and realay those by wifi and mqtt to my Home Assistant, I discounted that as unecessarily invasive. My approach was to install a tasmotised Sonoff R2 power monitor on the supply to the pump. The Sonoff device works with HA by the Tasmota integration. Everything is pretty standard:

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
and
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

This gave me a simple ooverview of my pump operation and status:

![image](https://github.com/user-attachments/assets/996b4e93-9c18-4a41-9cc9-92a99cc6c94b)

However I was still interested to know how much rainwater I had in my underground storage tank and whether or not the design of roof area was providing sufficient water to meet the house's demand.
About this time Rainwater Harvesting launched a new undergound, [in tank monitoring system](https://www.rainwaterharvesting.co.uk/product/rain-acculevel-level-sensor-for-f-line-tanks/) and although this looks quite neat, it is a cloud based sytem accessed via a proprietary app and therefore cannot be integrated into Home Assistant (or any other system)

So I decided to independently build my own tank level sensor based on the modbus version of the [QDY30A product](https://a.aliexpress.com/_EjQJvbW).
I chose this as wanted a waterpressure depthmeasurement, not an ultrasonic level probe, and I went for modbus because I had recent successful experience of an Arduino nano-33 based modbus interface to a Solaris solar inverter.
At the time Arduino products had become difficult to obtain and were also overkill for this application, so I decided to switch to a WemosD-1 product.
The QDY30A needs a 20V supply and the Wemos needs 5V so I got this [power supply](https://www.ebay.co.uk/itm/295877465206?var=594084681451) from eBay.
The [TTL-RS485 interface](https://a.aliexpress.com/_EHIAcRK) came from AliExpress and the [Wemos-D1](https://www.ebay.co.uk/itm/363891216533) also came from eBay.

After assembling the hardware as below: 

I stumbled across [this discussion](https://community.home-assistant.io/t/water-level-sensor-qdy30a-modbus-rs485-with-esp32-s2-mini/698712/4) regarding a very similar application.
Since I was having a few problems porting my arduino mqtt code to wemos, I decided to jump on the EspHome bus.


![2025-01-30 17 16 27](https://github.com/user-attachments/assets/f091f2ce-fabc-4518-b8aa-d546361701b8)

![image](https://github.com/user-attachments/assets/0c1adee1-340c-4464-98e8-3765311c9c77)

