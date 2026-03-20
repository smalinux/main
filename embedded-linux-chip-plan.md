# Embedded Linux Chip Interfacing — 2-Year Mastery Plan

> **Goal:** Confidently connect ANY electronic chip to Linux SBCs (Raspberry Pi, BeagleBone Black, AUBoard-15P) with full kernel driver + Yocto integration understanding.
> **Starting point:** Zero electronics knowledge.
> **Style:** One chip at a time. Measure. Understand. Move on.

---

## How To Use This Plan

- **Each chip** = 1–3 weeks depending on complexity
- **For every chip, always do:**
  1. Read the datasheet (at least the block diagram, registers, timing diagrams)
  2. Wire it on a breadboard (use a multimeter + logic analyzer)
  3. Test with userspace tools first (`i2cdetect`, `spidev`, `gpioset`, etc.)
  4. Find or write a Linux kernel driver (device tree binding + driver source)
  5. Build a Yocto layer/recipe that includes the driver + device tree overlay
  6. On AUBoard-15P: instantiate the interface in Vivado, expose via AXI, write or port the driver
- **Buy:** A logic analyzer (Saleae Logic 8 or clone), multimeter, oscilloscope (Rigol DS1054Z), breadboard kit, jumper wires, soldering iron

---

## Phase 0 — Foundations (Month 1–2)

> Learn enough electronics and Linux kernel basics to not destroy chips.

### 0.1 Electronics Survival Kit

| Topic | What To Learn | Google Keywords |
|---|---|---|
| Ohm's Law, voltage dividers | `V = IR`, current limiting resistors | `ohm's law tutorial embedded` |
| Pull-up / pull-down resistors | Why I2C needs pull-ups, internal vs external | `pull-up resistor i2c explained` |
| Decoupling capacitors | 100nF near every IC VCC pin | `decoupling capacitor why placement` |
| Level shifting | 3.3V ↔ 5V, 1.8V ↔ 3.3V | `logic level shifter i2c spi` |
| Reading datasheets | Block diagrams, pin descriptions, timing diagrams, register maps | `how to read a datasheet embedded` |
| Soldering basics | Through-hole first, then SMD | `soldering tutorial beginner electronics` |
| ESD safety | Ground straps, handling ICs | `ESD protection embedded development` |
| Using a multimeter | Continuity, voltage, current measurement | `multimeter tutorial beginners` |
| Using a logic analyzer | Capture I2C/SPI/UART waveforms | `saleae logic analyzer i2c spi tutorial` |
| Using an oscilloscope | Analog signals, rise times, noise | `oscilloscope tutorial embedded beginner` |

### 0.2 Linux Kernel & Yocto Foundations

| Topic | What To Learn | Google Keywords |
|---|---|---|
| Device Tree basics | nodes, properties, `compatible`, overlays | `linux device tree tutorial beginners` |
| Kernel config & build | `make menuconfig`, cross-compile | `linux kernel cross compile arm64` |
| Kernel modules | `insmod`, `modprobe`, `lsmod`, writing a hello-world module | `linux kernel module tutorial` |
| sysfs / debugfs | `/sys/class/`, `/sys/bus/i2c/`, `/sys/bus/spi/` | `linux sysfs interface explained` |
| Yocto basics | `bitbake`, layers, recipes, `local.conf`, machine conf | `yocto project tutorial beginner` |
| Yocto BSP layer | Creating a BSP for a custom board | `yocto custom bsp layer tutorial` |
| Yocto kernel recipes | `linux-yocto`, `defconfig`, device tree overlays in Yocto | `yocto linux kernel recipe device tree` |
| PetaLinux (for AUBoard-15P) | BSP build, MicroBlaze target, Vivado export | `petalinux tutorial artix ultrascale` |

### 0.3 Bus Protocol Fundamentals

| Bus | Key Concepts | Typical Speed | Google Keywords |
|---|---|---|---|
| **I2C** | SDA/SCL, 7-bit address, ACK/NACK, clock stretching | 100/400 kHz, 1 MHz | `i2c protocol tutorial embedded linux` |
| **SPI** | MOSI/MISO/SCLK/CS, modes 0–3, full duplex | 1–100 MHz | `spi protocol tutorial embedded linux` |
| **UART** | TX/RX, baud rate, start/stop bits, no clock | 9600–115200 baud | `uart serial protocol tutorial` |
| **1-Wire** | Single data line + ground, parasitic power | ~16 kbps | `1-wire protocol linux driver` |
| **I2S** | Audio: BCLK, LRCLK, DATA | varies | `i2s protocol audio linux` |
| **PWM** | Duty cycle, frequency, servo/motor control | varies | `pwm linux kernel sysfs` |
| **GPIO** | Input/output, interrupts, debouncing | — | `linux gpio subsystem libgpiod` |
| **CAN** | Differential pair, arbitration, frames | 125k–1M bps | `can bus linux socketcan tutorial` |
| **MDIO/MII** | Ethernet PHY management | — | `mdio mii ethernet phy linux` |
| **MIPI CSI-2** | Camera serial interface, lanes, D-PHY | 1–2.5 Gbps/lane | `mipi csi2 linux v4l2 driver` |
| **PCIe** | Lanes, BARs, DMA, endpoint vs root complex | Gen3/4 | `pcie endpoint linux driver tutorial` |

---

## Phase 1 — I2C Chips: The Foundation (Month 3–5)

> I2C is the most common embedded bus. Master it first.

### 1.1 Temperature Sensors

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 1 | **TMP102** | Texas Instruments | I2C | `tmp102` | Simplest possible I2C sensor, 2 registers | `tmp102 raspberry pi i2c linux driver` |
| 2 | **LM75** | NXP / TI / ON Semi | I2C | `lm75` | Industry standard, in-kernel `hwmon` driver | `lm75 linux hwmon device tree` |
| 3 | **STTS751** | STMicroelectronics | I2C | `stts751` | ST ecosystem, SMBus alert | `stts751 linux driver i2c` |
| 4 | **MCP9808** | Microchip | I2C | `mcp9808` (hwmon) | ±0.25°C accuracy, great datasheet | `mcp9808 linux kernel driver` |
| 5 | **ADT7410** | Analog Devices | I2C | `adt7410` | 16-bit resolution, ADI ecosystem | `adt7410 linux hwmon driver` |

**Yocto task:** Create a `meta-sensors` layer. Add device tree overlays for each sensor. Build an image with `lm-sensors` package.

**AUBoard-15P task:** Expose an AXI I2C controller in Vivado → connect TMP102 → boot MicroBlaze Linux → read temperature via `hwmon`.

### 1.2 IMU — Accelerometer / Gyroscope / Magnetometer

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 6 | **MPU-6050** | InvenSense (TDK) | I2C | `inv_mpu6050` | Most documented IMU ever, IIO subsystem | `mpu6050 linux iio driver device tree` |
| 7 | **LSM6DSO** | STMicroelectronics | I2C/SPI | `st_lsm6dsx` | ST's flagship 6-axis, great Linux support | `lsm6dso linux iio driver` |
| 8 | **ADXL345** | Analog Devices | I2C/SPI | `adxl345` | Classic 3-axis accel, IIO subsystem | `adxl345 linux iio driver` |
| 9 | **BMI270** | Bosch Sensortec | I2C/SPI | `bmi270` | Modern 6-axis, low power | `bmi270 linux driver iio` |
| 10 | **LIS3MDL** | STMicroelectronics | I2C/SPI | `st_magn` | 3-axis magnetometer, IIO | `lis3mdl linux iio magnetometer` |

**Linux kernel focus:** Learn the **IIO (Industrial I/O)** subsystem — `iio_info`, `iio_readdev`, triggered buffers, sysfs channels.

### 1.3 Pressure / Humidity / Environmental

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 11 | **BME280** | Bosch Sensortec | I2C/SPI | `bme280` | Temp + humidity + pressure, IIO | `bme280 linux iio driver` |
| 12 | **LPS22HB** | STMicroelectronics | I2C/SPI | `st_pressure` | Barometric pressure, IIO | `lps22hb linux iio driver` |
| 13 | **HDC1080** | Texas Instruments | I2C | `hdc100x` | Humidity + temp, simple registers | `hdc1080 linux driver i2c` |
| 14 | **SHT31** | Sensirion | I2C | `sht3x` | High-accuracy humidity, `hwmon` | `sht31 linux hwmon driver` |
| 15 | **ENS160** | ScioSense | I2C | `ens160` | Air quality (eCO2, TVOC), IIO | `ens160 linux iio driver` |

### 1.4 Light / Proximity / Color

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 16 | **TSL2561** | ams-OSRAM | I2C | `tsl2561` | Lux sensor, IIO | `tsl2561 linux iio light sensor` |
| 17 | **APDS-9960** | Broadcom/Avago | I2C | `apds9960` | Gesture + proximity + color + ALS | `apds9960 linux iio driver` |
| 18 | **VEML7700** | Vishay | I2C | `veml6030` | Ambient light, simple | `veml7700 linux iio driver` |
| 19 | **VL53L0X** | STMicroelectronics | I2C | `vl53l0x` | Time-of-Flight distance, up to 2m | `vl53l0x linux iio tof sensor` |
| 20 | **OPT3001** | Texas Instruments | I2C | `opt3001` | Lux sensor, human-eye response | `opt3001 linux iio driver` |

### 1.5 I/O Expanders

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 21 | **PCA9535** | NXP | I2C | `gpio-pca953x` | 16-bit GPIO expander, interrupt support | `pca9535 linux gpio expander device tree` |
| 22 | **MCP23017** | Microchip | I2C | `gpio-mcp23s08` | 16 GPIOs, very popular | `mcp23017 linux gpio driver` |
| 23 | **TCA9548A** | Texas Instruments | I2C | `i2c-mux-pca954x` | I2C multiplexer (8 channels) — learn I2C mux in kernel | `tca9548a linux i2c mux device tree` |
| 24 | **PCAL6416A** | NXP | I2C | `gpio-pca953x` | Latched interrupts, agglomerated | `pcal6416a linux gpio expander` |

### 1.6 EEPROM / RTC

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 25 | **AT24C256** | Microchip | I2C | `at24` | 32KB EEPROM, learn `nvmem` subsystem | `at24 eeprom linux nvmem driver` |
| 26 | **DS3231** | Analog Devices (Maxim) | I2C | `rtc-ds3231` | High-precision RTC, learn `rtc` subsystem | `ds3231 linux rtc driver device tree` |
| 27 | **PCF8563** | NXP | I2C | `rtc-pcf8563` | Common RTC in industrial designs | `pcf8563 linux rtc driver` |

### 1.7 Power Monitoring

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 28 | **INA219** | Texas Instruments | I2C | `ina2xx` | Current/power monitor, `hwmon` | `ina219 linux hwmon driver` |
| 29 | **INA260** | Texas Instruments | I2C | `ina2xx` | Integrated shunt, easier to wire | `ina260 linux power monitor` |
| 30 | **PAC1952** | Microchip | I2C | `pac1934` | Multi-channel power monitor | `pac1934 linux hwmon driver` |

---

## Phase 2 — SPI Chips: Speed & Precision (Month 6–8)

> SPI is faster, full-duplex, and used for ADCs, DACs, displays, and flash.

### 2.1 ADC (Analog-to-Digital Converters)

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 31 | **MCP3008** | Microchip | SPI | `mcp320x` | 10-bit, 8-channel, easiest SPI ADC | `mcp3008 linux iio spi adc` |
| 32 | **ADS1115** | Texas Instruments | I2C | `ads1015` | 16-bit delta-sigma, programmable gain | `ads1115 linux iio driver` |
| 33 | **AD7689** | Analog Devices | SPI | `ad7689` | 16-bit SAR, 8-channel, IIO | `ad7689 linux iio adc driver` |
| 34 | **ADS8688** | Texas Instruments | SPI | `ti-ads8688` | 16-bit, 8-channel, bipolar input | `ads8688 linux iio driver` |

### 2.2 DAC (Digital-to-Analog Converters)

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 35 | **MCP4725** | Microchip | I2C | `mcp4725` | 12-bit single DAC, simplest | `mcp4725 linux iio dac driver` |
| 36 | **AD5676** | Analog Devices | SPI | `ad5676` | 16-bit, 8-channel DAC | `ad5676 linux iio dac` |
| 37 | **DAC7578** | Texas Instruments | I2C | `ti-dac7578` | 12-bit, 8-channel | `dac7578 linux iio driver` |

### 2.3 SPI Flash / NOR

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 38 | **W25Q128** | Winbond | SPI | `spi-nor` | 128Mbit NOR flash, boot storage | `w25q128 linux spi-nor mtd` |
| 39 | **IS25LP512M** | ISSI | QSPI | `spi-nor` | 512Mbit, same family as AUBoard-15P flash | `issi qspi linux spi-nor driver` |
| 40 | **MT25QU256** | Micron | QSPI | `spi-nor` | Used in many SoC reference designs | `micron mt25qu linux spi-nor` |

### 2.4 Display — SPI TFT/OLED

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 41 | **SSD1306** | Solomon Systech | I2C/SPI | `ssd1306` (DRM) | 128×64 OLED, learn DRM/framebuffer | `ssd1306 linux drm driver device tree` |
| 42 | **ILI9341** | Ilitek | SPI | `ili9341` (DRM) | 320×240 TFT, common in embedded | `ili9341 linux drm spi display` |
| 43 | **ST7789** | Sitronix | SPI | `st7789v` (DRM) | 240×240/320 TFT | `st7789 linux drm driver` |

### 2.5 CAN Bus Transceiver (SPI-based controller)

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 44 | **MCP2515** | Microchip | SPI | `mcp251x` | CAN controller, learn SocketCAN | `mcp2515 linux socketcan driver` |
| 45 | **TCAN4550** | Texas Instruments | SPI | `tcan4x5x` | CAN FD + transceiver, modern | `tcan4550 linux socketcan can-fd` |

---

## Phase 3 — UART & 1-Wire Chips (Month 9–10)

### 3.1 UART Devices

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 46 | **MAX3232** | Analog Devices (Maxim) | UART (RS-232 level shift) | N/A (hardware only) | Understand RS-232 level shifting | `max3232 rs232 uart level shifter` |
| 47 | **SP3485** | MaxLinear | UART (RS-485) | `serial` + GPIO for DE/RE | Half-duplex RS-485, learn direction control | `rs485 linux serial driver gpio rts` |
| 48 | **SC16IS752** | NXP | I2C/SPI → UART | `sc16is7xx` | UART expander, adds 2 UARTs via I2C/SPI | `sc16is752 linux uart bridge driver` |
| 49 | **NEO-6M** | u-blox | UART | `gnss` subsystem | GPS/GNSS module, learn NMEA parsing | `ublox neo-6m linux gnss driver gpsd` |

### 3.2 1-Wire Devices

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 50 | **DS18B20** | Analog Devices (Maxim) | 1-Wire | `w1-therm` | Temperature, learn `w1` subsystem | `ds18b20 linux w1 1-wire driver` |
| 51 | **DS2431** | Analog Devices (Maxim) | 1-Wire | `ds2431` | 1Kbit EEPROM, 1-Wire memory | `ds2431 linux w1 eeprom` |

---

## Phase 4 — Power Management ICs (Month 11–12)

> PMICs are in every real product. Learn voltage regulation and power sequencing.

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 52 | **TPS65219** | Texas Instruments | I2C | `tps65219` (regulator + GPIO) | PMIC for AM62x, learn `regulator` subsystem | `tps65219 linux pmic regulator driver` |
| 53 | **PCA9450** | NXP | I2C | `pca9450` | PMIC for i.MX 8M, NXP reference | `pca9450 linux regulator driver` |
| 54 | **STPMIC1** | STMicroelectronics | I2C | `stpmic1` | PMIC for STM32MP1, learn power sequencing | `stpmic1 linux regulator driver` |
| 55 | **AXP313A** | X-Powers | I2C | `axp20x` | Used in Allwinner boards, learn MFD drivers | `axp313a linux mfd regulator` |
| 56 | **TPS62827** | Texas Instruments | I2C | `tps6286x` | Simple step-down regulator | `tps62827 linux regulator driver` |
| 57 | **LTC3589** | Analog Devices | I2C | `ltc3589` | Multi-output PMIC | `ltc3589 linux regulator driver` |

**Linux kernel focus:** Learn the **regulator framework** — `regulator_get()`, `regulator_enable()`, voltage constraints in device tree, power sequencing.

---

## Phase 5 — Ethernet, Wi-Fi, Bluetooth (Month 13–15)

> Network connectivity is critical for embedded Linux products.

### 5.1 Ethernet PHY

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 58 | **KSZ9031** | Microchip | RGMII + MDIO | `ksz9031` (phylib) | Gigabit PHY, common in ARM SBCs | `ksz9031 linux phy driver device tree rgmii` |
| 59 | **DP83867** | Texas Instruments | RGMII + MDIO | `dp83867` | GbE PHY, TI reference designs | `dp83867 linux phy driver` |
| 60 | **RTL8211F** | Realtek | RGMII + MDIO | `realtek` | Cheap GbE PHY, very common | `rtl8211f linux phy driver` |
| 61 | **LAN8720** | Microchip | RMII + MDIO | `smsc` | 10/100 PHY, simpler to start with | `lan8720 linux phy driver` |

**Linux kernel focus:** Learn **phylib** — PHY drivers, MDIO bus, `phy-handle` in device tree, `ethtool`, MAC ↔ PHY binding.

**AUBoard-15P task:** The board has a Microchip 10/100 PHY — study its Vivado IP integration and Linux driver binding via AXI Ethernet.

### 5.2 Wi-Fi / Bluetooth Modules

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 62 | **CC3301** | Texas Instruments | SDIO/SPI | `wlcore` / `cc33xx` | Your original chip! Wi-Fi 6 companion | `cc3301 linux wireless driver` |
| 63 | **WL1837MOD** | Texas Instruments | SDIO | `wl18xx` | Wi-Fi + Bluetooth combo, BeagleBone | `wl1837 linux wireless driver` |
| 64 | **CYW43455** | Infineon (Cypress) | SDIO | `brcmfmac` | Used in Raspberry Pi 3B+/4 | `cyw43455 linux brcmfmac driver` |
| 65 | **ATWILC1000** | Microchip | SPI/SDIO | `wilc1000` | Open-source friendly, SPI option | `wilc1000 linux wireless spi driver` |
| 66 | **nRF7002** | Nordic Semi | SPI | Zephyr (no mainline Linux) | Wi-Fi 6 companion, learn Zephyr comparison | `nrf7002 wifi companion ic` |

**Linux kernel focus:** Learn **cfg80211 / mac80211** wireless subsystem, firmware loading, regulatory domain, `wpa_supplicant`, `iw`.

### 5.3 Bluetooth Low Energy (standalone)

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 67 | **nRF52840** | Nordic Semi | UART (HCI) | `hci_uart` (btattach) | BLE 5.0, HCI over UART | `nrf52840 linux bluetooth hci uart` |
| 68 | **CC2652R** | Texas Instruments | UART | `hci_uart` | Zigbee + Thread + BLE, multiprotocol | `cc2652 linux bluetooth hci` |

---

## Phase 6 — Audio, Motor, PWM (Month 16–17)

### 6.1 Audio Codecs (I2S + I2C control)

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 69 | **WM8960** | Cirrus Logic | I2S + I2C | `wm8960` | Stereo codec, learn ASoC framework | `wm8960 linux asoc driver device tree` |
| 70 | **TLV320AIC3104** | Texas Instruments | I2S + I2C | `tlv320aic3x` | TI audio codec, common in industrial | `tlv320aic3104 linux asoc driver` |
| 71 | **SGTL5000** | NXP | I2S + I2C | `sgtl5000` | Used in NXP i.MX reference designs | `sgtl5000 linux asoc driver` |
| 72 | **MAX98357A** | Analog Devices (Maxim) | I2S (no I2C) | `max98357a` | Simplest: I2S amplifier, no control bus | `max98357a linux asoc driver` |

**Linux kernel focus:** Learn **ASoC (ALSA SoC)** — codec driver, DAI link, machine driver, `aplay`, `arecord`, `amixer`.

### 6.2 Motor Drivers

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 73 | **DRV8833** | Texas Instruments | GPIO + PWM | No in-kernel (use `pwm` + `gpio` sysfs) | Dual H-bridge, DC motors + steppers | `drv8833 linux pwm motor control` |
| 74 | **PCA9685** | NXP | I2C | `pca9685` (PWM) | 16-channel PWM, servo control | `pca9685 linux pwm driver i2c` |
| 75 | **TB6612FNG** | Toshiba | GPIO + PWM | No in-kernel (userspace) | Dual DC motor driver | `tb6612fng linux motor driver` |

### 6.3 LED Drivers

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 76 | **LP5024** | Texas Instruments | I2C | `lp50xx` (LED) | 8 RGB LEDs, learn `leds` subsystem | `lp5024 linux led driver` |
| 77 | **IS31FL3731** | Lumissil (ISSI) | I2C | `is31fl319x` | LED matrix driver, charlieplexing | `is31fl3731 linux led driver` |

---

## Phase 7 — High-Speed: USB, PCIe, Camera (Month 18–20)

> This is where real product complexity lives.

### 7.1 USB

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 78 | **FT232R** | FTDI | USB | `ftdi_sio` | USB-to-UART, understand USB serial | `ftdi ft232r linux usb serial driver` |
| 79 | **CP2102** | Silicon Labs | USB | `cp210x` | USB-to-UART alternative, compare with FTDI | `cp2102 linux usb serial driver` |
| 80 | **USB2514** | Microchip | USB (hub) | `usb2514` (platform) | 4-port USB hub, learn USB topology in DT | `usb2514 linux usb hub driver device tree` |
| 81 | **TUSB8041** | Texas Instruments | USB | `xhci` | USB 3.0 hub, learn xHCI | `tusb8041 linux usb3 hub` |
| 82 | **FE1.1s** | Terminus Tech | USB (hub) | Generic `hub` | Cheap 4-port hub, practice USB enumeration | `usb hub linux device tree` |

### 7.2 PCIe

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 83 | **ASM1182e** | ASMedia | PCIe | `pcieport` | PCIe switch, learn PCIe topology | `asmedia pcie switch linux` |
| 84 | **Study AUBoard-15P PCIe Gen4 x4** | AMD (FPGA) | PCIe | `xdma` / custom | Implement PCIe endpoint in FPGA fabric | `xilinx xdma pcie endpoint linux driver` |

**AUBoard-15P task:** Use Vivado to instantiate a PCIe Gen4 endpoint block → write a simple Linux PCIe driver on the host → DMA data to/from FPGA.

### 7.3 Camera / Image Sensors

| # | Chip | Manufacturer | Interface | Linux Driver | Why This One | Google Keywords |
|---|---|---|---|---|---|---|
| 85 | **OV5640** | OmniVision | MIPI CSI-2 / DVP | `ov5640` | 5MP, best Linux support of any camera sensor | `ov5640 linux v4l2 mipi csi driver` |
| 86 | **IMX219** | Sony | MIPI CSI-2 | `imx219` | Raspberry Pi Camera Module v2 sensor | `imx219 linux v4l2 driver` |
| 87 | **AR0234CS** | onsemi | MIPI CSI-2 | `ar0234cs` | Global shutter, industrial/automotive | `ar0234 linux v4l2 driver` |

**Linux kernel focus:** Learn **V4L2 (Video4Linux2)** + **Media Controller** — subdevs, media links, pipeline configuration, `v4l2-ctl`, `media-ctl`.

---

## Phase 8 — FPGA Integration Deep Dive (Month 21–22)

> Tie everything together with the AUBoard-15P.

| # | Task | Details | Google Keywords |
|---|---|---|---|
| 88 | **AXI GPIO** | Expose FPGA pins as Linux GPIOs via AXI GPIO IP | `xilinx axi gpio linux driver device tree` |
| 89 | **AXI I2C** | Connect I2C sensors to FPGA fabric, access from Linux | `xilinx axi iic linux i2c driver` |
| 90 | **AXI SPI** | Connect SPI chips to FPGA fabric | `xilinx axi spi linux driver` |
| 91 | **AXI UART** | UART controller in fabric | `xilinx axi uartlite linux driver` |
| 92 | **AXI DMA** | DMA engine for high-throughput data paths | `xilinx axi dma linux dmaengine driver` |
| 93 | **AXI Ethernet** | 10/100/1G Ethernet MAC in fabric + PHY | `xilinx axi ethernet linux driver` |
| 94 | **Custom AXI peripheral** | Write a custom Verilog/VHDL IP, expose registers via AXI Lite, write a Linux driver | `custom axi peripheral linux driver vivado` |
| 95 | **Yocto + PetaLinux** | Build a full Yocto image for MicroBlaze targeting AUBoard-15P with all your drivers | `petalinux yocto microblaze custom bsp` |

---

## Phase 9 — Capstone Projects (Month 23–24)

> Build real systems combining multiple chips.

| # | Project | Chips Involved | Skills Demonstrated |
|---|---|---|---|
| 96 | **Weather Station** | BME280 + SSD1306 + ESP32 (Wi-Fi) + DS3231 | I2C, SPI, UART, Yocto image, systemd service |
| 97 | **CAN Bus Data Logger** | MCP2515 + W25Q128 + INA219 + GPS (NEO-6M) | SPI, I2C, UART, SocketCAN, MTD, filesystem |
| 98 | **Audio Playback System** | WM8960 + PCA9685 (volume knob) + SSD1306 | I2S, I2C, ASoC, ALSA, DRM |
| 99 | **FPGA Accelerated Sensor Hub** | AUBoard-15P + TMP102 + ADXL345 + MCP3008 (all via FPGA AXI) | Vivado, AXI, device tree, custom drivers, PetaLinux |
| 100 | **Custom SBC Bring-Up** | Design a carrier board with 5+ chips, write full BSP in Yocto | PCB design (KiCad), full kernel BSP, Yocto distro |

---

## Quick Reference — Linux Kernel Subsystems Covered

| Subsystem | Chips That Teach It | Key Source Directory |
|---|---|---|
| **hwmon** | TMP102, LM75, INA219, SHT31 | `drivers/hwmon/` |
| **IIO** | MPU6050, BME280, ADXL345, MCP3008, ADS1115 | `drivers/iio/` |
| **GPIO** | PCA9535, MCP23017, AXI GPIO | `drivers/gpio/` |
| **RTC** | DS3231, PCF8563 | `drivers/rtc/` |
| **nvmem** | AT24C256 | `drivers/nvmem/` |
| **regulator** | TPS65219, PCA9450, STPMIC1 | `drivers/regulator/` |
| **PHY (phylib)** | KSZ9031, DP83867, LAN8720 | `drivers/net/phy/` |
| **wireless (cfg80211)** | CC3301, WL1837, CYW43455 | `drivers/net/wireless/` |
| **ASoC** | WM8960, TLV320AIC3104, SGTL5000 | `sound/soc/codecs/` |
| **DRM** | SSD1306, ILI9341, ST7789 | `drivers/gpu/drm/` |
| **V4L2** | OV5640, IMX219, AR0234 | `drivers/media/i2c/` |
| **USB** | FT232R, USB2514 | `drivers/usb/` |
| **SPI NOR / MTD** | W25Q128, IS25LP512M | `drivers/mtd/spi-nor/` |
| **SocketCAN** | MCP2515, TCAN4550 | `drivers/net/can/` |
| **w1 (1-Wire)** | DS18B20, DS2431 | `drivers/w1/` |
| **PWM** | PCA9685, DRV8833 | `drivers/pwm/` |
| **LED** | LP5024, IS31FL3731 | `drivers/leds/` |
| **MFD** | AXP313A, STPMIC1 | `drivers/mfd/` |

---

## Essential Tools Checklist

| Tool | Purpose | Google Keywords |
|---|---|---|
| **Saleae Logic 8** (or clone) | Protocol decoding: I2C, SPI, UART, CAN | `saleae logic analyzer embedded` |
| **Rigol DS1054Z** | Oscilloscope for analog signals | `rigol ds1054z review` |
| **Multimeter** (Fluke 115 or UNI-T) | Voltage, current, continuity | `best multimeter embedded development` |
| **Bus Pirate** | Quick bus probing without SBC | `bus pirate i2c spi tutorial` |
| **FTDI FT2232H** | USB to JTAG/SPI/I2C/UART | `ft2232h openocd jtag` |
| **Segger J-Link EDU** | JTAG/SWD debugger | `jlink edu jtag debugging` |
| **KiCad** | PCB design for capstone project | `kicad tutorial beginner pcb` |
| **i2c-tools** | `i2cdetect`, `i2cget`, `i2cset` | `i2c-tools linux tutorial` |
| **libgpiod** | `gpiodetect`, `gpioget`, `gpioset` | `libgpiod tutorial linux gpio` |
| **can-utils** | `candump`, `cansend`, `cangen` | `can-utils socketcan tutorial` |
| **v4l2-utils** | `v4l2-ctl`, `media-ctl` | `v4l2-ctl linux camera tutorial` |

---

## Essential References

| Resource | URL |
|---|---|
| Linux kernel source (driver examples) | https://elixir.bootlin.com/linux/latest/source |
| Bootlin embedded Linux training | https://bootlin.com/training/ |
| Device Tree specification | https://devicetree.org/specifications/ |
| Yocto Project documentation | https://docs.yoctoproject.org/ |
| Linux kernel documentation | https://docs.kernel.org/ |
| AUBoard-15P product page | https://www.avnet.com/americas/products/avnet-boards/avnet-board-families/auboard-15p-fpga-development-kit/ |
| AMD Vivado / PetaLinux docs | https://docs.amd.com/r/en-US/ug1144-petalinux-tools-reference-guide |
| Bootlin Elixir (kernel cross-ref) | https://elixir.bootlin.com/ |
| EEVblog Forum | https://www.eevblog.com/forum/ |
| Linux kernel mailing list archive | https://lore.kernel.org/ |
| Digi-Key reference designs | https://www.digikey.com/reference-designs |
| Mouser technical resources | https://www.mouser.com/technical-resources/ |

---

## 2-Year Timeline Summary

```
Month  1-2   ████  Phase 0: Electronics basics + kernel/Yocto foundations
Month  3-5   ██████  Phase 1: I2C chips (30 chips — sensors, expanders, EEPROM, RTC, power)
Month  6-8   ██████  Phase 2: SPI chips (13 chips — ADC, DAC, flash, displays, CAN)
Month  9-10  ████  Phase 3: UART + 1-Wire (6 chips — RS-485, GPS, temperature)
Month 11-12  ████  Phase 4: PMICs (6 chips — regulators, power sequencing)
Month 13-15  ██████  Phase 5: Networking (11 chips — Ethernet PHY, Wi-Fi, Bluetooth)
Month 16-17  ████  Phase 6: Audio + Motor + LED (9 chips — codecs, PWM, drivers)
Month 18-20  ██████  Phase 7: High-speed (10 chips — USB, PCIe, cameras)
Month 21-22  ████  Phase 8: FPGA integration (8 tasks — AXI peripherals, custom IP)
Month 23-24  ████  Phase 9: Capstone projects (5 system-level builds)
              ─────────────────────────────────────────────────
              Total: ~100 chips/tasks across 24 months
```
