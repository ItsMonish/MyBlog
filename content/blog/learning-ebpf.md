+++
date = '2026-03-07T20:32:58+05:30'
title = 'I wanted to poke at the Linux kernel. eBPF was the answer'
description = 'Why and how I started learning eBPF, extended Berkley Packet Filters, and what I liked about it'
categories = ['eBPF', 'Learning']
tags = ['Learning', 'Linux', 'eBPF']
keywords = ['eBPF', 'Linux']
summary = 'Why and how I started learning eBPF, extended Berkley Packet Filters, and what I liked about it'
+++
## Introduction
A while back I came across [Cloudflare's writeup](https://blog.cloudflare.com/how-cloudflare-auto-mitigated-world-record-3-8-tbps-ddos-attack/) on how they mitigated a 3.8 Tbps DDoS attack, the largest ever disclosed publicly at the time. The post goes deep into how their systems detected and dropped attack traffic autonomously (quite interesting stuff). One part that caught my eye: they used XDP and eBPF to drop packets directly at the NIC level, before the kernel network stack even saw them. I had zero clue on what eBPF was back then. I thought that was interesting, bookmarked it, and then forgot about it (Yeah I do that a lot).

When I eventually got around to it, I was looking to get into lower level Linux stuff anyway. How the kernel works, how you can poke at it from the outside. eBPF kept coming up. I also saw it framed as a safer alternative to writing kernel modules for certain use cases. That part made immediate sense to me, because writing kernel modules means one bad pointer dereference and your machine is just down. So I thought maybe I could look into it seriously.

## What even is eBPF
eBPF lets you run sandboxed programs inside the Linux kernel without modifying the kernel source or loading a module. Before your program runs, the kernel verifies it. No unbounded loops, no out-of-bounds memory access, every code path has to terminate. If it doesn't pass, it doesn't load. However it doesn't mean it's bullet-proof, I still found some bugs and weird things happen now and then. But it catches the most obvious ones.

Evidently, it started as Berkeley Packet Filter, which was just a way to filter network packets efficiently. The "extended" part came later and expanded it well beyond that when people wanted better tracing options when debugging inside the kernel. These days it shows up in observability tools, performance profilers, security products, and a lot of infrastructure tooling that runs quietly in the background of production systems.

## How I learned it
I started reading [Learning eBPF](https://www.oreilly.com/library/view/learning-ebpf/9781098135119/) by Liz Rice and went through it cover to cover. It builds up a solid mental model of how everything fits together, the verifier, maps, ring buffers, CO-RE, without throwing you into kernel internals from page one. Great book, would recommend it if you're starting out.

After that I spent time staring and reading through examples at [libbpf-bootstrap](https://github.com/libbpf/libbpf-bootstrap) and [cilium/ebpf](https://github.com/cilium/ebpf/tree/main/examples). That was just to see how things are done in real projects and scripts.

Then I started writing stuff. A per-process file access counter. A pure C rough version of what would eventually become [Barbwire](https://github.com/ItsMonish/barbwire). None of it was useful or anything. It was just to get familiar with the toolchain, get yelled at by the verifier, figure out why and repeat.

## What I liked about eBPF
A couple of things genuinely impressed me.

The first was CO-RE, which stands for Compile Once, Run Everywhere. Despite my reservations on Java, I always thought the idea of Code once, run everywhere was pretty neat (Thanks to JVM). CO-RE brings the same thing to Linux kernel programming. It handles struct field offset relocations across kernel versions, so your program isn't silently tied to the exact kernel it was compiled against. Without it, anything that walks kernel structs like `task_struct` breaks the moment you run it on a different machine. With it, the same binary works across versions. It is remarkable as far as I can tell.

The second was how dynamic the whole thing is. With kernel modules you write code, compile, load, and if something goes wrong you restart. With eBPF you attach programs to running systems and detach them without rebooting or disrupting anything. For observability and containers especially, that matters a lot. You can instrument a live system, see what you need, and pull the probe out. No downtime, no risk of leaving a bad module loaded.

## What came next
After a few weeks of experiments I felt ready to build something with an actual purpose. That became Barbwire, a lightweight behavioral correlator that watches for processes opening sensitive files and then making network connections. I'll write about that in the next post.

It's not a production security tool. It's an experiment. But building it was a good reason to use eBPF for something real instead of just counting file accesses forever.
