---
title:  "Tutorial 2: Ripping, restoring, and decompiling the factory operating system"
layout: post
tag: developer
---

# Ripping, restoring, and decompiling the factory operating system.

In this tutorial we're going to review the memory layout of the STM32F103.
We'll use this information and our tools to copy both the factory bootloader 
and factory operating system to our development computer. We'll finish
by opening both the bootloader and operating system in Ghidra, and 
writing some early observations about how they work.

# The memory layout of the STM32F103: Memory-mapped I/O

Here is what the memory layout of the STM32 looks like.

<p align="center">
  <img width="460" height="300" src="https://i.stack.imgur.com/ugTmg.png">
</p>

Memory addresses are represented in hexidecimal, so each digit goes 1 to 9, then A, B, C, D, E, F.

What's really fascinating is that every part of the chip is represented as a memory address. So when we see which wires are connected to which, we will literally be reading and writing memory addresses. When we write to or read from the USB, it will be abstracted as writing to or reading from memory. As we'll see, our entire operating system will use this abstraction over and over to communicate with the LCD screen, the keyboard buttons, and the USB module.

There are two pieces of software running on the device, the bootloader, and the operating system. The bootloader is executed first, then it finds the stored
copy of the operating system and loads it into ram.

The bootloader is located from memory address 0x0 to 0x6000, and it is mirrored at 0x08000000 to 0x08006000. Canonically, the operating
system is located starting at 0x08006000.

The bootloader is 0 to 0x6000 or 24576 bytes. That's not too bad! We have a reasonable chance of reading and understanding what it does if we were to decompile it in Ghidra.

Each memory address stores a byte, which is made of up of two hex digits, like 0xA5, or 0x01, for example, up to 0xFF.

The addressess typically run in sequences of 2 or 4 bytes. 2 byte sequences, like 0x04 0x06 (move the value of register 0 to register 4) are usually Cortex M3 instructions. 4 byte sequences are typically either "vectors," which means they store an address on the chip itself, or 4 byte Cortex M3 instructions.

It can be really hard to read through sequences of bytes to try to figure out which instructions are encoded, forget about whole functions. Decompilers like Ghidra or IDA load these byte sequences and display them in human-readable code so you can understand what is happening.


### Ripping the bootloader, the OS, and both.

GDB can copy sectors of memory from your MPK2 to your local filesystem using the [dump command](https://stuff.mit.edu/afs/athena/project/rhel-doc/3/rhel-gdb-en-3/dump-restore-files.html). We've made a script in the util/ folder of the Open Klave source tree to make this easier: "copy_os_from_device.sh".

```
 dump binary memory bootloader.bin 0 0x6000
```
 Will dump the bootloader to a bin file. Note you can also dump it to an ihex file format, which we'll discuss later.
 Make sure you keep this backup file in a safe place. It is essential to be able to quickly and cleanly load and wipe the chip memory.

 ```
 dump binary memory mpk2os.bin 0x08006000 0x08033fff
 ```

 Let's be safe and also dump the os in the ihex format so we can reload it using the bootloader

 ```
dump ihex memory mpk2os.ihex 0x08006000 0x08033fff
 ```

Now shockingly enough, this bare ihex dump is sufficient to completely restore the operating system if you bork it during development.
The bootloader has a built-in recovery mode which receives ihex as a sysex message and rerwrites the flash over usb.

# Use the factory bootloader to load the operating system back on to the keyboard

Now that we've copied the OS to our development computer, let's load it back onto the device. The factory bootloader does a lot with very little. It has a bare USB driver that is able to accept new software in the form of MIDI Sysex commands from the development computer. Note, if you are using these tutorials to learn about reverse engineering and bare metal OS development generally, you could always just flash an OS back your device using GDB. But throughout Open Klave we rely on the factory bootloader, because it can be accessed using just a USB cable, so non-developers can use Open Klave on their machine without a JTAG probe.

Earlier, we ripped the operating system and saved it as [IHex, or Intel Hex](https://en.wikipedia.org/wiki/Intel_HEX). IHex is an interesting binary protocol. It is a sequence of records in plaintext, specifying a start address on the chip, and then data to copy to that address. It is coded in ASCII to be human readable. So for example, if you wanted to copy the byte '0xE3' to the device, that would be represented in ihex as 0x69 0x51, the hexidecimal representations of 'e' and '3'. It takes 2n bytes to convey n bytes of information in the format.

Conveniently, this format is also byte-compatible with [Midi Sysex](https://en.wikipedia.org/wiki/MIDI_Machine_Control#MIDI_Universal_Real_Time_SysEx_Message_Format). So basically, you can take a binary written in ihex, and just add F0 and the machine status bytes at the top of the message, and F7 at the end, and the bootloader will understand and copy it to the device. You can even break up ihex messages into chunks for faster transmission.

Read through the code in 'util/flash_os.py', and use it to copy the operating system back on the device. First, turn the power off. Then, while holding the 'Push to Enter' button, turn the power back on. Then run the script to copy the OS to the chip.

# Decompile the factory bootloader in Ghidra

We have now copied the bootloader and operating system to our development computer, and used the factory bootloader to reload the operating system back onto the keyboard just using the USB cable. Let's take a moment and see if we can learn anything about the bootloader code.

Open Ghidra, and import the bootloader ihex file. The format is Intel Hex, and the language should be Arm Cortex 32-bit little endian. Do not analyze it yet. We need to make a few adjustments to see it in proper context. First, you should install the SVD-loader plugin for Ghidra from leveldown security. This allows you to take an SVD file, which maps memory areas to text descriptions of peripheral devices to your binary. You can find an XML description of the STM32F103 architecture in 'util/STM32F103xx.svd.txt", which you can load.

Next, we need to remap Ghidra's memory so it is properly aligning the binary addresses with its internal description of how the STM32F103 works. Finally, open the analyzer and do all the analyses. If everything worked you should see something like this:

<p align="center">
  <img src="/pics/Ghidra_start_vector.png">
</p>


This is the 'interrupt vector table'. It specifies 4 byte addresses that should be jumped to when different events happen on the device; for example if a timer goes off, a USB is connected, or the device is powered on. Let's jump to the 'reset' vector in Ghidra. This is the entry point the bootloader goes when the device is turned on:


<p align="center">
  <img src="/pics/Ghidra_bootloader_entry.png">
</p>

# Playing with Ghidra: where is ihex loaded in the bootloader?

Now we're cooking. We can start to do an educational loop: we query where a certain functionality takes place in Ghidra, and we can set a breakpoint in GDB and watch it live. Or we could set a breakpoint in GDB that happens when a value of interest changes, and write down the interesting instruction and examine it in Ghidra.

We learned above the bootloader starts a USB connection with the host computer, and then accepts ihex in the form of a stream of Sysex messages. How do we find out where it is?

We turn to our first of several thousand trips to the STM32f103 developer manual. You can find a mirrored copy [here](https://github.com/sillsm/openklave.com/raw/master/docs/stm32f103%20dev%20manual.pdf). I cannot overstate the importance of reading this manual - literally every single thing you could ever want to know about keyboard's chip is explained in excruciating detail. For these tutorials, we'll start by skimming just a bit at a time when it is relevant so we can develop familiarity with it.

Ok so according to the manual, the interrupt routine that should be called every time there's a USB event is at offset 0x90. 

<p align="center">
  <img src="/pics/USB_interrupt_man.png">
</p>

Ah! There's a routine defined there, at address 0x080036d0. As you can see, I hand renamed these addresses and functions, which you can do too in Ghidra by right or long pressing on any address or name and clicking rename.


<p align="center">
  <img src="/pics/USB_entry_routine.png">
</p>

# Essential Software 

1. Ghidra, for decompiling the system firmware and annotating it.
2. OpenOCD, for connecting your laptop debugger to the microcontroller and setting up a gdb instance.
3. GDB, for debugging.
4. Python for general scripting.
5. The [rtmidi](https://github.com/SpotlightKid/python-rtmidi) python package, available on pip.
6. The arm-none-eabi-gcc cross-compiler toolchain.

# Essential Tools

1. [An ST-Link/V2 debugger for the STM32 architecture.](https://www.amazon.com/ST-LINK-V2-circuit-debugger-programmer/dp/B00SDTEM1I/ref=sr_1_3?dchild=1&keywords=st-link%2Fv2+debugger&qid=1618172788&sr=8-3). It costs about $30. Get the real one.
2. A screw driver to open the keyboard and seperate the top plate from the bottom.

# Operating System Developer Only

If you don't want to open the device or buy anything beyond a USB cable, you can develop, compile, and burn Open Klave to the MPK2 series using just Ghidra, python, and the arm-none-eabi-gcc tools. However, it's highly reccommended to get your hands dirty and get all the tools, as there is an essential learning loop between poking the machine in gdb, and learning how to write good routines in c. 

## Diving in

You can do a lot with just python. You can send sysex commands to the MPK2 device and its factory operating system, for example by using rtmidi to send midi messages from your laptop to the device. Doing this, you can change the colors on the keypads, and even intercept and modify midi messages between the device and your software. You can do just about everything but send messages to the LCD or get extended keypad colors. [This is a great resource](http://practicalusage.com/akai-mpk261-mpk2-series-controlling-the-controller-with-sysex/) to play around with that, and [this is even better](https://github.com/nsmith-/mpk2).

The goal of these notes and operating system is to permit keyboard owners to execute arbitrary code with only a USB cable, on top of a lightweight and accommodating operating system. Go get your screw driver and let's see what we can learn.

### Investigating the firmware

Unscrew the screws at the bottom of your keyboard, and investigate the primary printed circuit board.

<p align="center">
  <img width="460" height="300" src="/pics/mpk249pcb.jpg">
</p>

We notice two things. First, its brain is a microcontroller called an STM32F103. Second, it has a 16 pin JTAG connector so you can debug it.
The ST-Link/V2 you bought earlier has a connector which fits snuggly over the JTAG port.

#### A note on the STM32F103

The STM32F103 runs an instruction set called ARM Cortex M3. The chip is 32-bit and little-endian, which will be important for setting up Ghidra and sending values to it later.

# Exercise: Establish a connection to the device and GDB in

Let's end the tutorial by making sure your tools work. Have you installed python and rtmidi? What happens when you type at the command prompt
```
python
import rtmidi
quit()
```

If there are error messages, try again. Please also verify you have installed openocd by typing 'openocd --version'.

## Get connected, for free

Here's what it should look like once you have the ST-Link/V2 plugged in to the MPK2 JTAG. Note that this method requires 2 active USB connections to work. So you'll have the ST-LINK/v2 connected to the JTAG port and plugged into your computer's USB, and you'll have a second USB connection to the back of the keyboard, to power and send midi back and forth, like you would usually if you were just trying to plug it into a DAW.

<p align="center">

  <img width="460" height="300" src="/pics/debugger-setup.jpg">
</p>

## Launching the GDB server

Let's verify all the wires are properly connected. Open up two terminal windows. In one, you will run OpenOCD to launch a GDB server.

```
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg
```

If everything is working properly you should see something like
```
...
Info : STLINK V2J29S7 (API v2) VID:PID 0483:3748
Info : Target voltage: 3.217323
Info : stm32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32f1x.cpu on 3333
Info : Listening on port 3333 for gdb connections
...
```

Now, in a second terminal, start gdb and connect to this local server.

```
gdb

(in gdb)
target remote localhost:3333
```
which should display
```
Remote debugging using localhost:3333
```

If you did everything right and sucessfully connected gdb to the MPK2, the unit will display cool debug stripes in the light pads indicating the keyboard is in debug mode. Notice how the MPK249 debug mode alters the pad colors between columns of blue and green.

<p align="center">
  <img width="460" height="300" src="/pics/debugstripes.jpg">
</p>

The reason for this is because each pad only has a single red, green, and blue LED. All other colors are simulated by rapidly turning these on and off. In default debug mode, the animation routine is silenced.

When tinkering with the keyboard, it is more comfortable to adjust the plastic hood mostly back over the keys, so you can play notes and test things out as you debug.

1. Press CTRL-C to stop the keyboard executing its current program, and enter debug mode.
2. Enter 'c' to continue execution. PC means 'program counter'. It is the current instruction that is executing.
3. Enter 'stepi' to step forward one instruction on the keyboard.
4. Reset the keyboard by entering 'monitor reset halt'. You will need to enter c to start it running like normal.

Whew! Good work! 

