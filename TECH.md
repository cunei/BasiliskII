# Basilisk II Technical Manual

## Table of Contents

1. Introduction
1. Modes of operation
1. Memory access
1. Calling native routines from 68k mode and vice-versa
1. Interrupts
1. Parts of Basilisk II
1. Porting Basilisk II

## 1. Introduction

Basilisk II can emulate two kind of Macs, depending on the ROM being used:

1. A Mac Classic
1. A Mac II series computer ("Mac II series" here means all 68020/30/40 based Macs with 32-bit clean ROMs (this excludes the original Mac II, the IIx/IIcx and the SE/30), except PowerBooks; in the following, "Mac II" is used as an abbreviation of "Mac II series computer", as defined above)

More precisely spoken, MacOS under Basilisk II behaves like on a Mac Classic or Mac II because, apart from the CPU, the RAM and the ROM, absolutely no Mac hardware is emulated. Rather, Basilisk II provides replacements (usually in the form of MacOS drivers) for the parts of MacOS that access hardware. As there are practically no Mac applications that access hardware directly (this is also due to the fact that the hardware of different Mac models is sometimes as different as, say, the hardware of an Atari ST and an Amiga 500), both the compatibility and speed of this approach are very high.

## 2. Modes of operation

Basilisk II is designed to run on many different hardware platforms and on many different operating systems. To provide optimal performance under all environments, it can run in four different modes, depending on the features of the underlying environment (the modes are selected with the REAL\_ADDRESSING, DIRECT\_ADDRESSING and EMULATED_68K defines in "sysdeps.h"):

1. Emulated CPU, "virtual" addressing (`EMULATED_68K = 1`, `REAL_ADDRESSING = 0`):  
This mode is designed for non-68k or little-endian systems or systems that don't allow accessing RAM at `0x0000..0x1fff`. This is also the only mode that allows 24-bit addressing, and thus the only mode that allows Mac Classic emulation. The 68k processor is emulated with the UAE CPU engine and two memory areas are allocated for Mac RAM and ROM. The memory map seen by the emulated CPU and the host CPU are different. Mac RAM starts at address 0 for the emulated 68k, but it may start at a different address for the host CPU.  
In order to handle the particularities of each memory area (RAM, ROM and Frame Buffer), the address space of the emulated 68k is broken down into banks. Each bank is associated with a series of pointers to specific memory access functions that carry out the necessary operations (e.g. byte-swapping, catching illegal writes to memory). A generic memory access function, get\_long() for example, goes through the table of memory banks (mem\_banks) and fetches the appropriate specific memory access fonction, lget() in our example. This slows down the emulator, of course.

1. Emulated CPU, "direct" addressing (`EMULATED_68K = 1`, `DIRECT_ADDRESSING = 1`):  
As in the virtual addressing mode, the 68k processor is emulated with the UAE CPU engine and two memory areas are set up for RAM and ROM. Mac RAM starts at address 0 for the emulated 68k, but it may start at a different address for the host CPU. Besides, the virtual memory areas seen by the emulated 68k are separated by exactly the same amount of bytes as the corresponding memory areas allocated on the host CPU. This means that address translation simply implies the addition of a constant offset (MEMBaseDiff). Therefore, the memory banks are no longer used and the memory access functions are replaced by inline memory accesses.

1. Emulated CPU, "real" addressing (`EMULATED_68K = 1`, `REAL_ADDRESSING = 1`):  
This mode is intended for non-68k systems that do allow access to RAM at `0x0000..0x1fff`. As in the virtual addressing mode, the 68k processor is emulated with the UAE CPU engine and two areas are allocated for RAM and ROM but the emulated CPU lives in the same address space as the host CPU. This means that if something is located at a certain address for the 68k, it is located at the exact same address for the host CPU. Mac addresses and host addresses are the same. The memory accesses of the CPU emulation still go through access functions but the address translation is no longer needed. The memory access functions are replaced by direct, inlined memory accesses, making for the fastest possible speed of the emulator. On little-endian systems, byte-swapping is still required, of course.  
A usual consequence of the real addressing mode is that the Mac RAM doesn't any longer begin at address 0 for the Mac and that the Mac ROM also is not located where it usually is on a real Mac. But as the Mac ROM is relocatable and the available RAM is defined for MacOS by the start of the system zone (which is relocated to the start of the allocated RAM area) and the MemTop variable (which is also set correctly) this is not a problem. There is, however, one RAM area that must lie in a certain address range. This area contains the Mac "Low Memory Globals" which (on a Mac II) are located at `0x0000..0x1fff` and which cannot be moved to a different address range. The Low Memory Globals constitute of many important MacOS and application global variables (e.g. the above mentioned "MemTop" variable which is located at `0x0108`). For the real addressing mode to work, the host CPU needs access to `0x0000..0x1fff`. Under most operating systems, this is a big problem. On some systems, patches (like PrepareEmul on the Amiga or the sheep\_driver under BeOS) can be installed to "open up" this area. On other systems, it might be possible to use access exception handlers to emulate accesses to this area. But if the Low Memory Globals area cannot be made available, using the real addressing mode is not possible.  
**Note:** currently, real addressing mode is known to work only on AmigaOS, NetBSD/m68k, FreeBSD/i386, Linux/ppc and Linux/i386.

1. Native CPU (`EMULATED_68K = 0`, this also requires `REAL_ADDRESSING = 1`)  
This mode is designed for systems that use a 68k (68020 or better) processor as host CPU and is the technically most difficult mode to handle. The Mac CPU is no longer emulated (the UAE CPU emulation is not needed) but MacOS and Mac applications run natively on the existing 68k CPU. This means that the emulator has its maximum possible speed (very close to that of a real Mac with the same CPU). As there is no control over the memory accesses of the CPU, real addressing mode is implied, and so the Low Memory area must be accessible (an MMU might be used to set up different address spaces for the Mac and the host, but this is not implemented in Basilisk II). The native CPU mode has some possible pitfalls that might make its implementation difficult on some systems:

   1. Implied real addressing (this also means that Mac programs that go out of control can crash the emulator or the whole system)
   1. MacOS and Mac applications assume that they always run in supervisor mode (more precisely, they assume that they can safely use certain priviledged instructions, mostly for interrupt control). So either the whole emulator has to be run in supervisor mode (which usually is not possible on multitasking systems) or priviledged instructions have to be trapped and emulated. The Amiga and NetBSD/m68k versions of Basilisk II use the latter approach (it is possible to run supervisor mode tasks under the AmigaOS multitasking kernel (ShapeShifter does this) but it requires modifying the Exec task switcher and makes the emulator more unstable).
   1. On multitasking systems, interrupts can usually not be handled as on a real Mac (or with the UAE CPU). The interrupt levels of the host will not be the same as on a Mac, and the operating systems might not allow installing hardware interrupt handlers or the interrupt handlers might have different stack frames and run-time environments than 68k hardware interrupts. The usual solution is to use some sort of software interrupts or signals to interrupt the main emulation process and to manually call the Mac 68k interrupt handler with a faked stack frame.
   1. 68060 systems are a small problem because there is no Mac that ever used the 68060 and MacOS doesn't know about this processor. Basilisk II reports the 68060 as being a 68040 to the MacOS and patches some places where MacOS makes use of certain 68040-specific features such as the FPU state frame layout or the PTEST instruction. Also, Basilisk II requires that all of the Motorola support software for the 68060 to emulate missing FPU and integer instructions and addressing modes is provided by the host operating system (this also applies to the 68040).
   1. The "EMUL\_OP" mechanism described below requires the interception and handling of certain emulator-defined instructions.

## 3. Memory access

There is often a need to access Mac RAM and ROM inside the emulator. As Basilisk II may run in "real" or "virtual" addressing mode on many different architectures, big-endian or little-endian, certain platform-independent data types and functions are provided:

   1. "sysdeps.h" defines the types int8, uint8, int16, uint16, int32 and uint32 for numeric quantities of a certain signedness and bit length
   1. "cpu\_emulation.h" defines the ReadMacInt\*() and WriteMacInt\*() functions which should always be used to read from or write to Mac RAM or ROM
   1. "cpu\_emulation.h" also defines the Mac2HostAddr() function that translates a Mac memory address to a (uint8 \*) in host address space. This allows you to access larger chunks of Mac memory directly, without going through the read/write functions for every access. But doing so you have to perform any needed endianness conversion of the data yourself by using the ntohs() etc. macros which are available on most systems or defined in "sysdeps.h".

## 4. Calling native routines from 68k mode and vice-versa

An emulator like Basilisk II requires two kinds of cross-platform function calls:

   1. Calling a native routine from the Mac 68k context
   1. Calling a Mac 68k routine from the native context

Situation 1. arises in nearly all Basilisk drivers and system patches while case 2. is needed for the invocation of Mac call-back or interrupt routines. Basilisk II tries to solve both problems in a way that provides the same interface whether it is running on a 68k or a non-68k system.

### 4.1. The EMUL\_OP mechanism

Calling native routines from the Mac 68k context requires breaking out of the 68k emulator or interrupting the current instruction flow and is done via unimplemented 68k opcodes (called "EMUL\_OP" opcodes). Basilisk II uses opcodes of the form `0x71xx` (these are invalid MOVEQ opcodes) which are defined in "emul\_op.h". When such an opcode is encountered, whether by the emulated CPU or a real 68k, the execution is interrupted, all CPU registers saved and the EmulOp() function from "emul\_op.cpp" is called. EmulOp() decides which opcode caused the interrupt and performs the required actions (mostly by calling other emulator routines). The EMUL\_OP handler routines have access to nearly all of the 68k user mode registers (exceptions being the PC, A7 and SR). So the EMUL\_OP opcodes can be thought of as extensions to the 68k instruction set. Some of these opcodes are used to implement ROM or resource patches because they only occupy 2 bytes and there is sometimes not more room for a patch.

### 4.2. Execute68k()

"cpu\_emulation.h" declares the functions Execute68k() and Execute68kTrap() to call Mac 68k routines or MacOS system traps from inside an EMUL\_OP handler routine. They allow setting all 68k user mode registers (except PC and SR) before the call and examining all register contents after the call has returned. EMUL\_OP and Execute68k() may be nested, i.e. a routine called with Execute68k() may contain EMUL\_OP opcodes and the EMUL\_OP handlers may in turn call Execute68k() again.

## 5. Interrupts

Various parts of Basilisk II (such as the Time Manager and the serial driver) need an interrupt facility to trigger asynchronous events. The MacOS uses different 68k interrupt levels for different events, but for simplicity Basilisk II only uses level 1 and does it's own interrupt dispatching. The "InterruptFlags" contains a bit mask of the pending interrupts. These are the currently defined interrupt sources (see main.h):

```
INTFLAG_60HZ    - MacOS 60Hz interrupt (unlike a real Mac, we also handle
                  VBL interrupts and the Time Manager here)
INTFLAG_1HZ     - MacOS 1Hz interrupt (updates system time)
INTFLAG_SERIAL  - Interrupt for serial driver I/O completion
INTFLAG_ETHER   - Interrupt for Ethernet driver I/O completion and packet
                  reception
INTFLAG_AUDIO   - Interrupt for audio "next block" requests
INTFLAG_TIMER   - Reserved for a future implementation of a more precise
                  Time Manager (currently not used)
INTFLAG\_ADB     - Interrupt for mouse/keyboard input
INTFLAG_NMI     - NMI for debugging (not supported on all platforms)
```

An interrupt is triggered by calling SetInterruptFlag() with the desired interrupt flag constant and then TriggerInterrupt(). When the UAE 68k emulator is used, this will signal a hardware interrupt to the emulated 680x0. On a native 68k machine, some other method for interrupting the MacOS thread has to be used (e.g. on AmigaOS, a signal exception is used). Care has to be taken because with the UAE CPU, the interrupt will only occur when Basilisk II is executing MacOS code while on a native 68k machine, the interrupt could occur at any time (e.g. inside an EMUL\_OP handler routine). In any case, the MacOS thread will eventually end up in the level 1 interrupt handler which contains an M68K\_EMUL\_OP\_IRQ opcode. The opcode handler in emul\_op.cpp will then look at InterruptFlags and decide which routines to call.

## 6. Parts of Basilisk II

The conception of Basilisk II is quite modular and consists of many parts which are relatively independent from each other:

- UAE CPU engine ("uae\_cpu/\*", not needed on all systems)
- ROM patches ("rom\_patches.cpp", "slot\_rom.cpp" and "emul\_op.cpp")
- resource patches ("rsrc\_patches.cpp" and "emul\_op.cpp")
- PRAM Utilities replacement ("xpram.cpp")
- ADB Manager replacement ("adb.cpp")
- Time Manager replacement ("timer.cpp")
- SCSI Manager replacement ("scsi.cpp")
- video driver ("video.cpp")
- audio component ("audio.cpp")
- floppy driver ("sony.cpp")
- disk driver ("disk.cpp")
- CD-ROM driver ("cdrom.cpp")
- external file system ("extfs.cpp")
- serial drivers ("serial.cpp")
- Ethernet driver ("ether.cpp")
- system-dependent device access ("sys\_\*.cpp")
- user interface strings ("user\_strings.cpp")
- preferences management ("prefs.cpp" and "prefs\_editor\_*.cpp")

Most modules consist of a platform-independant part (such as video.cpp) and a platform-dependent part (such as video\_beos.cpp). The "dummy" directory contains generic "do-nothing" versions of some of the platform-dependent parts to aid in testing and porting.

### 6.1. UAE CPU engine

All files relating to the UAE 680x0 emulation are kept in the "uae\_cpu" directory. The "cpu\_emulation.h" header file defines the link between the UAE CPU and the rest of Basilisk II, and "basilisk\_glue.cpp" implements the link. It should be possible to replace the UAE CPU with a different 680x0 emulation by creating a new "xxx\_cpu" directory with an appropriate "cpu\_emulation.h" header file (for the inlined memory access functions) and writing glue code between the functions declared in "cpu\_emulation.h" and those provided by the 680x0 emulator.

### 6.2. ROM and resource patches

As described above, instead of emulating custom Mac hardware, Basilisk II provides replacements for certain parts of MacOS to redirect input, output and system control functions of the Mac hardware to the underlying operating systems. This is done by applying patches to the Mac ROM ("ROM patches") and the MacOS system file ("resource patches", because nearly all system software is contained in MacOS resources). Unless resources are written back to disk, the system file patches are not permanent (it would cause many problems if they were permanent, because some of the patches vary with different versions of Basilisk II or even every time the emulator is launched).

ROM patches are contained in "rom\_patches.cpp" and resource patches are contained in "rsrc\_patches.cpp". The ROM patches are far more numerous because nearly all the software needed to run MacOS is contained in the Mac ROM (the system file itself consists mainly of ROM patches, in addition to pictures and text). One part of the ROM patches involves the construction of a NuBus slot declaration ROM (in "slot\_rom.cpp") which is used to add the video and Ethernet drivers. Apart from the CPU emulation, the ROM and resource patches contain most of the "logic" of the emulator.

### 6.3. PRAM Utilities

MacOS stores certain nonvolatile system parameters in a 256 byte battery backed-up CMOS RAM area called "Parameter RAM", "PRAM" or "XPRAM" (which refers to "Extended PRAM" because the earliest Mac models only had 20 bytes of PRAM). Basilisk II patches the ClkNoMem() MacOS trap which is used to access the XPRAM (apart from some routines which are only used early during system startup) and the real-time clock. The XPRAM is emulated in a 256 byte array which is saved to disk to preserve the contents for the next time Basilisk is launched.

### 6.4. ADB Manager

For emulating a mouse and a keyboard, Basilisk II patches the ADBOp() MacOS trap. Platform-dependent code reports mouse and keyboard events with the ADBMouseDown() etc. functions where they are queued, and the INTFLAG\_ADB interrupt is triggered. The ADBInterrupt() handler function sends the input events to MacOS by calling the ADB mouse and keyboard handlers with Execute68k().

### 6.5. Time Manager

Basilisk II completely replaces the Time Manager (InsTime(), RmvTime(), PrimeTime() and Microseconds() traps). A "TMDesc" structure is associated with each Time Manager task, that contains additional data. The tasks are executed in the TimerInterrupt() function which is currently called inside the 60Hz interrupt handler, thus limiting the resolution of the Time Manager to 16.6ms.

### 6.6. SCSI Manager

The (old-style) SCSI Manager is also completely replaced and the MacOS SCSIDispatch() trap redirected to the routines in "scsi.cpp". Under the MacOS, programs have to issue multiple calls for all the different phases of a SCSI bus interaction (arbitration, selection, command transfer etc.). Basilisk II maps this API to an atomic API which is used by most modern operating systems. All action is deferred until the call to SCSIComplete(). The TIB (Transfer Instruction Block) mini-programs used by the MacOS are translated into a scatter/gather list of data blocks. Operating systems that don't support scatter/gather SCSI I/O will have to use buffering if more than one data block is being transmitted. Some more advanced (but rarely used) aspects of the SCSI Manager (like messaging and compare operations) are not emulated.

### 6.7. Video driver

The NuBus slot declaration ROM constructed in "slot\_rom.cpp" contains a driver definition for a video driver. The Control and Status calls of this driver are implemented in "video.cpp".

The host-side initialization of the video system is done in VideoInit(). This function must fill the VideoModes vector with a list of supported video modes (combinations of color depth and resolution). It must then call video\_init\_depth\_list() and setup the VideoMonitor structure with the default mode information and the address of a frame buffer for MacOS. In real addressing mode, this frame buffer must be in a MacOS compatible layout (big-endian and 1, 2, 4 or 8 bits paletted chunky pixels, RGB 5:5:5 or xRGB 8:8:8:8). In virtual addressing mode, the frame buffer is located at address `0xa0000000` on the Mac side and you have to supply the host address, size and layout (BasiliskII will do an automatic pixel format conversion in virtual addressing mode) in the variables MacFrameBaseHost, MacFrameSize and MacFrameLayout.

There are two functions of the platform-dependent video driver code that get called during runtime: video\_set\_palette() to update the CLUT (for indexed modes) or gamma table (for direct color modes), and video\_switch\_to\_mode() to switch to a different color depth and/or resolution (in this case the frame buffer base in VideoMonitor must be updated).

### 6.8. Audio component

Basilisk II provides a Sound Manager 3.x audio component for sound output. Earlier Sound Manager versions that don't use components but 'snth' resources are not supported. Nearly all component functions are implemented in "audio.cpp". The system-dependent modules ("audio\_\*.cpp") handle the initialization of the audio hardware/driver, volume controls, and the actual sound output.

The mechanism of sound output varies depending on the platform but usually there will be one "streaming thread" (either a thread that continuously writes data buffers to the audio device or a callback function that provides the next data buffer) that reads blocks of sound data from the MacOS Sound Manager and writes them to the audio device. To request the next data buffer, the streaming thread triggers the INTFLAG\_AUDIO interrupt which will cause the MacOS thread to eventually call AudioInterrupt(). Inside AudioInterrupt(), the next data block will be read and the streaming thread is signalled that new audio data is available.

### 6.9. Floppy, disk and CD-ROM drivers

Basilisk II contains three MacOS drivers that implement floppy, disk and CD-ROM access ("sony.cpp", "disk.cpp" and "cdrom.cpp"). They rely heavily on the functionality provided by the "sys\_\*.cpp" module. BTW, the name ".Sony" of the MacOS floppy driver comes from the fact that the 3.5" floppy drive in the first Mac models was custom-built for Apple by Sony (this was one of the first applications of the 3.5" floppy format which was also invented by Sony).

### 6.10. External file system

Basilisk II also provides a method for accessing files and direcories on the host OS from the MacOS side by means of an "external" file system (henceforth called "ExtFS"). The ExtFS is built upon the File System Manager 1.2 interface that is built into MacOS 7.6 (and later) and available as a system extension for earlier MacOS versions. Unlike other parts of Basilisk II, extfs.cpp requires POSIX file I/O and this is not going to change any time soon, so if you are porting Basilisk II to a system without POSIX file functions, you should emulate them.

### 6.11. Serial drivers

Similar to the disk drivers, Basilisk II contains replacement serial drivers for the emulation of Mac modem and printer ports. To avoid duplicating code, both ports are handled by the same set of routines. The SerialPrime() etc. functions are mostly wrappers that determine which port is being accessed. All the real work is done by the "SERDPort" class which is subclassed by the platform-dependent code. There are two instances (for port A and B) of the subclasses.

Unlike the disk drivers, the serial driver must be able to handle asynchronous operations. Calls to SerialPrime() will usually not actually transmit or receive data but delegate the action to an independant thread. SerialPrime() then returns "1" to indicate that the I/O operation is not yet completed. The completion of the I/O request is signalled by calling the MacOS trap "IODone". However, this can't be done by the I/O thread because it's not in the right run-time environment to call MacOS functions. Therefore it will trigger the INTFLAG\_SERIAL interrupt which causes the MacOS thread to eventually call SerialInterrupt(). SerialInterrupt(), in turn, will not call IODone either but install a Deferred Task to do the job. The Deferred Task will be called by MacOS when it returns to interrupt level 0. This mechanism sounds complicated but is necessary to ensure stable operation of the serial driver.

### 6.12. Ethernet driver

A driver for Ethernet networking is also contained in the NuBus slot ROM. Only one ethernet card can be handled by Basilisk II. For Ethernet to work, Basilisk II must be able to send and receive raw Ethernet packets, including the 14-byte header (destination and source address and type/length field), but not including the 4-byte CRC. This may not be possible on all platforms or it may require writing special net drivers or add-ons or running with superuser priviledges to get access to the raw packets.

For situations in which access to raw Ethernet packets is not possible, Basilisk II implements a special "tunnelling" mode in which it sends and receives packets via UDP, using BSD socket functions. It simply wraps the Ethernet packets into UDP packets, using dummy Ethernet addresses that are made up of the IP address of the host. Ethernet broadcast and AppleTalk multicast packets are sent to the IP broadcast address. Because of this non-standard way of tunnelling, it is only possible to set up a "virtual" network amongst machines running Basilisk II in this way.

Writing packets works as in the serial drivers. The ether\_write() routine may choose to send the packet immediately (e.g. under BeOS) and return noErr or to delegate the sending to a separate thread (e.g. under AmigaOS) and return "1" to indicate that the operation is still in progress. For the latter case, a Deferred Task structure is provided in the ether\_data area to call IODone from EtherInterrupt() when the packet write is complete (see above for a description of the mechanism).

Packet reception is a different story. First of all, there are two methods provided by the MacOS Ethernet driver API to read packets, one of which (ERead/ ERdCancel) is not supported by Basilisk II. Basilisk II only supports reading packets by attaching protocol handlers. This shouldn't be a problem because the only network code I've seen so far that uses ERead is some Apple sample code. AppleTalk, MacTCP, MacIPX, OpenTransport etc. all use protocol handlers. By attaching a protocol handler, the user of the Ethernet driver supplies a handler routine that should be called by the driver upon reception of Ethernet packets of a certain type. 802.2 packets (type/length field of 0..1500 in the packet header) are a bit special: there can be only one protocol handler attached for 802.2 packets (by specifying a packet type of "0"). The MacOS LAP Manager will attach a 802.2 handler upon startup and handle the distribution of 802.2 packets to sub-protocol handlers, but the Basilisk II Ethernet driver is not concerned with this.

When the driver receives a packet, it has to look up the protocol handler installed for the respective packet type (if any has been installed at all) and call the packet handler routine. This must be done with Execute68k() from the MacOS thread, so an interrupt (INTFLAG\_ETHER) is triggered upon reception of a packet so the EtherInterrupt() routine can call the protocol handler. Before calling the handler, the Ethernet packet header has to be copied to MacOS RAM (the "ed\_RHA" field of the ether\_data structure is provided for this). The protocol handler will read the packet data by means of the ReadPacket/ReadRest routines supplied by the Ethernet driver. Both routines will eventually end up in EtherReadPacket() which copies the data to Mac address space. EtherReadPacket() requires the host address and length of the packet to be loaded to a0 and d1 before calling the protocol handler.

Does this sound complicated? You are probably right. Here is another description of what happens upon reception of a packet:

1. Ethernet card receives packet and notifies some platform-dependent entity inside Basilisk II
1. This entity will store the packet in some safe place and trigger the INTFLAG\_ETHER interrupt
1. The MacOS thread will execute the EtherInterrupt() routine and look for received packets
1. If a packet was received of a type to which a protocol handler had been attached, the packet header is copied to ed\_RHA, a0/d1 are loaded with the host address and length of the packet data, a3 is loaded with the Mac address of the first byte behind ed\_RHA and a4 is loaded with the Mac address of the ed\_ReadPacket code inside ether\_data, and the protocol handler is called with Execute68k()
1. The protocol handler will eventually try to read the packet data with a "jsr (a4)" or "jsr 2(a4)"
1. This will execute an M68K\_EMUL\_OP\_ETHER\_READ\_PACKET opcode
1. The EtherReadPacket() opcode handling routine will copy the requested part of the packet data to Mac RAM using the pointer and length which are still in a0/d1

For a more detailed description of the Ethernet driver, see the book "Inside AppleTalk".

### 6.13. System-dependent device access

The method for accessing floppy drives, hard disks, CD-ROM drives and files vary greatly between different operating systems. To make Basilisk II easily portable, all device I/O is made via the functions declared in "sys.h" and implemented by the (system-dependent) "sys\_\*.cpp" modules which provides a standard, Unix-like interface to all kinds of devices.

### 6.14. User interface strings

To aid in localisation, all user interface strings of Basilisk II are collected in "user\_strings.cpp" (for common strings) and "user\_strings\_\*.cpp" (for platform-specific strings), and accessed via the GetString() function. This way, Basilisk II may be easily translated to different languages.

### 6.15. Preferences management

The module "prefs.cpp" handles user preferences in a system-independant way. Preferences items are accessed with the PrefsAdd\*(), PrefsReplace\*() and PrefsFind\*() functions and stored in human-readable and editable text files on disk. There are two lists of available preferences items. The first one, common\_prefs\_items, is defined in "prefs\_items.cpp" and lists items which are available on all systems. The second one, platform\_prefs\_items, is defined in "prefs\_\*.cpp" and lists the prefs items which are specific to a certain platform.

The "prefs\_editor\_\*.cpp" module provides a graphical user interface for setting the preferences so users won't have to edit the preferences file manually.

## 7. Porting Basilisk II

Porting Basilisk II to a new platform should not be hard. These are the steps involved in the process:

1. Create a new directory inside the "src" directory for your platform. If your platform comes in several "flavours" that require adapted files, you should consider creating subdirectories inside the platform directory. All files needed for your port must be placed inside the new directory. Don't scatter platform-dependent files across the "src" hierarchy.
1. Decide in which mode (virtual addressing, real addressing or native CPU) Basilisk II will run.
1. Create a "sysdeps.h" file which defines the mode and system-dependent data types and memory access functions. Things which are used in Basilisk but missing on your platform (such as endianness macros) should also be defined here.
1. Implement the system-specific parts of Basilisk:  
main\_\*.cpp, sys\_\*.cpp, prefs\_\*.cpp, prefs\_editor\_\*.cpp, xpram\_\*.cpp, timer\_\*.cpp, audio\_\*.cpp, video\_\*.cpp, serial\_\*.cpp, ether\_\*.cpp, scsi\_\*.cpp and clip\_\*.cpp  
You may want to take the skeleton implementations in the "dummy" directory as a starting point and look at the implementation for other platforms before writing your own.
1. Important things to remember:
   - Use the ReadMacInt\*() and WriteMacInt\*() functions from "cpu\_emulation.h" to access Mac memory
   - Use the ntohs() etc. macros to convert endianness when accessing Mac memory directly
   - Don't modify any source files outside of your platform directory unless you really, really have to. Instead of adding "#ifdef PLATFORM" blocks to one of the platform-independent source files, you should contact me so that we may find a more elegant and more portable solution.
1. Coding style: indent -kr -ts4

Christian Bauer  
<Christian.Bauer@uni-mainz.de>
