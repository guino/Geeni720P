# Geeni Look 720P Smart Home Camera Root and Customization

This project contains information and files to Root(hack) and Customize the Geeni 720P and similar cameras (i.e. geeni, teco, etc) running older 2.7.x firmware on older hardware.

### TL;DR

To save you time: A **programmer** is required to hack this device. You will need to remove the flash chip from the board, read/patch/write the flash and solder it back. 

If you have what it takes to do the hack (or just want to read about it) follow below.

### The device

Since I don't have the device myself, the hands-on work was done by @Ne3Mx on his camera. The initial discussion/posts can be seen here: https://github.com/guino/Merkury720/issues/7

This is the device in question:

![device](https://user-images.githubusercontent.com/19650964/110670144-a1dbf300-81a7-11eb-9fe5-263b1865ec7a.png)

This devide is older, has a differen OS + file System + Boot Loader than the other devices I previously looked at. We tried the previously used methods and here's the bottom line:

* https://github.com/guino/BazzDoorbell/issues/2 -- does not work because the manufacturer forgot to comment out `MTDNUM=5` in the script used by this method.
* https://github.com/guino/BazzDoorbell/issues/13 -- does not work because the manufacturer did not include the `cut` command used in the script used by this method.
* https://github.com/guino/Merkury720 -- does not work because the bootloader lacks the `run` command used by this method.
* Serial port access was limited -- we could get a rooted terminal but limited resources available (i.e. no SD card access) and the watchdog would reset the board fairly quickly (prevented us from trying anything more advanced). It is likely possible to hack it with just the serial port by mounting /dev and the SD card and running a watchdog resetting application (i.e. https://github.com/guino/BazzDoorbell/issues/4#issuecomment-742434771 ) then it should allow for making changes to the OS partition in order to run a custom script at boot time. I would probably need the device in hands to try different things in the short amount of time available before the watchdog reboots the device.

Here's the internal board:

![board](https://user-images.githubusercontent.com/19650964/110670264-c89a2980-81a7-11eb-8235-ab31543a0e10.png)

You can see the flash chip at the bottom-left and the UART pins at the top-lef (above SD card slot).

On the original issue you can see some logs showing the ppsMmcTool.txt being executed but the env and run comamands fail, so there's really no point getting into that.

### Down to business

First thing to do was to remove the chip (heat gun):

![noflash](https://raw.githubusercontent.com/guino/Geeni720P/master/img/noflash.jpg)

Now we use a cheap SPI flash programmer (ch341a_spi) modded for 3.3V with to read the chip:

`flashrom -p ch341a_spi -r flash.bin`

Next we examine+extract the flash file:

```
$ binwalk -e -M flash.bin
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
132404        0x20534         CRC32 polynomial table, little endian
393216        0x60000         uImage header, header size: 64 bytes, header CRC: 0x5BFD85EB, created: 2021-02-07 13:22:17, image size: 2054864 bytes, Data Address: 0x80008000, Entry Point: 0x80008000, data CRC: 0xC87931CD, OS: Linux, CPU: ARM, image type: OS Kernel Image, compression type: none, image name: "Linux-3.4.35"
393280        0x60040         Linux kernel ARM boot executable zImage (little-endian)
409623        0x64017         gzip compressed data, maximum compression, from Unix, last modified: 1970-01-01 00:00:00 (null date)
2686976       0x290000        JFFS2 filesystem, little endian
2742152       0x29D788        Zlib compressed data, compressed
2744100       0x29DF24        Zlib compressed data, compressed
2745772       0x29E5AC        Zlib compressed data, compressed
2747328       0x29EBC0        Zlib compressed data, compressed
2749492       0x29F434        Zlib compressed data, compressed
2751976       0x29FDE8        Zlib compressed data, compressed
(... irrelevenat output removed ...)
```

Based on the layout of the device previously known from /proc/cmdline: `mem=23M console=ttyAMA0,115200 loglevel=0 ppsdebug=off mtdparts=hi_sfc:192k(bld)ro,64k(env)ro,64k(enc)ro,64k(sysflg),2240k(sys),5m(app),448k(cfg) ppsAppParts=5 ip=192.168.1.10:::255.255.255.0 eth=08:88:12:3a:22:10)` we know that the app partition starts at offset 2624kb of the flash (0x290000) so we'll mount the JFFS2 partition named 290000.jffs2 extracted by binwalk:

```
$ mkdir app
$ sudo modprobe mtdram total_size=8192 erase_size=256
$ sudo modprobe mtdblock
$ sudo dd if=290000.jffs2 of=/dev/mtdblock2
11136+0 records in
11136+0 records out
5701632 bytes (5.7 MB, 5.4 MiB) copied, 0.0204717 s, 279 MB/s
$ sudo mount -t jffs2 /dev/mtdblock2 app
$ cd app
$ ls
bin  drv  hisi  home  lib  sound
$ sudo ls home/app
network  ppsapp  ppsdsry
```

There's the old/familiar ppsapp we need to patch for RTSP. We have the option to patch and flash it directly but in order to provide extra features I chose to just add a new startup script `home/init.d/S70custom` with the following contents:

```
$ chmod +x S70custom
$ cat S70custom 
#!/bin/sh

cat /proc/mounts > /tmp/hack

(
while true; do
 sleep 10
 if [ -e /mnt/mmc01/custom.sh ]; then
  cp /mnt/mmc01/custom.sh /tmp/custom.sh
  chmod +x /tmp/custom.sh
  /tmp/custom.sh
 fi
done
) &
```

With the script created/saved it's now time to put back into the flash:

```
$ sudo umount app
$ cp flash.bin flash-custom.bin
$ dd conv=notrunc if=290000.jffs2 of=flash-custom.bin bs=1 seek=$((0x290000))
5701632+0 records in
5701632+0 records out
5701632 bytes (5.7 MB, 5.4 MiB) copied, 5.68915 s, 1.0 MB/s
$ flashrom -p ch341a_spi -w flash-custom.bin
```

### Success!

With the flash written and soldered back on the board, the last steps are to prepare the SD card with the files from here: https://github.com/guino/Merkury720/tree/main/mmc and patch ppsapp as described in https://github.com/guino/ppsapp-rtsp (mostly the same since it's older software).

The result is a thumbs up from @Ne3Mx while viewing the RTSP feed on VLC:

![vlc](https://raw.githubusercontent.com/guino/Geeni720P/master/img/VLC.png)

While it required a more drastic approach it is a viable solution if you have the equipment and are willing to do the work.

The patch for the ppsapp mentioned in this project is posted here: https://github.com/guino/ppsapp-rtsp/issues/1#issuecomment-797815070
