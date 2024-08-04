# CoreMP135-Debian-notes
Just some notes and issues I encountered while working with the M5-stack CoreMP135 and Debian Linux

## Useful links and resources
- https://docs.m5stack.com/en/core/M5CoreMP135
- https://docs.m5stack.com/en/guide/linux/coremp135/image
- https://community.m5stack.com/topic/6478/coremp135-debian-image/42
- https://community.m5stack.com/topic/6513/running-python-gui-apps-on-coremp135-with-debian

## Journey

1. Downloaded the Debian 12 image (M5_CoreMP135_debian12_20240628) from https://docs.m5stack.com/en/guide/linux/coremp135/image
2. Flashed with BalenaEtcher to 8GB SD card
3. Powered up through USB-C via laptop
4. Connected to system terminal via USB (`picocom --baud 115200 /dev/ttyACM1`, Arduino serial monitor or Putty can also be used, note baud rate of 115200)
5. Login with user: `root` , pass: `root`.

## Issues

| Issue | Solution |
| ---- | ----|
| `free -m` only displays 441 MB ram (not 4GB!) | Device only contains 512 MB ram, M5-stack wrote 4 Gb (giga-bit in their datasheet) |
| The root partion is not expanded to take up all space available on SD card (Debian image only allocates 1GB). | Use provided MMC resize tool (`/usr/local/m5stack/resize_mmc.sh`)

## To do

- Figure out how to interact with LCD screen in real-time / interactively
- Get tty terminal send to LCD screen
- WiFi dongle
- USB LTE dongle
- Setup time and RTC

## Interacting with CoreMP153 over USB-C

Connect USB-C to laptop (I am using Debian 11). 

1. Check that CoreMP153 serial interface shows up (`ls /dev/tty*`). In my case it shows up as `/dev/ttyACM0`.
2. Connect with serial monitor software (e.g. `picocom --baud 115200 /dev/ttyACM0`)
3. When prompted, login in with root, root. Sometimes 'serial gibberish' will cause a random user to be inputted when connecting with picocom -> just press enter to fault back to login dialogue

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

### Building drivers for USB WiFi dongle

1. Confirm which version of the kernel the CoreMP153 is using (`uname -r`). In my case, I am using `5.15.118`
3. Confirm which drivers you need to build (e.g. insert USB WiFi adapter and type `lsusb`, in my case it showed `Realtek Semiconductor Corp. RTL8188ETV Wireless LAN 802.11n Network Adapter`).
In my case, only Broadcom, ralink and marvell drivers where present in `/lib/modules/5.15.118/kernel/drivers/net/wireless/`, so I will be building drivers for Realtek - specifically rtl818x drivers.
5. Setup a environment for compiling the OS (see also https://docs.m5stack.com/en/guide/linux/coremp135/buildroot ). I am using a Cloud hosted VPC running Ubuntu 22.04.
6. Follow the steps in the buildroot guide to git clone the `CoreMP153_buildroot` and `external_st` directories
7. Follow steps on compilations (compilation took approx 30 min in my case)
