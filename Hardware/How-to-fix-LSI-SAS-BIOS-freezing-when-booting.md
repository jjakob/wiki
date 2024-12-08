# How to fix LSI SAS BIOS freezing when booting

You just installed your LSI SAS card in your machine, and now it won't boot any more, it freezes right when it should start executing the option ROM of the PCIe card, and hangs there indefinitely with just a blank black screen.

If the card has IT firmware (you don't need its RAID functions), turns out, you don't need the BIOS on the card at all - just erase it! The card will still work fine when booted into the OS (at least Linux will). Of course by erasing the BIOS, you lose the ability to boot any drive attached to the card, as it won't hook Int13h or Int19h. You also lose the ability to run the configuration utility during POST, but you can do everything you need to with lsiutil from the OS.

I don't know the reason why the card BIOS freezes, but it may be if you have the card in a smaller slot than its size. In my case I had the x8 card in a x1 slot. If you need the card BIOS, try putting it in a slot equal or larger than its connector, so x8 card in a x8 or x16 slot, then maybe it will boot. Or it might be somehow incompatible with your mobo's BIOS, then that won't work either.

Tested on the following cards:
- LSI SAS3442E-R (chip SAS1068E B1) with MPTFW-01.33.00.00-IT
- IBM M1015 (FRU: 46M0861) (chip SAS2008 B2) cross-flashed to LSI SAS9211-8i with fw 20.00.07.00-IT (card is actually SAS9220-8i but FW is the same) 
- Dell PERC H310 (chip SAS2008 B2) cross-flashed to LSI SAS9211-8i with fw 20.00.07.00-IT
- LSI SAS9200-8e with fw 20.00.07.00-IT

## Getting lsiutil

### Debian/Devuan/Ubuntu

[HWRAID](https://hwraid.le-vert.net/wiki/) has repos that you can add to apt. Note that as of the time of this writing, lsiutil is only available for Debian Buster (10) and not any newer version, but you can add the buster repo to bookworm and it will still work (at least lsiutil will).
It will work on Devuan  too, just use the Debian buster repo.
In the Ubuntu repo, lsiutil is only available up until bionic, but may work in newer Ubuntu releases, I haven't tested it.

### Gentoo

sys-block/lsiutil is in the main repo, just emerge it.

### Binary

It's possible to get a static binary version of lsiutil that will run on many distros. It may be available somewhere on Broadcom's support downloads website. One source for it is at [Fohdeesha Docs](https://fohdeesha.com/docs/perc.html). On the linked page, there is a link to the 'Dell Perc Flashing ZIP' that contains two iso images. lsiutil is in the linux iso in '/root/lsiutil'. You don't need to write the iso to media and boot from it, you can loop-mount it and grab the files you need from it. `mount deesh-Linux-v2.3.iso /mnt/deesh-linux-iso`

## Erasing the BIOS from the card

If you are [cross-flashing]() the card, you will be erasing its entire flash memory anyway, so when flashing the new firmware to it, just don't flash the BIOS.

If you have a card that already has the BIOS flashed, you can easily erase it using lsiutil.

You need a machine that will boot with the card. I had an old Intel GCT945-HM that booted fine with the card in the x16 slot.

Boot into an OS with lsiutil. I had a SSD with Devuan 11 with lsiutil already on it.

If none of your machines can boot with the card, you may try hot-plugging it while booted. I don't recommend it, but I've hotplugged countless times and never had an issue, even with a mobo without hotplug support, but the arcing on the power pins during hot plugging might wear out the pins eventually (I've seen them arc a couple times). But it might as well result in a dead mobo or card. YMMV, do it at your own risk. First install any card that the OS will boot with. Preferably the same connector size as your LSI card, but I've had success with smaller x1 cards too. Boot into Linux, use lspci to find the "domain" of the card (e.g. '06:00.0'), then remove it via sysfs: `echo 1 > /sys/bus/pci/devices/0000\:06\:00.0/remove`. Then yank the card out and plug in the LSI card. Then rescan the bus `echo 1 > /sys/bus/pci/rescan`, the card should get detected and the driver loaded.

In lsiutil, first select your controller, enter the expert menu `e`, then `4.  Download/erase BIOS and/or FCode (update the FLASH)`.
When prompted for a filename, don't enter it (just press enter), when asked if you want to preserve the image, answer 'no' for both BIOS and EFI BIOS.
Then confirm that you want to erase the BIOS.

Below is a copy of the lsiutil session, but the card already had the BIOS erased, so it didn't erase it again, so the exact process will be different. (TODO: get a copy of an actual flash erasing session)

```
# lsiutil

LSI Logic MPT Configuration Utility, Version 1.62, January 14, 2009

1 MPT Port found

     Port Name         Chip Vendor/Type/Rev    MPT Rev  Firmware Rev  IOC
 1.  /proc/mpt/ioc0    LSI Logic SAS1068E B1     105      01210000     0

Select a device:  [1-1 or 0 to quit] 1

 1.  Identify firmware, BIOS, and/or FCode
 2.  Download firmware (update the FLASH)
 4.  Download/erase BIOS and/or FCode (update the FLASH)
 8.  Scan for devices
10.  Change IOC settings (interrupt coalescing)
13.  Change SAS IO Unit settings
16.  Display attached devices
20.  Diagnostics
21.  RAID actions
22.  Reset bus
23.  Reset target
42.  Display operating system names for devices
45.  Concatenate SAS firmware and NVDATA files
59.  Dump PCI config space
60.  Show non-default settings
61.  Restore default settings
66.  Show SAS discovery errors
69.  Show board manufacturing information
97.  Reset SAS link, HARD RESET
98.  Reset SAS link
99.  Reset port
 e   Enable expert mode in menus
 p   Enable paged mode
 w   Enable logging

Main menu, select an option:  [1-99 or e/p/w or 0 to quit] e

Enabled expert mode in menus

 1.  Identify firmware, BIOS, and/or FCode
 2.  Download firmware (update the FLASH)
 3.  Upload firmware
 4.  Download/erase BIOS and/or FCode (update the FLASH)
 5.  Upload BIOS and/or FCode
 6.  Download SEEPROM
 7.  Upload SEEPROM
 8.  Scan for devices
 9.  Read/change configuration pages
10.  Change IOC settings (interrupt coalescing)
13.  Change SAS IO Unit settings
14.  Change IO Unit settings (multi-pathing, queuing, caching)
15.  Change persistent mappings
16.  Display attached devices
17.  Show expander routing tables
18.  Change SAS WWID
19.  Test configuration page actions
20.  Diagnostics
21.  RAID actions
22.  Reset bus
23.  Reset target
24.  Clear ACA
33.  Erase non-volatile adapter storage
34.  Remove device from initiator table
35.  Display Log entries
36.  Clear (erase) Log entries
37.  Force full discovery
40.  Display current events
42.  Display operating system names for devices
44.  Program manufacturing information
45.  Concatenate SAS firmware and NVDATA files
46.  Upload FLASH section
47.  Display version information
48.  Display chip VPD information
49.  Program chip VPD information
50.  Dump MPT registers
51.  Dump chip memory regions
52.  Read/modify chip memory locations
54.  Identify FLASH device
55.  Force firmware to fault (with C0FFEE)
56.  Read/write expander memory
57.  Read/write expander ISTWI device
59.  Dump PCI config space
60.  Show non-default settings
61.  Restore default settings
66.  Show SAS discovery errors
67.  Dump all port state
68.  Show port state summary
69.  Show board manufacturing information
70.  Dump all device pages
80.  Set SAS phy offline
81.  Set SAS phy online
90.  Send SCSI CDB
95.  Send SATA request
96.  Send SMP request
97.  Reset SAS link, HARD RESET
98.  Reset SAS link
99.  Reset port
 e   Disable expert mode in menus
 p   Enable paged mode
 w   Enable logging

Main menu, select an option:  [1-99 or e/p/w or 0 to quit] 4

To erase an image:
  1.  hit RETURN when asked for a image file name
  2.  answer No if asked to preserve an existing image

Enter x86 BIOS filename:
No x86 BIOS image exists in FLASH, and image won't be downloaded

Enter FCode filename:
No FCode image exists in FLASH, and image won't be downloaded

Enter EFI BIOS filename:
No EFI BIOS image exists in FLASH, and image won't be downloaded

Main menu, select an option:  [1-99 or e/p/w or 0 to quit] 0

     Port Name         Chip Vendor/Type/Rev    MPT Rev  Firmware Rev  IOC
 1.  /proc/mpt/ioc0    LSI Logic SAS1068E B1     105      01210000     0

Select a device:  [1-1 or 0 to quit] 0
```

That's it. Reboot or shutdown, install the card in the problematic machine, it should boot without freezing.
