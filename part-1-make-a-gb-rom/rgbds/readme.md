The *Game Boy* and *Game Boy Color* were very cool handhelds with lots of nifty games. Maybe at one point you've wondered "hmm, what was programming for the Game Boy like?" or you've wanted to make your own. Assembly is confusing and messy, but it's entirely possible to learn.

I plan to write a bunch of tutorials from the ground up, explaining how to use the Game Boy hardware to do different techniques. Before that, I should explain the bare minimum to get a ROM that "works", even if it does nothing yet.

If you don't understand everything here, don't worry, you don't have to right now. A lot of this can be copy-pasted verbatim and changed later, but I've taken time to break it down and explain different pieces.

This tutorial uses Z80 assembly and an assembler named [RGBDS](http://anthony.bentley.name/rgbds/), which is included with the download link at the bottom. 

### **Layout**

First thing to do is to tell the assembler where to put the code. Make a text file, name it **game.z80** (or game.txt or whatever you want really), then open it and put the following:

```assembler
SECTION "rom", HOME
```

&#9670; This defines a section named *rom* located in the **HOME** bank (ROM bank 0).

&#9670; RGBDS sections are kind of ugly, but you can read a bit about them here: http://otakunozoku.com/RGBDSdocs/asm/section.htm

### **Boot Header**

Now that the assembler knows enough information, there is also the matter of giving some information to the Game Boy. The Game Boy will only run cartridges that pass the boot verification, and everything else will lock on the startup screen. To pass the boot procedure, there needs to be a **Boot Header** that adheres to a Game Boy format. The header also has enough information about the hardware the catridge uses, which can be used by emulators.

You can skim this and change the headers later.

The first section in the header is pretty boring, but can be used to store really small subroutines aligned to 8-byte boundaries, called **RST Handlers**. (There's a series of optimized instructions called *rst* (short for restart) which take less CPU cycles and use less bytes in the ROM to call.)

```assembler
; $0000 - $003F: RST handlers.
ret
REPT 7
    nop
ENDR
; $0008
ret
REPT 7
    nop
ENDR
; $0010
ret
REPT 7
    nop
ENDR
; $0018
ret
REPT 7
    nop
ENDR
; $0020
ret
REPT 7
    nop
ENDR
; $0028
ret
REPT 7
    nop
ENDR
; $0030
ret
REPT 7
    nop
ENDR
; $0038
ret
REPT 7
    nop
ENDR
```

RGBDS requires you to pad areas manually, it has no "org" directive to skip ahead in the ROM, and you can't use the same SECTION with different addresses twice. So you need use the **REPT** (repeat) macro or **DS** (define storage) directive to pad bytes manually.

The next section is a series of **Interrupt Handlers**, for responding to different events that can be triggered by the Game Boy hardware. These are also aligned to 8-byte boundaries.

```assembler
; $0040 - $0067: Interrupt handlers.
jp draw
REPT 5
    nop
ENDR
; $0048
jp stat
REPT 5
    nop
ENDR
; $0050
jp timer
REPT 5
    nop
ENDR
; $0058
jp serial
REPT 5
    nop
ENDR
; $0060
jp joypad
REPT 5
    nop
ENDR
```

There are few different interrupts:

&#9670; *draw* is a **Vertical Blank** ("v-blank") handler. It happens once every frame when the LCD screen has drawn the final scanline. During this short time it's safe to mess with the video hardware and there won't be interference.

&#9670; *stat* is a **Status Interrupt** handler. It is fired when certain conditions are met in the LCD Status register. Writing to the LCD Status Register, it's possible to configure which events trigger the Status interrupt. One use is to trigger **Horizontal Blank** ("h-blank") interrupts, which occur when there's a very small window of time between scanlines of the screen, to make a really tiny change to the video memory.

&#9670; *timer* is a **Timer Interrupt** handler. It is fired when the Game Boy's 8-bit timer wraps around from `255` to `0`. The timer's update frequency is customizable.

&#9670; *serial* is a **Serial Interrupt** handler. It is triggered when the a serial link cable transfer has completed sending/receiving a byte of data.

&#9670; *joypad* is a **Joypad Interrupt** handler. Its primary purpose is to break the Game Boy from its low-power standby state, and isn't terribly useful for much else.

&#9670; The Interrupt Flag register can be used to determine which interrupt has happened.

&#9670; The Interrupt Enable register configures what events will triggered interrupts.





Then there is a small area with some free space which you can use for whatever you want, which is 152 bytes long. We ignore this for now.

```assembler
; $0068 - $00FF: Free space.
DS $98
```

Following that, there is an area that is 4 bytes in length that can hold code for the **Startup Routine**, which is executed directly after the cartridge passes the boot check.

```assembler
; $0100 - $0103: Startup handler.
nop
jp main
```

Since this is tiny, pretty much all that fits is an instruction to jump somewhere else in the ROM.

Next, there is some binary data that defines the **Nintendo logo**. This data on the cartridge must match the exact logo in Game Boy's internal boot ROM, or the Game Boy boot procedure will lock up after the logo is displayed.

```assembler
; $0104 - $0133: The Nintendo Logo.
DB $CE, $ED, $66, $66, $CC, $0D, $00, $0B
DB $03, $73, $00, $83, $00, $0C, $00, $0D
DB $00, $08, $11, $1F, $88, $89, $00, $0E
DB $DC, $CC, $6E, $E6, $DD, $DD, $D9, $99
DB $BB, $BB, $67, $63, $6E, $0E, $EC, $CC
DB $DD, $DC, $99, $9F, $BB, $B9, $33, $3E
```

Then there's the **Title** for the ROM, which is 11 characters in all caps. Any unusued bytes should be filled with 0, which we do with the DS directive.

```assembler
; $0134 - $013E: The title, in upper-case letters, followed by zeroes.
DB "TEST"
DS 7 ; padding
```

The **Manufacturer Code** is an uppercase 4 letter-string, but on older titles it's used for more letters in the game title. Lots of emulators will display this as part of the title, which is undesirable, so here it's just filled with `0` values.

```assembler
; $013F - $0142: The manufacturer code.
DS 4
```

The **GBC Compatibility Flag** decides whether the ROM is compatible with both the Game Boy Color and the original Game Boy, or if it was a GBC-exclusive. Monochrome games use it as another uppercase letter in the title. Saying the ROM is exclusive doesn't actually prevent the ROM from starting if it is run on an original Game Boy, but both compatible/exclusive settings have the effect of enabling extra features on the Game Boy Color.

```assembler
; $0143: Gameboy Color compatibility flag.    
GBC_UNSUPPORTED EQU $00
GBC_COMPATIBLE EQU $80
GBC_EXCLUSIVE EQU $C0
DB GBC_UNSUPPORTED
```

The **Licensee Code** is a two-character name in upper case. Which was used to indicate who developed/published the cartridge.

```assembler
; $0144 - $0145: "New" Licensee Code, a two character name.
DB "OK"
```

The **Super Game Boy** compatibility flag indicates whether this ROM has extra features on the Super Game Boy. If set, then a ROM can utilize the Super Game Boy transfer transfer protocol to add things like border images, custom palettes, screen recoloring, sprites, sound effects, and sometimes even data that can be sent to SNES RAM and run on the SNES (both 65816 and SPC programs).

```assembler
; $0146: Super Gameboy compatibility flag.
SGB_UNSUPPORTED EQU $00
SGB_SUPPORTED EQU $03
DB SGB_UNSUPPORTED
```

The **Cartridge Type** determines what Memory Bank Controller the cartridge has, as well other additional hardware that the Game Boy can use: rumble, extra RAM, battery-backed saves, a real-time clock, etc. Some carts are pretty unique, like Tamagotchi and Game Boy Camera.

```assembler
; $0147: Cartridge type. Either no ROM or MBC5 is recommended.
CART_ROM_ONLY EQU $00
CART_MBC1 EQU $01
CART_MBC1_RAM EQU $02
CART_MBC1_RAM_BATTERY EQU $03
CART_MBC2 EQU $05
CART_MBC2_BATTERY EQU $06
CART_ROM_RAM EQU $08
CART_ROM_RAM_BATTERY EQU $09
CART_MMM01 EQU $0B
CART_MMM01_RAM EQU $0C
CART_MMM01_RAM_BATTERY EQU $0D
CART_MBC3_TIMER_BATTERY EQU $0F
CART_MBC3_TIMER_RAM_BATTERY EQU $10
CART_MBC3 EQU $11
CART_MBC3_RAM EQU $12
CART_MBC3_RAM_BATTERY EQU $13
CART_MBC4 EQU $15
CART_MBC4_RAM EQU $16
CART_MBC4_RAM_BATTERY EQU $17
CART_MBC5 EQU $19
CART_MBC5_RAM EQU $1A
CART_MBC5_RAM_BATTERY EQU $1B
CART_MBC5_RUMBLE EQU $1C
CART_MBC5_RUMBLE_RAM EQU $1D
CART_MBC5_RUMBLE_RAM_BATTERY EQU $1E
CART_POCKET_CAMERA EQU $FC
CART_BANDAI_TAMA5 EQU $FD
CART_HUC3 EQU $FE
CART_HUC1_RAM_BATTERY EQU $FF
DB CART_ROM_ONLY
```

The **ROM Size** determines how much read-only memory is available on the cartridge. The simplest has two 16 KB banks, and each setting is some multiple of 32 KB.

```assembler
; $0148: Rom size.
ROM_32K EQU $00
ROM_64K EQU $01
ROM_128K EQU $02
ROM_256K EQU $03
ROM_512K EQU $04
ROM_1024K EQU $05
ROM_2048K EQU $06
ROM_4096K EQU $07
ROM_1152K EQU $52
ROM_1280K EQU $53
ROM_1536K EQU $54
DB ROM_32K
```

The **RAM Size** indicates how much random-access memory is on the cartridge. This is additional RAM on top of the RAM that GB / GBC provides, and it might be battery-backed to persist data as saves and so on. Apparently for the MBC2, you need to say it has "no RAM" even though the MBC2 actually does have its own RAM memory.

```assembler
; $0149: Ram size.
RAM_NONE EQU $00
RAM_2K EQU $01
RAM_8K EQU $02
RAM_32K EQU $03
DB RAM_NONE
```

Then finally some other stuff, that doesn't really need to be explained too much. The data located for the **Header Checksum** and **Global Checksum** is calculated by an external patching tool, because it's too tedious to calculate by hand. The Header Checksum needs to be correct or the cartridge won't pass the boot check. Running **rgbfix** will patch your ROM for you.

```assembler
; $014A: Destination code.
DEST_JAPAN EQU $00
DEST_INTERNATIONAL EQU $01
DB DEST_INTERNATIONAL
; $014B: Old licensee code.
; $33 indicates new license code will be used.
; $33 must be used for SGB games.
DB $33
; $014C: ROM version number
DB $00
; $014D: Header checksum.
; Assembler needs to patch this.
DB $FF
; $014E- $014F: Global checksum.
; Assembler needs to patch this.
DW $FACE
```

Whew, and we're done!

For more information, there are some resources:

&#9670; The Cartridge Header: http://nocash.emubase.de/pandocs.htm#thecartridgeheader

&#9670; Reverse engineering the Game Boy's internal boot code: http://gbdev.gg8.se/wiki/articles/Gameboy_Bootstrap_ROM

### **Loose Ends**

After the boot header, there needs to a little bit more going on, so the program can actually compile.

```assembler
; $0150: Code!
main:
.loop:
    halt
    jr .loop

draw:
stat:
timer:
serial:
joypad:
    reti
```

This creates a *main* function that just loops forever, sleeping periodically to use minimal battery life. This will also use placeholder interrupt handlers that do nothing, for now.

### **Putting it All Together**

Finally, you need to run a few different RGBDS tools to build the project: 

```assembler
rgbasm -ogame.obj game.z80
rgblink -mgame.map -ngame.sym -ogame.gb game.obj
rgbfix -p0 -v game.gb
```

&#9670; "game.z80" is the name of your input file

&#9670; "game.map", "game.sym" and "game.obj" are build artifacts.

&#9670; "game.sym" can be used for debugging in BGB.

&#9670; "game.gb" is the name you want the resulting ROM file to have.


It should spit out text similar to this:

```assembler
Output filename game.obj
Assembling game.z80
Pass 1...
Pass 2...
Success! 244 lines in 0.00 seconds (4879999 lines/minute)
```

And there you go! If all went well, you should see **game.gb**, your very own Game Boy ROM. It does nothing useful yet, but it's now an empty template which boots up and can be used to make a game.

&#9670; You can [download the code and assembler here](http://make.vg/get/assemblydigest/part-1-make-a-gb-rom.zip).

&#9670; You can also [view the code and this tutorial on Github](https://github.com/assemblydigest/gameboy/tree/master/part-1-make-a-gb-rom/rgbds)!

&#9670; This download does not include a Game Boy emulator, but there are plenty available online.

&#9670; I really recommend [BGB](http://bgb.bircd.org/) if you're on Windows, as it is probably the most accurate GB emulator and it contains many developer-friendly features.

&#9670; You can also buy cartrdiges like the [Drag N Derp](http://derpcart.com/) to test on real hardware.

### **Next Steps**

In another part, there will be a tutorial to make a ROM that actually does more than just pass the boot check. Things like:

&#9670; How to make graphics for the Game Boy

&#9670; Tiles and the Background Layer

&#9670; The Window Layer

&#9670; Sprites

&#9670; Scrolling

&#9670; Game Boy Color hardware

&#9670; Scanline Effects

&#9670; Math

&#9670; Algorithms

&#9670; Bank Switching

&#9670; Neat Programming Tricks


Expect a new tutorial soon!
