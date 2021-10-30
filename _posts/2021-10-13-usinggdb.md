---
title:  "Tutorial 3: Development techniques in GDB"
layout: post
tag: developer
---

# Development techniques.

We will cover three techniques to reverse engineer a running chip using GDB. 
First, we will learn how to examine the state of all the incoming and outgoing wires in the chip (GPIO). 
Then, we will practice selectively turning off device peripherals to see what happens.
Finally, we will introduce *function prototyping*. Using GDB, we can reset the device and establish the *minimal routine* that is necessary 
to accomplish some objective, for example, make a pad change a color, or write a character to the LCD screen.

As the STM32F103 has a memory mapped peripheral, most operating system stuff boils down to reading from an address, writing to that address,
and looping. We can prototype in GDB first, and then translate the minimal routine almost line for line to C and build on it.




# Technique One: Look at what's happening on the IO wires.
