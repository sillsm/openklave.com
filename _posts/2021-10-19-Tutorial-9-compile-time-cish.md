---
title:  "Tutorial 9: Compile-time testing in C-ish"
layout: post
tag: developer
---

# Compile-time testing in C-ish

Open Klave is written in C, but its target architecture is a piano, not the development computer. It can be difficult
creating data structures and abstractions for a different chip. You don't know stuff works until you try to run it.

In this tutorial we review C-ish, which is my term for writing in C, except we also permit the constexpr keyword from C++17 on, 
and we use a C++ compiler. By writing in C-ish, we can dramatically reduce the complexity of our operating system
by mostly sticking to C conventions. But the constexpr keyword will permit some compiler testing and sanity-checking
before anything is transmitted to the device.

We will finish by examining Open Klave's "Global Event Stack," and write some compile time tests. And we'll
consider some caveat and limitations to this approach.




# Blah 
