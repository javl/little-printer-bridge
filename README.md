
![Little Printer and a custom bridge device](https://github.com/javl/little-printer-zigbee-bridge/blob/main/header.jpg?raw=true)

# Little Printer Bridge

## What is this about?

The Little Printer is an internet-connected thermal receipt printer released by BERG in 2012. But when their servers went down in 2015, the printers stopped working.

There is altered firmware available for the bridge - the device that sits between the printer and your router - that makes it use an alternative server, but flashing this firmware requires some technical knowhow and it won't help users who's bridge broke or got lost over the years.

This project means to offer an alternative to the original bridge by using a Raspberry Pi + Zigbee dongle, or an ESP32-C6 microcontroller. It also extends the system with support for USB receipt printers.


---

- [Little Printer Bridge](#little-printer-bridge)
  - [What is this about?](#what-is-this-about)
  - [Ecosystem](#ecosystem)
    - [1. Python bridge (this repository)](#1-python-bridge-this-repository)
    - [2. ESP32 bridge](#2-esp32-bridge)
    - [3. New server](#3-new-server)
  - [Support](#support)
  - [Installation](#installation)
    - [Option 1: Install the bridge on a Raspberry Pi](#option-1-install-the-bridge-on-a-raspberry-pi)
      - [What you'll need:](#what-youll-need)
      - [Installation steps:](#installation-steps)
    - [Option 2: Run the Python bridge locally](#option-2-run-the-python-bridge-locally)
  - [Settings and arguments](#settings-and-arguments)
    - [Supported servers](#supported-servers)
    - [Arguments](#arguments)
  - [Fake printer (local testing)](#fake-printer-local-testing)
    - [Arguments](#arguments-1)
    - [Examples](#examples)
  - [config.json](#configjson)
  - [Troubleshooting](#troubleshooting)
  - [Thanks](#thanks)
  - [License](#license)

---

## Ecosystem

The project contains three parts:

### 1. Python bridge (this repository)

A Python script that can fully replace the original bridge device. You need a Zigbee USB dongle and a computer to run the bridge on, or you can use a Raspberry Pi to create a new standalone bridge device. This bridge also works with USB receipt printers (two Epson models have been tested).

### 2. ESP32 bridge

An alternative to this Python bridge that runs on a Seeed Studio XIAO ESP32-C6 microcontroller. This device is even cheaper and smaller than a Raspberry Pi Zero, but does not support USB printers. More information and a flashing tool can be found at [littleprinter.jaspervanloenen.com/esp-bridge](https://littleprinter.jaspervanloenen.com/esp-bridge).

### 3. New server

A new privacy-friendly server available at [littleprinter.jaspervanloenen.com](https://littleprinter.jaspervanloenen.com). This server reintroduces the ability to update the printer's personality - the face printed after every message - and is in active development.

---
## Support

Did you find this tool useful? Feel free to support my open source tools - especially when using them commercially:

[![GitHub Sponsor](https://img.shields.io/badge/_-sponsor_on_Github-blue?logo=github)](https://github.com/sponsors/javl) [![BMC](https://img.shields.io/badge/Buy_Me_a_Coffee-orange?logo=buymeacoffee)](https://www.buymeacoffee.com/javl)

---

## Installation

> If you don't have a Raspberry Pi and/or Zigbee dongle, or want a cheaper and smaller alternative, you can get the [Seeed Studio XIAO ESP32-C6](https://www.seeedstudio.com/Seeed-Studio-XIAO-ESP32C6-p-5884.html) and flash the bridge firmware via the online tool at [littleprinter.jaspervanloenen.com/esp-bridge](https://littleprinter.jaspervanloenen.com/esp-bridge).

There are two ways to run this Python bridge:

1. Set up and run the Python bridge on a Raspberry Pi
2. Run it directly on your local machine

### Option 1: Install the bridge on a Raspberry Pi

Setting up this bridge on a Raspberry Pi doesn't require any programming. Below are the basic instructions, or [read the full instructions](https://github.com/javl/little-printer-bridge/wiki/Install-on-Raspberry-Pi/_edit) (including screenshots).

#### What you'll need:

* A Raspberry Pi - the older Zero W model works fine
* Power adapter for the Raspberry Pi
* An SD card (8 GB is sufficient) and an SD card reader
* A Zigbee adapter - the Sonoff ZBDongle-E works well
* A USB-A to micro-USB adapter if your Raspberry Pi model doesn't have full-size USB ports
* A short USB extension cable - Zigbee adapters are prone to interference when plugged directly into a Raspberry Pi



#### Installation steps:
1. Insert your SD card into your reader.
2. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/), use it to flash the latest Raspberry Pi OS Lite image to the SD card. When you get to the `customisation` section, press skip.
3. When flashing is done, open the newly created `boot` partition and paste the scripts below into the `network-config` and `user-data` files.

   **network-config** - update with your WiFi credentials:

   ```yaml
   network:
     version: 2
     wifis:
       wlan0:
         dhcp4: true
         # Change this to your country code (ISO 3166-1 alpha-2)
         regulatory-domain: "NL"
         access-points:
           "YourNetworkName":
             password: "YourPassword"
         optional: false
   ```

   **user-data:**

   ```yaml
     #cloud-config

     hostname: little-printer-bridge
     manage_etc_hosts: true

     timezone: Europe/Amsterdam
     keyboard:
       layout: us

     # Create bridge user
     users:
       - name: bridge
         groups: users,adm,dialout,audio,netdev,video,plugdev,cdrom,games,input,gpio,spi,i2c,render,sudo
         shell: /bin/bash
         lock_passwd: false
         passwd: "$6$boqkDuR6gaVxXbFA$glw/miVRIqHulbFK0Bebyyov0NKQ3z7tlyUS7RYkXGjDozcEiSG.jSw2IDSI/ghItMpaJx/92Zajr.v6ebqmU1"

     rpi:
       interfaces:
         i2c: true
         spi: true

     # SSH disabled by default for security
     # enable_ssh: true

     # Update system packages
     package_update: true
     packages:
       - libopenjp2-7
       - python3-pip
       - git
       - libfreetype-dev

     # Create service file
     write_files:
       - path: /etc/systemd/system/little-printer-bridge.service
         permissions: '0644'
         owner: root:root
         content: |
           [Unit]
           Description=Little Printer Bridge Script
           After=network-online.target
           Wants=network-online.target

           [Service]
           Type=simple
           User=bridge
           WorkingDirectory=/opt/little-printer-bridge
           ExecStartPre=/bin/sh -c 'until ping -c1 1.1.1.1 > /dev/null 2>&1; do sleep 2; done'

           ExecStart=/opt/little-printer-bridge/venv/bin/python3 -m bridge.main

           Restart=on-failure
           RestartSec=10

           [Install]
           WantedBy=multi-user.target

     runcmd:
       # Clone the repository into /opt
       - git clone https://github.com/javl/little-printer-bridge.git /opt/little-printer-bridge

       # Transfer folder ownership to the bridge user so it can generate config.json
       - chown -R bridge:bridge /opt/little-printer-bridge

       # Create virtual env
       - python3 -m venv /opt/little-printer-bridge/venv

       # Install python requirements into venv
       - /opt/little-printer-bridge/venv/bin/pip install -r /opt/little-printer-bridge/bridge/requirements.txt

       # Tell systemd to find the file created by write_files, then boot it up
       - systemctl daemon-reload
       - systemctl enable little-printer-bridge.service
       - systemctl start little-printer-bridge.service

       # Disable cloud-init for subsequent boots
       - touch /etc/cloud/cloud-init.disabled
   ```

4. Eject the SD card, insert it into the Raspberry Pi, and power it on.
5. The script will automatically run and install everything. The Raspberry Pi will reboot multiple times during this process. Depending on your internet speed and Raspberry Pi model this can take a while - around 20 minutes on a Raspberry Pi Zero W.

To verify it worked, put your Little Printer into pairing mode by holding the button on the inside until the light turns off. The light will blink while the printer searches for a bridge, then turn solid when it finds one and has a claim code ready.

Press the button on top to print the code, then visit [littleprinter.jaspervanloenen.com](https://littleprinter.jaspervanloenen.com) to claim your printer and start printing.

### Option 2: Run the Python bridge locally

* Tested with **Python 3.9** and up, on Linux and macOS
* For use with Little Printer, you need a Zigbee adapter - for example, the Sonoff ZBDongle-E
* For USB printer support you need `libusb` installed (see [Troubleshooting](#troubleshooting))

Get the code, install the Python dependencies and run the bridge. Depending on your system you may need to replace `python3` and `pip3` with `python` and `pip` - verify with `python --version` that the version is >= 3.9. Using a virtual environment is recommended:

```bash
git clone git@github.com:javl/little-printer-bridge.git
cd little-printer-bridge
python3 -m venv venv
source venv/bin/activate
pip install -r bridge/requirements.txt

python3 -m bridge.main

# or, if you don't have a zigbee adapter and want USB only:
python3 -m bridge.main --no-zigbee
```

To verify it worked, put your Little Printer into pairing mode by holding the button on the inside until the light turns off. The light will blink while the printer searches for a bridge, then turn solid when it finds one and has a claim code ready.

Press the button on top to print the code, then visit [littleprinter.jaspervanloenen.com](https://littleprinter.jaspervanloenen.com) to claim your printer and start printing.

---

## Settings and arguments

### Supported servers

Both the Raspberry Pi bridge and the ESP32-C6 bridge support two types of servers:

1. **The new server** at [littleprinter.jaspervanloenen.com](https://littleprinter.jaspervanloenen.com). A privacy-friendly alternative with support for both original Little Printers and USB receipt printers, including `personalities` (the face printed after every message).
2. **Sirius-based servers**, such as the one hosted by [Nord Projects](https://littleprinter.nordprojects.co/). This has been the main replacement for most people but has some limitations - most notably requiring an X account to log in. To use a Sirius server, add the `--sirius` flag. By default this uses Nord Projects' server, but you can specify a different one with `--sirius-server-url <URL>`.

### Arguments

| Argument | Default | Description |
|---|---|---|
| `--port PORT` | from config | Serial port of the EZSP dongle (e.g. `/dev/ttyUSB0`) |
| `--lp-server-url URL` | production URL | WebSocket URL of the LP server |
| `--sirius` | - | Connect to a Sirius server as a Berg bridge client |
| `--sirius-server-url URL` | Nord Projects instance | WebSocket URL of the Sirius server |
| `--no-usb` | - | Disable USB ESC/POS printer discovery and printing |
| `--no-zigbee` | - | Skip Zigbee init (for USB-only setups without a Zigbee dongle) |
| `--setup-udev` | - | Write udev rule for any inaccessible USB printer (Linux only, requires root) |
| `--clear-devices` | - | Remove all paired devices from NCP key table and config, then exit |
| `--new-network` | - | Discard stored network and form a new one with fresh EPAN and key |
| `--debug` | - | Enable DEBUG-level logging |

---

## Fake printer (local testing)

`fake_printer.py` emulates a Little Printer over WebSocket - no Zigbee dongle or physical hardware needed - useful for testing. Incoming prints are saved as `receipt.png`.

```bash
python3 fake_printer.py
```

On first run it prints a claim code to the terminal. Enter it on the server to pair the device. Once paired, any print job sent to it is decoded and saved as `receipt.png` in the current directory (overwritten each time).

### Arguments

| Argument | Default | Description |
|---|---|---|
| `--device-id HEX16` | `deadbeef00000001` | EUI64 as 16-char little-endian hex. Change to run multiple fake printers simultaneously |
| `--server-url URL` | LP server default | WebSocket URL of the LP server |
| `--config PATH` | `bridge/config.json` | Config file to read/write pairing state |

### Examples

```bash
# Start a fake printer (claim code printed on first run):
python3 fake_printer.py

# Run two fake printers at the same time:
python3 fake_printer.py --device-id deadbeef00000001
python3 fake_printer.py --device-id deadbeef00000002

# Point at a local server:
python3 fake_printer.py --server-url ws://localhost:8080/api/v1/connection
```

Pairing state is stored in `bridge/config.json` under the `ws_devices` key. Delete that entry to force a new claim code on the next run.

---

## config.json

Generated automatically on first run:

| Field | Description |
|---|---|
| `ezsp_port` | Serial port of the EZSP dongle (e.g. `/dev/ttyUSB0`) |
| `ezsp_baud` | Baud rate (typically 115200) |
| `channel` | Zigbee channel (one of 11, 14, 15, 19, 20, 24, 25) |
| `extended_pan_id` | 8-byte hex. First 4 bytes are always `42455247` ("BERG") - the printer scans for this prefix |
| `network_key` | 16-byte hex AES network key, randomly generated |
| `print_id` | Auto-incrementing counter used to match print confirmations |
| `devices` | Dict of EUI64 → `{claim_code, link_key}` for each paired Zigbee printer |
| `usb_devices` | Dict of `vendor:product` → device info for each paired USB printer |
| `ws_devices` | Dict of BE device address → `{claim_code, link_key}` for each paired fake (WebSocket) printer |

> **Note:** Do not change `extended_pan_id` or `network_key` after a printer has been paired. The printer will need to be re-paired.

---

## Troubleshooting

**`FileNotFoundError: [Errno 2] No such file or directory: '/dev/ttyUSB0'`**

Dongle not detected at the given port. Make sure it is plugged in and update the port in `config.json` if needed.

---

**Windows: drivers for the Sonoff Zigbee dongle**

1. Download `CP210x Universal Windows Driver` from [silabs.com](https://www.silabs.com/software-and-tools/usb-to-uart-bridge-vcp-drivers?tab=downloads).
2. Extract the `.zip` file somewhere on your system.
3. Open Device Manager, right-click your dongle in the list and select `Update Driver`. Select the directory you extracted the driver to.
4. Find the Sonoff device under `Ports` and note the COM port (e.g. `COM3`). Update `bridge/config.json` accordingly or pass `--port COMx` to the script.

---

**`ImportError: libopenjp2.so.7: cannot open shared object file: No such file or directory`**

Install the missing module:

```bash
sudo apt-get install libopenjp2-7
```

---

**USB ESC/POS printer: "insufficient permissions" or claim slip not printing**

Access to USB devices requires the right permissions. The bridge logs the exact command to fix this.

**Linux / Raspberry Pi:** Create a udev rule for your printer's vendor and product ID (shown in the error log):

```bash
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="XXXX", ATTRS{idProduct}=="XXXX", MODE="0666"' \
  | sudo tee /etc/udev/rules.d/99-usb-printer.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Replace `XXXX:XXXX` with the actual vendor/product ID shown in the log. Unplug and replug the printer afterwards.

**macOS:** libusb usually works without extra steps, but macOS may claim the device with its built-in printer driver, blocking pyusb. If you get a permissions or "device busy" error, unload the Apple USB printer driver temporarily:

```bash
sudo kextunload -b com.apple.driver.AppleUSBPrinter
```

Reload it with `sudo kextload -b com.apple.driver.AppleUSBPrinter` when done.

**Windows - this is currently untested! So do let me know how you get on!**: pyusb requires the WinUSB driver instead of the default Windows printer driver. Use [Zadig](https://zadig.akeo.ie/) to replace the driver:

1. Open Zadig, select your printer from the list.
2. Choose `WinUSB` as the target driver.
3. Click `Replace Driver`.

Note: this replaces the Windows print driver, so the printer will no longer work with normal Windows print dialogs until you restore the original driver.

---

## Thanks

- Thanks to [BERG](https://berglondon.com/projects/) for creating the Little Printer in the first place
- Huge thanks to [Nord Projects](https://nordprojects.com) for reviving the cloud service, providing instructions for updating the bridge device, and creating a new mobile app
- Anyone who donated to support my open source projects

---

## License

In the spirit of open source and to prevent users getting locked out of their printers again, this project is shared under a GNU GPLv3 license. This means you can use it pretty much in any way you like (including commercially) as long as you give proper attribution and share any changes you make. If you do make any changes that might benefit others, please share them here as a pull request as well.
