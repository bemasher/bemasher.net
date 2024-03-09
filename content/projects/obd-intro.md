+++
title = 'Trip Logger'
date = 2024-03-09T12:06:00-07:00
draft = false
tags = ['Trip Logger']
+++

I like data, especially discovering interesting patterns in it. My car, a 2019 Honda Civic Hatchback produces an awful lot of it without even having to ask for it. This project is about collecting that data and having a look at it.

Vehicles manufactured after 1996 in the United States are required to implement OBD-II, an on-board diagnostics reporting capability.[^OBD] Luckily the standard requires a specific connector, placement, pinout, and permits a set of signaling protocols.

My Civic implements a few of these protocols, but the one of interest is Controller Area Network (CAN). One way to interact with your vehicle is using an ELM327-based[^ELM] device. These affordable devices abstract away the complexity of OBD-II behind a device with an AT command set.[^AT]

Years ago I purchased an ELM327 knockoff, but I never got very far into the weeds with it because it required a Wi-Fi connection, and in my case an app on my iPhone, none of which were open-source, free, or well-made. The device was also always on (draining my vehicle's battery), and the Wi-Fi access point had a hard-coded SSID and PSK, so I only used it when I needed a known-working implementation of OBD-II.

What I really wanted was trip logs that began at engine start, and ended at engine shutoff. Bonus points if these logs can be sync'd to my server when WiFi is in range.

The design requirements are as follows:

 * Automatic boot at engine start.
 * Automatic logging at boot.
 * Log both CAN and GPS data.
 * Automatic (safe) shutdown.

# Power

For automatic boot and shutdown, the Pi will boot whenever power is present, and shutdown (unsafely) when power is absent.

To power the Pi, there are a few options:

 * USB, there's a port in the center console that supplies enough current to run the Pi. This port is powered when the engine is running, or in accessory mode. This provides automatic boot at engine start, and automatic (unsafe) shutdown at engine stop.
 * 12V accessory port, this port only provides power when the engine is running.
 * OBD-II provides a direct connection to the battery, so I'll need a way to handle boot and shutdown on the Pi.

Powering the Pi directly from the battery through the OBD-II port makes the most sense. This requires a DC-DC converter to get 5V from the 12V-ish battery. I've already got some cheap DC-DC buck converter modules[^LM2596] rated for the necessary current.

A power button can be emulated using `dtoverlay=gpio-shutdown` which triggers when a specified GPIO pin is driven low. One of the two push-buttons on the PiCAN 2 can be used for this, and it allows triggering both boot and safe shutdown. I'll have to measure how much power the device and peripherals draw when everything is shutdown or asleep.

I've not yet determined how I'll handle automatic boot, for now I think I'll manually start the Pi with the push button. Automatic shutdown is easy: everything on the CAN bus stops transmitting shortly after the engine stops, so use a timeout to trigger shutdown after the last message is received.

# CAN Interface

There are quite a few accessory boards for platforms like the Raspberry Pi that marry a CAN controller,[^MCP2515] and transceiver[^MCP2551] to the pi. For this project, the most suitable for development I found is the PiCAN 2.[^PICAN2] This board includes both screw terminals and a DB-9 connector for connecting to the bus.

There are also some unpopulated footprints on the board for a switch-mode power supply, and momentary push buttons, which I'll populate.

Be careful when setting up this board for first use, a triplet of solder jumpers need to be set for the specific cable you'll use. I purchased an OBD-II to DB-9 cable from Adafruit.[^CABLE] It's pinout corresponds to the "CAN Cable" instead of the "OBDII Cable" described in the user guide.

# GPS

The vehicle doesn't know where it is in space and time, and GPS is a relatively easy way to get this data.

The Pi doesn't have a real-time clock, so when it first boots, it's not oriented to time. GPS is effectively a collection of very accurate clocks, so that problem is solved.

There are a few benefits to the device always having power: GPS time to first fix from cold boot is around 30 seconds, but if the GPS always has power and is instead put to sleep, TTFF is around 1 second, so the Pi has current time sooner.

For this application, I already have a u-blox MAX-M8Q module[^GPS] from Uputronics. I may replace this with the GPS/RTC hat[^HAT] from Uputronics to simplify the assembled package.

# That's all for now.

Tune in at some indeterminate future time for some software.

[^OBD]: [Wikipedia: On-board Diagnostics](https://en.wikipedia.org/wiki/On-board_diagnostics)
[^ELM]: [Wikipedia: ELM327](https://en.wikipedia.org/wiki/ELM327)
[^AT]: [Wikipedia: Hayes AT command set](https://en.wikipedia.org/wiki/Hayes_AT_command_set)
[^MCP2515]: [MCP2515 Stand-Alone CAN Controller with SPI Interface](https://ww1.microchip.com/downloads/en/DeviceDoc/MCP2515-Stand-Alone-CAN-Controller-with-SPI-20001801J.pdf)
[^MCP2551]: [MCP2551 High-speed CAN Transceiver](https://ww1.microchip.com/downloads/aemDocuments/documents/APID/ProductDocuments/DataSheets/20001667G.pdf)
[^PICAN2]: [PiCAN 2 CAN Bus Interface for Raspberry Pi](https://copperhilltech.com/pican-2-can-bus-interface-for-raspberry-pi/)
[^GPS]: [uBLOX MAX-M8Q Breakout for Active Antennas 5V](https://store.uputronics.com/index.php?route=product/product&path=60_64&product_id=84)
[^HAT]: [Raspberry Pi GPS/RTC Expansion Board](https://store.uputronics.com/index.php?route=product/product&path=60_64&product_id=81)
[^CABLE]: [OBD Plug (16-pin) to DE-9 (DB-9) Socket Adapter Cable - 1 meter long](https://www.adafruit.com/product/4841)
[^LM2596]: [LM2596 DC-DC Buck Converter Module](https://a.co/d/e0n8oBu)
