# ESPHome on M5Stamp ESP32S3 Module (M5StampS3)

Starting from scratch and no experience with ESPHome and an ESP32 controller can be challenging and is often frustrating. Picking one of the hugh amount of available ESP32 boards and find a first basic ESPHome configuration for a specific board is one of the first and toughest challenge.
Therefore I created a simple example configuration for the M5StampS3 as a starting point to save time and to prevent typical frustrating pitfalls.

> [!TIP]
> TL;DR Just use the example [m5stamps3-device.yaml](m5stamps3-device.yaml) and replace all `[..]`-placeholders with valid settings to create your first firmware.

Personally, I like the products and controllers of [M5Stack](https://m5stack.com/), especially their [STAMP-series](https://shop.m5stack.com/collections/m5-controllers/STAMP). There are definitely cheaper ESP32 boards available but for me the STAMP-series provide the strong advantage of having a much better onboard antenna (not those standard PCB-antennas) while keeping a good tradeoff between number of GPIOs and size which makes them suitable for a lot of use cases.

At the time of writing this (Nov 2024), I prefer using the ESP32-S3 because of higher computing power, more flash and a native USB interface. M5Stack names the corresponding board [M5Stamp ESP32S3 Module](https://docs.m5stack.com/en/core/StampS3) for the base model and additionally offers the two variants [M5StampS3 with 2.54 Header Pin](https://docs.m5stack.com/en/core/M5StampS3%20PIN2.54) and [M5StampS3 with 1.27 Header Pin](https://docs.m5stack.com/en/core/M5StampS3%20PIN1.27) with presoldered header pins.

Technically they are all the same and feature 
* ESP32-S3FN8
* 8MB flash
* 5V input voltage with onboard 3.3V 1A step-down converter (MUN3CAD01-SC)
* single RGB LED (WS2812B)
* single physical button
* 10 GPIOs (23 GPIOs with 1.27 header pin)
* USB-C interface
<br/>

## Basic yaml configuration
Let's assume you start with the official documentation [Getting Started with ESPHome and Home Assistant](https://esphome.io/guides/getting_started_hassio.html), you most probably will try to create a first configuration using the wizard provided by ESPHome. The wizard will ask for the used board and you will encounter the first challenge as M5StampS3 is not listed. Choose the general setting `esp32-s3-devkitc-1`. In addition, I prefer to use `esp-idf` framework instead of `arduino` framework.

> [!TIP]
> Before finishing the wizard, connect the M5StampS3 via USB. Luckily, as the ESP-S3 supports native USB serial connection, there shouldn't be any specific driver required. You have to press the button down while connecting to USB. USB connection is only required the first time, afterwards OTA can be used when M5StampS3 has wifi connection.

> [!NOTE]
> If you don't want to use the wizard, you can also copy the example configuration [m5stamps3-device.yaml](m5stamps3-device.yaml) and replace all `[..]`-placeholders with valid settings.


After finishing the wizard, ESPHome will try to upload a firmware via WebSerial (only Chrome or Edge) which most probably will lead to the M5StampS3 being stuck in an endless boot-loop. This is because the default flash mode is not compatible with ESP32-S3. Add the following to the `espome`-section in your configuration to set a working flash mode:
```YAML
esphome:
  platformio_options:
    board_build.flash_mode: dio
```

The M5StampS3 features 8MB flash which needs to be configured explicitly. Your `esp32`-section should look like:

```YAML
esp32:
  board: esp32-s3-devkitc-1
  flash_size: 8MB
  framework:
    type: esp-idf
```

Having this basic configuration, the M5StampS3 won't be stuck in a boot-loop anymore and all flash memory an be utilized.
<br/>
<br/>
<br/>

## Configure onboard RGB light
The M5StampS3 has a single RGB WS2812 (Neopixel) led connected to GPIO 21. Add the following to your configuration:
```YAML
light:
  - platform: esp32_rmt_led_strip
    id: status_led
    internal: true
    pin: GPIO21
    num_leds: 1
    rmt_channel: 0
    rgb_order: GRB
    chipset: ws2812
    color_correct: [30%, 40%, 50%]
    default_transition_length: 0s
```
I prefer to use `color_correct`-setting to reduce the brightness and adjust the color mixture of the led as I experience the led as to bright when running at 100%. 

>[!NOTE]
>Set `internal: false` if you want to control the led from outside, e.g. from Home Assistant. It will then be a normal light-entity.

### Use led to visualize wifi connection state
It is possible to define some actions for signaling the state of the wifi connection. Let's say, led should be red when wifi is disconnected and green as soon as there is a wifi connection.

First, set the led to red light upon power-on by adding a `on_boot`-action to the `esphome`-section:
```YAML
esphome:
  ...
  on_boot:
    - priority: 600
      then:
        - light.turn_on:
            id: status_led
            red: 100%
            green: 0%
            blue: 0%
```
Setting `priority: 600` leads to activate the led as early as possible after power-on.

Now add the actions for connect and disconnect to the `wifi`-section of your configuration:
```YAML
wifi:
  ...
  on_connect:
    - light.turn_on:
        id: status_led
        red: 0%
        green: 100%
        blue: 0%
  on_disconnect:
    - light.turn_on:
        id: status_led
        red: 100%
        green: 0%
        blue: 0%
```
Update your M5StampS3 and on power-on you will now see the led first being red and then switching to green after a few seconds if your wifi credentials are correct and your WIFI is in range.

## Configure onboard button
The M5StampS3 has a single button connected to GPIO 0. A button is usually represented as a binary sensor. Add the following to your configuration:
```YAML
binary_sensor:
  - platform: gpio
    id: surface_button
    internal: true
    pin:
      number: GPIO0
      inverted: true
      mode:
        input: true
        pullup: false
        pulldown: false
    filters:
      - delayed_on_off: 20ms
```
The `deleyed_on_off: 20ms`-filter is strongly recommended for any physical button or switch, as it is some kind of software debouncing.

>[!NOTE]
>Set `internal: false` if you want to receive a button press externally, e.g. in Home Assistant.

### Show button press with led
To see if the button works correctly, another action can be used to change the color of the led for a short time.

First, add a `flicker`-effect to the defined light:
```YAML
light:
  - platform: esp32_rmt_led_strip
    id: status_led
    ...
    default_transition_length: 0s
    effects:
      - flicker:
```

Then add an `on_press`-action to the binary sensor:
```YAML
binary_sensor:
  - platform: gpio
    id: surface_button
    ...
    filters:
      - delayed_on_off: 20ms
    on_press:
      then:
        - light.turn_on:
            id: status_led
            flash_length: 200ms
            red: 0%
            green: 0%
            blue: 100%
```
Setting the `flash_length` triggers the `flicker`-effect and automatically changes to led back to color before the `on_press`-action.
<br/>
<br/>

**That's it, you now have the foundation to continue adding functionality and extending your configuration.**

**Have fun with your M5StampS3 and ESPHome** :smile: