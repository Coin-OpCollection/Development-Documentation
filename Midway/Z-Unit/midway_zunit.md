# Midway Z-Unit
## Introduction
This document is a schematics-based overview of the Midway Z-Unit System, which is supported by images scanned by myself from the NARC service manual. In this overview, you will see sections of the schematics presented alongside technical information and implementation details from the Midway Z-Unit Core for the Analogue Pocket we have released.

### Legal Mumbo-Jumbo
Unfortunately, with the way everything has become now at days, I have to state that it is perfectly OK to link to this site and use materials from it ONLY if proper attribution is made. If you want to do something else with it and have any doubts, please contact us and we can answer. But please, do not be like certain people and just randomly create YouTube videos and posts without proper attribution, and always link back to the source.

## High Level Architecture
<img src="images/Scan_20241121-topaz-enhance-4x-cropped.png" width="1200">

The Z-Unit architecture is composed totally of 4 boards.
- Main, CPU Board
- Image/ ROM Board
- I/O Board
- Sound Board

All of these boards must be connected to eachother in a specific way, as outlined by the diagram above. Therefore, the most practical way to connect the system in a cabinet or home scenario is to have a mounting board and lay out all the pieces as shown. Then, connect the ribbon cables and also the power supply/ transformer to the side.

As it is very difficult to do this, and I might be missing some of the interface cables, during development, I did not hook up the board and observe it normally as I usually do as it was impractical for me to do so. I physically inspected the boards and reviewed certain parts as development commenced, but did not go through the expense of actually hooking things up. The transformer/ power supply I have does not support NARC as it needs enough power to power all the boards.

### Main CPU Board
<img src="images/Scan_20241121 (5)-topaz-enhance-4x-cropped.png" width="800">

#### Bill of Materials
|   Item No. | Part No.      | Part Description   | Description                 |   Qty |
|-----------:|:--------------|:-------------------|:----------------------------|------:|
|         15 | 5880-11056-00 | B1                 | Battery - Lithium 3V Button |     1 |
|         14 | A-5346-3036-7 | U28                | IC, PLD-COLOR RAM CNTL      |     1 |
|         13 | A-5346-3036-6 | U78                | IC, PLD-LOCAL CONTROL       |     1 |
|         12 | A-5346-3036-5 | U79                | IC, PLD-VIDEO RAM CNTL      |     1 |
|         11 | A-5346-3036-4 | U80                | IC, PLD-ADDRESS DECODE      |     1 |
|         10 | A-5346-3036-3 | U83                | IC, PLD-IMAGE ROM CNTL      |     1 |
|          9 | A-5346-3036-2 | U12                | IC, PLD-VIDEO RAM SEQ.      |     1 |
|          8 | A-5346-3036-1 | U20                | IC, PLD-AUTOERASE CNTL      |     1 |
|          7 | 5340-12019-00 | U65                | IC, RAM/S 5564 8K x 8       |     1 |
|          6 | 5340-12213-00 | U42-U49            | IC, RAM/V 4464 64K x 4      |    16 |
|          4 | 5400-12220-00 | U18                | IC, TMS34010 GSP            |     1 |
|          3 | 5410-12239-00 | U77                | IC, CUSTOM ASIC (DMA)       |     1 |
|          2 | 16-8850-210   | LABEL              | PCB IDENT.                  |     1 |
|          1 | C-11878       | SYS-Z              | CPU PCB SUB-ASSEMBLY        |     1 |

The above is a BOM which is listed in the manual. It indicates the major components on the CPU Board C-11879-3036. Below, I will go over some of the major components and what role they play in the FPGA Core and system.

#### U18, TMS34010 GSP/ CPU
This is the main CPU of the system, the [Texas Instruments TMS34010 CPU](https://en.wikipedia.org/wiki/TMS34010). The CPU was a powerhouse at the time this game was released, and Midway also used it in subsequent games on the Y, T and Wolf Units. Additionally, this CPU is not to be confused with the TMS34020, which was released later, functions a little bit differently, and is more suited towards 3D operations practically speaking.

As it indicates in the Wikipedia article, the chip was both a CPU and GPU. The CPU includes instructions for moving data, mathematics and other functional operations. The GPU side includes instructions for drawing data in VRAM, and bulk clearing. In addition, the chip also could generate sync and blank video signals, or, optionally, sync with an external video circuit that would do the same. It could also operate in interleaved and progressive mode.

From a development perspective, it took about 1 year total to implement the chip. I redesigned it and rewrote it multiple times for the MiSTer, and then rewrote it again for the Analogue Pocket. I have implemented the CPU and GPU parts totally. For the Analogue Pocket, I have eliminated the GPU as it is not used in games except for the POST screens and some diagnostic messages that are outside of the normal gameplay loop. I eliminated the GPU from the Analogue Pocket to get the CPU to fit in the smaller FPGA unit.

Other than replicating the functionality from the developer's guide published by TI directly, I have also used current emulator sources to test issues and adjudicate when the manual was unclear. As in the original design, the instruction executions were implemented with a custom microcode engine I designed. The GPU did not use microcode, and so I implemented that using standard verilog. I have only ever tested proper workings on the MiSTer, and intend to reintroduce the GPU back in the implementation for MARS as it has the space. The FPGA implementation has the CPU generating the video line counter and interesting to note, the video timings come from the game program, so there is no need to actually setup static video timings via measurement like we would normally do on other cores or systems.

#### U77, DMA
The DMA Chip is a very important chip in the system that handles all bitmap rendering. The DMA has a direct line to the Image/ ROM Board. It also has a direct line to the Bitmap and Palette RAM. Again, the graphics are not rendered by the CPU even though the CPU could technically do it. The reason why is because the CPU bus is too small. It can only take in and output 16 bits. That's not enough to power the requirements of the system, so they needed something that operated on a 32 bit bus.

In the games, there is a concept of "software sprites" and "software backgrounds". All the layering, object management and queues are managed by game software. DMA is simply responsible for pulling data from the ROM, and plotting it according to commands (also called modes) that are sent to it which describe the ways of processing the ROM Image Data before getting output to the Bitmap and Palette RAM. Normally, in other systems of the period, many had hardware sprites and graphics that would normally work by a custom chip.

#### U20, Autoerase PLD
The Autoerase PLD performs a very simple, but important function in the system and it exists in the Z and Y Units on most games. Pixel data goes from either the CPU or DMA to Bitmap and Palette RAM. Following that, there are line counters that run and determine what lines to output to the CRT. After the lines get pulled from Bitmap and Palette RAM to the line buffer and then eventually get read out to the CRT, there is a process for clearing the line that got outputted from the source memory to set the stage for the next frame - that's where the Autoerase PLD comes in.

The Autoerase PLD gets automatically invoked following a line output to the CRT. While it does its work, it acts as the complete bus master and locks out everything else from accessing the RAMs it touches (including the CPU). It is a very high priority to clear the line in memory to set the stage for the next frame.

It is interesting to note that although no games really take advantage or use it, the last 2 lines of the Bitmap and Palette RAM are intended to provide a value to clear the lines with. Usually all these values are the exact same values, but technically a game could write different values in each of the pixel positions, and it should be cleared with that line pattern.

#### U78, Local Control PLD & U80, Address Decoder PLD
These two chips provide control mechanisms for accessing devices external to CPU on the bus. For example, it handles addressing of I/O, ROMs and RAMs.

#### U28, Color RAM Control PLD & U79, Video RAM Control PLD
The local control and address decoder handle arbitration to individual devices on the bus, and the Color RAM and Video RAM controllers handle access to their respective ColRAM and VRAM components.

It is important to note that sometimes we use the term "Palette RAM" to refer to the part of the memory that holds the Palettes which are used to determine the colors outputted to the screen. However, in this system, that's a different thing. The Palette RAM is the upper part of a pixel. The lower part is called Bitmap RAM. Both are apart of Video RAM.

Color RAM, is really referring to the source of the data where the master palette entries and reference are located.

#### B1, Lithium 3V Battery, for CMOS
All settings for the game persist in CMOS RAM. In order to do this, nonvolatile memory is kept alive by a Lithium Battery while the system does not have power to it. This is common in most computer motherboards now at days, and even back then for the BIOS. The FPGA Core of course saves CMOS RAM, persisting it on the SD Card, and loads it back up when the core is started.

### Image/ ROM Board
<img src="images/Scan_20241121 (18)-topaz-enhance-4x-cropped.png" width="800">

#### Bill of Materials
|   Item No. | Part No.      | Part Description   | Description                 |   Qty |
|-----------:|:--------------|:-------------------|:----------------------------|------:|
|          3 | 3036*         | U23-U94            | GAME ROM SET                |    72 |
|          2 | 16-8850-219   | LABEL              | PCB IDENT.                  |     1 |
|          1 | C-12261       | SYS-Z              | ROM 2 SUB-ASSEMBLY          |     1 |

#### U23-U94, Image ROM Data
The entire board contains the Image ROM Data for the game. That is, the bitmap data that has the sprites, backgrounds, objects, etc. This has a 32 bit and 16 bit interface to it's respective connected components. The 32 bit interface is for the DMA Chip, and the 16 bit interface is for the CPU. Whenever possible and practical, the CPU too gets involved in pulling data from the Image ROM and using that either for collision detection, plotting pixels, or other sorts of operations.

### I/O Board
<img src="images/Scan_20241121 (27)-topaz-enhance-4x-cropped.png" width="800">

#### Bill of Materials
|   Item No. | Part No.      | Part Description             | Description                                |   Qty |
|-----------:|:--------------|:-----------------------------|:-------------------------------------------|------:|
|         15 | 16-8850-212   | LABEL                        | PCB IDENT.                                 |     1 |
|         14 | 5792-10026-00 | P4                           | CONNECTOR, 64P EURO-DIN CARD CONN., FEMALE |     1 |
|         13 | 5791-10862-10 | J5, J3, J4, J5, J6           | CONNECTOR, 10P MOLEX .156" CENTER PINS     |     5 |
|         12 | 5791-10862-04 | J1                           | CONNECTOR, 4P MOLEX, .156" CENTER PINS     |     1 |
|         11 | 5043-09845-00 | C2, C3                       | CAPACITOR, .001 MFD.                       |     2 |
|         10 | 5040-08986-00 | C1                           | CAPACITOR, 100 MFD.                        |     1 |
|          9 | 5043-08980-00 | B                            | CAPACITOR, 0.01 MFD.                       |     9 |
|          8 | 5010-08991-00 | R1                           | RESISTOR, 4.7K, 1/4W, 5%                   |     1 |
|          7 | 5060-10396-00 | SCR1, SCR2, SCR3, SRC4, SRC5 | SIP, 10P 8-RES/8-CAP NETWORK, 4.7K, 470 PF |     5 |
|          6 | 5551-09822-00 | L1                           | INDUCTOR, 4.7 UH                           |     1 |
|          5 | 5311-10948-00 | U9                           | 74HC138, H-CMOS, 3/8 DECODER               |     1 |
|          4 | 5311-12208-00 | U7, U8                       | 74ALS245, ALS TTL, OCTAL BUS TRANSCEIVER   |     2 |
|          3 | 5311-10945-00 | U6                           | 74HC32, H-CMOS, QUAD 2-INPUT OR GATE       |     1 |
|          2 | 5311-12287-00 | U1, U2, U3, U4, U5           | 74HC541, HC TTL, OCTAL BUFFER              |     5 |
|          2 | 5779-12265-00 |                              | BARE PC BOARD                              |     1 |

Nothing special with this, all this board does is handle the input interface for player 1 and 2.

### Sound Board
<img src="images/Scan_20241121 (31)-topaz-enhance-4x-cropped.png" width="800">

#### Bill of Materials
| Item No. | Part No.      | Part Designation               | Description                     | Qty |
|----------|---------------|--------------------------------|---------------------------------|-----|
| 7        | 3036*         | U3, U4, U5, U35, U38, U37, U36 | 27512/1001 EPROM 250 NS         | 7   |
| 6        | 5371-11087-00 | U8                             | YM3012 DUAL SERIAL DAC          | 1   |
| 5        | 5370-11088-00 | U7                             | YM2151 SOUND GENERATOR          | 1   |
| 4        | 5340-12278-00 | U2, U34                        | 8K X 8 STATIC RAM, 150NS        | 2   |
| 3        | 5400-10320-00 | U1, U33                        | 68B09E MICROPROCESSOR           | 2   |
| 2        | 18-8850-230   | LABEL                          | PCB IDENT.                      | 1   |
| 1        | C-12350       | SYS-Z                          | SOUND BOARD SUB-ASSY            | 1   |

#### U3, U4, U5, U35, U38, U37, U36, Sound ROMs
The game has multiple sets of sound ROMs. One set corresponds to the master sound CPU, and one corresponds to the slave CPU.

#### U1, U33, 68B09E Microprocessor
The game has 2 6809 CPUs, one is considered the Master, and the other is considered the Slave. The Master is responsible for music and FM. The Slave is responsible for speech synthesis.

#### U30, HC55536 CVSD Module
This is not actually included on the printed BOM, but it is on the sound board. This is responsible for playback of speech by the Slave CPU.

#### U7, U8, YM3012 & YM2151 Sound Generator & DAC
The function of this is to play back music and FM sounds, as it normally is in other systems.

#### U12, MC1408 DAC
This is something else that's not on the printed BOM, but there is another DAC on the board that is responsible for some sound effects. Namely, the famous BONG noises in the VRAM Test that happens in the POST.