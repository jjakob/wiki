# Upgrading firmware on Emulex OCe10100 (IBM 49Y4201 49Y4202)

The IBM 49Y4201/49Y4202 use the OCe10100 chip and can be flashed with Emulex firmware. Unfortunately, I don't have compatible SFP transceivers for them so can't test them, but the card is detected in Linux and the Option ROM shows the new firmware version, so I assume it works. I need to get some compatible SFP+ (OCE10100-OPT) or maybe just one of them, read out its EEPROM and write it to a different SFP. (if anyone can make a dump of the OCE10100-OPT, please send it to me!)

## getting the firmware

The firmware can be found on Broadcom's support downloads website for the OCe10100 NIC.
[OneConnect Flash ISO Image version 10.2.370.19 x86](https://docs.broadcom.com/docs/12357080)
For some reason they also list ISOs with firmwares for the OCe11xxx on the same page, those have a version '11.x.x.x' and don't have the right `oc10-x.x.x.x.ufi` firmware file.

I couldn't get the above ISO to boot after copying it to a USB flash drive. Inspecting it, it had no MBR and so was probably meant just for CD booting.
For some reason isolinux's isohybrid tool refused to work on it to convert it to a hybrid ISO that could boot from USB. The 11.x ISO did convert, so I wrote it to the flash drive.
Then I mounted the 10.x ISO and copied oc10-10.0.803.31.ufi out of it onto a FAT32 formatted drive. Booted the 11.x ISO, mounted the flash drive with 10.x firmware file and flashed it.

## flash with ethtool

This might work. The one card I tested it on already had the same firmware as I was flashing to it, so I'm not sure, but it didn't complain.

```
cp /mnt/cd/oc10-10.0.803.31.ufi /lib/firmware/
ethtool -f enp1s0f0 oc10-10.0.803.31.ufi
```

## flash with OneConnect Flash ISO version 11.0.154.11

- convert the downloaded ISO to a hybrid iso with isohybrid (from isolinux)
- dd the iso to a usb flash drive
- boot the 11.x iso in the machine with the NIC
- plug in and mount the flash drive with the oc10-x.x.x.x.ufi file
- run the flash command with the oc10 file (TODO: look up the actual command)

## non-standard PCIe connector

The card has a non-standard connector that is 4 pins longer than a regular x8 connector (4 on one side, 8 pins if you count both). The extra pins have 2 traces wired to them whose function is unknown. They might be used in the IBM server for some sort of diagnostic or extra communication port, like I2C or SMBus (even though the PCIe standard has pins for SMBus and JTAG in standard locations and they are also wired on this IBM card). It didn't seem to affect its operation when plugged into a x16 slot, but the two traces land on two separate pairs of PCIe balanced data lanes, so they might confuse your mobo's PCIe bridge or cause signal integrity issues. To be safe, cover them up with tape. Or if you want to make the card fit into a x8 slot, cut them off with a dremel (carefully!).
