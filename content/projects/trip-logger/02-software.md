+++
draft = true
title = 'Software'
date = 2024-03-17T15:45:00-07:00
tags = ['Raspberry Pi','CAN']
+++

## Power Button

The PiCAN 2 has footprints for push-button switches, including the two 4.7k pull-up resistors. Below you'll see `S1`, `S2`, `R14`, and `R15` now crookedly populated. I prefer not to hand-solder SMD components smaller than 0805, but here we are.

![PICAN 2 PCB](../images/solder.webp "PiCAN 2 PCB")

## First Issue

First issue was a documentation misinterpretation. The newly populated switches don't do anything without configuration. In `/boot/firmware/config.txt` configuration for the overlay is as follows:

    dtoverlay=gpio-shutdown,gpio_pin=24,active_low=1

Since `gpio_pin` is configurable, I interpreted this to mean that any GPIO pin could be configured for both shutdown and power-up.

However, `dtoverlay -h gpio-shutdown` states:

 > This overlay only handles shutdown. After shutdown, the system can be powered up again by driving GPIO3 low. The default configuration uses GPIO3 with a pullup, so if you connect a button between GPIO3 and GND (pin 5 and 6 on the 40-pin header), you get a shutdown and power-up button. Please note that Raspberry Pi 1 Model B rev 1 uses GPIO1 instead of GPIO3.

Testing shows this does indeed trigger a (safe) shutdown, but has no effect after shutdown. A jumper between `GPIO24` and `GPIO3` confirms that only `GPIO3` can trigger a boot, regardless of what is specified in `config.txt`.

## PiCAN 2 Configuration

The PiCAN 2 documentation is quite thorough and helpfully provides the following configuration for `config.txt`:

    dtparam=spi=on
    dtoverlay=mcp2515-can0,oscillator=16000000,interrupt=25
    dtoverlay=spi-bcm2835-overlay

I'm running the lite-flavor of Raspberry Pi's Debian Bookworm, so this just works.

## Frame Capture

The interface needs to be brought up before interacting with it:

    ip link set can0 up type can bitrate 500000

Without writing any software, frames can be captured with pre-built software. Installing `can-utils` provides `candump` which does what you think it does. The invocation looks like:

    candump -l -D can0

 * `-l` logs can-frames into a file.
 * `-D` doesn't exit if the interface goes down.
 * `can0` is the interface to listen on.

These can be simply placed in a shell script and executed at boot with a systemd service `/usr/lib/systemd/system/candump.service`:

    [Unit]
    Description=Candump

    [Service]
    ExecStart=/home/bemasher/capture.sh
    WorkingDirectory=/home/bemasher

    [Install]
    WantedBy=multi-user.target

This is enough for a test-drive with some jury-rigging for power. The logs `candump` produces are fairly simple to parse and have the form: `(timestamp) interface id#data`

    (1710727895.060047) can0 158#0000000000000037
    (1710727895.048982) can0 191#010000A9A921013F
    (1710727895.049581) can0 1AB#800028
    (1710727895.051942) can0 130#0000FFFF00000422
    (1710727895.052131) can0 17C#0000060001002029
    (1710727895.052250) can0 1D6#0013
    (1710727895.052491) can0 1DC#02060000000015
    (1710727895.052738) can0 1DD#0045082D0000001C

It may not be the case for all vehicles, but mine is quite chatty without having to prod it for information with OBD-II. With a DBC[^DBC] as a starting point, I can extract nearly all of the information I'd want. Below is a chart showing fuel consumed, distance traveled, and speed.

![PICAN 2 PCB](../images/trip_log.svg "PiCAN 2 PCB")

[^DBC]: A file describing signals and how to decode them from canbus frames. [https://github.com/commaai/opendbc](https://github.com/commaai/opendbc)