use OpenWRT on MAC1200Rv2 
==========
Files to used in my openwrt devices.

```
MAC1200R_V2.0-factory-20161124.zip    Firmware download from Mercury website.
MAC1200R-V2.0-factory-20150228.zip    Firmware download from Mercury website.
mac1200r-uboot1-factory-20161124.bin  First Uboot extracted from MAC1200Rv2 device. (Thanks drbrains)
MAC1200R-uboot1-skipcheck-fakeMAC.bin Modified Uboot skips check firmware RSA sign. Modified from factory uboot.


ATTENTION: To update the first uboot, you need to be especially careful, otherwise your device may become bricked.

This document describes how to unlock MAC1200Rv2 to accept any firmware.

Prerequisites:
1. A computer with an ethernet port, IP set to 192.168.1.100, connect ethernet cable with the computer and MAC1200R LAN port.
2. A USB-serial connection, connect MAC1200R serial interface to the computer.
3. Putty, TFTPd, Hex-Editor. 

To update uboot and firmware, it includes four steps:
1.Connect serial to MAC1200R and enter recevery mode.
2.Query information from your own device, and prepare your own uboot image.
3.Flash new uboot into your device.
4.Restart device and update to your favorite firmware.

Step 1: Connect serial to MAC1200R and enter rescue mode.
  1.Open MAC1200R, there is 4-PIN serial interface available, connect it with USB-serial adaptor to computer.
  2.Run Putty, new connection with serial and 8-N-1 57600. 
  3.Power on MAC1200R, then input 'slp' as soon as possible because it only wait 5 seconds. The boot process should stopped and enter 'MT7628 #' prompt. 
    MT7628 # version
    U-Boot 1.1.3 (Nov 24 2016 - 09:50:44)

Step 2: Query information from your own device, and prepare your own uboot image.
  Note: This step can be omitted and you can directly to use MAC1200R-uboot1-skipcheck-fakeMAC.bin, if you don't care to use fake MAC address and device ID.

  MAC1200Rv2 has 8MB flash, MX25L6405D, 2048 pages * 4KB per page, 64KB page is used during erase.
  The base address of firmware is 0xbc000000,
    0xbc000000-0xbc7fffff: 8MB FLASH
      0xbc000000-0xbc01d800 : "factory_boot". --> This is the first uboot that we need to change.
      0xbc01d800-0xbc01e000 : "factory_info"  --\ These two parts will be erase during update, 
      0xbc01e000-0xbc020000 : "art"           --/ so you must backup data from your own device. 
      0xbc020000-0xbc030000 : "config"        : The saved configuation, clear when reset device.
      0xbc030000-0xbc040000 : "normal_boot"   ==\ This is second uboot. Factory firmware includes it.
      0xbc040000-0xbc172814 : "kernel"        ==| Linux kernel. Factory firmware includes includes it.
      0xbc172814-0xbc800000 : "rootfs"        ==/ Root filesystem. Factory firmware includes includes it.
      0xbc4b0000-0xbc800000 : "rootfs_data"   : The saved file, clear when reset device.
  Factory_info includes special information of your device, such as MAC address and device SN.
  My uboot file does not includes these data, that's why you need to prepare your own uboot image.
  
  1.Use "md.b" command to query information from your own device, you need to save them.
    Example:
      MT7628 # md.b bc01d800
      bc01d800: 4D 31 43 54 01 F2 00 01 7B 6D 61 63 3A BC 5F F6    M1CT....{mac:... 
      bc01d810: 5E 5F 5E 2C 70 69 6E 3A 00 00 00 00 00 00 00 00    ^_^,pin:        
      bc01d820: 2C 64 65 76 49 64 3A 30 31 32 33 34 35 36 37 38    ,devId:012345678
      bc01d830: 39 41 42 43 44 45 46 30 31 32 33 34 35 36 37 38    9ABCDEF012345678
      bc01d840: 39 41 42 43 44 45 46 30 31 32 33 34 35 36 37 2C    9ABCDEF01234567,
      bc01d850: 68 77 49 64 3A 6D 33 93 7F AC C0 08 CE D4 19 1B    hwId:........... 
      bc01d860: 8C CF DA 94 C1 2C 68 77 49 64 44 65 73 3A 7B 64    .....,hwIdDes:{d 
      bc01d870: 65 76 4D 6F 64 65 6C 3A 4D 41 43 31 32 30 30 52    evModel:MAC1200R
      bc01d880: 2C 32 2E 34 67 43 68 69 70 3A 4D 54 37 36 32 38    ,2.4gChip:MT7628
      bc01d890: 41 2C 35 67 43 68 69 70 3A 4D 54 37 36 31 32 45    A,5gChip:MT7612E
      bc01d8a0: 2C 68 77 56 65 72 3A 32 2E 30 7D 00 00 00 00 00    ,hwVer:2.0}     
      ... omitted ...
    You need to store bc01d800-bc020000.
  2.Edit MAC1200R-uboot1-skipcheck-fakeMAC.bin and replace part 0x01d800-0x020000 with your own data of bc01d800-bc020000, this is your own uboot binary.

Step 3: Flash new uboot into your device.
  1.Prepare tftpd on a computer, IP set to 192.168.1.100, I use Tftpd32 version 4.52(from http://tftpd32.jounin.net). 
  2.Enter tollowing command in 'MT7628 #' prompt(rescue mode).
    setenv ipaddr 192.168.1.1
    setenv serverip 192.168.1.100
    tftpboot 0x82000000 MAC1200R-uboot1-skipcheck-fakeMAC.bin
    md.b 0xbc003e80
      # To check current uboot version, it should be:
      # bc003e80: 55 2d 42 6f 6f 74 20 31 2e 31 2e 33 20 28 4e 6f    U-Boot 1.1.3 (No
      # bc003e90: 76 20 32 34 20 32 30 31 36 20 2d 20 30 39 3a 35    v 24 2016 - 09:5
      # bc003ea0: 31 3a 31 30 29 00 00 00 00 00 00 00 00 00 00 00    1:10)...........
    erase uboot
    cp.b 0x82000000 0 0x20000
    md.b 0xbc0141b0
      # To check update whether successed, it should be:
      # bc0141b0: 55 2d 42 6f 6f 74 20 31 2e 31 2e 33 20 28 4a 61    U-Boot 1.1.3 (Ja
      # bc0141c0: 6e 20 32 30 20 32 30 31 38 20 2d 20 30 30 3a 31    n 20 2018 - 00:1
      # bc0141d0: 35 3a 30 30 29 00 00 00 00 00 00 00 00 00 00 00    5:00)...........
      # bc0141e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................

   (!!!NOTE!!!: If flash not success, do not power off the device before you recover it, otherwise your device will be bricked.

Step 4: Restart device and update to your favorite firmware.
  You have several methods to update firmware:
    Factory firmware can be updated via Web GUI after normal start, http://192.168.100.1.
    Other image can be updated via recovery mode, tftp from 192.168.1.2.
      Recovery mode can be enter during boot time£¬press the reset button immediately and hold it for 10 seconds when power on.
    Use serial connection, enter rescue mode to update.

```