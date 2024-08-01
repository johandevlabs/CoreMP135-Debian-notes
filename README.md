# CoreMP135-Debian-notes
Just some notes and issues I encountered while working with the M5-stack CoreMP135 and Debian Linux

## Journey

1. Downloaded the Debian 12 image (M5_CoreMP135_debian12_20240628) from https://docs.m5stack.com/en/guide/linux/coremp135/image
2. Flashed with BalenaEtcher to 8GB SD card
3. Powered up through USB-C via laptop
4. Connected to system terminal via USB (`picocom --baud 115200 /dev/ttyACM1`, Arduino serial monitor or Putty can also be used, note baud rate of 115200)
5. Login with user: `root` , pass: `root`.

## Issues

| Issue | Solution |
| ---- | ----|
| `free -m` only displays 441 MB ram (not 4GB!) | TBD |
| The root partion is not expanded to take up all space available on SD card (Debian image only allocates 1GB). | Use provided MMC resize tool (`/usr/local/m5stack/resize_mmc.sh`)

## To do

- Figure out how to interact with LCD screen in real-time / interactively
- Get tty terminal send to LCD screen
- WiFi dongle
- USB LTE dongle
- Setup time and RTC


## Enabeling WiFi through USB dongle

I connected a cheap (old) USB WiFi I had lying around.

Output from `dmesg` and `lsusb` showed that the system recognized the device
```
root@CoreMP135:~# dmesg | tail -n3
[   31.848927] vdd_adc: disabling
[   31.848991] v1v8_periph: disabling
[  313.258603] usb 2-1.3: new high-speed USB device number 4 using ehci-platform
```
and
```
root@CoreMP135:~# lsusb 
Bus 002 Device 004: ID 0bda:0179 Realtek Semiconductor Corp. RTL8188ETV Wireless LAN 802.11n Network Adapter
Bus 002 Device 002: ID 05e3:0610 Genesys Logic, Inc. Hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
Everything looks good so far!

Checked `ip link status` if new `wlan` interface is created
```
root@CoreMP135:~# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: can0: <NOARP,ECHO> mtu 16 qdisc noop state DOWN mode DEFAULT group default qlen 10
    link/can 
3: can1: <NOARP,ECHO> mtu 16 qdisc noop state DOWN mode DEFAULT group default qlen 10
    link/can 
4: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether b6:a2:69:ea:2f:c7 brd ff:ff:ff:ff:ff:ff
5: eth1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN mode DEFAULT group default qlen 1000
    link/ether ce:c0:9f:83:25:d7 brd ff:ff:ff:ff:ff:ff
```
Unfortunately, it shows no wlan0 interface

After digging around some more, it would seem that the M5 Debian OS does not include support for my WiFi dongle ... 

The M5 guide (https://docs.m5stack.com/en/guide/linux/coremp135/buildroot) actually talks about this. Their suggested solution is recompile OS image with buildroot and configure to include needed wifi drivers (rtlxxxx in my case)
