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
| The root partion is not expanded to take up all space available on SD card (Debian image only allocates 1GB). | Use provided MMC resize tool (`/usr/local/m5stack/resize_mmc.sh`) |
| CoreMP135 does not support WiFi dongle (no wifi interface shown in `ifconfig -a`) | See section "Enabeling WiFi through USB dongle" below |

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
7. Follow steps on compilations (compilation took more than 60 min in my case)
8. Reconfigure the build (`make ARCH=arm menuconfig`), go to `Target Packages -> Hardware` and select all the relevant Realtek drivers (see screenshot).
9. Re-compile with `make`.
10. The new drivers for Realtek dongle can now be found in `output/target/lib/modules/5.15.118/extra/`
```
$ ls output/target/lib/modules/5.15.118/extra
8188eu.ko  8189fs.ko  8723bu.ko  8812au.ko  dtbocfg.ko
8189es.ko  8192eu.ko  8723ds.ko  8821cu.ko
```
11. Copy the 8*.ko files to CoreMP135 using ssh or usb-drive and place in `/lib/modules/5.15.118/extra/`
12. On the CoreMP135, run `depmod -a` and `modprobe 8188eu` to load the driver (note: in my case 8188eu was the relavant driver, other drivers were not used)
13. Confirm that drivers are loaded
```
root@CoreMP135:~# dmesg
...
[ 1731.069172] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[ 1731.131069] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[ 1731.141324] cfg80211: loaded regulatory.db is malformed or signature is missing/invalid
[ 1731.145945] 8188eu: loading out-of-tree module taints kernel.
[ 1731.391181] RTW: rtl8188eu v5.2.2.4_25483.20171222
[ 1731.391432] usbcore: registered new interface driver rtl8188eu
```
and 
```
root@CoreMP135:~# lsmod 
Module                  Size  Used by
8188eu               1540096  0
cfg80211              651264  1 8188eu
...
```
14. Insert USB WiFi dongle in CoreMP135, output from `dmesg` and `ifconfig -a` shows that devices is detected, correctly loaded, and a new WiFi network interface is present (great success!)
```
root@CoreMP135:~# dmesg
...
[ 1753.338597] usb 2-1.4: new high-speed USB device number 4 using ehci-platform
[ 1753.533557] RTW: hal_com_config_channel_plan chplan:0x20
[ 1753.534556] RTW: rtw_regsty_chk_target_tx_power_valid return false for band:0, path:0, rs:0, t:-1
```
and
```
root@CoreMP135:~# ifconfig -a
...
wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.119  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::195a:3c18:1f9d:610b  prefixlen 64  scopeid 0x20<link>
        inet6 fdd9:dc0f:bcd:e7eb:80d0:fd82:4e49:c658  prefixlen 64  scopeid 0x0<global>
        ether e0:b2:f1:80:0c:63  txqueuelen 1000  (Ethernet)
        RX packets 1192  bytes 450683 (440.1 KiB)
        RX errors 0  dropped 160  overruns 0  frame 0
        TX packets 174  bytes 22196 (21.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
15. Configure WiFi to connect to your access point (e.g. using `nmtui`)
16. Check connection to internet
```
root@CoreMP135:~# ping google.com 
PING google.com (142.251.36.14) 56(84) bytes of data.
64 bytes from ams15s44-in-f14.1e100.net (142.251.36.14): icmp_seq=1 ttl=114 time=14.0 ms
64 bytes from ams15s44-in-f14.1e100.net (142.251.36.14): icmp_seq=2 ttl=114 time=15.2 ms
64 bytes from ams15s44-in-f14.1e100.net (142.251.36.14): icmp_seq=3 ttl=114 time=24.4 ms
```
### Notes and after thoughs
- I had to deviate from the M5Stack guide (https://docs.m5stack.com/en/guide/linux/coremp135/buildroot) because I could get all the steps to work correctly - in particular the custom firmware compilation)
- It accured to me that a much simpler way of getting needed drivers would be `apt install firmware-realtek` (I have not tested but it should provide all the needed drivers). This method would require internet access on the CoreMP135, e.g. through ethernet. Alternatively download (https://packages.debian.org/bookworm/all/firmware-realtek/download) and transfer via USB-Drive.

### Unorganized copy pastes

A script to build Debian image is now available in the M5Stack repository.

CoreMP135_buildroot-external-st
https://github.com/m5stack/CoreMP135_buildroot-external-st/blob/st/2023.02.10/tools/creat_coremp135_debian12_image.sh

How to implement (Japanese)
https://qiita.com/nnn112358/items/44921e2470353653058e

#### framebuffer notes
- FBTFT drivers under kernel extensions should not be enabled "this extra
	  package is only needed for linux kernels until v3.19, since
	  v4.0 the drivers are included in the staging area" see https://giters.com/notro/fbtft/issues/590
- https://community.milkv.io/t/spi-milk-v-duo-st7735/625 a thread on working with framebuffer TFT displays directly from Linux console (using Milk V DUO S)
- https://github.com/notro/fbtft?tab=readme-ov-file
- https://milkv.io/docs/duo/resources/spilvgl
- https://community.milkv.io/t/milk-v-duo-spi-st7789/131 all up and running on milk v duo!!
- https://stackoverflow.com/questions/48627344/what-is-the-relationship-between-framebuffer-vt-and-tty some explanations regarding fb and tty
