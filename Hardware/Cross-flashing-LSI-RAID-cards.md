# Cross-flashing LSI RAID cards

It's widely known that a lot of vendor-branded RAID cards are actually just rebadged LSI cards, and can be cross-flashed to an OEM LSI firmware, for example to turn a RAID card into a HBA (using the LSI IT firmware).

Here I document the cards I personally had success with cross-flashing and the methods I used to do so.

## Cross-flashing basics

I mainly followed the guide at [Fohdeesha Docs](https://fohdeesha.com/docs/perc.html).

He provides two iso images with scripts for different cards. The H310 scripts worked for me for the H310 card. I had to slightly modify the H310 script to get it to work on the IBM M1015.

### Dell PERC H310

- Boot the fohdeesha DOS iso
- run 'info', write down the SAS address
- run '310FLCRS' (this will flash a modified LSI SBR with Dell PCI subsystem VID/PID that is required for it to work in a storage slot in a Dell server - otherwise it won't harm anything)
- power off

This is where I deviated from his guide a bit, but still used his flashing script.
- mount the fohdeesha linux iso, mount the squashfs under '/live/filesystem.squashfs' and copy '/root' and '/usr/local/bin' from it onto your bootable Linux of choice (I had a SSD ready with Devuan)
- boot into Linux, cd to the directory where you put the copied files into
- edit 'bin/H310' and adjust all paths from '/root' to './root'
- run `bin/H310`, this will flash SAS2118 IT firmware
- set SAS address with `./root/sas2flash -o -sasadd <address>`
- (optionally) flash option ROM BIOS images with `/root/sas2flash -c 0 -b ./root/Bootloaders/mptsas2.rom -b ./root/Bootloaders/x64sas2.rom`

### IBM M1015

Same as Dell H310. I haven't tested compatibility with IBM or Dell servers. It might potentially work in a Dell, since the PCI subsystem VID/PID in the SBR are now for the Dell H310. For an IBM, if it doesn't want to boot with these VID/PID, you might need to make and flash your own SBR with IBM's VID/PID. You can find instructions on how to do this [here](https://github.com/marcan/lsirec/issues/1#issuecomment-574971959).
If your server doesn't complain if it sees a different subsystem VID/PID than its OEM wants (for vendor lock-in), the card otherwise works fine (maybe even in a generic PCIe slot not reserved for the RAID card).

## Updating firmware after cross-flashing

Simply use sas2flash.
