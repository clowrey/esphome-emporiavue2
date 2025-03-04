For issues, please go to [the discussion board](https://github.com/emporia-vue-local/esphome/discussions).

⚠️NOTICE⚠️: If you have already flashed your device with a previous version of the config, I'd strongly encourage you to add `flash_write_interval: "48h"` from below to your config to preserve the flash memory's health and your ability to update the device in the future.

**ESPHome Documentation:** https://esphome.io/

<details>
<summary>Instructions changelog</summary>

- 2023-09-11: reduce logging verbosity
- 2023-09-03: revamp configuration for improved accuracy, thanks to [adam](https://www.technowizardry.net/2023/02/local-energy-monitoring-using-the-emporia-vue-2/) and [@kahrendt](https://github.com/kahrendt)
- 2023-06-11: fix buzzer with GND, move LED to HA config section, add template classes
- 2023-03-08: configuration example for net metering
- 2023-02-20: update style to modern home assistant, add buzzer support, add led support
- 2023-01-28: add frequency support
- 2023-01-18: increase flash write interval
- 2022-12-07: switch suggested branch back to dev
- 2022-07-30: add home assistant instructions & MQTT FAQ.
- 2022-07-16: mention using UART adaptor's RTS pin, thanks to @PanicRide
- 2022-07-02: mention mqtt is now supported
- 2022-04-30: bump software version number to 2022.4.0
- 2022-05-04: mention 64-bit ARM issues in FAQ

</details>

# Setting up Emporia Vue 2 with ESPHome

![example of hass setup](https://i.imgur.com/hC26j2M.png)

## What you need

- USB to serial converter module
  - I tested this with a cheap & generic CH340G adapter
- 4 male-to-female jumper wires
- 4 male pcb-mount headers
- Soldering iron & accessories
  - [some recommendations here](https://www.reddit.com/r/AskElectronics/wiki/soldering)
- [esptool.py](https://github.com/espressif/esptool) ([windows instructions](https://cyberblogspot.com/how-to-install-esptool-on-windows-10/), [generic instructions](https://docs.espressif.com/projects/esptool/en/latest/esp32/installation.html))
- ESPHome image

## Panel installation, part 1

You'll want to install the clamps & wiring harness into your panel following the instructions at https://www.emporiaenergy.com/installation-guides. At this time, place a label on each wire using masking tape & a pen rather than connecting them to the energy monitor.

Next, we need to figure out which circuits are on which phases, and in the case of multi-pole breakers, the multiplier. There should be a label like the following on your panel:
![panel phase diagram](https://i.imgur.com/GkoaLzp.jpeg)
For each clamp, you want to make a note of the following information:

- clamp number
- circuit number
- phase
- multiplier, if it is a multi-pole breaker

For the wiring harness, you'll want to make a note of which color cable matches which service main clamp (A, B, C).

## Writing configuration

Here's a starting point for a configuration, save it to `<yourfilename>.yaml` into project folder:

```yaml
esphome:
  name: emporiavue2
  friendly_name: vue2

external_components:
  - source: github://emporia-vue-local/esphome@dev
    components:
      - emporia_vue

esp32:
  board: esp32dev
  framework:
    type: esp-idf
    version: recommended

# Enable Home Assistant API
api:
  encryption:
    # Encryption key is generated by the new device wizard.
    key: "<generated_key_from_new_device_wizard>"

  services:
    - service: play_rtttl
      variables:
        song_str: string
      then:
        - rtttl.play:
            rtttl: !lambda 'return song_str;'

ota:
  # Create a secure password for pushing OTA updates.
  password: "<secure_password>"

# Enable logging
logger:
  logs:
    # by default, every reading will be printed to the UART, which is very slow
    # This will disable printing the readings but keep other helpful messages
    sensor: INFO

wifi:
  # Wifi credentials are stored securely by new device wizard.
  ssid: !secret wifi_ssid
  password: !secret wifi_password

preferences:
  # the default of 1min is far too short--flash chip is rated
  # for approx 100k writes.
  flash_write_interval: "48h"

output:
  - platform: ledc
    pin: GPIO12
    id: buzzer
  - platform: gpio
    pin: GPIO27
    id: buzzer_gnd

rtttl:
  output: buzzer
  on_finished_playback:
    - logger.log: 'Song ended!'

button:
  - platform: template
    name: "Two Beeps"
    on_press:
      - rtttl.play: "two short:d=4,o=5,b=100:16e6,16e6"

light:
  - platform: status_led
    name: "D3_LED"
    pin: 23
    restore_mode: ALWAYS_ON
    entity_category: config

i2c:
  sda: 21
  scl: 22
  scan: false
  frequency: 200kHz  # recommended range is 50-200kHz
  id: i2c_a

time:
  - platform: sntp
    id: my_time

# these are called references in YAML. They allow you to reuse
# this configuration in each sensor, while only defining it once
.defaultfilters:
  - &throttle_avg
    # average all raw readings together over a 5 second span before publishing
    throttle_average: 5s
  - &throttle_time
    # only send the most recent measurement every 60 seconds
    throttle: 60s
  - &invert
    # invert and filter out any values below 0.
    lambda: 'return max(-x, 0.0f);'
  - &pos
    # filter out any values below 0.
    lambda: 'return max(x, 0.0f);'
  - &abs
    # take the absolute value of the value
    lambda: 'return abs(x);'

sensor:
  - platform: emporia_vue
    i2c_id: i2c_a
    phases:
      - id: phase_a  # Verify that this specific phase/leg is connected to correct input wire color on device listed below
        input: BLACK  # Vue device wire color
        calibration: 0.022  # 0.022 is used as the default as starting point but may need adjusted to ensure accuracy
        # To calculate new calibration value use the formula <in-use calibration value> * <accurate voltage> / <reporting voltage>
        voltage:
          name: "Phase A Voltage"
          filters: [*throttle_avg, *pos]
        frequency:
          name: "Phase A Frequency"
          filters: [*throttle_avg, *pos]
      - id: phase_b  # Verify that this specific phase/leg is connected to correct input wire color on device listed below
        input: RED  # Vue device wire color
        calibration: 0.022  # 0.022 is used as the default as starting point but may need adjusted to ensure accuracy
        # To calculate new calibration value use the formula <in-use calibration value> * <accurate voltage> / <reporting voltage>
        voltage:
          name: "Phase B Voltage"
          filters: [*throttle_avg, *pos]
        phase_angle:
          name: "Phase B Phase Angle"
          filters: [*throttle_avg, *pos]
    ct_clamps:
      - phase_id: phase_a
        input: "A"  # Verify the CT going to this device input also matches the phase/leg
        power:
          id: phase_a_power
          filters: [*pos]
      - phase_id: phase_b
        input: "B"  # Verify the CT going to this device input also matches the phase/leg
        power:
          id: phase_b_power
          filters: [*pos]
      # Pay close attention to set the phase_id for each breaker by matching it to the phase/leg it connects to in the panel
      - { phase_id: phase_a, input:  "1", power: { id:  cir1, filters: [ *pos ] } }
      - { phase_id: phase_b, input:  "2", power: { id:  cir2, filters: [ *pos ] } }
      - { phase_id: phase_a, input:  "3", power: { id:  cir3, filters: [ *pos ] } }
      - { phase_id: phase_a, input:  "4", power: { id:  cir4, filters: [ *pos ] } }
      - { phase_id: phase_a, input:  "5", power: { id:  cir5, filters: [ *pos, multiply: 2 ] } }
      - { phase_id: phase_a, input:  "6", power: { id:  cir6, filters: [ *pos, multiply: 2 ] } }
      - { phase_id: phase_a, input:  "7", power: { id:  cir7, filters: [ *pos, multiply: 2 ] } }
      - { phase_id: phase_b, input:  "8", power: { id:  cir8, filters: [ *pos ] } }
      - { phase_id: phase_b, input:  "9", power: { id:  cir9, filters: [ *pos ] } }
      - { phase_id: phase_b, input: "10", power: { id: cir10, filters: [ *pos ] } }
      - { phase_id: phase_a, input: "11", power: { id: cir11, filters: [ *pos, multiply: 2 ] } }
      - { phase_id: phase_a, input: "12", power: { id: cir12, filters: [ *pos, multiply: 2 ] } }
      - { phase_id: phase_a, input: "13", power: { id: cir13, filters: [ *pos ] } }
      - { phase_id: phase_a, input: "14", power: { id: cir14, filters: [ *pos ] } }
      - { phase_id: phase_b, input: "15", power: { id: cir15, filters: [ *pos ] } }
      - { phase_id: phase_a, input: "16", power: { id: cir16, filters: [ *pos ] } }
    on_update:
      then:
        - component.update: total_power
        - component.update: balance_power
  - { platform: copy, name: "Phase A Power", source_id: phase_a_power, filters: *throttle_avg }
  - { platform: copy, name: "Phase B Power", source_id: phase_b_power, filters: *throttle_avg }
  - { platform: copy, name: "Total Power", source_id: total_power, filters: *throttle_avg }
  - { platform: copy, name: "Balance Power", source_id: balance_power, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 1 Power", source_id:  cir1, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 2 Power", source_id:  cir2, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 3 Power", source_id:  cir3, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 4 Power", source_id:  cir4, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 5 Power", source_id:  cir5, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 6 Power", source_id:  cir6, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 7 Power", source_id:  cir7, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 8 Power", source_id:  cir8, filters: *throttle_avg }
  - { platform: copy, name:  "Circuit 9 Power", source_id:  cir9, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 10 Power", source_id: cir10, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 11 Power", source_id: cir11, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 12 Power", source_id: cir12, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 13 Power", source_id: cir13, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 14 Power", source_id: cir14, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 15 Power", source_id: cir15, filters: *throttle_avg }
  - { platform: copy, name: "Circuit 16 Power", source_id: cir16, filters: *throttle_avg }
  - platform: template
    lambda: return id(phase_a_power).state + id(phase_b_power).state;
    update_interval: never   # will be updated after all power sensors update via on_update trigger
    id: total_power
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
  - platform: total_daily_energy
    name: "Total Daily Energy"
    power_id: total_power
    accuracy_decimals: 0
    filters: *throttle_time
  - platform: template
    lambda: !lambda |-
      return max(0.0f, id(total_power).state -
        id( cir1).state -
        id( cir2).state -
        id( cir3).state -
        id( cir4).state -
        id( cir5).state -
        id( cir6).state -
        id( cir7).state -
        id( cir8).state -
        id( cir9).state -
        id(cir10).state -
        id(cir11).state -
        id(cir12).state -
        id(cir13).state -
        id(cir14).state -
        id(cir15).state -
        id(cir16).state);
    update_interval: never   # will be updated after all power sensors update via on_update trigger
    id: balance_power
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
  - platform: total_daily_energy
    name: "Balance Daily Energy"
    power_id: balance_power
    accuracy_decimals: 0
    filters: *throttle_time
  - { power_id:  cir1, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 1 Daily Energy", filters: *throttle_time }
  - { power_id:  cir2, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 2 Daily Energy", filters: *throttle_time }
  - { power_id:  cir3, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 3 Daily Energy", filters: *throttle_time }
  - { power_id:  cir4, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 4 Daily Energy", filters: *throttle_time }
  - { power_id:  cir5, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 5 Daily Energy", filters: *throttle_time }
  - { power_id:  cir6, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 6 Daily Energy", filters: *throttle_time }
  - { power_id:  cir7, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 7 Daily Energy", filters: *throttle_time }
  - { power_id:  cir8, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 8 Daily Energy", filters: *throttle_time }
  - { power_id:  cir9, platform: total_daily_energy, accuracy_decimals: 0, name:  "Circuit 9 Daily Energy", filters: *throttle_time }
  - { power_id: cir10, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 10 Daily Energy", filters: *throttle_time }
  - { power_id: cir11, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 11 Daily Energy", filters: *throttle_time }
  - { power_id: cir12, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 12 Daily Energy", filters: *throttle_time }
  - { power_id: cir13, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 13 Daily Energy", filters: *throttle_time }
  - { power_id: cir14, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 14 Daily Energy", filters: *throttle_time }
  - { power_id: cir15, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 15 Daily Energy", filters: *throttle_time }
  - { power_id: cir16, platform: total_daily_energy, accuracy_decimals: 0, name: "Circuit 16 Daily Energy", filters: *throttle_time }
```

You'll want to replace `<ota password>`, `<wifi ssid>`, and `<wifi password>` with a unique password, and your wifi credentials, respectively.

You'll also want to update the `sensor` section of the configuration using the information you've collected in _Panel installation, part 1_.

Note the `sliding_window_moving_average`. This is optional, but since we get a reading every 240ms, it is helpful to average these readings together so that we don't need to store such dense, noisy, data in Home Assistant.

Note the "Total Power", "Total Daily Energy", and "Circuit x Daily Energy". This is needed for the Home Assistant energy system, which requires daily kWh numbers.

To configure energy returned to the grid for NET metering ([more info here](https://www.nrel.gov/state-local-tribal/basics-net-metering.html)), you need to add the following configuration:

```yaml
sensor:
  - platform: emporia_vue
    ct_clamps:
      - phase_id: phase_a
        input: "A"  # Verify the CT going to this device input also matches the phase/leg
        power:
          name: "Phase A Power Return"
          id: phase_a_power_return
          filters: [*moving_avg, *invert]  # This measures energy uploaded to grid on phase A
      - phase_id: phase_b
        input: "B"  # Verify the CT going to this device input also matches the phase/leg
        power:
          name: "Phase B Power Return"
          id: phase_b_power_return
          filters: [*moving_avg, *invert]  # This measures energy uploaded to grid on phase B
  - platform: template
    name: "Total Power Return"
    lambda: return id(phase_a_power_return).state + id(phase_b_power_return).state;
    update_interval: 1s
    id: total_power_return
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
  - platform: total_daily_energy
    name: "Total Daily Energy Return"
    power_id: total_power_return
    accuracy_decimals: 0

```

Your solar sensors' configuration depends on your setup (single phase, split phase, 3-phase). The following example shows a split-phase installation using ct clamps 15 and 16:

```yaml
sensor:
  - platform: template
    name: "Solar Power"
    lambda: return id(cir15).state + id(cir16).state;
    id: solar_power
    device_class: power
    state_class: measurement
    unit_of_measurement: "W"
  - platform: total_daily_energy
    name: "Solar Daily Energy"
    power_id: solar_power
    accuracy_decimals: 0
```

Do not use the `web_server` since it is not compatible with the `esp-idf` framework, and you will get odd error messages.

It's not too critical to get this right on the first try, because you can update the board over WiFi using [the ESPHome Dashboard](https://esphome.io/guides/getting_started_command_line.html#bonus-esphome-dashboard).

## Backing up & flashing the Vue 2

Pry the lever on one of the jumper cables up using a pencil or a needle or some other sharp thing. If your cables don't have a lever, cut one end of the cable & strip it using scissors or a knife.
![prying the lever on the jumper cable](https://i.imgur.com/BZJGdKq.jpg)![separated cable](https://i.imgur.com/eOc29M7.jpg)
You will then need to solder a serial header onto the programming port, so that it looks like this:

![closeup of the debug header pinout](https://i.imgur.com/NetVsQo.jpeg)
Plug the USB adapter in. Connect RX to RX, TX to TX, and GND to GND. Do not connect 5V or 3.3V at this time.

Plug in the unmodified end of the cable we modified above into the IO0 pin of the Emporia Vue 2.

Open a console window and test that `esptool.py version` works.

![photo of connected jumpers](https://i.imgur.com/TmB5PPV.jpeg)
Hold the modified end of the cable in IO0 to the metal shield on the ESP32. If you'd like, you can tape it down so that you have both hands free.

While holding it in place, connect 5V on your UART adapter to the `VCC_5V0` pin on the board.

If your TTL adapter has both the DTR and RTS pins exposed, you can let it automatically reboot the board and put the chip into flash mode when necessary. IO0 connects to DTR, and EN connects to RTS. In this case, you don't need to hold anything down.

### Doing a backup

With your other hand, run the following in the console: `esptool.py -b 921600 read_flash 0 0x800000 flash_contents.bin`. Successful completion of this step is _critical_ in case something goes wrong later. This file is necessary to restore the device to factory function.

If the above command fails, try again using `esptool.py -b 115200 read_flash 0 0x800000 flash_contents.bin`.

### Flashing new software

With your other hand, kick off the upload process. If you're using the command-line, `esphome run <yourfilename>.yaml`, otherwise click the button in the GUI. This will take a few minutes and install the new software on the Vue 2!

You'll see a bunch of errors like `Failed to read from sensor due to I2C error 3`, but that's fine, since they'll go away when it is installed into into the wall.

## Panel installation, part 2

Reassemble to Vue 2, and follow the instructions to plug everything in & started up!

## Getting a GUI

This project works best with Home Assistant. Follow these instructions to [connect the Vue 2 to Home Assistant](https://esphome.io/guides/getting_started_hassio.html#connecting-your-device-to-home-assistant).

Once you connect the Vue to Home Assistant, you can [configure the Home Assistant energy monitor functionallity](https://my.home-assistant.io/redirect/config_energy), as well as a variety of automations.

## FAQ

### What is MQTT?

MQTT is an alternative way of communicating the readings. If you need it, you already know, and it is not required for use with Home Assistant.

### How do I use this with MQTT?

There's now support for MQTT with this integration thanks to the hard work of the ESPHome folks! Please reference [MQTT Client Component](https://esphome.io/components/mqtt.html) for how to get this set up.

### I'm getting negative values

- You may have put that clamp on the wire backwards
- You may have selected the wrong phase in the configuration

### I've recorded negative energy values and I want to reset them

Sensor values are saved to the esp32 flash. You can reset all sensors by implementing a [factory reset button](https://esphome.io/components/button/factory_reset.html).

### The readings on one or two of my sensors are crazy

Sometimes the CTs aren't fully plugged into the 3.5mm jacks on the Vue. It's often not an issue with the initial install, but with stuff getting jostled around as you put things back together.

This issue will often manifest as jumps between 0W and some other wattage for no reason.

Open up the panel, and make sure every connector is fully inserted into the Vue. Check if the problem is solved before putting the panel cover back on.

### My data readings go up and down

If your readings are within ±1W, then they're within the expected margin of error. The filters are designed to smooth out noise like this, and it's expected as no physical system can be perfect.

If the readings are significant outside of that, there may be a problem.

### I'm seeing zeros on certain current clamps

First off, you will want to remove all filters for that sensor. Replace `filters: [ *moving_avg, *pos ]`, etc, with `filters: []`.

If your data is hovering around 0, then you either don't have any load on that circuit or there's some other issue that hasn't come up before.

If you're seeing negative data, it could be a few things:

- First off, make sure you've properly installed the clamps according to the instructions. The L side of the clamp should point towards the load. For solar systems or similar, keep in mind that current flows from the solar panel to your electrical panel, not the other way.
- Make sure you've selected the correct phase in the configuration. You will get negative _and_ nonsense power readings if you select the wrong phase. You can't negate the data through a filter and expect it to be correct.

When you're done troubleshooting, remember to place the filters back.

### I'm using a 64-bit Pi & can't compile!

Some users have successfully managed to build this on a 64-bit Pi: https://github.com/emporia-vue-local/esphome/discussions/147

~If you're using a 64-bit ARM OS, unfortunately you are unable to build this. It's not a limitation with this project, but a limitation with the upstream PlatformIO toolchains.~

You'll see an error like

```
Could not find the package with 'platformio/toolchain-esp32ulp @ ~1.22851.0' requirements for your system 'linux_aarch64'
```

You can try using a different computer. 32-bit and 64-bit x86 computers are both compatible (most laptops & desktops).
