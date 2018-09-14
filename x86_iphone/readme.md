# x86 find its way into your iPhone

### Introduction
In one of my several lifes, I'm supposed to be a vulnerability researcher working on baseband exploitation. As every vulnerability researcher knows, being up to date with recent developments is of utmost importance for the success of your job. So of course, after Apple announced its new, shiny, big, bigger and biggest line of iPhone smartphones, I downloaded some OTA firmwares from ipsw.me and started to look into the new baseband firmware.

What I discovered sent a shiver of horror down my spine, the kind of horror that only playing Doom at nighttime, alone in your room, without lights, can produce. Stay with me and I'll tell you what I found...

*NOTE*: this is just some preliminary analysis and I'm not going to describe anything related to baseband rerversing, it's not the scope of this document.

### Who/What/Where?
The baseband cpu is a standalone core that lives in your phone and is responsible for managing 2g/3g/4g/cdma/5g wireless communications. Given the absurd complexity of these standards, today a baseband cpu must be very powerful and enough general purpose, so the days of custom FPGA based IPs are long gone, at least for the main part. A lot has been said and written about basebands on modern smartphones, so I won't repeat it. For our purpose, you just need to know that usually basebands are implemented using embedded friendly CPUs, like for example ARM (Cortex-M, Cortex-R or something inbetween), Qualcomm Hexagon (a kind of general purpose, VLIW dsp) or other more or less known architectures.

Apple is nothing special in this regard, up until the iPhone8/iPhoneX, they used to have two different basebands, one for CDMA markets and one for everything else. The CDMA one was based on Qualcomm Hexagon dsp, while the GSM one was based on Intel XMMxxxx architecture. For those that like to play around with iPhone firmwares, you might have seen MAVxxx and ICExxx files in the ipsw, well those two files contain the firmware respectively for Qualcomm based devices (MAV) and Intel based ones (ICE).

As you may know, Apple decided to drop Qualcomm and now they're using exclusively Intel based basebands, so we will concentrate on this.

### The old Intel baseband
Let's take a look at the old generation of Intel baseband, the one used up to iPhoneX. If you download an ipsw GSM firmware and unzip it (ipsw files are just zip files), there will be, amongst other things, a "Firmware" folder. Inside this folder you will see a couple of files: "ICE17-1.04.52.Release.bbfw" and "Mav17-1.91.00.Release.bbfw". The version will change of course, these are for iPhone8 with iOS 11beta3. Those files are again zip files and we're interested in ICE17-xxx, let's unzip it and see the contents:

```console
John-MacBook-Air:ICE user$ ls -al
total 76800
drwxr-xr-x  16 user  staff       544 Sep 14 11:08 .
drwxr-xr-x@ 15 user  staff       510 Sep 14 11:08 ..
-rw-r--r--@  1 user  staff      9098 Apr 20 03:29 2GFW.elf
-rw-r--r--@  1 user  staff    753552 Apr 20 03:29 3GFW.elf
-rw-r--r--@  1 user  staff     13358 Apr 20 03:29 AudioFW.elf
-rw-r--r--@  1 user  staff       274 Apr 20 03:29 Debug_info.elf
-rw-r--r--@  1 user  staff      1031 Apr 20 03:29 Info.plist
-rw-r--r--@  1 user  staff   5146700 Apr 20 03:29 LTEFW.elf
-rw-r--r--@  1 user  staff         0 Oct 30  2017 Options.plist
-rw-r--r--@  1 user  staff    423592 Apr 20 03:29 RFFW.elf
-rw-r--r--@  1 user  staff  30633772 Apr 20 03:29 SYS_SW.elf
-rw-r--r--@  1 user  staff   1834194 Apr 20 03:29 TDSFW.elf
-rw-r--r--@  1 user  staff     83170 Apr 22 12:26 bbcfg.bin
-rw-r--r--@  1 user  staff     73496 Apr 20 03:28 ebl.bin
-rw-r--r--@  1 user  staff    163836 Apr 20 03:28 psi_ram.bin
-rw-r--r--@  1 user  staff    163836 Apr 20 03:29 restorepsi.bin
```

lot of stuff here. People familiar with embedded firmware reversing, will set their eyes on "psi_ram.bin" and "ebl.bin" of course. A quick hex dump of their content, will immediately reveal that they contain ARM code. As a matter of fact, "psi_ram.bin" is the stage0 bootloader, that will load "ebl.bin" at some address and jump to its entry point. ebl.bin will then load all other files and jump to the main firmware, contained in SYS_SW.elf. Just to be sure, let us fire IDA and open psi_ram.bin for example, load address does not matter, just set it to 0:

```
ROM:00000000                             CODE32
ROM:00000000 6C 04 00 EA                 B               loc_11B8
ROM:00000000             ; ---------------------------------------------------------------------------
ROM:00000004 6C 69 48 55                 DCD 0x5548696C
ROM:00000008 01 51 FE FF                 DCD 0xFFFE5101
[...]
ROM:000011B8 10 40 2D E9                 STMFD           SP!, {R4,LR}
ROM:000011BC 1D 0D 00 FB                 BLX             sub_463A
ROM:000011C0 08 41 9F E5                 LDR             R4, =0x80400
ROM:000011C4 00 00 A0 E3                 MOV             R0, #0
ROM:000011C8 04 00 84 E5                 STR             R0, [R4,#4]
ROM:000011CC 05 3A 00 FA                 BLX             sub_F9E8
ROM:000011D0 01 00 50 E3                 CMP             R0, #1
ROM:000011D4 37 00 00 0A                 BEQ             loc_12B8
ROM:000011D8 F4 10 9F E5                 LDR             R1, =0x230000
ROM:000011DC 08 00 84 E2                 ADD             R0, R4, #8
ROM:000011E0 00 30 A0 E3                 MOV             R3, #0
ROM:000011E4 00 21 04 E3                 MOV             R2, #0x4100
ROM:000011E8 27 3B 00 FA                 BLX             sub_FE8C
ROM:000011EC
ROM:000011EC             loc_11EC                                ; CODE XREF: ROM:000012CC↓j
ROM:000011EC 00 20 A0 E3                 MOV             R2, #0
ROM:000011F0 8E 0F 84 E2                 ADD             R0, R4, #0x238
ROM:000011F4 02 10 A0 E1                 MOV             R1, R2
ROM:000011F8 DB 3B 00 FA                 BLX             sub_1016C
ROM:000011FC D4 10 8F E2                 ADR             R1, aBootrom ; "bootrom"
ROM:00001200 00 00 A0 E3                 MOV             R0, #0
ROM:00001204 F3 3B 00 FA                 BLX             sub_101D8
ROM:00001208 01 00 A0 E3                 MOV             R0, #1
ROM:0000120C E4 3B 00 FA                 BLX             sub_101A4
ROM:00001210 C8 00 8F E2                 ADR             R0, aPsiPsiStartup ; "psi: PSI Startup"
```

nothing strange here, it's your standard embedded ARM boot code, with some custom setup related to this specific SoC. Moving on, psi_ram will load ebl and jump into it, you can find the code yourself, is not that complex.
The important thing is, this is ARM code.

### The new Intel baseband...the horror...the horror...
Waking up this morning, I wanted to analyse the new baseband firmware, to get an idea of what changed and where. So I downloaded a random OTA for the new iPhones from ipsw.me, in particular: https://ipsw.me/otas/iPhone11,2 , which should be the new iPhone Xs. I went through the usual stages (downloading, unzipping, unzipping, unzipping, copying, etc...), I load psi_ram.bin into IDA, I select ARM little endian, and...huge chunk of undefined data. I press "C" like an idiot for about 200 times, I reload the file multiple times, I try another IDA version, I try to disassemble the bytes with Capstone, I DISASSEMBLE THEM MANUALLY, nothing...all I got was invalid code. So I went back to sleep because obviously it was not a good day.

I wake up again after about an hour, I download again the ipsw for the same model and I go through all the same steps as usual. Still invalid code. My first guess was that the code is encrypted, so I use some random tool to analyse the entropy, nothing, it's very low. Ok, you cannot trust tools these days, so I wrote my own script to calculate the entropy and again, is too low to be encrypted and anyway, it was also clear by a quick "eye" analysis, even though you should not trust your eyes that much.

I stare at the screen for several minutes, thinking what the fu** is going on. But of course, as a reverser you should never give up, so I try EVERY POSSIBLE RISC ARCHITECTURE I KNOW, and trust me, they're a lot (no I mean, A LOT, I reversed everything from Dreamcast SH4 to Fujitsu/Siemens running inside my Nikon D90), and...

Nothing, all I get is garbage. My next guess is: ok then, it's encrypted with some entropy-preserving algorithm. I try some cryptoanalisis approaches I know, even though is not my main job and I'm no expert at it, still nothing.

I make another coffee and I start to think, in my mind, as a joke: fuck it, this is x86 because Intel has gone insane. I start to laugh maniacally at the idea, but you know, I had tried everything, so I reopen "psi_ram.bin" and I leave the default "MetaPC" architecture in IDA, and...

```
seg000:0000143A sub_143A        proc near               ; CODE XREF: sub_12EC+6F↑p
seg000:0000143A                                         ; sub_160B+182↓p
seg000:0000143A
seg000:0000143A var_4           = dword ptr -4
seg000:0000143A
seg000:0000143A                 push    esi
seg000:0000143B                 mov     [esp+4+var_4], eax
seg000:0000143E                 lea     edx, [esp+4+var_4]
seg000:00001441                 push    44h ; 'D'
seg000:00001443                 push    edx
seg000:00001444                 push    25h ; '%'
seg000:00001446                 call    sub_2644
seg000:0000144B                 add     esp, 0Ch
seg000:0000144E                 pop     ecx
seg000:0000144F                 retn
seg000:0000144F sub_143A        endp
```

I could not believe my eyes...I was looking at embedded x86 code running inside a baseband processor. I thought that it was just random chance, that somehow I ended up with valid x86 code, so I look at another function (note: IDA automatically resolved most of the code) and....
```
seg000:0000DFD0 sub_DFD0        proc near               ; CODE XREF: sub_DF4D+7A↑p
seg000:0000DFD0
seg000:0000DFD0 arg_0           = dword ptr  4
seg000:0000DFD0
seg000:0000DFD0                 mov     eax, [esp+arg_0]
seg000:0000DFD4                 lgdt    fword ptr [eax]
seg000:0000DFD7                 mov     ax, 10h
seg000:0000DFDB                 mov     ds, eax
seg000:0000DFDD                 assume ds:nothing
seg000:0000DFDD                 mov     es, eax
seg000:0000DFDF                 assume es:nothing
seg000:0000DFDF                 mov     ss, eax
seg000:0000DFE1                 assume ss:nothing
seg000:0000DFE1                 mov     ax, 18h
seg000:0000DFE5                 mov     gs, eax
seg000:0000DFE7                 assume gs:nothing
seg000:0000DFE7                 jmp     far ptr 8:0FF90DFEEh
seg000:0000DFE7 sub_DFD0        endp
```

holy sh*t...lgdt...mov ds, eax...jmp far ptr 8:xxx, you can't get more x86 than this. This is no coincidence, this MUST be valid x86 code...I was totally speechless, and partially in awe, because this reminds me of my younger days when I was writing bootloaders and operating systems that run in protected mode.

But my rational part told me "hey, probably they just decided to boot the system with an embedded x86 mcu, OBVIOUSLY the main firmware still runs on ARM", so I turn my attention to "SYS_SW.elf", the main firmware. I look at the ELF header, and it says ARM...good. I load it in IDA, and I get garbage...AGAIN!!!! I think "what.the.fuck" and I try to load it with MetaPC again and...BOOM, valid x86 code! Random snippet:

```
seg000:010029E4
seg000:010029E4
seg000:010029E4 sub_10029E4     proc near               ; CODE XREF: seg000:0001016C↑p
seg000:010029E4                                         ; seg000:00010175↑p ...
seg000:010029E4                 push    esi
seg000:010029E5                 push    edi
seg000:010029E6                 push    ebx
seg000:010029E7                 push    ebp
seg000:010029E8                 mov     edi, [esp+14h]
seg000:010029EC                 cmp     edi, 8Ah
seg000:010029F2                 lea     eax, ds:218DB44h[edi*8]
seg000:010029F9                 mov     esi, [eax]
seg000:010029FB                 mov     ebp, [eax-4]
seg000:010029FE                 jb      short loc_1002A05
seg000:01002A00                 push    0FFFFFFFFh
seg000:01002A02                 pop     eax
seg000:01002A03                 jmp     short loc_1002A68
```

this is awesome...

### Conclusions
Nothing really, I just found this funny and wanted to share.
If you ever felt like you missed an x86 core in your life, now you have one (or several? I still need to investigate) always with you, so you won't feel alone ever again.
I still did no thave time to analyse the other models, but my guess is that they're also running on x86.

What will happen next? Are we going to get a z80 inside our phones? Why not a 6502? Hell, I would love to run C64 basic interpreter natively inside my phone....

No really, I'm not criticizing Apple or Intel for using an x86 core, I don't really care, but...my PERSONAL opinion is that something is going really wrong in this world.