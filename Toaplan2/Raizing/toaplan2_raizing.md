# **Toaplan2 - Raizing**
## **Introduction**
This document is a technical overview of the Toaplan 2 Raizing System. It covers the high level architecture and subsystems of games Sorcer Striker, Kingdom Grandprix, Battle Garegga, Batrider and Battle Bakraid. All the boards are different and cannot be converted to eachother as well.

### **Legal Mumbo-Jumbo**
Unfortunately, with the way everything has become now at days, I have to state that it is perfectly OK to link to this site and use materials from it ONLY if proper attribution is made. If you want to do something else with it and have any doubts, please contact us and we can answer. But please, do not be like certain people and just randomly create YouTube videos and posts without proper attribution, and always link back to the source.

## **High Level Architecture**
The Raizing series of games is based for the most part on the Toaplan2 Platform architecture. Central to the graphics system is the GP9001 chip. The chip is used as a GCU (Graphics Coordination Unit) in the platform, this is not the same as a GPU, however. The system boasts a large graphics rom and banking capability as well as 256 sprites on screen at any given point. The sprites can get quite large in the games, and there is a concept of linked and chained sprites which are smaller sprites that make up a larger whole. All the games use ADPCM sound and the YM2151 in conjunction with a Z80. It is a single board platform where all the necessary components are integrated.

### **CPU**
We start our journey on this platform with the CPU: A Motorola 68000. The CPU in the platform handles all coordination aspects between various devices on the bus including the GP9001, sound components, memory and I/O. There is an extensive POST test at the beginning of each game in the series which pretty much checks everything, including the ROM data. As a result, the CPU also has access to the ROM data as well, including the sound CPU program, which is somewhat uncommon.

In the POST process, the text ROM data is also unpacked to memory so it is in a usable form by the text layer.

### **GP9001 GCU**
The primary and novel device that is used by the system is the GP9001 GCU. It is responsible for handling coordination of all the rendering of the graphics in the system short of actually outputting the pixels on the screen. It handles the majority of the video chain and also utilizes several custom chips to achieve its work.

#### **Interface with 68K CPU**
On the address bus, the 68K performs operations and makes requests to the GP9001. The requests are the following, and there is an acknowledgment it receives back that the request was completed:

|Address (8 bit addressing)|Read or Write|Function|
|-|-|-|
|0x400008|Read|Read RAM (Even Word)|
|0x40000A|Read|Read RAM (Odd Word)|
|0x400000|Write|Write GP9001 Register|
|0x400004|Write|Select GP9001 Register|
|0x400008/A|Write|Write GP9001 RAM|
|0x40000A|Write|Set GP9001 RAM Pointer|

The RAM in question that is connected to the GP9001 is basically a contiguous RAM that comprises:
- Sprite RAM (Double/Triple Buffered, depending on game)
- Scroll Layer 0 RAM
- Scroll Layer 1 RAM
- Scroll Layer 2 RAM

The entirety of the RAM is 0x2000 words that are 16 bits in length. Scroll 0-2 RAM is 0x800 each in length, and Sprite RAM is 0x400 in length.

So basically, the CPU interfaces with the RAMs that provide the main graphics queues for the scroll and object layers via the GP9001 interface on the address bus. How it works is the CPU sets a starting pointer at any time, and with each subsequent write, that pointer auto increments by 1 on the GP9001 side thereby allowing the CPU to fill contiguous blocks of data. It can, of course, reset the pointer at any time to something else and start the process all over again. It must also do so when reading back address contents from it's memory.

#### **GP9001 Registers**
As the GP9001 is a controller for graphics operations, it has a number of registers that direct the function of subsystems such as the scroll layers and object layers. The registers basically control the rendering properties of the layers as a whole, whereas specific properties that are located in the RAM control the rendering properties of an individual component or object on that layer.

Below is a list of registers in the GP9001 and what they do.

| Address  | Register                | Function                                                    |
|----------|-------------------------|-------------------------------------------------------------|
| 0x00 / 0x80 | Background Scroll X   | Sets the horizontal scroll position for the background layer. |
| 0x01 / 0x81 | Background Scroll Y   | Sets the vertical scroll position for the background layer.   |
| 0x02 / 0x82 | Foreground Scroll X   | Sets the horizontal scroll position for the foreground layer. |
| 0x03 / 0x83 | Foreground Scroll Y   | Sets the vertical scroll position for the foreground layer.   |
| 0x04 / 0x84 | Text Scroll X         | Sets the horizontal scroll position for the text layer.       |
| 0x05 / 0x85 | Text Scroll Y         | Sets the vertical scroll position for the text layer.         |
| 0x06 / 0x86 | Sprite Scroll X       | Sets the horizontal scroll position for sprites.              |
| 0x07 / 0x87 | Sprite Scroll Y       | Sets the vertical scroll position for sprites.                |
| 0x0E        | VINT Control          | Configures vertical synchronization and control settings. Initialization?     |
| 0x8*        | Scroll Flip Control   | Configures flipping of all layers and sprites (horizontal/vertical). |

The 0x0e (VINT control) one is a special register that controls the VINT pin going out. This is SUPPOSED to go into the CPU as the VINT interrupt, but I really haven't found any practical function for it in my testing, so I don't think it's being used, at least in the Raizing games. It would normally be used if the V interrupt that goes into the CPU (IPL1) is controlled under the supervision of the game program instead of by the line counters, but no game is really doing that, so I left it disconnected in my core. It seemed to mess up more things than it fixed. They say in the MAME documentation that it is used as an initialization value to tell the program the GP9001 is ready, but I am not sure that's true either.

For the Scroll Flip Control (mask 0x80, 8th bit), this too is kind of a special option to be used in the selecting of the scroll registers when writing to them. If the 8th bit is high, then it means that flip on that particular layer and axis is on, else it is off. But, I never actually used these functions (I tie the FLIPX and FLIPY pins down in the video module). I think it is supposed to be used when you have the dip switch for inverting the monitor video on. I never quite got that to work properly, and so I do it by inverting the line counters instead to flip the video in both ways as it is a less invasive and simple solution. So essentially, if the user selects flip, it outputs the pixels to the screen in a different, inverted, way.

One other interesting thing is about the "extra" text layer, which is the top most layer in the system. The GP9001 actually does NOT interface with that RAM directly nor does it control the rendering of it in any way. The Text RAM is actually written to by the CPU directly. So, if you see text displaying on the screen, that indeed is the work of the 68K CPU, not the GP9001. The GP9001 only gets involved in scroll and object queues/ layers. The layer called "Text" above is for larger text graphics, like the Raizing logo or other bigger bitmap font stuff. It is a regular scroll layer like the other scroll layers.

#### **Large Graphics ROM Banking**
There is a custom chip, labeled AFBK_CT2 on the board that controls the current banking position of the graphics ROMs. Because the ROMs are large, only a certain area is addressable at any given time, and so the banking is also a request from the CPU to the GP9001, but on a different address range, 0x5000C*.

The lower 3 bits of the address are sent to the GP9001 for this request, and that is the bank that is selected (total 8 banks). The custom chip maintains the current bank status of all the ROMs under it's guidance. Additionally, I perform byteswapping of the graphics data requested, and decoding for 32 bits at a time (this is the size of the interface on the real boards). The decoding function for graphics is universal for all games, the following:

```
integer i,m,npix;
function [31:0] decode_gfx;
	input [7:0] a,b,c,d;
	begin
        decode_gfx = 32'b0;
		for(i = 0; i < 4; i= i + 1) begin
			m = 7 - (i << 1);
			npix = ((a >> m) & 1) << 0;
			npix = npix | ((c >> m) & 1) << 1;
			npix = npix | ((b >> m) & 1) << 2;
			npix = npix | ((d >> m) & 1) << 3;
			npix = npix | ((a >> (m - 1)) & 1) << 4;
			npix = npix | ((c >> (m - 1)) & 1) << 5;
			npix = npix | ((b >> (m - 1)) & 1) << 6;
			npix = npix | ((d >> (m - 1)) & 1) << 7;
			decode_gfx = decode_gfx<<8 | npix;
		end
        //data is really little endian
        decode_gfx = {decode_gfx[7:0], decode_gfx[15:8], decode_gfx[23:16], decode_gfx[31:24]};
	end
endfunction
```

So essentially, the parent module AFBK_CT2 is an interface to the graphics data requested for the scroll and object layers independently.

### **Rendering Chain**
Now that we have covered the core functions of the GP9001, let's talk about the rest of the rendering process and chain. The GP9001 gets involved in the setup work prior to the layers actually rendering, and then coordinates the rendering itself. So, there are multiple modules inside the GP9001. One is a high level coordinator, then you have specific logic components for rendering objects, reading from the queue, keeping track of the scroll layer rendering and managing that, doing prioritization, etc. All of these things are highly extensive and complex in the system.

Just talking about priority levels alone, each layer has 16 priority levels within themselves for adjudication. So each layer has to be drawn carefully considering and ranking the priority level of the tiles within it that have to be drawn from the queue. all 3 scroll layers and the object layer have this scheme.

On the real board, things are actually drawn on a per frame basis, as taking a look at the lag between when sprites are drawn and when they are presented to the player is about 1-2 frames delayed (depending on the game). I think that since there is a very large set of DRAMs connected to the GP9001 (TMS45160) at least on the Batrider board, this is used as a staging area for the framebuffer. There are 2 of these, so it is about 512KB. Note that Battle Garegga actually has an additional frame of lag because presumably it could not afford a full 512KB buffer for the whole frame and used a different approach to pre-stage the frame.

As we cannot afford a huge 512KB space in today's technology on the FPGA boards in addition to other memories required for operation, I went with converting the logic to a line buffer approach. This is basically rendering all the scroll layers and object layer for the upcoming scanline in the blanking period of each line. As such, this requires intersection routines to determine what tiles need to be drawn, and determining where the current line of those tiles in ROM exist, pulling them, and drawing just the one line of the tile as opposed to the whole tile at once.

Doing this approach means the FPGA core is in fact faster than the real board from a performance and responsiveness perspective. In fact - it is ahead because the sprites lag noticeably. It means that when the player presses buttons or performs input, those reactions are seen live, as opposed to being presented 1 or 2 frames after they do that thing. Sprites are double buffered in the system, however. To get around this and synchronize the gameplay action and graphics, I artificially buffer the sprites in the vertical blanking period, and work off of past copies of the sprite RAM. This way, I can emulate the frame delays seen on the actual board and synchronize the gameplay.

However, there still is another problem - SDRAM performance. Because all 4 layers are rendered at the same time in the horizontal blanking period for the subsequent line, this causes a huge hit to SDRAM. This, in effect, creates some small delay from a frame by frame perspective, however, the gameplay will indeed be synchronized. In order to resolve this, I plan to run the SDRAM at a higher clock than normally done. I was limited to 96@CL2 on the MiSTer when originally programming this core. I ported it to the pocket and used the same clocks. However, I have done 128Mhz@CL2 on the Pocket for the Midway core, and want to see if that resolves latency here. So, this is my plan next in addition to some cleanup and refactoring on the core.

#### **Scroll Layers**
All 3 scroll layers have the exact same logic in terms of rendering and processing, and all 3 are executed at the exact same time in addition to object rendering.

The scroll layers are made up of 16x16 tiles in a map formation. The layers are panned by using the GP9001's offsetting registers for the x and y. As the layer scrolls, the game program will send updated values to the GP9001 to reflect the position of the tile map so that it can be scrolled and rendered correctly. In addition, each game has base offsets for all scroll layers and object layer that must be taken into account, and added to the current scroll x or y value for that layer.

The general process for rendering the scroll layers is as follows:
- First, the data is scanned and retrieved from the appropriate RAM for the scroll layer. It is done in 2 subsequent cycles, and the total size of the attribute for the tile on that layer is 32 bits.
- For each tile that is gotten, the attributes are unpacked and indexed into priority levels. I mentioned above that it is very important for all layers that they be rendered in order of priority level adjudication with other tiles on the same layer. Therefore, the plan calls for tiles to be rendered by iterating through the tiles for the respective priority levels, one by one on top of eachother.
- The priority level is located in the lower 4 bits of the second upper half of the attributes. The tile ID is the lower 16 bits. The palette ID is the 7 bits following the priority level. All these attributes, as well as the tile x and y position are combined into a 64 bit data segment which is apart of the indexed data queue.
- Once all the data in the RAM is indexed by priority level, the priority levels are iterated through one by one up to 16. The higher the priority level, the higher that tile appears in relation to other tiles on the same layer. So I go through each priority level and each tile within that priority level one by one and draw one on top of the other.
- In this process, I also know whether a priority level contains any tiles at all, so I just skip over them if they contain none.
- Clipping also has to be done after the data is fetched from graphics ROM via the AFBK_CT2 module. Because the coordinate system is conventional, and things are drawn from left to right, there is a possibility of negative values. The rendering system uses signed numbers for all the values. Those pixels that are clipped simply wont be drawn to the line buffer.
- The color comes from the Graphics ROM. The palette ID comes from the tile attributes. The x and y coordinate of the tile comes from a fixed grid position in the 320x240 space and corresponds to a fixed location in the RAM.

So, at the end of this process, the output is to a line buffer for the subsequent scanline that contains a final pixel value for each of the pixels in that line.

#### **Object Layer**
The object layer is similar in nature to rendering the scroll layers, but with a few differences. The total amount of sprites that can possibly appear in the frame is 256. The number of priority levels, as with tiles, is 16.

Objects are multi-buffered (double, with an exception in Battle Garegga which is triple). So this means that the copy of the sprite RAM you use as a basis for rendering is an old copy that could be 1 or 2 frames behind.

In addition to priority levels of a sprite, there is also another feature that is notable called "multiconnected" sprites. This is a feature in the engine where multiple different sprites can be connected together to form one big sprite.

Clipping is somewhat complicated to perform as the game uses a traditional coordinate system. You have to account for sprites that are partially on the screen on any edge and the locations for objects could be negative, so you have to be careful of signed values in the calculations.

The below is the process of rendering the object layer:
- The sprite RAM is scanned to go through all 256 sprite entries. 
- The key attributes are retrieved and index according to the priority level, as in the scroll layers. These attributes include:
  - Whether the sprite is active (a gating item for the queue, else it is skipped over).
  - Whether the sprite is xflipped or y flipped
  - Whether the sprite is a multiconnected sprite
- After retrieving the key items, I determine if that particular sprite has any business being rendered on this scanline considering the properties above. If it does, it gets entered into the queue for the priority level. If not, it is skipped over.
- After the priority queue is established, like the scroll layers, I iterate through each priority level and render the objects in that priority level one after the other. Higher priority level items are rendered on top of the previous rendered items. this is important because there are scenarios where objects can overlap eachother on the screen or with other things.
- When rendering happens, there are additional properties to retrieve. Total there are about 64 bits worth of properties for a sprite to consider. These are:
  - Sprite ID
  - Palette ID
  - Priority Level
  - Flip X/Y
  - Multiconnected Sprite
  - Enabled
  - Width/ Height
  - X/Y Position

As a note for this and also for the scroll layers, only non blank pixels should be rendered. Also, since things are line buffer based with a queue, there is no need to care about clearing of the source queue for the objects or tiles. The line buffers are automatically cleared following rendering to the CRT.

#### **Extra Text Layer**
The extra text layer is fairly simple compared to the other layers. It is a fixed grid of text that floats on top of everything. The ROM is unpacked by the CPU on startup, and the characters in this layer are 8x8. The CPU puts items in the text VRAM for the characters it wants to render on the screen.

The rendering process is basically iterating through the entirety of the RAM and just rendering out characters that are non blank. There are no priority levels, no scrolling or shifting. There is a static offset, however that is applied to the whole layer. Again, this is not to be confused with the TEXT scroll layer that is controlled by the GP9001.

#### **Mixing**
After all 4 layers are rendered, there is a final adjudication process that determines the pixel that actually appears on top for that coordinate. In each of the line buffers for the 4 layers, I include a priority level along with them. The priority level at the pixel level is used to determine in relation to other final pixels on the other layers, which pixels should be on top. This is important for foreground and background mixing effects with sprites.

One example of this is the opening scene in Batrider where there is a bridge in the background, and the ship flies under that bridge, being covered by the bridge, and comes back out again. This is where the background can become a foreground, and the sprite can be wedged between 2 different scroll layers for a time, and come back out again some frames later.

As in the individual layers, I iterate through all 16 priority levels and draw one pixel on top of another. I use a function to perform the mixing:

```
integer i;
function [10:0] pixel_priority_mux;
    input [4:0] pri;
    input [10:0] et;
    input [14:0] obj,scr2,scr1,scr0;
    begin
        pixel_priority_mux = blank_pixel;
        for(i=0;i<16;i=i+1) begin
            if(pri[0] && scr0[14:11] == i[3:0]) pixel_priority_mux = scr0[10:0]; 
            if(pri[1] && scr1[14:11] == i[3:0]) pixel_priority_mux = scr1[10:0];
            if(pri[2] && scr2[14:11] == i[3:0]) pixel_priority_mux = scr2[10:0];
            if(pri[3] && obj[14:11] == i[3:0]) pixel_priority_mux = obj[10:0];
        end
        
        if(pri[4]) pixel_priority_mux = et;
    end
endfunction
```

The text layer always appears on top no matter what, so it is by default the highest priority. It is used for HUD elements and stuff like that.

### **Sound**
For the most part, most of the games use a single Z80 in combination with a YM2151 for FM synthesis and music as well as an MSM6295 OKI for speech and sound effects. The only exception is Battle Bakraid where the OKI was swapped out for a newer YMZ280B chip. It is also ADPCM, just does things differently than the usual MSM6295.

#### **Interface with CPU**
The interface with the CPU is primarily done through "sound latches". These are really just registers that get passed back and forth between the main CPU and the sound CPU. 

The main CPU sends sound commands to the Z80, and then a command is issued to the appropriate devices to play the sound. An acknowledgment is sent back to the CPU to indicate completion of the command. It is important though that the CPU wait for the command acknowledgment from the Z80 because otherwise there could be desynchronization issues between the game program and sound and will result in skipping or artifacts in the audio.

#### **Issues with Sorcer Striker & Kingdom Grandprix**
Originally, I had problems with the speed of the sound on both of these games. After tracing back the relevant parts of the board concerning the Z80, we determined that the issue was with the WAIT generation for these games on the Z80.

For these games, the WAIT should be supplied by the YM2151 DOUT pin 7 inverted.

#### **Sound ROM placement on Battle Bakraid**
The FPGA core was designed originally for a 32MB SDRAM. Battle Bakraid does in fact fit just about exactly within 32MB. However, if we take a look at the SDRAM configuration from a bank perspective, each bank has about 8MB to work with. Since that is the case, and with other data needed in the system like the program, graphics and sound ROMs, I had to split up the data in such a way that everything would fit properly and utilize all the space in every bank efficiently.

So, I had to create a split banking system for ADPCM data that goes into the YMZ280B. The YMZ280B has a larger addressable region, so no programmatic banking is needed in the game like previous entries using the MSM6295. However, I had to create one for the purpose of splitting the data and utilizing all the space.

### **High Score Saving/ NVRAM**
From a board perspective, only Battle Bakraid utilizes an IC that uses serial communications to load and save high score data. None of the other boards have this functionality. However, for the FPGA Core, I built one into Batrider and Battle Garegga.

This was not too complicated to do, it essentially boils down to finding the memory address region that stores the high score table and declaring that an NVRAM region that saves and loads.

## License

The documentation provided on this site is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-nc-sa/4.0/). 

This means you are free to:

- **Share**: Copy and redistribute the material in any medium or format.
- **Adapt**: Remix, transform, and build upon the material.

However, the following terms apply:

- **Attribution**: You must give appropriate credit to "Coin-Op Collection," provide a link to the license, and indicate if changes were made. You may do so in any reasonable manner, but not in any way that suggests the licensor endorses you or your use.
- **Non-Commercial**: You may not use the material for commercial purposes.
- **ShareAlike**: If you remix, transform, or build upon the material, you must distribute your contributions under the same license as the original.

For any exceptions or further inquiries, please contact Coin-Op Collection directly.