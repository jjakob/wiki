# Thinkpad T420 killed by decomposing foam corrosion

T420 that was completely dead, no response to the power button. The DC power cable sparked when plugged in, which is abnormal, indicating a possible short on the power input.

Disassembly revealed that the foam strips from the bottom cover had decomposed into a sticky goo that was also stuck to parts of the motherboard. After cleaning them off with label remover (containing Limonene, I found it works best for this) followed by isopropanol and drying in the oven, I found that where the foam residue was covering two caps, the cap leads were very corroded. Testing with a multimeter showed a dead short across the caps. Looking at the schematic the caps were on the main VINT20 power rail. After removing the two caps the short was gone, two replacement caps with the same value were fitted from a donor board. Since the main power rail was shorted I suspected other damaged components in the path from the DC input jack and the battery, which were found: shorted Q34 (SI7129DN), blown F12 (10A), shorted Q9 (SI7121DN). The MOSFETS were replaced with equivalent P-channel MOSFETS in SO-8 package, bodged in place of the much smaller original footprints.

The foam also corroded the pins of the EC chip and components near it, but the corrosion wasn't severe enough to cause damage, and was cleaned up with flux and hot air.

There was also corrosion on the SATA HDD connector pins which was cleaned with a fiberglass pencil and alcohol.

## Pictures

[T420 motherboard, top, annotated](t420_mobo_top_annotated.jpg)

[T420 motherboard, bottom, annotated](t420_mobo_bottom_annotated.jpg)

Pictures of the bottom cover with the bad foam circled in red:

[T420 bottom cover, annotated](t420_bottom_cover_annotated.jpg)

Original pictures:

[T420 motherboard, top](t420_mobo_top.jpg)

[T420 motherboard, bottom](t420_mobo_bottom.jpg)

[T420 bottom cover](t420_bottom_cover.jpg)

## Repair pictures

[Bodged Q9](t420_mobo_repair_1.jpg)

[Removed Q34 and scraped off solder mask to fit SO-8 replacement](t420_mobo_repair_2.jpg)

[Fitted replacement Q34 and F12](t420_mobo_repair_3.jpg)

## Flashing Libreboot

[Flashing setup with Raspberry Pi 3B](t420_flashing_1.jpg)

The wires are loosely held in place in the vias in the motherboard by the pressure of the wires only, no soldering required.

[Closeup of wires connected to motherboard](t420_flashing_2.jpg)
