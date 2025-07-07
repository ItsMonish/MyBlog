+++
date = '2025-06-22T13:45:21+05:30'
title = 'Run Run, Ember Run! - EchoCTF Writeup'
description = 'An interesting challenge involving a MBR boot sector'
categories = ['Writeup', 'EchoCTF']
tags = ['forensics','challenge-writeup','echoCTF']
keywords = ['forensics', 'echoctf', 'writeup', 'mbr boot sector']
summary = 'Writeup for an EchoCTF challenge - Run Run, Ember Run!'
+++

## Introduction
This writeup deals with the [Run Run, Ember Run!](https://echoctf.red/challenge/9) challenge from [EchoCTF](https://echoctf.red/). This challenge focused on understanding the MBR boot record format and the bootstrap code it contains. A compressed gunzip archive [run_run_ember_run.img.gz](https://echoctf.red/uploads/run_run_ember_run.gz) contains challenge file.

Challenge Statement:
```
A small challenge with only four questions for you to answer.

An attacker gained access to one of the systems leaving behind the file attached to this challenge.

Not many details are known about the file but there are at least two ways to get what you want. Depending on the road you'll take it can be as easy as walking in the park or hard as climbing the Himalayas.

You need to analyze the file and answer the following questions...

This is not a filesystem, even though you'll find it on disks... and although its not executable you can still run it!!!

Q1: What is the first thing you'd see from running this image? (200 pts)
The first thing that would greet you is a flag. Give it for answer to this question.

Q2: What do you get from address 0? (300 pts)
What are the values of the first 16 bytes starting from address 0. Remove any spaces from your answer.

Q3: What is the address backdoored by the attacker? (400 pts)
The attacker seemed to have added a backdoor at a certain address. When you try to print this address you will get a flag. What is the backdoored address in hex (prefixed by 0x).

Q4: What is the flag from the backdoored address? (500 pts)
What is the flag being displayed from the backdoored address?

```

## Writeup
The question makes it look like the challenge has something to do with a binary reverse engineering analysis. But also there are points that suggest that it a disk? So first we need to understand what it is we are given.

![file utility output](/images/echoctf_runrunemberrun/1.png)

It says that the file is a DOS/MBR boot sector. So we can't "run" a boot sector right? That's what I thought and decided to take a look at the hex dump of the file.

![hex dump](/images/echoctf_runrunemberrun/2.png)

That's all of the hex dump. So taking a preliminary look at it revealed nothing. I mean no hidden files, no corrupted data, nothing. But what caught my attention is how it is exactly 512 bytes. The boot partitions are indeed 512 bytes. So that suggested we really do have a MBR boot sector at our hands. Hence we should be able to look at the entries using something like `fdisk`.

![fdisk output](/images/echoctf_runrunemberrun/3.png)

So we have something. The partition data that the sector holds. But this is of no use without the actual disk (except for now we know what kinda partitions the computer this record belonged to had). But that can't be it. I was missing something. And then looking at the wikipedia page for [MBR boot record](https://en.wikipedia.org/wiki/Master_boot_record) had the following table:

![mbr wikipedia](/images/echoctf_runrunemberrun/4.png)

Huh, so there IS something that I was missing. There is an entire 446 bytes of "Bootstrap code area". That should have been obivous but it wasn't for me. After all this code region is usually targetted by malicious softwares called [Bootkits](https://en.wikipedia.org/wiki/Rootkit#bootkit).

According to the wiki at [OSDev](https://wiki.osdev.org/MBR_(x86)) (not explicitly mentioned, the example provided there indicated it), this code is of 16-bit x86 instruction set. So all I have to do is disassemble the image provided. Easy.

But before going head first into assembly, recall that the challenge statement explicitly stated to "run" the image. Since we have a valid boot sector image, perhaps we can? Quick search on the internet showed [QEMU](https://www.qemu.org/) is good for it. Using the command, 

`qemu-system-i386 -drive format=raw,file=run_run_ember_run.img`

we are able to boot into the boot sector on a virtualized machine console.

![first boot and answer](/images/echoctf_runrunemberrun/6.png)

Yeah we are indeed greeted with a message. Hence our answer for Q1. It looks like a prompt waiting for input. 

![second answer](/images/echoctf_runrunemberrun/7.png)

Hitting enter on the prompt gave some interesting output. It displays the memory dump of the sector. Since our second question is literally what are the values of first 16 bytes of address 0, we have our answer for that as well.

And for Q3, we apparently have a backdoor of sorts. I saw no other option than having to dive into the assembly code. We can dump the entire image using `objdump`, I specifically used the following command:

`objdump -D -b binary -mi386 -Maddr16,data16 -Mintel run_run_ember_run.img`

The options being:
```
-D : Disassemble the file
-b binary : target object is binary
-mi386 : target architecture is x86
-Maddr16,data16 : Disassembler options
    addr16 : Addresses are in 16-bit
    data16 : Data is in 16-bit
-Mintel : Use intel style assembly code
```

![disassembly](/images/echoctf_runrunemberrun/5.png)

There was a lot of disassembly. As a matter of fact, we have dumped the entire image file, but the actual code is only 446 bytes, which would mean the offset upto 1bd is the valid code.

And now we read assembly.

![backdoor](/images/echoctf_runrunemberrun/8.png)

Skimming through the assembly dump, the instruction at offset 86 caught my attention. A compare statement that compares the value at the register `bx` with `0x3750`. As far as I could tell `0x3750` is nothing special. But the instruction checks for it, and then branches. So the question arises, what are the two branches?

```
86:   81 fb 50 37             cmp    bx,0x3750
8a:   75 30                   jne    0xbc
8c:   b0 6d                   mov    al,0x6d

```

Assuming the likely case that `bx` won't be equal to 0x3750, it seems the code we want to execute is at `bc` offset. That begs the next question, how could `bx` be set to `0x3750`.

```
74:   e8 c5 00           call   0x13c    
77:   72 05              jb     0x7e     
79:   93                 xchg   bx, ax   
```
A little above the offset of 86, at offset 74, we can see that a call is made to the address `0x13c`. In x86 convention a call's return value is usally found in `ax` register (if it returns). Then instruction `xchg` exchanges or swaps the value between `bx` and `ax` registers. So bx depends on whatever is returned from the instructions at `0x13c`.

```
13c:   e8 04 00                call   0x143
13f:   72 15                   jb     0x156
141:   88 c4                   mov    ah,al
143:   ac                      lods   al,BYTE PTR ds:[si]
144:   e8 10 00                call   0x157
147:   72 0d                   jb     0x156
149:   c0 e0 04                shl    al,0x4
14c:   88 c2                   mov    dl,al
14e:   ac                      lods   al,BYTE PTR ds:[si]
14f:   e8 05 00                call   0x157
152:   72 02                   jb     0x156
154:   08 d0                   or     al,dl
156:   c3                      ret
```
To be quite honest, I don't exactly understand what's going in here. The call to `0x13c` in turn makes a call to `0x143` which is 3 instruction away and kinda confusing and we are missing a ret instruction. 

But in a gist the instruction at `0x143` loads a string, an input string to be specific, and calls to another `0x157`. It does the same set of instruction twice, but the instruction `shl al, 0x4` (shift the contents of al\[lower 8 bits of ax\] to the left by 4 places), suggests it just processes an input of 16 bits, 8 bits each time. And then the processed contents is stored at the register `ax`.

So looking at `0x157` we have:
``` 
                    ; turns strings to ascii. Like '0' -> 0
157:   2c 30                   sub    al,0x30    
                    ; jump to ret if subtraction is negative. For cases less than '0'
159:   72 0e                   jb     0x169      
                    ; compare al and 10(in decimal)
15b:   3c 0a                   cmp    al,0xa
                    ; compliment carry bit, not sure why
15d:   f5                      cmc
                    ; jump to ret(0x169) if al is above or equal to 10.
                    ; so al must have value between 0 and 9
15e:   73 09                   jae    0x169
160:   2c 31                   sub    al,0x31
162:   72 05                   jb     0x169
164:   04 0a                   add    al,0xa
166:   3c 10                   cmp    al,0x10
168:   f5                      cmc
169:   c3                      ret
```
Now just looking at the first few lines of the routine, we can see that the call converts the string input to number input. So if we want `0x3750` as value of `ax` register, we just need to input it?

![backdoor](/images/echoctf_runrunemberrun/9.png)

Yes. We just had to put it there and the flag is given. Could've tried that first, nevertheless it was good. So we have our answer for Q4.

We still need answer for Q3. Now this part, I think the question is kinda misleading. It asks for the "backdoored address". Technically, the backdoor is at offset 86. MBR base address starts at 0x7c00, making the "backdoored address" as 0x7c86. But this is not it. The answer is the value we need to meet at this address. That is our answer for Q3. Or maybe I'm mistaken, but as far as I can tell, that is not an address.

### Note
You can actually find the both of the flags in the hex dump from the beginning if you look for it (with some 'pattern recognition'). I didn't.But it's just the flags.

## Conclusion
It did take me quite some time to figure out Q3, but nevertheless, the challenge was fun and unique. There ain't many challenges that require you to boot a boot sector. So all things considered it was a solid.
