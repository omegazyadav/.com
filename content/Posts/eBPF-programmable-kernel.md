---
title: "eBPF: A Programmable Kernel"
date: 2024-03-17
author: Yadav Lamichhane
description: "Basic of eBPF"
tags:
- Linux
---

In this blog post we will be exploring eBPF, a linux kernel feature which has revolutinized in the area of networking, security, tracing, observability, profiling, etc. Sicne it is a kernel feature, we will be talking about how feature lands into kenrel space, basic distinction about userpace and kernel space.
After it, we will delve into the eBPF technology and use cases of it to solve the real world problems.

{{< img "/img/ebpf.png" >}}

# Contents
- The Linux Kernel
- Kernel Development Lifecycle
- Basic fundamentals of eBPF
- Classical BPF
- Extended BPF(eBPF)
- XDP(Express Data Path)
- eBPF in Cloud Native World

## The Linux Kernel

The Linux kernel is a free and open-source, monolithic, modular, multitasking, Unix-like operating system kernel. It was originally written in 1991 by Linus Torvalds for his i386-based PC, and it was soon adopted as the kernel for the GNU operating system, which was written to be a free replacement for Unix.

The kernel offers a set of APIs that applications issue which are generally referred as "System Calls". These APIs acts as the interface between hardware and the processes which is responsible for managing the low-level functions of the computer.

{{< img "/img/linux-kernel.png" >}}
