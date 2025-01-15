# ZPA ZE110 meter reading using IEC 62056-21 with ESPHOME

## tl;dr

Reading PREdistribuce ZPA ZE110 powre meter from 2009 data to into Home Assistant<br>
Buy the HW, copy the example code, you're good to go

## HW Used: 

- [ESP32-C3 MINI](https://a.aliexpress.com/_mKSMj9N)
- [TTL to IR Infrared IEC62056 optical probe](https://a.aliexpress.com/_mqWDXiR) (too lazy to hack this together and I needed to have at least "not a bomb" look for my apartment building)
- random headers from bottom of my parts drawer, but you can solder the 4 wires directly

If anything is sold out, use the keyword to find similar parts

###  HW example ESP32C3 Supermini
![110](https://github.com/user-attachments/assets/18b5d382-b6fc-4262-b59d-773d75efb912)

### Pinout 
```
## ESP32 - IR head
GND    - GND black
3V3    - VCC red
GPIO20 - RXD yellow
GPIO21 - TXD white
```

# The long read

This example exposes values which I was interested in during my research, always check with your energy distributor what's allowed to be read by the consumer

These serial settings are the main thing you will need to make metter talk to you
```
  baud_rate: 300
  data_bits: 7
  stop_bits: 1
  parity: EVEN
```

There is a IEC handshake required, the meter will simply not spit out the information like some newer models. 
I hacked together a handshake reply according to this [blog post](https://www.gurux.fi/Gurux.DLMS.Iec#ModeECPlusPlus&quot)

```
/XXXZ Ident and a new line
XXX is the manufacturer name
Z is the maximum baud rate
Ident is the Identification info of the device.
Values of Z are:
0 - 300 Bd
1 - 600 Bd
2 - 1 200 Bd
3 - 2 400 Bd
4 - 4 800 Bd
5 - 9 600 Bd
6 - 19200Bd
7, 8, 9 - reserved for later extensions.
```

- First you need to send ``/?!\r\n``
- Then the meter will introduce itself  ``/ZPA4ZE110.v20_003\r\n``
- I was too lazy to implement baud rate switching, so despite support for 4800baud, we continue reading at default 300 ``\x06100\r\n``

All data is read within 19 seconds, this should be enough for comfortable 30 second refresh rate.

As the meter only reports whole numbers, I added two ``template`` sensors to calculate VA (U x I) and Watts (U x I x phase)


# Example ESPHome code

```
esphome:
  name: esp-meter
  friendly_name: esp-meter

esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino
    version: 2.0.3
    
wifi:
  ssid: XXX
  password: XXXX
  power_save_mode: light

external_components:
  - source: github://ondraondra/esphome_ZPA-ZE110
    components: [obis]
    refresh: 0s

api:
  encryption:
    key: "XXXX"

ota:
  - platform: esphome
    password: "XXXX"

logger:
  baud_rate: 0
  #level: VERY_VERBOSE
web_server:
  version: 2
captive_portal:

uart:
  id: uart_bus
  tx_pin: GPIO21
  rx_pin: GPIO20
  rx_buffer_size: 2048  
  baud_rate: 300
  data_bits: 7
  stop_bits: 1
  parity: EVEN
  # dev leftovers, handy for figuring out the OBIS codes
  #debug: 
    #direction: BOTH
    #dummy_receiver: false
    #after:
    #  delimiter: "\n"
    #sequence:
    #  - lambda: UARTDebug::log_string(direction, bytes);

interval:
  - interval: 30sec
    then:
      - uart.write:
          data: [0x2F, 0x3F, 0x21, 0x0D, 0x0A]
      - delay: 2000ms
      - uart.write:
          id: uart_bus
          data: [0x06, 0x31, 0x30, 0x30, 0x0D, 0x0A]          

obis:
  uart_id: uart_bus

sensor:
  - platform: obis
    name: "esphome total"
    channel: "1.8.0"
    unit_of_measurement: kWh
    accuracy_decimals: 3
    device_class: energy
    state_class: total_increasing
  
  - platform: obis
    name: "esphome T1 amp"
    channel: "11.4"
    id: "esphome_T1_amp"
    unit_of_measurement: A
    accuracy_decimals: 3
    device_class: energy
  - platform: obis
    name: "esphome T1 voltage"
    channel: "12.4"
    id: "esphome_T1_voltage"
    unit_of_measurement: V
    accuracy_decimals: 3
    device_class: energy
  - platform: obis
    name: "esphome T1 phase"
    channel: "13.4"
    id: "esphome_T1_phase"
    unit_of_measurement: cos
    accuracy_decimals: 3
    device_class: energy
  - platform: template
    name: "esphome T1 (Watts)"
    id: power_sensor_watt
    unit_of_measurement: "W"
    accuracy_decimals: 2
    lambda: |-
      return id(esphome_T1_voltage).state * id(esphome_T1_amp).state * id(esphome_T1_phase).state;
    update_interval: 10s
  - platform: template
    name: "esphome T1 (VA)"
    id: power_sensor_va
    unit_of_measurement: "VA"
    accuracy_decimals: 2
    lambda: |-
      return id(esphome_T1_voltage).state * id(esphome_T1_amp).state;
    update_interval: 10s

text_sensor:
    - platform: obis
      channel: "0.0"
      name: "Serial Number"
```


## Total energy for energy usage

The reported total value is missing unit for T1 and is overall imprecisse, rounded up to whole kWh.

To get more precision for Home Assisntats energy usage tab, I pool current consumption values and compute the values instead. <br>
You can hide the volts, amps and power factor, they are only used to compute the resulting Watts.

1. Settings > Devices & Services > Helpers tab.
2.	Click + Create Helper > Integration - Riemann sum integral.
3.	Name it something like "Total Energy Consumption".
4.	Choose the "esphome T1 (Watts)" computed consumption sensor as the input sensor.
5.	Set the Method to trapezoidal for better accuracy.
6.	Choose Unit Prefix as k (for kWh).
7.	Set Unit of Measurement to kWh.
8.  Set accuracy to 3 decimal points

## ESPHome

https://esphome.io/

## OBIS

Yet another fork of a fork, thank you guys for your work so far! 🙌<br>
https://github.com/evlo/esphome_ZPA-ZE312<br>
https://github.com/jplitza/esphome_components

Good reading on OBIS codes can be found [here](https://onemeter.com/docs/device/obis/) and [here](https://www.promotic.eu/cz/pmdoc/Subsystems/Comm/PmDrivers/IEC62056_OBIS.htm)


## Power Meter ZE110.D
Manufacturer leaflet and specs for the meter<br>
https://www.ceha-kdc.cz/eshop/cat/55/55-06-548-05480.pdf

### Reported values by the meter

These are valid for my meter, reverse engineered via random Google searches & experiments, your mileage may vary

```
Found field '<0x02>F.F' with value '000000' and unit '' # ???
Found field '0.0' with value 'CXXXXXX' and unit ''	# meter ID (on sticker)
Found field 'C.1' with value 'XXXXXXXX' and unit '' # serial number 
Found field '0.1' with value '000000' and unit '' # ?
Found field '1.8.1' with value '0035749#kWh' and unit '' #  active power T1 from grid, unit is messed up = unsuable
Found field '1.8.2' with value '0000001' and unit 'kWh' #  active power T2 from grid
Found field '1.8.3' with value '0000001' and unit 'kWh' #  active power T3 from grid
Found field '1.8.4' with value '0000001' and unit 'kWh' #  active power T4 from grid
Found field '1.8.0' with value '0035753' and unit 'kWh' # total active power from grid 
Found field '2.8.1' with value '0000001#kWh' and unit '' #  active power T1 to grid, unit is messed up = unsuable
Found field '2.8.2' with value '0000000' and unit 'kWh' #  active power T2 to grid
Found field '2.8.3' with value '0000000' and unit 'kWh' #  active power T3 to grid
Found field '2.8.4' with value '0000000' and unit 'kWh' #  active power T4 to grid
Found field '2.8.0' with value '0000001' and unit 'kWh' # total active power from grid 
Found field '0.9.2' with value '24-10-09 11:05' and unit '' # current date & time Y-M-D h:m (off by ~3 months after 16 years on my meter)
Found field 'C.7.0' with value '0068' and unit '' # Event power down - counter
Found field '0.C.1' with value '09-03-20 09:12' and unit '' # installation date & time Y-M-D h:m
Found field '0.2.0' with value '0063' and unit '' # Firmware version
Found field '0.2.1' with value 'PRE-ZE110-000001' and unit '' # Parameters scheme ID
Found field '0.2.4' with value '4441' and unit '' # ???
Found field 'C.8.1' with value '33931:21#h:min' and unit '' # ??? possibly total time in T1 from grid
Found field 'C.8.2' with value '00000:06' and unit 'h:min' # ??? possibly total time in T2 from grid
Found field 'C.8.3' with value '00000:06' and unit 'h:min' # ??? possibly total time in T3 from grid
Found field 'C.8.4' with value '00000:06' and unit 'h:min' # ??? possibly total time in T4 from grid
Found field 'C.8.0' with value '33931:39' and unit 'h:min' # ??? possibly total meter time from grid
Found field 'C.9.1' with value '00000:08#h:min' and unit '' # ??? possibly total time in T1 to grid
Found field 'C.9.2' with value '00000:00' and unit 'h:min' # ??? possibly total time in T2 to grid
Found field 'C.9.3' with value '00000:00' and unit 'h:min' # ??? possibly total time in T3 to grid
Found field 'C.9.4' with value '00000:00' and unit 'h:min' # ??? possibly total time in T4 to grid
Found field 'C.9.0' with value '00000:08' and unit 'h:min' # ??? possibly total meter time to grid
Found field 'C.50' with value '33934:18' and unit 'h:min' # ???
Found field '12.4' with value '237.5' and unit 'V' # current voltage
Found field '11.4' with value '000.968' and unit 'A' # current current :))
Found field '13.4' with value '0.621' and unit 'cos' # current  power factor
Found field '11.6' with value '010.835' and unit 'A' # all time high current through meter
Found field '1.6.0' with value '02.544' and unit 'kW' # all time high power through meter 
Found field '0.6.0' with value '02341:52' and unit 'h:min' # ??? time stamp for ATH event
```

*******


