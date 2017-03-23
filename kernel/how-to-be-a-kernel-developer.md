---
layout : default
---

# Strategies for getting started with kernel programming

1. Strace all thing!
2. Read some kernel code!
3. Write a kernel module!
4. Do the Eudalypta challenge
5. Write your own operating system
6. Do a Linux Kernel internship
	- Google Summer of Code
	- Outreach Program
7. Linux Weekly News!

# Why you should Lean It

* Understanding Your Operation System Makes You A Better Programmer
* Treat Your Programs as A Black Box
* Your Operating System Is a Tool. Learn It.

# Learn by Doing

## PERF: TRACK L1 CACHE MISSES!

1. Track L1 Cache Misses
2. Flame Graph (function calling)

##  Ftrace: Tracing Kernel Functions

- tracking TCP retransmits

## `/proc`

1. Pretty fun simple demo
	- run a program
	- get its PID
	- delete the executable
	- YOU CAN STILL RECOVER IT
	- ls /proc/<PID>/exe
	- cat /proc/<PID>/exe > demo_recover
	- ^C the program
	- grant execution permission to demo_recover
	- running demo_recover (Cool!)

2. `/proc/*/fd`

# References

- [10 things you need to do before sending your patch to LKML](https://yaapb.wordpress.com/2012/12/14/10-things-you-need-to-do-before-sending-your-patch-to-lkml/)
- [The linux-kernel mailing list FAQ](http://vger.kernel.org/lkml/)

[back](../)


