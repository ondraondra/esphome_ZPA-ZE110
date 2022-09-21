# ZPA ZE312 meter reading using IEC 62056-21 with ESPHOME

This example only exposes values allowed to be exposed by ČEZ distribuce

these settings are the main thing you will need to make metter talk to you

```
  baud_rate: 300
  data_bits: 7
  stop_bits: 1
  parity: EVEN
```





# EXAMPLE SETTINGS

```
esphome:
  name: elektromer
esp32:
  board: esp32-c3-devkitm-1
  framework:
    type: arduino
    version: 2.0.3
    platform_version: 4.4.0  

wifi:
  ap:

external_components:
  - source: github://evlo/esphome_ZPA-ZE312
    components: [obis]
    refresh: 0s

api:
logger:
  baud_rate: 0
  #level: VERY_VERBOSE
ota:
web_server:
  version: 2
captive_portal:
mdns:
  disabled: true

uart:
  id: uart_bus
  tx_pin: GPIO21
  rx_pin: GPIO20
  rx_buffer_size: 2048  
  baud_rate: 300
  data_bits: 7
  stop_bits: 1
  parity: EVEN
  debug: 
    direction: BOTH
    dummy_receiver: false
    after:
      delimiter: "\n"
    sequence:
      - lambda: UARTDebug::log_string(direction, bytes);

obis:
  uart_id: uart_bus
  force_update:  # optional
    interval: 10000
    payload: "\x2F\x3F\x21\x0D\x0A"


sensor:
  - platform: obis
    name: "esphome L1"
    channel: "21.8.0"
    unit_of_measurement: kWh
    accuracy_decimals: 3
    device_class: energy
    state_class: total_increasing
  - platform: obis
    name: "esphome L2"
    channel: "41.8.0"
    unit_of_measurement: kWh
    accuracy_decimals: 3
    device_class: energy
    state_class: total_increasing
  - platform: obis
    name: "esphome L3"
    channel: "61.8.0"
    unit_of_measurement: kWh
    accuracy_decimals: 3
    device_class: energy
    state_class: total_increasing

text_sensor:
    - platform: obis
      channel: "0.0.0"
      name: "Serial Number"
```


## ESPHOME

https://esphome.io/

## OBIS

basically using this https://github.com/jplitza/esphome_components

*******

# Tasmota stuff and description of the readouts

Here is what you need to get the metter readouts in tasmota

```
#define WEB_SERVER 2

#undef  STA_PASS1
#define STA_PASS1

#ifndef USE_SCRIPT
#define USE_SCRIPT
#endif

#ifndef USE_SML_M
#define USE_SML_M
#endif

#ifdef USE_RULES
#undef USE_RULES
#endif

#ifdef USE_MP3_PLAYER 
#undef USE_MP3_PLAYER
#undef MP3_VOLUME
#endif

#define SML_OBIS_LINE
//#define USE_SCRIPT_SUB_COMMAND
#define SetOption142 1
#define Timezone 99
#define TimeStd 0,0,10,1,3,60
#define TimeDst 0,0,3,1,2,120
```

```
platformio run -e tasmota32c3
```
```
{"NAME":"T-01C3","GPIO":[0,0,1,544,0,0,0,0,1,1,1,0,0,0,0,0,0,0,0,0,1,1],"FLAG":0,"BASE":1}
```

this is also usefull to understand what data you get from the meter, check ČEZ distribuce documentation what you are allowed to read and what you should not read from the metter 

```
>D
>B
->sensor53 r
>M 1
+1,20,o,0,300,Ele,21,200,2F3F210D0A
1,C.1.0(@1,SN,,sn,0
1,0.0.0(@1,Cust n.,,cn,0

;broken and will parse L1, L2, L3 into this as they also contain 1.8.0
;1,1.8.0(@1,Total in,KWh,Total_in,3
1,1.8.1(@1,T1 in,KWh,T1_in,3
1,1.8.2(@1,T2 in,KWh,T2_in,3

1,21.8.0(@1,L1 in,KWh,L1_in,3
1,41.8.0(@1,L2 in,KWh,L2_in,3
1,61.8.0(@1,L3 in,KWh,L3_in,3

1,2.8.0(@1,Total out,KWh,Total_out,3
;1,22.8.0(@1,L1 out,KWh,L1_out,3
;1,42.8.0(@1,L2 out,KWh,L2_out,3
;1,62.8.0(@1,L3 out,KWh,L3_out,3

;RRMMDDhhmm
;1,C.8.1(@1,T1 period,,t1_period,0
;1,C.8.2(@1,T2 period,,t2_period,0
;1,C.8.0(@1,Operating period in,,period_in,0
;1,C.82.0(@1,Operating period out,,period_out,0

;1,C.7.1(@1,Outages L1,,outages_L1,0
;1,C.7.2(@1,Outages L2,,outages_L2,0
;1,C.7.3(@1,Outages L3,,outages_L3,0

;1,0.2.1(@#4:4,SW,,sw_ver,0
;RRMMDDhhmm
;1,C.2.1(@1,last parametrisation,,last_parametrisation,0
;1,C.2.9(@1,last readout,,last_readout,0

;1,C.3.9(@1,Attacks,,attacks,0
#
````
