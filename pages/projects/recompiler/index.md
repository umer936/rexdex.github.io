---
layout: project_page
title: Xbox360 to Win64 recompiler
permalink: /projects/recompiler/
project_root: recompiler
project_logo: /images/projects/xbox_remcompiler.jpg
project_index: true
project_info: "What if you could take an Xbox360 executable and run it on your PC? I made this project as a Proof of Concept that this can indeed be done. It's far from being anything practical, and honestly, it's not marketable in any way ATM but it's still a lot of fun"
---

## Story of porting Xbox360 executables to Windows

The idea is simple: *what if you could take an Xbox360 game and run it on your PC?* Is this even possible in principle? I was pondering this question a few years ago and it should not come as a surprise that there are some obvious technical difficulties in getting this done:

- **Different CPUs** - Xbox360 uses a PowerPC based CPU, our PCs are based on x86 architecture. They are different in so many ways that I don't even know where to start :) PowerPC is RISC based, has shitloads of registers but very simple instructions. x86 is totally different on the other hand - not so many registers and many more instructions that are more complicated (addressing modes...). It's obvious that a simple transcription is not feasible.

- **Memory Layout** - Xbox360 uses BigEndian byte ordering, x86 CPUs use LittleEndian. To be compatible with incoming data that is being read from files and read/written into the memory all memory based operands must be byteswapped. This may pose a significant performance issue.

- **Encrypted executable image** - Yup. For various reasons, the executables for Xbox360 are encrypted. There are some clever guys in Russia though that figured that out :)

- **Different and outdated GPU architecture** - If we want to see any graphics rendered, the GPU needs to be emulated. There are two hard nuts to crack: first, the shaders we see will be complied into the GPU compatible format. No HLSL on input, sorry. Those shaders will have to be reverse engineered as well. Secondly, the Xbox360 GPU was using ~10MB of internal memory called EDRAM that was serving as a temporary storage of render target for the duration of rendering. Although some cards today still use a similar concept, this is never exposed directly to the user. Since there are a lot of different ways people used the EDRAM on Xbox, this part has to be emulated. To be honest, it's probably implemented differently for every game.

- **Inlining of graphics/kernel functions** - Some of the functions used while compiling the executable were inlined directly into the compiled code, making it much harder to write a simple API level wrapper. This kills the dream of making a "function level" wrapper where we could just go and wrap the "d3d->DrawPrimitive" call directly. Nope, this is not going to happen.

Fortunately, every problem is solvable and the answer is ***YES*** in principle. If you want to know how, keep reading :)

## Current state of the project

Currently, the published branch of the project allows to run simple Xbox360 demo apps (samples). I've not yet  to run it with any real game as it probably would not work with anything big and serious. Also, on the legal side, this is a fine line because getting anything bigger is tricky as it requires basically going to torrent sites and digging through old Xbox Live Arcade content or pirated games. The Xbox360 is not yet abandonware :) For the same reason, there are no source executables given, you'd need to get one "from somewhere". Sorry :(

Stuff currently implemented:

***Backend*** (offline processing):

+ XEX image loading, decryption, and decompression
+ PowerPC instruction disassembly 
+ Program blocks reconstruction
+ Generation C++ equivalent code for whole executable ("recompilation")
+ 96% of PowerPC CPU instructions implemented
 
***Runtime*** (host):

+ Loading and running recompiled image (as DLL)
+ Basic kernel with threads, synchronization
+ Basic IO with file support
+ GPU command queue bootstrap
+ Tracing functionality (offline debugging)

![DolphinDemoScreenshot](/images/projects/xbox_remcompiler.jpg)
 
***GPU***:

+ AMD Microcode shader disassembly and recompilation into DX11 HLSL shaders
+ Command buffer parser and executor
+ Very simple EDRAM simulator
+ Trace dump functionality
 
***Debugging tools***

+ Basic IDE that allows to view the disassembly
+ Basic offline (trace based) debugger that allows to inspect every executed instruction
+ Basic GPU trace viewer that allows to inspect internal GPU state at each point
+ *Time Machine* tool that makes it possible to find previous instruction that touched given register or memory

{% include subpages.html %}
More pages (Debugging, GPU emulation) are coming.

## How do I run it?

- You will need Visual Studio 2015 (sorry, Windows only)
- Get the wxWidgets in 3.1.0 and compile the x64 DLL libs, place them in dev\external\wxWidgets-3.1.0\
- Compile the whole solution from dev\src\recompile.sln 
- Run the "framework\frontend" project
- Open the project "projects\xenon\doplhin\dolphin.px"
- Select the "Final" configuration
- Click the "Build" button
- Assuming you've installed the project in C:\recompiler, run the "launcher\frontend" project with following parameters: "-platform=Recompiler.Xenon.Launcher.dll -image=C:\recompiler\projects\xenon\doplhin\Dolphin.px.Final.VS2015.dll -dvd=C:\recompiler\projects\xenon\doplhin\data -devkit=C:\recompiler\projects\xenon\doplhin\data"
- To exit the app, close the GPU output window

## References

- [XEX informations](http://www.openrce.org/forums/posts/111)
- [PowerPC ISA](http://fileadmin.cs.lth.se/cs/education/EDAN25/PowerISA_V2.07_PUBLIC.pdf)
- [Free60 description of the XEX](http://www.free60.org/wiki/XEX)
- [Sourcecode of the Free60 project](https://github.com/Free60Project)
- [Radeon R600 ISA](http://developer.amd.com/wordpress/media/2012/10/R600_Instruction_Set_Architecture.pdf)
- [Radeon microcode decompiler](https://github.com/freedreno/freedreno/blob/master/includes/instr-a2xx.h)
- [Radeon R6xx/R7xx Acceleration](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/10/R6xx_R7xx_3D.pdf)
- [Radeon Linux Driver](http://cgit.freedesktop.org/xorg/driver/xf86-video-radeonhd/tree/src/r6xx_accel.c?id=3f8b6eccd9dba116cc4801e7f80ce21a879c67d2#n454)
- [Radeon Driver parts](https://github.com/freedreno/amd-gpu/blob/master/include/reg/yamato/22/yamato_offset.h])
- [Actual open source Radeon Driver](https://github.com/freedreno/amd-gpu/blob/master/)
- [Mesa source code](http://fossies.org/dox/MesaLib-10.3.5/fd2__gmem_8c_source.html])
- [Radeon Evergreen 3D Register Reference Guide](http://www.x.org/docs/AMD/old/evergreen_3D_registers_v2.pdf)
- [Radeon R200 OpenGL driver](https://github.com/freedreno/mesa/blob/master/src/mesa/drivers/dri/r200/r200_state.c)
- [Radeon R600 driver parts](http://ftp.tku.edu.tw/NetBSD/NetBSD-current/xsrc/external/mit/xf86-video-ati/dist/src/r600_reg_auto_r6xx.h)
- [Legacy rendering formats](http://msdn.microsoft.com/en-us/library/windows/desktop/cc308051(v=vs.85).aspx)
- [Crunch texture decompression](https://code.google.com/p/crunch/source/browse/trunk/inc/crn_decomp.h#4104)
- [CRC32 calculation](http://create.stephan-brumme.com/disclaimer.html])
- [CRC calculation](http://www.intel.com/technology/comms/perfnet/download/CRC_generators.pdf)
- [Slicing-by-8 CRC algorithm](http://sourceforge.net/projects/slicing-by-8/)
- [Kernel objects on Windows Vista](http://www.nirsoft.net/kernel_struct/vista/KOBJECTS.html)
- [More Windows kernel stuff](http://www.nirsoft.net/kernel_struct/vista/SLIST_HEADER.html)
- [APC on Windows](http://www.drdobbs.com/inside-nts-asynchronous-procedure-call/184416590?pgno=1)
- [RtlFillMemoryUlong reference](http://msdn.microsoft.com/en-us/library/ff552263)
- [Dashboard for Xbox360](http://code.google.com/p/vdash/source/browse/trunk/vdash/include/kernel.h)
- [Tiled rendering patent](https://www.google.com/patents/US20060055701)
- [Deflate compression](http://tools.ietf.org/html/rfc1951)

## Other notable Xbox360 related projects

- [Xenia - Very nice Xbox Emulator, runs some games](https://github.com/benvanik/xenia)

## Future work

- Getting a simple game to work :)
