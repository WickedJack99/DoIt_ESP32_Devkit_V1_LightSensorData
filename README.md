# DoIt_ESP32_Devkit_V1_LightSensorData
Repository to receive light sensor data.

Note: the custom date calculation has known bugs. The repository is kept as an archive of the original one-week implementation.

Built in one week as a small ESP32 light sensor data logger.

Not actively developing.

# Device
> The device used is a DoIt ESP32 Devkit V1 development board.

# Measurements
Measurements are done with a 10 Lux resolution light resistor.
It is connected to GPIO34 (ADC1_ch6) since ADC2 doesn't work when using WiFi module.

> Each hour a new measurement is sent to the repository into the Measurements directory into the specified date directory.

# Sources
- [Setting up Arduino IDE for ESP32](https://randomnerdtutorials.com/installing-the-esp32-board-in-arduino-ide-windows-instructions/)
- [Determine pin functionality](https://randomnerdtutorials.com/esp32-pinout-reference-gpios/)
- [HTTPS with ESP32](https://randomnerdtutorials.com/esp32-https-requests/#esp32-https-requests-httpclient)
