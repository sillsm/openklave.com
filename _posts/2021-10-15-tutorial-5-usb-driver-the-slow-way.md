---
title:  "Tutorial 5: Writing a USB driver for a chip the hard way"
layout: post
tag: developer
---

# Write a USB driver the hard way

USB is a really fast protocol. Unlike driving the LCD, USB messages transact too fast for us to be able to send or receive 
specific messages in real time using GDB and stepping through the results. In this tutorial, we're going to 
write a USB driver the slow way. We're going to learn about every single register that makes USB work, as well
as deep-dive into the protocol itself. We'll master one transaction at a time, from device enumeration to sending midi back and forth
to the computer.

A note: this technique is excellent for learning, but creates barely-legible spaghetti code if you take it too seriously.
A USB driver is one of the areas where abstraction is really important. Once you learn the basics, you should do 
your best to abstract away from them in your code so a reader can understand what is going on.



# Turn the USB peripheral on and clock it

Blah blah
